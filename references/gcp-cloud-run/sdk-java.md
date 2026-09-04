# Java SDK on GCP Cloud Run

<!-- Source: docs/develop/java/workers/serverless-workers/cloud-run.mdx -->

Use this reference for Java-specific Worker construction, versioning behavior, connection configuration, image packaging, and scale-in safety. For the shared Cloud Run deployment lifecycle, permissions, versioning model, observability, and diagnostics, see `setup.md`, `iam.md`, `versioning.md`, `observability.md`, and `diagnostics.md`.

**There is no Cloud Run Worker package.** This is an ordinary long-lived Java Worker plus Worker Versioning, which Serverless Workers require. Nothing in `../aws-lambda/sdk-java.md` applies.

## Inspect the versioning API before generating code

Worker Versioning is a Public Preview surface and the option names differ between SDKs. Read the installed version's API rather than writing from memory:

```bash
mvn -q dependency:get -Dartifact=io.temporal:temporal-sdk:<version>:jar:sources
unzip -o ~/.m2/repository/io/temporal/temporal-sdk/<version>/temporal-sdk-<version>-sources.jar \
  'io/temporal/worker/WorkerDeploymentOptions.java' 'io/temporal/common/WorkerDeploymentVersion.java' -d /tmp/src
```

## Versioned Worker

Set `WorkerDeploymentOptions` on `WorkerOptions`. The Worker reads its connection settings and Task Queue from the environment so one image runs against any Namespace:

```java
package example;

import io.temporal.client.WorkflowClient;
import io.temporal.client.WorkflowClientOptions;
import io.temporal.common.VersioningBehavior;
import io.temporal.common.WorkerDeploymentVersion;
import io.temporal.serviceclient.WorkflowServiceStubs;
import io.temporal.serviceclient.WorkflowServiceStubsOptions;
import io.temporal.worker.Worker;
import io.temporal.worker.WorkerDeploymentOptions;
import io.temporal.worker.WorkerFactory;
import io.temporal.worker.WorkerOptions;
import java.util.concurrent.TimeUnit;

public final class Main {
  public static void main(String[] args) {
    String apiKey = System.getenv("TEMPORAL_API_KEY");

    WorkflowServiceStubs service =
        WorkflowServiceStubs.newServiceStubs(
            WorkflowServiceStubsOptions.newBuilder()
                .setTarget(System.getenv("TEMPORAL_ADDRESS"))
                .setEnableHttps(true)
                .addApiKey(() -> apiKey)
                .build());

    WorkflowClient client =
        WorkflowClient.newInstance(
            service,
            WorkflowClientOptions.newBuilder()
                .setNamespace(System.getenv("TEMPORAL_NAMESPACE"))
                .build());

    WorkerFactory factory = WorkerFactory.newInstance(client);
    Worker worker =
        factory.newWorker(
            System.getenv("TEMPORAL_TASK_QUEUE"),
            WorkerOptions.newBuilder()
                .setDeploymentOptions(
                    WorkerDeploymentOptions.newBuilder()
                        .setUseVersioning(true)
                        .setVersion(new WorkerDeploymentVersion("my-app", "build-1"))
                        .setDefaultVersioningBehavior(VersioningBehavior.PINNED)
                        .build())
                .build());

    worker.registerWorkflowImplementationTypes(GreetingWorkflowImpl.class);
    worker.registerActivitiesImplementations(new GreetingActivitiesImpl());

    Runtime.getRuntime().addShutdownHook(new Thread(() -> {
      factory.shutdown();
      factory.awaitTermination(8, TimeUnit.SECONDS);
    }));
    factory.start();
  }
}
```

`WorkerDeploymentVersion`'s two arguments are the deployment name and the build ID, and both must match the version created with `temporal worker deployment create-version` exactly. → `setup.md` Step 6.

No `LambdaWorker`, no `define`, no shaded uber-jar requirement — this is `WorkerFactory` as in any long-lived Java Worker.

## Versioning behavior

Every Workflow needs `VersioningBehavior.PINNED` or `AUTO_UPGRADE`. `setDefaultVersioningBehavior` covers every Workflow; to set it per Workflow, annotate the Workflow method.

```java
import io.temporal.common.VersioningBehavior;
import io.temporal.workflow.WorkflowVersioningBehavior;

public class GreetingWorkflowImpl implements GreetingWorkflow {
  @Override
  @WorkflowVersioningBehavior(VersioningBehavior.PINNED)
  public String run(String name) {
    // ...
  }
}
```

