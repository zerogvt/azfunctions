# Azure Functions (Java) → OpenTelemetry → Dynatrace

A minimal Java 17 HTTP-triggered Azure Function (`HttpExample`) on the **Flex Consumption** plan that sends logs, traces and metrics to **Dynatrace** using **OpenTelemetry only** (no OneAgent).

Use it as a reference to add OpenTelemetry and Dynatrace ingestion to an existing Java function app.

## Setting up OpenTelemetry → Dynatrace for an existing Java Function App

### How it works

```
Java worker (your code)                Functions host (.NET)                    Dynatrace
context.getLogger().info(...)  ──gRPC──►  telemetryMode: OpenTelemetry  ──OTLP/HTTP──►  /api/v2/otlp
                                          (logs, spans, metrics)                        (logs, spans, metrics)
```

The **Functions host** sends the telemetry, not your Java code. When `host.json` sets `telemetryMode: OpenTelemetry`, the host exports its own logs, request spans and runtime metrics to whatever endpoint the standard `OTEL_EXPORTER_OTLP_*` app settings point at. Log lines written through `context.getLogger()` travel through the host and arrive with a `trace_id`, so in Dynatrace they're linked to the matching invocation span.

You don't need any OpenTelemetry dependency in `pom.xml` for this.

### 1. `host.json`: switch the host to OpenTelemetry mode

```json
{
    "version": "2.0",
    "telemetryMode": "OpenTelemetry",
    "logging": {
        "logLevel": {
            "default": "Warning",
            "Function": "Information",
            "Host.Results": "Information"
        }
    },
    "extensionBundle": {
        "id": "Microsoft.Azure.Functions.ExtensionBundle",
        "version": "[4.*, 5.0.0)"
    }
}
```

- `telemetryMode: OpenTelemetry` is required. Without it the host uses the classic Application Insights pipeline and ignores the `OTEL_*` settings.
- `"Function": "Information"` lets your `context.getLogger().info(...)` lines through. With only `"default": "Warning"` they'd be dropped.

### 2. Code: log through the execution context

```java
@FunctionName("HttpExample")
public HttpResponseMessage run(
        @HttpTrigger(name = "req", methods = {HttpMethod.GET, HttpMethod.POST},
                     authLevel = AuthorizationLevel.ANONYMOUS)
        HttpRequestMessage<Optional<String>> request,
        final ExecutionContext context) {

    // context logger -> Functions host -> OTLP -> Dynatrace (correlated with the invocation trace)
    context.getLogger().info("DT-OTEL-CHECK HttpExample invoked name=" + name
            + " invocationId=" + context.getInvocationId());
    ...
}
```

Use `context.getLogger()`. A separately created `java.util.logging.Logger` does reach Application Insights, but in our tests the host didn't forward it to Dynatrace.

### 3. Dynatrace: create an access token

In Dynatrace, go to **Access tokens → Generate new token** and give it these scopes:

| Scope | Needed for |
|---|---|
| `openTelemetryTrace.ingest` | spans |
| `metrics.ingest` | metrics |
| `logs.ingest` | logs |

This gives you a classic API token, which starts with `dt0c01.`.

### 4. Azure app settings

| Setting | Value | Notes |
|---|---|---|
| `OTEL_EXPORTER_OTLP_ENDPOINT` | `https://<env-id>.live.dynatrace.com/api/v2/otlp` | Base URL. The exporter adds `/v1/traces`, `/v1/metrics` and `/v1/logs` itself. |
| `OTEL_EXPORTER_OTLP_HEADERS` | `Authorization=Api-Token dt0c01.XXXX.YYYY` | See the format warning below. |
| `OTEL_EXPORTER_OTLP_PROTOCOL` | `http/protobuf` | **Required.** The .NET host defaults to gRPC, which Dynatrace doesn't accept. |
| `OTEL_EXPORTER_OTLP_METRICS_TEMPORALITY_PREFERENCE` | `delta` | Dynatrace only accepts delta metrics. |
| `OTEL_RESOURCE_ATTRIBUTES` | `service.name=java-quickstart,service.version=1.0.1` | Becomes `service.name` in Dynatrace. Don't put quotes around the value. |
| `APPLICATIONINSIGHTS_ENABLE_AGENT` | `false` | Stops the Application Insights Java agent from also being injected into the worker. |

> ⚠️ **Header format.** `OTEL_EXPORTER_OTLP_HEADERS` uses the OpenTelemetry `key=value` format, **not** HTTP `Key: value`, and must **not** be wrapped in quotes.
>
> - ✅ `Authorization=Api-Token dt0c01.ABC.DEF`
> - ❌ `"Authorization: Bearer dt0s16.ABC.DEF"`
>
> In OpenTelemetry mode a malformed value doesn't produce a readable error. The host just never finishes starting: every request returns **502 after about 30s**, `/admin/host/status` stays in `"state": "Default"`, and nothing is logged anywhere.

Set them with the Azure CLI:

```bash
az functionapp config appsettings set -g <rg> -n <app> --settings \
  "OTEL_EXPORTER_OTLP_ENDPOINT=https://<env-id>.live.dynatrace.com/api/v2/otlp" \
  "OTEL_EXPORTER_OTLP_HEADERS=Authorization=Api-Token <dt0c01-token>" \
  "OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf" \
  "OTEL_EXPORTER_OTLP_METRICS_TEMPORALITY_PREFERENCE=delta" \
  "OTEL_RESOURCE_ATTRIBUTES=service.name=<your-service>,service.version=<x.y.z>" \
  "APPLICATIONINSIGHTS_ENABLE_AGENT=false"
```

On Flex Consumption, changing an app setting automatically replaces the running instances, so you don't need to restart manually. If you want to force a restart anyway, run `az functionapp restart -g <rg> -n <app>`.

