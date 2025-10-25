# Claude Code OpenTelemetry Monitoring

Monitor your Claude Code usage with a complete observability stack: OpenTelemetry Collector, Prometheus, Loki, and Grafana. Track costs, token usage, cache efficiency, active time, and code changes in real-time with pre-built dashboards. Watch and check back for updates to this repo for step by step instructions for setting this up. Still fine-tuning the charts and panels. 

An accompanying Claude Code usage metrics MCP server allows Claude Code access to it's own usage metrics (see below example along with example screenshots).

**Based on Official Claude Code Documentation:** Claude Code supports OpenTelemtry as described in Anthropic's [official monitoring documentation](https://docs.claude.com/en/docs/claude-code/monitoring-usage).

---

## Cache Performance Optimization

Effective prompt caching can reduce your Claude Code costs by up to **90%**. This setup helps you monitor and optimize cache performance in real-time.

### Quick Cache Insights

**Example Performance Metrics:**
- **Cache Hit Rate:** 93.53% (40.28M cached reads vs 2.49M cache creation)
- **Cost Savings:** ~80% reduction ($129/day → $27/day)
- **Cache ROI:** 16.2:1 (savings vs cache creation cost)

### Key Optimization Strategies

1. **Stable Context First** - Place large, unchanging context at the beginning of prompts
2. **Long Sessions** - Group related tasks to leverage warm cache (5-minute TTL)
3. **Batch Operations** - Process similar tasks together to maximize cache reuse
4. **Monitor Efficiency** - Track cache hit rate via Grafana dashboards and MCP server

### Cache Monitoring Dashboard Panels

This setup includes pre-built panels for:
- **Cache Hit Rate %** - Real-time cache efficiency gauge
- **Cached Write Cost (24h)** - Cost savings from cache reuse
- **Token Usage by Type** - Breakdown of cache reads vs creation
- **Cache Efficiency Trend** - Historical cache performance

### Target Performance Goals

| Metric | Target | Current Example | Status |
|--------|--------|-----------------|--------|
| Cache Hit Rate | > 90% | 93.53% | ✅ Excellent |
| Cache ROI | > 10:1 | 16.2:1 | ✅ Excellent |
| Daily Cost Efficiency | > 75% savings | ~80% | ✅ Excellent |

For detailed optimization strategies, cost analysis, and advanced techniques, see [CACHE_OPTIMIZATION.md](CACHE_OPTIMIZATION.md).

---

## Grafana Dashboards

![Claude Code Monitoring](screenshots/claude-code-opentelemetry-grafana-prometheus-loki-1.png)

![Claude Code Monitoring](screenshots/claude-code-opentelemetry-grafana-prometheus-loki-2.png)

![Claude Code Monitoring](screenshots/claude-code-opentelemetry-grafana-prometheus-loki-3.png)

## Claude Code Usage Metrics MCP Server

```bash
claude mcp add --transport stdio metrics -s user -- uv run --directory /path/to/your/mcp-server metrics-server
```

```bash
claude mcp list
Checking MCP server health...

context7: https://mcp.context7.com/sse (SSE) - ✓ Connected
cf-docs: https://docs.mcp.cloudflare.com/sse (SSE) - ✓ Connected
metrics: uv run --directory /path/to/your/mcp-server metrics-server - ✓ Connected
```

### Claude Code Usage Metrics MCP Server Demo

This section showcases the actual JSON responses from all 14 MCP tools, demonstrating the format and data you can expect.

#### 1. get_current_cost

Returns today's total USD cost from Prometheus.

```json
{
  "metric": "Total Cost Today",
  "value": 27.149809833783127,
  "formatted": "$27.1498",
  "unit": "currencyUSD"
}
```

#### 2. get_token_usage

Returns token breakdown by type (input, output, cacheCreation, cacheRead).

```json
{
  "metric": "Token Usage (24h)",
  "breakdown": {
    "cacheCreation": 2487212.387455661,
    "cacheRead": 40278591.98958367,
    "input": 16407.023220696385,
    "output": 283598.58088185877
  },
  "total": 43065809.98114189
}
```

#### 3. get_cache_efficiency

