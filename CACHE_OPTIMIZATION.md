# Claude Code Cache Performance Optimization Guide

## Executive Summary

This guide provides strategies to optimize Claude Code's prompt caching performance based on OpenTelemetry metrics analysis. Effective cache optimization can reduce costs by up to 90% while maintaining response quality.

## Current Performance Metrics

Based on real-world data from this monitoring setup:

```json
{
  "cache_hit_rate": "93.53%",
  "daily_cost": "$27.15",
  "24h_token_breakdown": {
    "cache_read": "40,278,592 tokens (93.5%)",
    "cache_creation": "2,487,212 tokens (5.8%)",
    "input": "16,407 tokens (0.04%)",
    "output": "283,599 tokens (0.66%)"
  },
  "cache_efficiency_ratio": "16.2:1",
  "total_tokens_24h": "43,065,810"
}
```

## Understanding Cache Metrics

### Cache Hit Rate Formula

```promql
sum(increase(claude_code_token_usage_tokens_total{type="cacheRead"}[24h]))
/ clamp_min(sum(increase(claude_code_token_usage_tokens_total[24h])), 1) * 100
```

**What it means:** Percentage of tokens served from cache vs. total tokens processed.

### Token Types Explained

| Token Type | Description | Cost Impact | Optimization Goal |
|------------|-------------|-------------|-------------------|
| `cacheRead` | Tokens retrieved from cache | 90% cheaper than input tokens | **Maximize** |
| `cacheCreation` | Tokens used to populate cache | Same cost as input tokens | Optimize for reuse |
| `input` | New user input tokens | Standard API pricing | Keep concise |
| `output` | Model response tokens | Standard API pricing | N/A (quality-driven) |

### Cost Breakdown

**Cache Read Pricing:**
- Input tokens: $3.00 per 1M tokens
- Cache creation: $3.75 per 1M tokens (25% markup)
- Cache read: $0.30 per 1M tokens (90% discount)

**Your Current Savings:**
```
Without caching: 43.07M tokens × $3.00/1M = $129.21/day
With caching:
  - Cache reads: 40.28M × $0.30/1M = $12.08
  - Cache creation: 2.49M × $3.75/1M = $9.33
  - Input: 16.4K × $3.00/1M = $0.05
  - Output: 283.6K × $15.00/1M = $4.25
  Total: ~$25.71/day

Savings: $103.50/day (80% reduction)
Monthly savings: ~$3,105
```

## Cache Optimization Strategies

### 1. Maximize Cache Reuse

**Strategy:** Structure prompts and context to leverage prompt caching effectively.

**Best Practices:**

#### Use Stable Context at the Beginning
```markdown
Good: Large, stable context first
❌ [Changing user query]
❌ [Dynamic timestamp]
✅ [Large codebase context - 50KB]
✅ [Documentation - 100KB]
✅ [System instructions - 10KB]

Better: Reorderable sections
✅ [Large stable context - 50KB]
✅ [Semi-stable context - 20KB]
❌ [Volatile user input - 1KB]
```

**Why:** Claude caches prompt prefixes. Stable content at the start maximizes cache hits.

#### Minimize Cache-Breaking Changes
- Avoid timestamps in cached sections
- Keep dynamic content at the end of prompts
- Use consistent formatting and whitespace
- Batch similar queries to leverage warm cache

### 2. Session Management

**Current Data:** Multiple sessions detected in metrics

**Optimization:**
```bash
# Check session-level cache efficiency
sum by (session_id) (increase(claude_code_token_usage_tokens_total{type="cacheRead"}[24h]))
/
sum by (session_id) (increase(claude_code_token_usage_tokens_total[24h])) * 100
```

**Best Practices:**
- Keep long-running sessions for related tasks
- Reuse sessions when working on the same codebase
- Cache persists for 5 minutes of inactivity
- New sessions require cache warm-up (higher initial cost)

### 3. Context Window Optimization

