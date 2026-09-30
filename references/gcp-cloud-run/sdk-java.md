# Java SDK on GCP Cloud Run

<!-- Source: docs/develop/java/workers/serverless-workers/cloud-run.mdx -->

Use this reference for Java-specific Worker construction, versioning behavior, connection configuration, image packaging, and scale-in safety. For shared Cloud Run execution constraints, deployment lifecycle, permissions, versioning, observability, and diagnostics, see `constraints.md`, `setup.md`, `iam.md`, `versioning.md`, `observability.md`, and `diagnostics.md`.

## Install and scaffold

Use Java 21 and pin the SDK in `pom.xml`:

```xml
<properties>
  <maven.compiler.release>21</maven.compiler.release>
</properties>

<dependency>
  <groupId>io.temporal</groupId>
  <artifactId>temporal-sdk</artifactId>
  <version>1.40.0</version>
</dependency>
```

The `temporal-serviceclient` classes used below are a transitive dependency of `temporal-sdk`; do not omit them when inspecting or shading the application.

## Inspect the versioning API before generating code

```bash
mvn -q dependency:get -Dartifact=io.temporal:temporal-sdk:1.40.0:jar:sources
mvn -q dependency:get -Dartifact=io.temporal:temporal-serviceclient:1.40.0:jar:sources
unzip -o ~/.m2/repository/io/temporal/temporal-sdk/1.40.0/temporal-sdk-1.40.0-sources.jar \
  'io/temporal/worker/WorkerDeploymentOptions.java' 'io/temporal/common/WorkerDeploymentVersion.java' -d /tmp/src
```

If the project intentionally uses another SDK version, substitute that exact version in all three places and inspect its sources before generating code.

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
    String deploymentName = requireEnv("TEMPORAL_DEPLOYMENT_NAME");
    String buildId = requireEnv("TEMPORAL_BUILD_ID");
    String taskQueue = requireEnv("TEMPORAL_TASK_QUEUE");
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
            taskQueue,
            WorkerOptions.newBuilder()
                .setDeploymentOptions(
                    WorkerDeploymentOptions.newBuilder()
                        .setUseVersioning(true)
                        .setVersion(new WorkerDeploymentVersion(deploymentName, buildId))
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
    System.out.printf(
        "Worker started deployment=%s build=%s taskQueue=%s%n",
        deploymentName, buildId, taskQueue);
  }

  private static String requireEnv(String name) {
    String value = System.getenv(name);
    if (value == null || value.isBlank()) {
      throw new IllegalStateException(name + " must be set");
    }
    return value;
  }
}
```

`WorkerDeploymentVersion`'s two arguments are the deployment name and the build ID, and both must match the version created with `temporal worker deployment create-version` exactly. → `setup.md` Step 6.

Use `WorkerFactory` as for any long-lived Java Worker. Cloud Run imposes no Temporal-specific artifact format.

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

Create an executable fat jar. The service descriptor transformer is required; without it, shaded gRPC providers can disappear at runtime:

```xml
<build>
  <finalName>worker</finalName>
  <plugins>
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-shade-plugin</artifactId>
      <version>3.6.1</version>
      <executions>
        <execution>
          <phase>package</phase>
          <goals><goal>shade</goal></goals>
          <configuration>
            <createDependencyReducedPom>false</createDependencyReducedPom>
            <transformers>
              <transformer implementation="org.apache.maven.plugins.shade.resource.ServicesResourceTransformer" />
              <transformer implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
                <mainClass>example.Main</mainClass>
              </transformer>
            </transformers>
          </configuration>
        </execution>
      </executions>
    </plugin>
  </plugins>
</build>
```

Build and run it in a Java 21 image:

```dockerfile
FROM maven:3.9.11-eclipse-temurin-21 AS build
WORKDIR /src
COPY pom.xml ./
RUN mvn -q dependency:go-offline
COPY src ./src
RUN mvn -q -DskipTests package

FROM eclipse-temurin:21-jre
WORKDIR /app
COPY --from=build /src/target/worker.jar ./worker.jar
CMD ["java", "-XX:MaxRAMPercentage=75", "-jar", "/app/worker.jar"]
```

```gitignore
.git
.gitignore
.idea
target
*.iml
```

Cloud Run Worker Pools run this image as `linux/amd64`. The JVM reads the container memory limit but defaults the maximum heap to a quarter of it, leaving most of a small instance unused. A pool defaults to 512 MiB per instance, so raise `--memory` when creating it if the Worker needs more. → `setup.md` Step 2.

## Graceful shutdown on scale-in

Register the JVM shutdown hook before `factory.start()`, as in the versioned Worker example. Cloud Run's `SIGTERM` starts the hook; `factory.shutdown()` stops polling, and `awaitTermination` keeps the hook alive while received Tasks drain. `shutdown()` alone is asynchronous, so omitting the wait lets the JVM exit before draining. Do not use `shutdownNow()` as the normal signal path.

## Keep Activities safe across scale-in

Apply the Heartbeat-resume invariant from `constraints.md` in Java:

```java
import io.temporal.activity.ActivityOptions;
import io.temporal.common.RetryOptions;
import io.temporal.workflow.Workflow;
import java.time.Duration;

private final GreetingActivities activities =
    Workflow.newActivityStub(
        GreetingActivities.class,
        ActivityOptions.newBuilder()
            .setStartToCloseTimeout(Duration.ofMinutes(10))
            .setHeartbeatTimeout(Duration.ofSeconds(10))
            .setRetryOptions(
                RetryOptions.newBuilder()
                    .setInitialInterval(Duration.ofSeconds(1))
                    .setMaximumAttempts(5)
                    .build())
            .build());
```

The Activity records the next item to process:

```java
public class GreetingActivitiesImpl implements GreetingActivities {
  @Override
  public String process(List<String> items) {
    ActivityExecutionContext context = Activity.getExecutionContext();
    int startIndex = context.getHeartbeatDetails(Integer.class).orElse(0);

    for (int i = startIndex; i < items.size(); i++) {
      // ... process items.get(i)
      context.heartbeat(i + 1);
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

## Observability

For Java SDK configuration, see `docs/develop/java/platform/observability`. For shared Cloud Run behavior and provider-specific scaling signals, see `observability.md`.