Returns cache hit rate percentage.

```json
{
  "metric": "Cache Hit Rate",
  "value": 93.52789643057625,
  "formatted": "93.53%",
  "unit": "percent"
}
```

#### 4. get_available_metrics

Returns comprehensive reference of all Prometheus metrics and Loki log events (truncated for brevity).

```json
{
  "prometheus_metrics": {
    "claude_code_cost_usage_USD_total": {
      "description": "Session cost in USD",
      "type": "counter",
      "labels": ["session_id", "model", "deployment_environment", "service_name"],
      "example_query": "sum by (session_id) (increase(claude_code_cost_usage_USD_total[24h]))"
    },
    "claude_code_token_usage_tokens_total": {
      "description": "Token usage by type",
      "type": "counter",
      "labels": ["type (input|output|cacheCreation|cacheRead)", "session_id", "model"],
      "example_query": "sum by (type) (increase(claude_code_token_usage_tokens_total[24h]))"
    }
  },
  "loki_log_events": {
    "tool_result": {
      "event_name_label": "tool_result",
      "description": "Tool execution results",
      "attributes": ["tool_name", "duration_ms (numeric)", "success (true|false)", "error", "decision_source", "session_id"],
      "example_query": "{service_name=\"claude-code\", event_name=\"tool_result\"} | unwrap duration_ms",
      "note": "Use label selectors, NOT | json filter (OTLP format)"
    },
    "user_prompt": {
      "event_name_label": "user_prompt",
      "description": "User prompt submissions (privacy-sensitive)",
      "attributes": ["prompt (full text)", "prompt_length (numeric)", "session_id", "model"],
      "example_query": "{service_name=\"claude-code\", event_name=\"user_prompt\"}",
      "note": "Requires OTEL_LOG_USER_PROMPTS=1 environment variable"
    }
  },
  "common_patterns": {
    "promql_patterns": [
      "Cache efficiency: sum(increase(...{type=\"cacheRead\"}[24h])) / clamp_min(sum(increase(...[24h])), 1) * 100",
      "Per-session cost: sum by (session_id) (increase(claude_code_cost_usage_USD_total[24h]))"
    ],
    "logql_patterns": [
      "Slow tools: {service_name=\"claude-code\", event_name=\"tool_result\"} | duration_ms > 5000",
      "Tool failures: {service_name=\"claude-code\", event_name=\"tool_result\", success=\"false\"}"
    ]
  }
}
```

#### 5. get_recent_prompts

Returns recent user prompt submissions (requires `OTEL_LOG_USER_PROMPTS=1`).

```json
{
  "count": 10,
  "prompts": [
    {
      "prompt": "metrics MCP server is installed @mcp-server/README.md at claude code user scope, I need for you to test each mcp tool to ensure it's working",
      "length": "140",
      "timestamp": "2025-10-23T18:23:01.865Z",
      "model": "N/A",
      "session_id": "dcc1d3a8-fded-48db-827b-fcf1c1f3a9d3"
    },
    {
      "prompt": "what is my token usage breakdown",
      "length": "32",
      "timestamp": "2025-10-23T17:56:27.830Z",
      "model": "N/A",
      "session_id": "fd9afd28-606a-4e78-b115-5ad5443082fb"
    }
  ],
  "truncated": false
}
```

#### 6. get_tool_results

Returns recent tool execution logs with performance metrics (truncated for brevity).

```json
{
  "count": 20,
  "results": [
    {
      "tool_name": "mcp__metrics__get_cache_efficiency",
      "duration_ms": "26",
      "success": "true",
      "error": "",
      "decision_source": "user_permanent",
      "timestamp": "2025-10-23T18:24:23.528Z",
      "session_id": "dcc1d3a8-fded-48db-827b-fcf1c1f3a9d3"
    },
    {
      "tool_name": "Bash",
      "duration_ms": "1354",
      "success": "true",
      "error": "",
      "decision_source": "config",
      "timestamp": "2025-10-23T18:20:40.931Z",
      "session_id": "e24ee0d8-4c7a-416e-aa8e-767fbbadb75c"
    }
  ],
  "truncated": false
}
```

#### 7. query_prometheus