**Cache Tiers (Claude 3.5 Sonnet):**
- Minimum cacheable context: 1,024 tokens
- Optimal cache block size: 2,048+ tokens
- Maximum context: 200,000 tokens

**Strategy:**
```
Total prompt: 150,000 tokens
├─ Cached prefix (128,000 tokens) ✅
│  ├─ Codebase context (100,000 tokens)
│  ├─ Documentation (20,000 tokens)
│  └─ System instructions (8,000 tokens)
└─ Dynamic suffix (22,000 tokens) ❌
   ├─ User query (2,000 tokens)
   └─ Recent history (20,000 tokens)
```

### 4. Model Selection Impact

**Cache Efficiency by Model:**

| Model | Context Window | Cache Support | Use Case |
|-------|---------------|---------------|----------|
| Claude 3.5 Sonnet | 200K tokens | Yes | Best for large codebases |
| Claude 3 Opus | 200K tokens | Yes | Complex reasoning tasks |
| Claude 3 Haiku | 200K tokens | Yes | Fast iterations |

**Recommendation:** Use Sonnet for optimal cache/performance balance.

### 5. Monitoring and Alerting

**Key Metrics to Track:**

```yaml
Critical Alerts:
  - cache_hit_rate < 70%: "Cache efficiency degraded"
  - daily_cost > $50: "Cost threshold exceeded"
  - cache_creation_ratio > 20%: "Excessive cache churn"

Warning Alerts:
  - cache_hit_rate < 85%: "Cache performance below optimal"
  - cost_trend_7d > 20%: "Cost trending upward"
```

**Dashboard Panels to Monitor:**

1. **Cache Hit Rate Trend (7 days)**
```promql
avg_over_time(
  (sum(increase(claude_code_token_usage_tokens_total{type="cacheRead"}[1h]))
   / clamp_min(sum(increase(claude_code_token_usage_tokens_total[1h])), 1) * 100
  )[7d:1h]
)
```

2. **Cache Efficiency by Session**
```promql
sum by (session_id) (increase(claude_code_token_usage_tokens_total{type="cacheRead"}[24h]))
/
sum by (session_id) (increase(claude_code_token_usage_tokens_total[24h])) * 100
```

3. **Cost Savings from Cache**
```promql
# Savings = what you would have paid without cache
(sum(increase(claude_code_token_usage_tokens_total{type="cacheRead"}[24h])) * 0.003
- sum(increase(claude_code_token_usage_tokens_total{type="cacheRead"}[24h])) * 0.0003)
```

4. **Cache Warm-up Cost**
```promql
sum(increase(claude_code_token_usage_tokens_total{type="cacheCreation"}[24h])) * 0.00375
```

### 6. Advanced Optimization Techniques

#### A. Cache-Aware Prompt Engineering

**Before Optimization:**
```
User query: "Fix bug in auth.js"
Context: [Entire codebase - 500 files]
Result: 50% cache hit (inconsistent file ordering)
```

**After Optimization:**
```
Stable context tier 1: [Core framework files - alphabetically sorted]
Stable context tier 2: [Project configuration files]
Stable context tier 3: [Related modules - auth/*]
Dynamic context: [User query + auth.js]
Result: 95% cache hit
```

#### B. Batch Similar Operations

```bash
# Poor: Each query breaks cache
claude code "Review api/users.js"
claude code "Review api/posts.js"
claude code "Review api/comments.js"

# Better: Batch in single session
claude code "Review all API endpoints: users.js, posts.js, comments.js"
```

#### C. Cache Invalidation Awareness

**Cache TTL:** 5 minutes of inactivity

**Strategy:**
- Complete related tasks within 5-minute windows
- Use MCP server to monitor real-time cache efficiency
- Track cache hit rate per session to identify optimization opportunities

#### D. Cost-Effective Development Workflows

