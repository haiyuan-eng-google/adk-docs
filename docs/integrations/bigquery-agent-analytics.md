---
catalog_title: BigQuery Agent Analytics Plugin
catalog_description: In-depth agent analytics for behavior analysis and logging
catalog_icon: /integrations/assets/bigquery.png
catalog_tags: ["observability", "google"]
---

# BigQuery Agent Analytics plugin for ADK

<div class="language-support-tag">
  <span class="lst-supported">Supported in ADK</span><span class="lst-python">Python v1.21.0</span><span class="lst-go">Go v1.0.0</span><span class="lst-java">Java v1.5.0</span><span class="lst-kotlin">Kotlin v0.8.0</span>
</div>

The BigQuery Agent Analytics plugin logs ADK lifecycle, LLM, tool, HITL, and
multi-agent workflow events into BigQuery via the asynchronous **BigQuery
Storage Write API**. It provides automatic table creation, additive schema
upgrades, per-event-type views, tool provenance, `event_id` deduplication,
optional Cloud Storage offloading, and `trace_id` correlation with Cloud Trace.

This guide covers ADK SDK installation, configuration, runtime deployment, and
language-specific behavior. For BigQuery platform architecture, schemas,
governance, and SQL analytics, see the **BigQuery Cloud Docs**:

| Topic | BigQuery Cloud Docs |
| --- | --- |
| Overview & architecture | [BigQuery Agent Analytics overview](https://docs.cloud.google.com/bigquery/docs/bigquery-agent-analytics) |
| Setup, IAM, GCS offloading, CMEK, VPC-SC & troubleshooting | [Configure BigQuery Agent Analytics](https://docs.cloud.google.com/bigquery/docs/configure-agent-analytics) |
| `agent_events` 17-column schema, `v_*` views & `v_eval_*` views | [Agent analytics schema reference](https://docs.cloud.google.com/bigquery/docs/agent-analytics-schema) |
| Complete 28-event catalog & JSON payload schemas | [Agent analytics event reference](https://docs.cloud.google.com/bigquery/docs/agent-analytics-events) |
| SQL recipes, `event_id` dedup, `ObjectRef` multimodal AI, Context Graph & Looker | [Analyze agent analytics data](https://docs.cloud.google.com/bigquery/docs/analyze-agent-analytics) |

!!! warning "BigQuery Storage Write API"

    This feature uses **BigQuery Storage Write API**, which is a paid service.
    For information on costs, see the [BigQuery
    documentation](https://cloud.google.com/bigquery/pricing?e=48754805&hl=en#data-ingestion-pricing).

??? note "Kotlin support"

    The **Kotlin** plugin logs invocation lifecycle events. It writes an
    `INVOCATION_STARTING` row when an invocation begins and an
    `INVOCATION_COMPLETED` row when it ends, and creates the partitioned,
    clustered events table on first use if it does not already exist.

    Rows are inserted one at a time through `tabledata.insertAll`, synchronously
    on the invocation path, rather than through the Storage Write API used by
    Python, Go, and Java.

    The following are not implemented in Kotlin: LLM, tool, agent, state, HITL
    and A2A events; the ADK 2.0 workflow events; automatic view creation; Auto
    Schema Upgrade; tool provenance; GCS offloading; and drop statistics.

## Prerequisites {#prerequisites}

- **Google Cloud project & dataset**: Enable the **BigQuery API** and create
  the destination dataset (the plugin creates the `agent_events` table on first use).
- **Cloud Storage bucket (optional)**: For offloading large text and multimodal payloads.
- **Authentication & IAM**: Run `gcloud auth application-default login` locally
  or attach a service account with `roles/bigquery.jobUser` (project),
  `roles/bigquery.dataEditor` (dataset, or table-level if table/views are
  pre-created and view auto-creation is disabled), and
  `roles/storage.objectCreator` on the bucket if offloading (grant
  `roles/storage.objectViewer` on the bucket to your BigQuery Cloud resource
  connection's service account for `ObjectRef` reads). See
  [Configure BigQuery Agent Analytics](https://docs.cloud.google.com/bigquery/docs/configure-agent-analytics)
  for CMEK, VPC-SC, and cross-project setup.

## Installation

=== "Python"

    Install the `bigquery-analytics` extra (includes BigQuery, Cloud Storage,
    and `pyarrow`, which is no longer included in `gcp`):

    ```bash
    pip install "google-adk[bigquery-analytics]>=2.7.0"
    ```

=== "Go"

    The Go plugin ships as an independent submodule
    (`google.golang.org/adk/plugin/agentanalytics`, tagged
    `plugin/agentanalytics/vX.Y.Z`) so core ADK Go apps avoid BigQuery dependencies:

    ```bash
    go get google.golang.org/adk/v2@latest
    go get google.golang.org/adk/plugin/agentanalytics@v1.0.0
    ```

=== "Java"

    Included in core ADK Java (`com.google.adk:google-adk`, v1.5.0+ for the
    plugin, v1.10.0+ for `event_id` parity):

    ```groovy title="build.gradle"
    implementation 'com.google.adk:google-adk:1.14.0'
    ```

=== "Kotlin"

    JVM-only integrations artifact:

    ```kotlin title="build.gradle.kts"
    implementation("com.google.adk:google-adk-kotlin-integrations:1.5.0")
    ```

## Quickstart

=== "Python"

    ```python title="agent.py"
    import os
    from google.adk.agents import Agent
    from google.adk.apps import App
    from google.adk.models.google_llm import Gemini
    from google.adk.plugins.bigquery_agent_analytics_plugin import BigQueryAgentAnalyticsPlugin

    os.environ["GOOGLE_CLOUD_PROJECT"] = "your-gcp-project-id"
    os.environ["GOOGLE_CLOUD_LOCATION"] = "us-central1"
    os.environ["GOOGLE_GENAI_USE_ENTERPRISE"] = "True"

    plugin = BigQueryAgentAnalyticsPlugin(
        project_id="your-gcp-project-id",
        dataset_id="your-big-query-dataset-id",
    )

    root_agent = Agent(
        model=Gemini(model="gemini-flash-latest"),
        name="my_agent",
        instruction="You are a helpful assistant.",
    )

    app = App(name="my_agent", root_agent=root_agent, plugins=[plugin])
    ```

=== "Go"

    Always call `defer bqPlugin.Close()` because ADK Go's runner does not close plugins:

    ```go title="main.go"
    package main

    import (
    	"context"
    	"fmt"
    	"log"

    	"google.golang.org/adk/plugin/agentanalytics"
    	"google.golang.org/adk/v2/agent"
    	"google.golang.org/adk/v2/agent/llmagent"
    	"google.golang.org/adk/v2/model/gemini"
    	"google.golang.org/adk/v2/plugin"
    	"google.golang.org/adk/v2/runner"
    	"google.golang.org/adk/v2/session"
    	"google.golang.org/genai"
    )

    func run(ctx context.Context) error {
    	bqPlugin, err := agentanalytics.NewBigQueryAgentAnalyticsPlugin(
    		ctx, "your-gcp-project-id", "your-big-query-dataset-id", "agent_events",
    	)
    	if err != nil {
    		return fmt.Errorf("create BigQuery plugin: %w", err)
    	}
    	defer bqPlugin.Close()

    	model, err := gemini.NewModel(ctx, "gemini-flash-latest", &genai.ClientConfig{
    		Project: "your-gcp-project-id", Location: "us-central1", Backend: genai.BackendVertexAI,
    	})
    	if err != nil {
    		return fmt.Errorf("create model: %w", err)
    	}

    	myAgent, err := llmagent.New(llmagent.Config{
    		Name: "my_agent", Model: model, Instruction: "You are a helpful assistant.",
    	})
    	if err != nil {
    		return fmt.Errorf("create agent: %w", err)
    	}

    	sessionService := session.InMemoryService()
    	r, err := runner.New(runner.Config{
    		AppName: "my_agent", Agent: myAgent, SessionService: sessionService,
    		PluginConfig: runner.PluginConfig{Plugins: []*plugin.Plugin{bqPlugin}},
    	})
    	if err != nil {
    		return fmt.Errorf("create runner: %w", err)
    	}

    	sessResp, err := sessionService.Create(ctx, &session.CreateRequest{
    		AppName: "my_agent", UserID: "demo_user",
    	})
    	if err != nil {
    		return fmt.Errorf("create session: %w", err)
    	}

    	userMsg := genai.NewContentFromText("Hello!", genai.RoleUser)
    	for _, err := range r.Run(ctx, "demo_user", sessResp.Session.ID(), userMsg, agent.RunConfig{}) {
    		if err != nil {
    			return fmt.Errorf("run error: %w", err)
    		}
    	}
    	return nil
    }

    func main() {
    	if err := run(context.Background()); err != nil {
    		log.Fatal(err)
    	}
    }
    ```

=== "Java"

    ```java title="Agent.java"
    import com.google.adk.agents.LlmAgent;
    import com.google.adk.models.Gemini;
    import com.google.adk.plugins.Plugin;
    import com.google.adk.plugins.agentanalytics.BigQueryAgentAnalyticsPlugin;
    import com.google.adk.plugins.agentanalytics.BigQueryLoggerConfig;
    import com.google.adk.runner.InMemoryRunner;
    import com.google.common.collect.ImmutableList;

    public final class Agent {
      public static void main(String[] args) throws Exception {
        Plugin bqLoggingPlugin = new BigQueryAgentAnalyticsPlugin(
            BigQueryLoggerConfig.builder()
                .projectId("your-gcp-project-id")
                .datasetId("your-big-query-dataset-id")
                .tableName("agent_events") // Optional; default in v1.8.0+
                .build());

        InMemoryRunner runner = new InMemoryRunner(
            LlmAgent.builder()
                .model(Gemini.builder().modelName("gemini-2.5-flash").build())
                .name("my_agent")
                .instruction("You are a helpful assistant.")
                .build(),
            "my_agent",
            ImmutableList.of(bqLoggingPlugin));

        // Use runner, then flush and close plugin
        runner.close().blockingAwait();
      }
    }
    ```

=== "Kotlin"

    ```kotlin title="build.gradle.kts"
    implementation("com.google.adk:google-adk-kotlin-integrations:1.5.0")
    ```

    ```kotlin title="BigQueryAnalyticsExample.kt"
    --8<-- "examples/kotlin/snippets/integrations/BigQueryAnalyticsExample.kt:quickstart"
    ```

    Set `location` to match your dataset (defaults to `"US"`). Logging failures
    never fail the turn; inspect logs under
    `com.google.adk.kt.plugins.agentanalytics.BigQueryAgentAnalyticsPlugin`.

### Run and test agent

Run your agent and verify logged rows in the [BigQuery Console](https://console.cloud.google.com/bigquery) (for full SQL recipes, see [Analyze agent analytics data](https://docs.cloud.google.com/bigquery/docs/analyze-agent-analytics)):

```sql
SELECT timestamp, event_type, content
FROM `your-gcp-project-id.your-big-query-dataset-id.agent_events`
ORDER BY timestamp DESC
LIMIT 20;
```

## Configuration options {#configuration-options}

Full option and method references live in each language's SDK API docs:

- **Python**: [/api-reference/python/google.adk.plugins.bigquery_agent_analytics_plugin.html](/api-reference/python/google.adk.plugins.bigquery_agent_analytics_plugin.html)
- **Go**: [pkg.go.dev/google.golang.org/adk/plugin/agentanalytics](https://pkg.go.dev/google.golang.org/adk/plugin/agentanalytics)
- **Java**: [/api-reference/java/com/google/adk/plugins/agentanalytics/package-summary.html](/api-reference/java/com/google/adk/plugins/agentanalytics/package-summary.html)
- **Kotlin**: [/api-reference/kotlin/google-adk-kotlin-integrations-bigquery/com.google.adk.kt.plugins.agentanalytics/index.html](/api-reference/kotlin/google-adk-kotlin-integrations-bigquery/com.google.adk.kt.plugins.agentanalytics/index.html)

In **Go**, initialize from `cfg := agentanalytics.DefaultConfig()` (a zero-value
`agentanalytics.Config{}` sets `Enabled: false`), or use
`NewBigQueryAgentAnalyticsPluginWithClients` with `bqClient == nil` for
pre-provisioned append-only deployments (`cfg.ViewDefinitions()` exports view
DDL). In **Java**, `createViews` defaults to `false` (Python and Go default to
`true`).

| Capability | Python (`BigQueryLoggerConfig`) | Go (`agentanalytics.Config`) | Java (`BigQueryLoggerConfig.builder()`) |
| --- | --- | --- | --- |
| Table & location | `project_id`, `dataset_id`, `table_id`, `location="US"` | `ProjectID`, `DatasetID`, `TableName="agent_events"` | `projectId`, `datasetId`, `tableName`, `location("us")` |
| Batching & flush | `batch_size=1`, `batch_flush_interval=1.0`, `flush_on_run_end=True`, `queue_max_size=10000`, `shutdown_timeout=10.0` | `BatchSize=100`, `BatchFlushIntv=1s`, `DisableFlushOnRunEnd=false`, `QueueMaxSize=10000`, `ShutdownTimeout=10s` | `batchSize(1)`, `batchFlushInterval(1s)`, `queueMaxSize(10000)`, `shutdownTimeout(10s)` |
| GCS offloading | `gcs_bucket_name`, `connection_id`, `max_content_length=500*1024` | `GCSBucketName`, `GCSPathPrefix`, `ConnectionID`, `MaxContentLen`, `MaxInlineTextSize=32*1024` | `gcsBucketName`, `connectionId`, `maxContentLength(500*1024)` |
| Delivery | `exactly_once_delivery=False` | `ExactlyOnceDelivery=false` | *(default stream with `event_id` in v1.10.0+)* |
| Views & eval views | `create_views=True`, `create_eval_views=False`, `view_prefix="v"` | `DisableCreateViews=false`, `CreateEvalViews=false`, `ViewPrefix="v"` | `createViews(false)`, `viewPrefix("v")` |
| Filtering & metadata | `event_allowlist`, `event_denylist`, `payload_column_denylist`, `custom_metadata_allowlist`, `enable_otel_correlation` | `EventAllowlist`, `EventDenylist`, `PayloadColumnDenylist`, `CustomMetadataAllowlist`, `EnableOTelCorrelation` | `eventAllowlist`, `eventDenylist`, `customTags` |
| Classifiers & callbacks | `tool_result_classifier`, `on_schema_ready`, `on_schema_error` | `ToolResultClassifier`, `OnSchemaReady`, `OnSchemaError` | — |

**Cloud Storage prefix (`GCSPathPrefix`, Go `v1.0.0+`)**: Valid UTF-8 prefixes
(`<= 750` bytes, no `~`, control chars, or Unicode line/paragraph separators,
and not starting with `.well-known/acme-challenge`) are used verbatim; all
others are rewritten to `<base>~<sha256>` (`<= 467` bytes) to satisfy
hierarchical-namespace folder limits.

### Shared background writer {#shared-background-writer}

In **Python (`v2.10.0+`)**, `flush_on_run_end=False` (or
`use_dedicated_background_loop=True`) runs writes on a shared background thread
keyed by table and credential identity (`credentials_identifier` overrides
sharing). Default `close()` / `shutdown()` detaches without stopping the shared
writer; pass `close_background_transport=True` or call
`await BigQueryAgentAnalyticsPlugin.close_shared_background_transports(timeout=10.0)`
to drain all writers at exit.

### Delivery and deduplication {#delivery-and-deduplication}

Every row in **Python (`v2.7.0+`)**, **Go (`v0.1.0+`)**, and **Java
(`v1.10.0+`)** receives a 32-hex `event_id` before enqueue, preserved across
retries for downstream `QUALIFY` deduplication (see [Analyze agent analytics data](https://docs.cloud.google.com/bigquery/docs/analyze-agent-analytics)).

Enabling `exactly_once_delivery=True` (`ExactlyOnceDelivery: true` in Go) uses
committed streams and explicit offsets to avoid duplicate appends on ambiguous
retries within a live processor (at extra `CreateWriteStream` quota cost), but is
still best-effort under retry exhaustion, offset conflicts, or rotation backoff
(30s in Go). On `OutOfRange`, `NotFound`, or unexpected `AlreadyExists`, Go
rotates in-place once and resends when safe, while Python drops the batch and
rotates before the next batch; Python also treats `ResourceExhausted` after
`Send` as an unknown outcome.

### Table readiness callbacks {#table-readiness-callbacks}

In **Python (`v2.12.0+`)**, `on_schema_error(exc)` runs on a worker thread if
table read/create/upgrade fails (not view-only errors), and `on_schema_ready()`
runs once the table and requested views exist. Both must be synchronous, fast,
and picklable if you pickle the plugin. After success, later setups skip the
check unless cancelled or `create_analytics_views()` fails.

#### Go callbacks (`OnSchemaReady` / `OnSchemaError`) {#go-table-readiness-callbacks}

In **Go (`v1.0.0+`)**, the constructor checks the table synchronously (calling
`OnSchemaError(err)` before returning on failure) and creates views in a
background goroutine (`OnSchemaReady()` runs once startup views finish, or
synchronously if no startup views are configured). Do not call `Close()`,
`SyncViews()`, or `SyncEvalViews()` inside `OnSchemaReady`.

### Tool result error classification {#tool-result-error-classification}

In **Python (`v2.13.0+`)**, `after_tool_callback` classifies returned payloads in
order: (1) `ReflectAndRetryToolPlugin` error responses (`TOOL_ERROR`, avoiding
duplicates when the analytics plugin is registered before retry plugins),
(2) synchronous `tool_result_classifier(tool=..., tool_args=..., tool_context=..., result=...)`
returning `ToolResultClassification(status="ERROR" | "OK", error_message=...)`
or `None`, and (3) built-in MCP `isError` / `is_error == True`. For `ERROR`,
the raw result is omitted from `content` and `content_formatter` does not run
on `error_message`.

#### Go classifier (`ToolResultClassifier`) {#go-tool-result-error-classification}

In **Go (`v1.0.0+`)**, `AfterToolCallback` checks (1) `OnToolErrorCallback` /
unrecovered errors, (2) `plugin/retryandreflect` responses, (3)
`cfg.ToolResultClassifier(ctx, tool, args, result)` returning
`agentanalytics.ToolResultVerdict{Decision, ErrorMessage}` (`ToolResultError`,
`ToolResultOK`, or `ToolResultAbstain`), and (4) `isError` / `is_error == true`.
Keep the callback fast, concurrent-safe, and read-only.

### Agent evaluation views {#agent-evaluation-views}

In **Python (`v2.13.0+`)**, set `create_eval_views=True` (or call
`await plugin.create_eval_views()`) to create `{view_prefix}_eval_session` and
`{view_prefix}_eval_turn` reconstructing `AgentData` evaluation traces (`trace`
JSON) and metrics. Requires the `content` column and all 15 core/HITL/workflow
events (filtering them raises `ValueError`). Filter or materialize a
`timestamp`-partitioned slice on large tables; see [Agent analytics schema reference](https://docs.cloud.google.com/bigquery/docs/agent-analytics-schema).

#### Go evaluation views (`CreateEvalViews` / `SyncEvalViews`) {#go-agent-evaluation-views}

In **Go (`v1.0.0+`)**, set `CreateEvalViews: true`, call
`bqPlugin.SyncEvalViews(ctx)`, or generate DDL via `cfg.EvalViewDefinitions()`.
The same `content` and 15-event requirements apply (including rejecting `"EVENT"`
in `EventDenylist`).

## Event types and payloads {#event-types}

- **Python (`v2.13.0+`) & Go (`v1.0.0+`)**: All **28 event types** and 25
  per-event views (`HITL_*_COMPLETED` rows query the base table).
- **Java (`v1.10.0+`)**: **21 of 28 event types** (excludes `INVOCATION_ERROR`,
  `AGENT_ERROR`, `AGENT_TRANSFER`, `AGENT_STATE_CHECKPOINT`,
  `EVENT_COMPACTION`, `NODE_OUTPUT`, and `NODE_ERROR`) with `event_id` parity.
- **Kotlin (`v0.8.0+`)**: `INVOCATION_STARTING` and `INVOCATION_COMPLETED` only.

| Category | Event Types |
| --- | --- |
| Lifecycle | `USER_MESSAGE_RECEIVED`, `INVOCATION_STARTING`, `INVOCATION_COMPLETED`, `INVOCATION_ERROR`, `AGENT_STARTING`, `AGENT_COMPLETED`, `AGENT_ERROR`, `AGENT_RESPONSE` |
| LLM & Tools | `LLM_REQUEST`, `LLM_RESPONSE`, `LLM_ERROR`, `TOOL_STARTING`, `TOOL_COMPLETED`, `TOOL_ERROR` |
| State & A2A | `STATE_DELTA`, `A2A_INTERACTION` |
| HITL | `HITL_CREDENTIAL_REQUEST`, `HITL_CONFIRMATION_REQUEST`, `HITL_INPUT_REQUEST`, `HITL_*_REQUEST_COMPLETED` |
| ADK 2.0 Workflows | `AGENT_TRANSFER`, `AGENT_STATE_CHECKPOINT`, `EVENT_COMPACTION`, `TOOL_PAUSED`, `NODE_OUTPUT`, `NODE_ERROR` |

See [Agent analytics event reference](https://docs.cloud.google.com/bigquery/docs/agent-analytics-events)
for the complete 28-event catalog and JSON payload schemas.

### Human-in-the-Loop (HITL) events {#hitl-events}

The plugin detects ADK's synthetic HITL calls (`adk_request_credential`,
`adk_request_confirmation`, `adk_request_input`) in addition to `TOOL_*` rows.
Auto-created views exist only for request events (`v_hitl_*_request`); query
`*_COMPLETED` rows directly from `agent_events` (`WHERE event_type LIKE 'HITL_%_COMPLETED'`).

### Agent workflow and pause/resume events (ADK 2.0) {#adk-2-events}

Python (`v2.3.0+`) and Go (`v0.1.0+`) attach an `attributes.adk` envelope
(`schema_version`, `source_event_id`, `node`, `branch`, `scope`, `route`,
`pause_kind`, `function_call_id`) to trace workflows and pair `TOOL_PAUSED`
with its resumed `TOOL_COMPLETED` via `function_call_id`. Java (`v1.7.0+`)
supports `TOOL_PAUSED` only, with `pause_kind` and `function_call_id` at the
top level of `attributes`.

In **Go `v1.0.0+`**, `AGENT_STATE_CHECKPOINT` is emitted whenever reserved
checkpoint keys appear in `CustomMetadata`, without requiring
`CustomMetadataAllowlist` (matching Python). To suppress checkpoints, add
`agentanalytics.EventAgentStateCheckpoint` (`"AGENT_STATE_CHECKPOINT"`) to
`EventDenylist`.

### Go and Python plugin differences {#go-python-differences}

Both plugins write the same table and view schemas; keep these intentional
cross-language differences in mind when writing SQL:

#### Rows and views {#go-python-row-differences}

- **Streamed `LLM_RESPONSE`**: Go writes one final row per call (setting
  `time_to_first_token_ms` from chunks); Python writes a row per partial chunk.
- **Error rows**: Go infers `AGENT_ERROR` / `INVOCATION_ERROR` at run end from
  open spans and unrecovered errors; Python logs on exception unwind.
- **`function_call_id` & `AGENT_RESPONSE`**: Go sets `function_call_id` on all
  tool rows (filter resumed pauses with `pause_kind = 'tool'`) and writes
  `AGENT_RESPONSE` on final events with both text and function responses.
- **Filters, `temp:`, OTel & GCS**: Go supports `"EVENT"` as an event-filter
  alias, drops top-level lowercase `temp:` keys from `STATE_DELTA` (instead of
  `[REDACTED]`), ignores partial session events, records ambient OTel spans as
  `parent_span_id`, and omits `LLM_RESPONSE` text above `MaxInlineTextSize`
  (`32 KiB`) from `content.response` when offloaded to GCS.

#### Payloads, metadata, and defaults {#go-python-payload-differences}

- **Metadata**: Go adds `event_id`, `author`, `adk_version`, `invocation_id`,
  `session_id`, `user_id`, `isolation_scope`, `artifact_delta`,
  `skip_summarization`, and `escalate` to `attributes.adk`, omits empty `node`,
  sorts tools by name with camelCase schemas, and adds `function_call` in
  `content_parts`.
- **Defaults & APIs**: Go defaults to `BatchSize: 100`, `InitialDelay: 100ms`
  (Python: `1`, `1.0s`), passes typed values to `ContentFormatter` (skipping
  empty payloads), and returns a typed `DropStats` struct.

## Security: Avoid logging sensitive credentials {#security-credentials}

!!! warning "Do not log OAuth tokens, API keys, or client secrets"

    In **Python**, built-in redaction normalizes keys by lowercasing and
    mapping hyphens to underscores, so **camelCase keys (`clientSecret`,
    `accessToken`) are not matched**. ADK serializes `adk_request_credential`
    arguments with camelCase aliases (Python v2.9.0+ strips client secrets and
    v2.12.0+ strips API keys, HTTP Basic passwords, and bearer tokens, but
    **OAuth2 access tokens and custom camelCase tool arguments remain
    unredacted**). In **Go**, key normalization strips both `_` and `-`,
    matching `snake_case`, `kebab-case`, and `camelCase` variants.

### Built-in redaction {#built-in-redaction}

Built-in redaction (Python, Go `v0.1.0+`, Java `v1.7.0+`) recursively masks
sensitive keys (`client_secret`, `access_token`, `refresh_token`, `id_token`,
`api_key`, `password`, `private_key`, `authorization`, `token`, `secret`, `sig`,
`signature`, and cloud credential headers) with `"[REDACTED]"` and scrubs
URIs, headers, and tracebacks fail-closed.

- **State scope (`temp:` vs `secret:`)**: ADK defines `app:`, `user:`, and
  `temp:`. Only **`temp:`** is redacted (`[REDACTED]` in Python/Java; removed
  from `state_delta` in Go). **`secret:` has no special meaning**—do not store
  secrets under `secret:`, and audit existing tables if relied on previously.
- **Custom `content_formatter`**: Runs before built-in redaction on raw `content`
  only (does not mask `attributes` such as `session_metadata`, `state_delta`, or
  `user_id`). Errors write `"[FORMATTER_FAILED]"` in Python/Go (and increment
  `formatter_failed` / `FormatterFailureDrops`) or drop content in Java. Go
  formatters must deep-clone modified values; Java formatters must be
  thread-safe and non-blocking.
- **Skip dedicated HITL or checkpoint rows with `event_denylist`** (suppresses
  `HITL_CREDENTIAL_REQUEST` and `HITL_CREDENTIAL_REQUEST_COMPLETED` rows, but
  does not strip suspended/resumed payloads from `TOOL_PAUSED`,
  `USER_MESSAGE_RECEIVED`, or subsequent `LLM_REQUEST` turns—pair with
  pre-redaction or `content_formatter` for end-to-end credential suppression):
    - **Python**: `BigQueryLoggerConfig(event_denylist=["HITL_CREDENTIAL_REQUEST", "HITL_CREDENTIAL_REQUEST_COMPLETED"])`
    - **Go**: `cfg.EventDenylist = []string{agentanalytics.EventHITLCredentialRequest, agentanalytics.EventHITLCredentialCompleted, agentanalytics.EventAgentStateCheckpoint}`
    - **Java**: `BigQueryLoggerConfig.builder().eventDenylist(ImmutableList.of("HITL_CREDENTIAL_REQUEST", "HITL_CREDENTIAL_REQUEST_COMPLETED")).build()`

## Deploy to Agent Runtime with the plugin {#deploy-agent-runtime}

Requires **ADK Python v1.24.0+** for synchronous end-of-run flushing before the
serverless runtime pauses. Complete [Agent Runtime setup](/deploy/agent-runtime/deploy/#setup-cloud-project),
define `app = App(..., plugins=[plugin])` in `my_bq_agent/agent.py`, include
`google-adk[bigquery-analytics]>=2.7.0` in `requirements.txt`, and deploy:

```shell
adk deploy agent_engine \
    --project=$PROJECT_ID \
    --region=us-central1 \
    --staging_bucket=gs://your-staging-bucket \
    --display_name="My BQ Analytics Agent" \
    --adk_app=agent.app \
    my_bq_agent
```

Or deploy via `client.agent_engines.create(agent=app, ...)` and test with
`agent.stream_query(...)`. If rows are missing, inspect Cloud Logging
(`resource.type="reasoning_engine"`) with `google_adk` at `DEBUG`, verify
`roles/bigquery.dataEditor` and `roles/bigquery.jobUser` on the runtime service
account, and test with `batch_size=1`, `batch_flush_interval=0.1`.

## Operations and downstream analytics

### Tracing and fork safety

- **No duplicate Cloud Trace spans**: The plugin tracks spans on an internal
  16-hex stack (never calling `tracer.start_span`) and inherits `trace_id` from
  any ambient OTel span for Cloud Trace correlation.
- **Fork safety (Python)**: Enables `GRPC_ENABLE_FORK_SUPPORT=1` and resets
  inherited channels/streams via `os.register_at_fork`. Prefer `spawn` (or
  Gunicorn `--preload` + lazy init / `post_fork`), and call `await plugin.flush()`
  before forking since parent-queued rows are not replayed in child processes.

### Dropped-event observability {#dropped-event-observability}

BigQuery logging is best-effort. Poll and alert on non-zero deltas:

- **Python (`plugin.get_drop_stats()`)**: Dict of loss reasons (`queue_full`,
  `arrow_prep_failed`, `retry_exhausted`, `permission_denied`, `not_found`,
  `non_retryable`, `unexpected_error`, `shutdown_timeout`, `shutdown_cancelled`,
  `offset_conflict`, `setup_unavailable`, `shutdown_race`, `stale_loop`) and
  sentinel incident counts (`formatter_failed`, `content_parse_failed`).
- **Go (`bqPlugin.DropStats()` via `agentanalytics.AsBigQueryAgentAnalyticsPlugin`)**:
  `agentanalytics.DropStats` (`QueueFullDrops`, `AppendFailureDrops`,
  `RowErrorDrops`, `ShutdownDrops`, `TotalDrops`, and `FormatterFailureDrops`).
- **Java (`plugin.getDropStats()`, v1.7.0+)**: `ImmutableMap<String, Long>`
  (`queue_full`, `append_error`, `serialization_error`, `after_close`,
  `shutdown_timeout`, `writer_permit_exhausted`, `writer_create_error`,
  `late_after_finalize`).

### The context graph and downstream analytics {#context-graph}

The [BigQuery Agent Analytics SDK](https://github.com/GoogleCloudPlatform/BigQuery-Agent-Analytics-SDK/tree/main)
provides `bqaa context-graph --property-graph` to materialize a BigQuery
[property graph](https://cloud.google.com/bigquery/docs/graph-overview) of
agent decisions (queryable with GQL), trajectory evaluation, trace
visualization, a [Looker Studio template](https://googlecloudplatform.github.io/BigQuery-Agent-Analytics-SDK/)
(switch the copied data source to **Viewer's credentials** before sharing), and
a [Looker Block](https://marketplace.looker.com/marketplace/detail/agent_analytics).
See [Analyze agent analytics data](https://docs.cloud.google.com/bigquery/docs/analyze-agent-analytics)
for SQL recipes and dashboards.

- **Feedback**: [bqaa-feedback@google.com](mailto:bqaa-feedback@google.com)
- **Plugin source**: [Python](https://github.com/google/adk-python/blob/main/src/google/adk/plugins/bigquery_agent_analytics_plugin.py) · [Go](https://github.com/google/adk-go/tree/main/plugin/agentanalytics)