**A Version set with no behavior fails at runtime**, not at build time.

## Connection configuration

Read `TEMPORAL_ADDRESS`, `TEMPORAL_NAMESPACE`, `TEMPORAL_API_KEY`, and `TEMPORAL_TASK_QUEUE` from the environment set on the pool, and mount the key from Secret Manager rather than passing it in plaintext. → `setup.md` Step 4.

`addApiKey` takes a **supplier**, called on every request, so a key can be rotated by returning a new value instead of restarting the Worker — worth using on a long-lived pool instance, where a restart is not free.

Java uses gRPC/Netty and the JVM truststore, so it is unaffected by the `NativeCertsNotFound` failure the Rust-core SDKs hit.

## Image packaging

Fat jar on a JRE image, with the heap sized to the instance:

```dockerfile
CMD ["java", "-XX:MaxRAMPercentage=75", "-jar", "/app/worker.jar"]
```

The JVM reads the container memory limit but defaults the maximum heap to a quarter of it, leaving most of a small instance unused. A pool defaults to 512 MiB per instance, so raise `--memory` when creating it if the Worker needs more. → `setup.md` Step 2.

## Graceful shutdown on scale-in

Register the JVM shutdown hook before `factory.start()`, as in the versioned Worker example. Cloud Run's `SIGTERM` starts the hook; `factory.shutdown()` stops polling, and `awaitTermination` keeps the hook alive while received Tasks drain. `shutdown()` alone is asynchronous, so omitting the wait lets the JVM exit before draining. Do not use `shutdownNow()` as the normal signal path.

Cloud Run can send `SIGKILL` ten seconds later. Shutdown therefore improves draining but cannot guarantee that a long Activity finishes; Activities must finish promptly or record Heartbeats so a retry can resume.

## Keep Activities safe across scale-in

The WCI removes instances based on Task Queue activity, not on what an individual instance is doing, so **an instance running a long Activity can be stopped mid-execution.** Record Heartbeats so a retry resumes from the last recorded progress:

```java
public class GreetingActivitiesImpl implements GreetingActivities {
  @Override
  public String process(List<String> items) {
    for (int i = 0; i < items.size(); i++) {
      Activity.getExecutionContext().heartbeat(i);
      // ... process items.get(i)
    }
    return "done";
  }
}
```

→ `constraints.md` for what else follows from the pool model.

## Logging and diagnostic signatures

If the application uses `logback-classic`, include an explicit `src/main/resources/logback.xml`. Without one, Logback's basic configuration sets the root logger to DEBUG; grpc-java's Netty transport has a DEBUG frame logger that can emit outbound HTTP/2 headers. Keep transport categories above DEBUG wherever bearer credentials are used:

```xml
<configuration>
  <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
    <encoder>
      <pattern>%date %-5level %logger{36} - %msg%n</pattern>
    </encoder>
  </appender>

  <logger name="io.grpc" level="WARN" />
  <logger name="io.netty" level="WARN" />
  <logger name="io.temporal" level="INFO" />

  <root level="INFO">
    <appender-ref ref="STDOUT" />
  </root>
</configuration>
```

Use a logging provider compatible with the SLF4J API version selected by the installed Temporal SDK. If Cloud Logging ever contains an `authorization` or `Bearer` value, restrict the transport logger immediately and rotate the exposed Temporal API key.

| Log signature | Meaning / action |
|---|---|
| No application or SDK logs | No compatible SLF4J provider is bound, or the configuration was not packaged in the jar. |
| `NettyClientHandler ... OUTBOUND HEADERS` | Unsafe transport DEBUG logging is enabled. Raise `io.grpc` and `io.netty` to WARN and inspect for credential exposure. |
| `UNAUTHENTICATED` | Check the Secret Manager mount and the key's Namespace permissions. |
| Worker starts but the intended Workflow does not progress | Check the deployment name, build ID, Task Queue, and that the version is current. |

## Observability

A Cloud Run Worker emits the same traces and metrics as a Worker anywhere else — no Cloud Run-specific wiring, and none of Lambda's ADOT layer or collector configuration. Use the SDK's normal metrics export and OpenTelemetry tracing interceptors. → `observability.md`, and `docs/develop/java/platform/observability`.