**High Cache Efficiency Workflows:**
1. **Iterative Debugging**: Same codebase, multiple queries
2. **Code Review Sessions**: Reviewing multiple files
3. **Documentation Writing**: Stable context, varied output
4. **Refactoring Projects**: Same files, incremental changes

**Low Cache Efficiency Workflows:**
1. **Cross-Project Switching**: Different codebases per query
2. **One-Off Queries**: No context reuse
3. **Exploratory Analysis**: Constantly changing file sets

## Optimization Playbook

### Scenario 1: Cache Hit Rate < 80%

**Diagnosis:**
```promql
# Check cache creation rate
sum(increase(claude_code_token_usage_tokens_total{type="cacheCreation"}[1h]))
/
sum(increase(claude_code_token_usage_tokens_total[1h])) * 100
```

**If cache creation > 15%:**
- Too many new sessions
- Context changing too frequently
- Session timeout issues

**Solutions:**
1. Consolidate tasks into longer sessions
2. Structure prompts with stable prefix
3. Batch related queries

### Scenario 2: Daily Cost Increasing

**Diagnosis:**
```promql
# Cost trend over 7 days
increase(claude_code_cost_usage_USD_total[7d])
```

**If cost increasing but cache hit rate stable:**
- More overall usage (expected)
- Larger context windows
- More complex tasks requiring longer outputs

**If cost increasing and cache hit rate dropping:**
- Inefficient session management
- Cache-breaking workflow changes
- Investigate with session-level metrics

**Solutions:**
1. Review session patterns via Loki logs
2. Analyze prompt structure changes
3. Optimize context window size

### Scenario 3: High Cache Creation Cost

**Diagnosis:**
```promql
sum(increase(claude_code_token_usage_tokens_total{type="cacheCreation"}[24h])) * 0.00375
```

**If cache creation cost > 30% of total:**
- Cache churn (creating but not reusing)
- Sessions too short
- Context too dynamic

**Solutions:**
1. Extend session lifetime
2. Group related queries
3. Identify and stabilize frequently changing context

## Monitoring with MCP Server

Use the Claude Code Usage Metrics MCP server to track optimization in real-time:

```bash
# Monitor cache efficiency during development
claude code "What's my current cache hit rate?"
# Uses: mcp__metrics__get_cache_efficiency

# Check if optimization is working
claude code "Compare my cache efficiency today vs yesterday"
# Uses: mcp__metrics__query_prometheus with custom time ranges

# Identify expensive sessions
claude code "Show me cost by session for the last 24h"
# Uses: mcp__metrics__query_prometheus with session grouping
```

## Best Practices Summary

| Practice | Impact | Difficulty | Priority |
|----------|--------|------------|----------|
| Stable context at prompt start | High | Low | 🔴 Critical |
| Long-running sessions | High | Low | 🔴 Critical |
| Batch similar tasks | Medium | Low | 🟡 High |
| Monitor cache metrics | Medium | Low | 🟡 High |
| Optimize context window | Medium | Medium | 🟡 High |
| Cache-aware workflows | High | Medium | 🟡 High |
| Session management strategy | Medium | High | 🟢 Medium |
| Custom alerting rules | Low | High | 🟢 Medium |

## Recommended PromQL Queries

### 1. Cache Efficiency Trend (Hourly)
```promql
sum(increase(claude_code_token_usage_tokens_total{type="cacheRead"}[1h]))
/
clamp_min(sum(increase(claude_code_token_usage_tokens_total[1h])), 1) * 100
```

### 2. Daily Cost Savings from Cache
```promql
(
  sum(increase(claude_code_token_usage_tokens_total{type="cacheRead"}[24h])) * 0.003
  - sum(increase(claude_code_token_usage_tokens_total{type="cacheRead"}[24h])) * 0.0003
)
```

### 3. Cache ROI (Return on Investment)
```promql
# ROI = Savings / Cache Creation Cost
(sum(increase(claude_code_token_usage_tokens_total{type="cacheRead"}[24h])) * 0.003
 - sum(increase(claude_code_token_usage_tokens_total{type="cacheRead"}[24h])) * 0.0003)
/
(sum(increase(claude_code_token_usage_tokens_total{type="cacheCreation"}[24h])) * 0.00375)
```

