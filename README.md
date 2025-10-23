# Claude Code OpenTelemetry Monitoring

Monitor your Claude Code usage with a complete observability stack: OpenTelemetry Collector, Prometheus, Loki, and Grafana. Track costs, token usage, cache efficiency, active time, and code changes in real-time with pre-built dashboards. Watch and check back for updates to this repo for step by step instructions for setting this up. Still fine-tuning the charts and panels. 

An accompanying Claude Code usage metrics MCP server allows Claude Code access to it's own usage metrics (see below example along with example screenshots).

**Based on Official Claude Code Documentation:** Claude Code supports OpenTelemtry as described in Anthropic's [official monitoring documentation](https://docs.claude.com/en/docs/claude-code/monitoring-usage).

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