Execute arbitrary PromQL queries against Prometheus.

**Example query:** `sum(increase(claude_code_cost_usage_USD_total[24h]))`

```json
{
  "query": "sum(increase(claude_code_cost_usage_USD_total[24h]))",
  "value": 27.303288749445116,
  "raw_result": {
    "status": "success",
    "data": {
      "resultType": "vector",
      "result": [
        {
          "metric": {},
          "value": [1761243919.881, "27.303288749445116"]
        }
      ]
    }
  }
}
```

#### 8. query_loki

Execute arbitrary LogQL queries against Loki.

**Example query:** `{service_name="claude-code"} |= "tool_result"` (limit 5)

```json
{
  "query": "{service_name=\"claude-code\"} |= \"tool_result\"",
  "count": 5,
  "log_lines": [
    "claude_code.tool_result",
    "claude_code.tool_result",
    "claude_code.tool_result",
    "claude_code.tool_result",
    "claude_code.tool_result"
  ],
  "truncated": false
}
```

### Dashboard Analysis Tools

#### 9. list_dashboard_panels

Lists all panels from both Grafana dashboards (truncated for brevity - shows 103 total panels).

```json
{
  "total_panels": 103,
  "panels": [
    {
      "id": 28,
      "title": "Real-Time Cost Burn Rate ($/hour)",
      "type": "stat",
      "description": "Current cost burn rate in dollars per hour based on last 5 minutes of activity",
      "has_queries": true
    },
    {
      "id": 27,
      "title": "Total Cost Today",
      "type": "stat",
      "description": "Total Cost Today",
      "has_queries": true
    },
    {
      "id": 3,
      "title": "Cache Hit Rate %",
      "type": "gauge",
      "description": "Cache Hit Rate %",
      "has_queries": true
    }
  ]
}
```

#### 10. find_panel_by_name

Search panels by keyword (case-insensitive, partial match).

**Example search:** `"cost"`

```json
{
  "search": "cost",
  "matches": 42,
  "results": [
    {
      "id": 28,
      "title": "Real-Time Cost Burn Rate ($/hour)",
      "type": "stat",
      "description": "Current cost burn rate in dollars per hour based on last 5 minutes of activity"
    },
    {
      "id": 27,
      "title": "Total Cost Today",
      "type": "stat",
      "description": "Total Cost Today"
    },
    {
      "id": 36,
      "title": "Cost Forecast",
      "type": "stat",
      "description": "Projected daily and monthly cost based on current usage patterns"
    }
  ]
}
```

#### 11. get_panel_query

Extract exact PromQL/LogQL queries from a specific panel.

**Example panel:** `"Total Cost Today"`

```json
{
  "panel": "Total Cost Today",
  "queries": [
    "sum(increase(claude_code_cost_usage_USD_total[24h]))",
    "avg_over_time((sum(increase(claude_code_cost_usage_USD_total[24h])))[7d:24h])"
  ],
  "primary_query": "sum(increase(claude_code_cost_usage_USD_total[24h]))"
}
```

#### 12. explain_panel_query

Explain a panel's query with breakdown and live value.

**Example panel:** `"Cache Hit Rate"`

```json
{
  "panel": "Cache Hit Rate %",
  "description": "Cache Hit Rate %",
  "query": "sum(increase(claude_code_token_usage_tokens_total{type=\"cacheRead\"}[24h])) /\nclamp_min(sum(increase(claude_code_token_usage_tokens_total[24h])), 1) * 100",
  "explanation": {
    "query": "sum(increase(claude_code_token_usage_tokens_total{type=\"cacheRead\"}[24h])) /\nclamp_min(sum(increase(claude_code_token_usage_tokens_total[24h])), 1) * 100",
    "components": [
      "sum - Total across all series",
      "increase - Total change over time range",
      "clamp_min - Prevent division by zero",
      "[24h] - Over the last 24h",
      "* 100 - Multiply by 100",
      "Filter: type=\"cacheRead\""
    ],
    "meaning": "This query calculates metrics from claude_code_token_usage_tokens_total. sum - Total across all series increase - Total change over time range clamp_min - Prevent division by zero"
  },
  "current_value": 93.53188202200037,
  "unit": null
}
```