### 4. Optimal Session Length Detection
```promql
# Sessions with best cache efficiency
topk(10,
  sum by (session_id) (increase(claude_code_token_usage_tokens_total{type="cacheRead"}[24h]))
  /
  sum by (session_id) (increase(claude_code_token_usage_tokens_total[24h])) * 100
)
```

### 5. Cache Warm-up Period Analysis
```promql
# First hour cache efficiency vs later hours
sum(increase(claude_code_token_usage_tokens_total{type="cacheRead"}[1h] offset 23h))
/
sum(increase(claude_code_token_usage_tokens_total[1h] offset 23h)) * 100
```

## LogQL Queries for Workflow Analysis

### 1. Find Cache-Inefficient Sessions
```logql
# Sessions with low cache reads
{service_name="claude-code", event_name="token_usage"}
| type="cacheRead"
| unwrap tokens
| sum by (session_id) < 1000000
```

### 2. Session Duration Analysis
```logql
# Correlate session length with cache efficiency
{service_name="claude-code", event_name="session"}
| unwrap duration_ms
| quantile by (session_id) 0.95
```

### 3. Prompt Pattern Analysis
```logql
# Find prompts that break cache (requires OTEL_LOG_USER_PROMPTS=1)
{service_name="claude-code", event_name="user_prompt"}
| prompt_length > 100000
```

## Optimization Roadmap

### Week 1: Measurement
- [ ] Set up all recommended Grafana panels
- [ ] Establish baseline metrics (current cache hit rate, daily cost)
- [ ] Configure alerts for cache efficiency < 85%
- [ ] Install and test MCP server for real-time monitoring

### Week 2: Quick Wins
- [ ] Implement stable context ordering in frequent workflows
- [ ] Consolidate short sessions into longer sessions
- [ ] Batch similar operations
- [ ] Review and optimize prompt templates

### Week 3: Advanced Optimization
- [ ] Analyze session-level cache patterns
- [ ] Implement cache-aware development workflows
- [ ] Optimize context window sizing
- [ ] Set up automated cost anomaly detection

### Week 4: Continuous Improvement
- [ ] Weekly cache efficiency reviews
- [ ] Identify and fix cache anti-patterns
- [ ] Document team best practices
- [ ] Measure and report cost savings

## Success Metrics

**Target Performance (within 30 days):**
- Cache hit rate: > 95%
- Daily cost reduction: 15-25%
- Cache ROI: > 10:1
- Average session cache efficiency: > 90%

**Your Current Performance:**
- ✅ Cache hit rate: **93.53%** (near target)
- 🟡 Cache ROI: **16.2:1** (excellent)
- ✅ Token efficiency: **93.5% cached**

**Optimization Potential:**
- Increase cache hit rate from 93.53% to 95%+ could save additional **$50-100/month**
- Extending average session duration by 10 minutes could improve cache reuse by **5-10%**
- Optimizing prompt structure could reduce cache creation costs by **20-30%**

## Additional Resources

- [Claude Code Monitoring Documentation](https://docs.claude.com/en/docs/claude-code/monitoring-usage)
- [Anthropic Prompt Caching Guide](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching)
- [OpenTelemetry Collector Configuration](https://opentelemetry.io/docs/collector/)
- [Prometheus Query Examples](https://prometheus.io/docs/prometheus/latest/querying/examples/)
- [Grafana Dashboard Best Practices](https://grafana.com/docs/grafana/latest/best-practices/)

## Support and Feedback

For questions, issues, or suggestions:
- GitHub Issues: https://github.com/centminmod/claude-code-opentelemetry-setup/issues
- Claude Code Documentation: https://docs.claude.com/claude-code

---

**Last Updated:** 2025-10-25
**Version:** 1.0.0