Before deploying, you can check the endpoint and token with an empty payload. Expect HTTP 200 for each signal:

```bash
for s in traces metrics logs; do
  curl -s -o /dev/null -w "$s %{http_code}\n" -X POST \
    -H "Authorization: Api-Token <dt0c01-token>" -H "Content-Type: application/x-protobuf" \
    --data-binary "" "https://<env-id>.live.dynatrace.com/api/v2/otlp/v1/$s"
done
```

### 5. Build and deploy

```bash
mvn clean package                        # stages to target/azure-functions/<functionAppName>
```

`pom.xml` → `functionAppName`, `resourceGroup` and `region` must match the **existing** app. Otherwise `mvn azure-functions:deploy` creates a new one. The VS Code settings (`.vscode/settings.json` → `azureFunctions.deploySubpath`) point at the same staging folder.

Deploy with VS Code (*Azure Functions: Deploy to Function App*), `mvn azure-functions:deploy`, or the Flex publish endpoint:

```bash
cd target/azure-functions/<functionAppName> && zip -r ../../deploy.zip . && cd -
curl -X POST -H "Authorization: Bearer $(az account get-access-token --query accessToken -o tsv)" \
  -H "Content-Type: application/zip" --data-binary @target/deploy.zip \
  "https://<app-scm-host>/api/publish?RemoteBuild=false"
```

### 6. Check the data in Dynatrace (DQL)

**Logs**, including the marker line:
```sql
fetch logs, from: now()-1h
| filter service.name == "java-quickstart"
| fields timestamp, loglevel, content, trace_id
| sort timestamp desc
```

**Traces:**
```sql
fetch spans, from: now()-24h
| filter service.name == "java-quickstart"
| fields start_time, trace.id, span.id, span.parent_id, span.name, span.kind,
         duration, http.response.status_code, faas.invocation_id
| sort start_time desc
| limit 100
```

**Metrics** (values are in seconds):
```sql
timeseries {
    avg_request_s = avg(http.server.request.duration),
    max_request_s = max(http.server.request.duration),
    avg_invoke_s  = avg(faas.invoke_duration)
  },
  filter: service.name == "java-quickstart",
  from: now()-24h, interval: 5m
```

The host also exports .NET runtime metrics (`dotnet.*`, `kestrel.*`, `aspnetcore.*`). To list everything the service sends:
`fetch metric.series | filter service.name == "java-quickstart" | summarize count(), by:{metric.key}`.

### What you get

| Signal | Source | Examples |
|---|---|---|
| Logs | host + `context.getLogger()` | your log lines and `Executing/Executed 'Functions.HttpExample'`, with `trace_id` |
| Spans | host | `GET api/HttpExample` (server), `init` |
| Metrics | host | `http.server.request.duration`, `faas.invoke_duration`, `dotnet.*` |

### Optional: OpenTelemetry Java agent (JVM-level telemetry)

The jar isn't committed (`libs/*.jar` is in `.gitignore`). To use the agent, download it into `libs/` before you build:

```bash
mkdir -p libs
curl -fL -o libs/opentelemetry-javaagent.jar \
  https://repo1.maven.org/maven2/io/opentelemetry/javaagent/opentelemetry-javaagent/2.33.1/opentelemetry-javaagent-2.33.1.jar
```

The file name must stay exactly `opentelemetry-javaagent.jar`. You can get other versions from [Maven Central](https://repo1.maven.org/maven2/io/opentelemetry/javaagent/opentelemetry-javaagent/) or the [GitHub releases](https://github.com/open-telemetry/opentelemetry-java-instrumentation/releases). If the jar isn't there, the build still succeeds: the copy step finds nothing to copy, so don't set the app setting below.

At build time `maven-resources-plugin` copies the jar into the deploy package, and this app setting loads it into the worker JVM:

```
languageWorkers__java__arguments = -javaagent:/home/site/wwwroot/libs/opentelemetry-javaagent.jar
```

- Use the **absolute** path `/home/site/wwwroot/libs/...`. The relative `/libs/...` crashes the worker with `Error opening zip file or JAR manifest missing`.
- **Caveat:** on Flex Consumption the agent loaded but **exported nothing** in our tests: no `jvm.*` metrics, and no logs even with `OTEL_JAVAAGENT_DEBUG=true`. It also can't create spans for function invocations, because the worker's gRPC library is shaded (`com.microsoft.azure.functions.shaded.io.grpc`). Everything listed above comes from the host. If you don't need JVM metrics, leave the agent out: that saves about 26 MB of package size and some cold-start time.

### Troubleshooting

| Symptom | Likely cause |
|---|---|
| Every request returns 502 after ~30s, nothing in any log, `/admin/host/status` stuck at `"state": "Default"` | Malformed `OTEL_EXPORTER_OTLP_HEADERS` (quotes, or `:` instead of `=`) |
| App is healthy, but Dynatrace receives nothing | `OTEL_EXPORTER_OTLP_PROTOCOL` not set to `http/protobuf` (host is using gRPC) |
| Data reaches Application Insights but not Dynatrace | `telemetryMode: OpenTelemetry` missing from `host.json` |
| `service.name` looks like `"service.name` | Quotes around the `OTEL_RESOURCE_ATTRIBUTES` value |
| Worker exits with `Error opening zip file … /libs/opentelemetry-javaagent.jar` | Relative agent path. Use `/home/site/wwwroot/libs/...` |
| A new log line doesn't appear right after deploying | Flex was still serving the previous instance. Wait a minute and send another request |

To check host state, get the master key with `az functionapp keys list -g <rg> -n <app>` and run:
```bash
curl -H "x-functions-key: <master-key>" https://<app-host>/admin/host/status
```
`"state": "Running"` is healthy. `"Default"` means the host never finished starting.