#### 13. explain_promql_query

Break down the structure of a PromQL query.

**Example query:** `rate(claude_code_token_usage_tokens_total{type="output"}[5m])`

```json
{
  "query": "rate(claude_code_token_usage_tokens_total{type=\"output\"}[5m])",
  "components": [
    "rate - Per-second rate of change",
    "[5m] - Over the last 5m",
    "Filter: type=\"output\""
  ],
  "meaning": "This query calculates metrics from claude_code_token_usage_tokens_total. rate - Per-second rate of change [5m] - Over the last 5m Filter: type=\"output\""
}
```

#### 14. explain_logql_query

Break down the structure of a LogQL query.

**Example query:** `{service_name="claude-code"} |= "tool_result" | duration_ms > 1000`

```json
{
  "query": "{service_name=\"claude-code\"} |= \"tool_result\" | duration_ms > 1000",
  "components": [
    "Stream selector: service_name=\"claude-code\"",
    "Line contains: 'tool_result'",
    "Filter: duration_ms > 1000"
  ],
  "meaning": "This query searches logs where Line contains: 'tool_result'"
}
```

![Claude Code Usage Metrics MCP Server Demo](screenshots/claude-code-opentelemetry-mcp-server-demo-2.png)

![Claude Code Usage Metrics MCP Server Demo](screenshots/claude-code-opentelemetry-mcp-server-demo-3.png)

![Claude Code Usage Metrics MCP Server Demo](screenshots/claude-code-opentelemetry-mcp-server-demo-4.png)

![Claude Code Usage Metrics MCP Server Demo](screenshots/claude-code-opentelemetry-mcp-server-demo-5.png)

![Claude Code Usage Metrics MCP Server Demo](screenshots/claude-code-opentelemetry-mcp-server-demo-6.png)

![Claude Code Usage Metrics MCP Server Demo](screenshots/claude-code-opentelemetry-mcp-server-demo-7.png)

![Claude Code Usage Metrics MCP Server Demo](screenshots/claude-code-opentelemetry-mcp-server-demo-8.png)

![Claude Code Usage Metrics MCP Server Demo](screenshots/claude-code-opentelemetry-mcp-server-demo-9.png)

![Claude Code Usage Metrics MCP Server Demo](screenshots/claude-code-opentelemetry-mcp-server-demo-11.png)

![Claude Code Usage Metrics MCP Server Demo](screenshots/claude-code-opentelemetry-mcp-server-demo-12.png)

![Claude Code Usage Metrics MCP Server Demo](screenshots/claude-code-opentelemetry-mcp-server-demo-13.png)

## Claude Code Metrics MCP Server Inspector Tests

![MCP Inspector](screenshots/claude-code-opentelemetry-mcp-server-inspector-tests-1.png)

![MCP Inspector](screenshots/claude-code-opentelemetry-mcp-server-inspector-tests-2.png)

![MCP Inspector](screenshots/claude-code-opentelemetry-mcp-server-inspector-tests-3.png)

![MCP Inspector](screenshots/claude-code-opentelemetry-mcp-server-inspector-tests-4.png)

![MCP Inspector](screenshots/claude-code-opentelemetry-mcp-server-inspector-tests-5.png)

![MCP Inspector](screenshots/claude-code-opentelemetry-mcp-server-inspector-tests-6.png)

![MCP Inspector](screenshots/claude-code-opentelemetry-mcp-server-inspector-tests-7.png)

![MCP Inspector](screenshots/claude-code-opentelemetry-mcp-server-inspector-tests-8.png)

![MCP Inspector](screenshots/claude-code-opentelemetry-mcp-server-inspector-tests-9.png)

## Star History

[![Star History Chart](https://api.star-history.com/svg?repos=centminmod/claude-code-opentelemetry-setup&type=date&legend=top-left)](https://www.star-history.com/#centminmod/claude-code-opentelemetry-setup&type=date&legend=top-left)

---

![Alt](https://repobeats.axiom.co/api/embed/d6ceab9f7729d7422c4e0079c5cfcd80a1878934.svg "Repobeats analytics image")
