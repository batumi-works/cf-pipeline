# Observability and Monitoring Guide

This document outlines the observability integration for the Cloudflare Workers CI/CD platform, including Datadog integration, metrics collection, and monitoring best practices.

## Datadog Integration

### Setup

1. **Datadog Account Configuration**:
```bash
# Set environment variables
export DATADOG_API_KEY="your-api-key"
export DATADOG_APP_KEY="your-app-key"
export DATADOG_SITE="datadoghq.com"  # or datadoghq.eu for EU
```

2. **GitHub Secrets Configuration**:
```bash
# Add to repository secrets
DATADOG_API_KEY=<your-datadog-api-key>
DATADOG_APP_KEY=<your-datadog-application-key>
```

### Deployment Event Tracking

Events are automatically sent to Datadog during CI/CD pipeline execution:

```javascript
// Example deployment event payload
{
  "title": "Deployment Success: production",
  "text": "Worker deployed successfully to production environment",
  "tags": [
    "source:cf-pipeline",
    "env:production",
    "service:my-worker",
    "version:1.2.3",
    "status:success",
    "deployment:cloudflare-workers"
  ],
  "alert_type": "info",
  "source_type_name": "github",
  "date_happened": 1699123456,
  "aggregation_key": "deployment-my-worker-production"
}
```

### Custom Metrics Collection

```javascript
// In your Worker code - send custom metrics
async function sendMetric(metricName, value, tags = []) {
  const apiKey = env.DATADOG_API_KEY;
  if (!apiKey) return;

  const metric = {
    series: [{
      metric: metricName,
      points: [[Date.now() / 1000, value]],
      tags: [
        'source:cloudflare-worker',
        'env:' + (env.ENVIRONMENT || 'unknown'),
        ...tags
      ]
    }]
  };

  try {
    await fetch('https://api.datadoghq.com/api/v1/series', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'DD-API-KEY': apiKey
      },
      body: JSON.stringify(metric)
    });
  } catch (error) {
    console.error('Failed to send metric:', error);
  }
}

// Usage in Worker
export default {
  async fetch(request, env, ctx) {
    const startTime = Date.now();
    
    try {
      const response = await handleRequest(request, env, ctx);
      
      // Track response time
      const duration = Date.now() - startTime;
      ctx.waitUntil(sendMetric('worker.response_time', duration, [
        'method:' + request.method,
        'status:' + response.status
      ]));
      
      // Track request count
      ctx.waitUntil(sendMetric('worker.request_count', 1, [
        'method:' + request.method,
        'status:' + response.status
      ]));
      
      return response;
    } catch (error) {
      // Track errors
      ctx.waitUntil(sendMetric('worker.error_count', 1, [
        'error_type:' + error.name
      ]));
      
      throw error;
    }
  }
};
```

### Datadog Dashboard Configuration

```json
{
  "title": "Cloudflare Workers - CI/CD Pipeline",
  "description": "Monitoring dashboard for CF Workers deployments",
  "widgets": [
    {
      "definition": {
        "title": "Deployment Events",
        "type": "event_stream",
        "query": "sources:cf-pipeline",
        "event_size": "s"
      }
    },
    {
      "definition": {
        "title": "Deployment Success Rate",
        "type": "query_value",
        "requests": [{
          "q": "sum:deployment.success{service:*}.as_rate()",
          "aggregator": "avg"
        }],
        "precision": 2
      }
    },
    {
      "definition": {
        "title": "CI Pipeline Duration",
        "type": "timeseries",
        "requests": [{
          "q": "avg:github.actions.workflow_run.duration{workflow_name:ci}",
          "display_type": "line"
        }]
      }
    },
    {
      "definition": {
        "title": "Worker Response Time",
        "type": "timeseries",
        "requests": [{
          "q": "avg:worker.response_time{*} by {env}",
          "display_type": "line"
        }]
      }
    }
  ]
}
```

## GitHub Actions Metrics

### Workflow Performance Tracking

```yaml
# In deploy.yml workflow
- name: Send workflow metrics
  if: always()
  run: |
    WORKFLOW_DURATION=$(($(date +%s) - ${{ github.event.workflow_run.created_at }}))
    STATUS="${{ job.status }}"
    
    curl -X POST "https://api.datadoghq.com/api/v1/series" \
      -H "Content-Type: application/json" \
      -H "DD-API-KEY: ${{ secrets.DATADOG_API_KEY }}" \
      -d '{
        "series": [{
          "metric": "github.workflow.duration",
          "points": [['$(date +%s)', '${WORKFLOW_DURATION}']],
          "tags": [
            "workflow:deploy",
            "status:'${STATUS}'",
            "repo:${{ github.repository }}",
            "env:${{ inputs.environment }}"
          ]
        }]
      }'
```

### CI/CD Pipeline Monitoring

```yaml
# Custom action for pipeline metrics
- name: Track pipeline stage
  uses: ./.github/actions/track-metric
  with:
    metric: 'pipeline.stage.duration'
    value: '${{ steps.stage-timer.outputs.duration }}'
    tags: |
      stage:test
      status:${{ steps.test.outcome }}
      environment:${{ inputs.environment }}
```

## Worker Runtime Observability

### Performance Monitoring

```javascript
// Worker performance monitoring
class PerformanceTracker {
  constructor(env) {
    this.env = env;
    this.startTime = Date.now();
    this.metrics = new Map();
  }
  
  startTimer(name) {
    this.metrics.set(name + '_start', Date.now());
  }
  
  endTimer(name) {
    const start = this.metrics.get(name + '_start');
    if (start) {
      const duration = Date.now() - start;
      this.metrics.set(name + '_duration', duration);
      return duration;
    }
    return null;
  }
  
  async sendMetrics(ctx) {
    const metrics = [];
    
    // Convert collected metrics to Datadog format
    this.metrics.forEach((value, key) => {
      if (key.endsWith('_duration')) {
        metrics.push({
          metric: 'worker.performance.' + key.replace('_duration', ''),
          points: [[Date.now() / 1000, value]],
          tags: [
            'env:' + (this.env.ENVIRONMENT || 'unknown'),
            'worker:' + (this.env.WORKER_NAME || 'unknown')
          ]
        });
      }
    });
    
    if (metrics.length > 0) {
      ctx.waitUntil(this.sendToDatadog(metrics));
    }
  }
  
  async sendToDatadog(metrics) {
    if (!this.env.DATADOG_API_KEY) return;
    
    try {
      await fetch('https://api.datadoghq.com/api/v1/series', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'DD-API-KEY': this.env.DATADOG_API_KEY
        },
        body: JSON.stringify({ series: metrics })
      });
    } catch (error) {
      console.error('Failed to send metrics to Datadog:', error);
    }
  }
}

// Usage
export default {
  async fetch(request, env, ctx) {
    const tracker = new PerformanceTracker(env);
    
    try {
      tracker.startTimer('total_request');
      
      // Database operation
      tracker.startTimer('database_query');
      const data = await fetchFromDatabase();
      tracker.endTimer('database_query');
      
      // External API call
      tracker.startTimer('external_api');
      const apiResponse = await callExternalAPI();
      tracker.endTimer('external_api');
      
      // Response generation
      tracker.startTimer('response_generation');
      const response = generateResponse(data, apiResponse);
      tracker.endTimer('response_generation');
      
      tracker.endTimer('total_request');
      
      // Send metrics asynchronously
      await tracker.sendMetrics(ctx);
      
      return response;
    } catch (error) {
      await tracker.sendMetrics(ctx);
      throw error;
    }
  }
};
```

### Error Tracking and Alerting

```javascript
// Error tracking with context
async function trackError(error, context, env, ctx) {
  const errorData = {
    message: error.message,
    stack: error.stack,
    context: {
      url: context.url,
      method: context.method,
      userAgent: context.userAgent,
      timestamp: new Date().toISOString(),
      environment: env.ENVIRONMENT || 'unknown',
      worker: env.WORKER_NAME || 'unknown'
    }
  };
  
  // Send to Datadog as log
  const logPayload = {
    ddsource: 'cloudflare-worker',
    ddtags: `env:${env.ENVIRONMENT},worker:${env.WORKER_NAME}`,
    hostname: 'cloudflare-worker',
    message: JSON.stringify(errorData),
    level: 'ERROR',
    timestamp: Date.now()
  };
  
  ctx.waitUntil(fetch('https://http-intake.logs.datadoghq.com/v1/input/' + env.DATADOG_API_KEY, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(logPayload)
  }));
  
  // Send error metric
  ctx.waitUntil(sendMetric('worker.error_count', 1, [
    'error_type:' + error.name,
    'endpoint:' + new URL(context.url).pathname
  ]));
}
```

## Alerting Configuration

### Datadog Monitors

```json
{
  "name": "CF Worker - High Error Rate",
  "type": "metric alert",
  "query": "avg(last_5m):avg:worker.error_count{*}.as_rate() > 0.05",
  "message": "Error rate for Cloudflare Worker is above 5%\n@slack-alerts",
  "tags": ["service:cloudflare-worker", "team:platform"],
  "options": {
    "thresholds": {
      "critical": 0.05,
      "warning": 0.02
    },
    "notify_audit": false,
    "require_full_window": false,
    "notify_no_data": true,
    "no_data_timeframe": 10
  }
}
```

### Deployment Failure Alerts

```json
{
  "name": "CF Pipeline - Deployment Failure",
  "type": "event alert",
  "query": "events('sources:cf-pipeline status:failure')",
  "message": "Cloudflare Worker deployment failed\n@pagerduty-platform",
  "tags": ["service:cf-pipeline", "severity:high"],
  "options": {
    "notify_audit": true,
    "include_tags": true,
    "aggregation": {
      "group_by": ["service", "env"],
      "metric": "count",
      "time": "5m"
    }
  }
}
```

### Performance Degradation Alerts

```json
{
  "name": "CF Worker - High Response Time",
  "type": "metric alert",
  "query": "avg(last_10m):avg:worker.response_time{*} > 1000",
  "message": "Worker response time is above 1 second\n@team-platform",
  "tags": ["service:cloudflare-worker", "severity:medium"],
  "options": {
    "thresholds": {
      "critical": 1000,
      "warning": 500
    },
    "evaluation_delay": 60
  }
}
```

## Log Management

### Structured Logging

```javascript
// Structured logging utility
class Logger {
  constructor(env) {
    this.env = env;
    this.context = {
      environment: env.ENVIRONMENT || 'unknown',
      worker: env.WORKER_NAME || 'unknown',
      version: env.WORKER_VERSION || '1.0.0'
    };
  }
  
  log(level, message, data = {}) {
    const logEntry = {
      timestamp: new Date().toISOString(),
      level: level.toUpperCase(),
      message,
      ...this.context,
      ...data
    };
    
    console.log(JSON.stringify(logEntry));
    
    // Send to Datadog if API key available
    if (this.env.DATADOG_API_KEY) {
      this.sendToDatadog(logEntry);
    }
  }
  
  info(message, data) { this.log('info', message, data); }
  warn(message, data) { this.log('warn', message, data); }
  error(message, data) { this.log('error', message, data); }
  debug(message, data) { this.log('debug', message, data); }
  
  async sendToDatadog(logEntry) {
    try {
      await fetch('https://http-intake.logs.datadoghq.com/v1/input/' + this.env.DATADOG_API_KEY, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          ddsource: 'cloudflare-worker',
          ddtags: `env:${this.context.environment},worker:${this.context.worker}`,
          hostname: 'cloudflare-worker',
          ...logEntry
        })
      });
    } catch (error) {
      console.error('Failed to send log to Datadog:', error);
    }
  }
}

// Usage in Worker
export default {
  async fetch(request, env, ctx) {
    const logger = new Logger(env);
    const requestId = crypto.randomUUID();
    
    logger.info('Request started', {
      requestId,
      method: request.method,
      url: request.url,
      userAgent: request.headers.get('user-agent')
    });
    
    try {
      const response = await handleRequest(request, env, ctx);
      
      logger.info('Request completed', {
        requestId,
        status: response.status,
        duration: Date.now() - startTime
      });
      
      return response;
    } catch (error) {
      logger.error('Request failed', {
        requestId,
        error: error.message,
        stack: error.stack
      });
      
      throw error;
    }
  }
};
```

## Cost Optimization

### Metrics Sampling

```javascript
// Sample metrics to reduce costs
function shouldSample(sampleRate = 0.1) {
  return Math.random() < sampleRate;
}

// Usage
if (shouldSample(0.05)) { // 5% sampling
  await sendMetric('worker.detailed_metric', value);
}

// Always send critical metrics
await sendMetric('worker.error_count', 1); // No sampling for errors
```

### Efficient Batch Reporting

```javascript
// Batch metrics to reduce API calls
class MetricsBatcher {
  constructor(env, batchSize = 100, flushInterval = 30000) {
    this.env = env;
    this.batchSize = batchSize;
    this.flushInterval = flushInterval;
    this.metrics = [];
    this.lastFlush = Date.now();
  }
  
  add(metric, value, tags = []) {
    this.metrics.push({
      metric,
      points: [[Date.now() / 1000, value]],
      tags: ['source:worker', ...tags]
    });
    
    if (this.shouldFlush()) {
      this.flush();
    }
  }
  
  shouldFlush() {
    return this.metrics.length >= this.batchSize || 
           (Date.now() - this.lastFlush) > this.flushInterval;
  }
  
  async flush() {
    if (this.metrics.length === 0) return;
    
    const batch = this.metrics.splice(0);
    this.lastFlush = Date.now();
    
    try {
      await fetch('https://api.datadoghq.com/api/v1/series', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'DD-API-KEY': this.env.DATADOG_API_KEY
        },
        body: JSON.stringify({ series: batch })
      });
    } catch (error) {
      console.error('Failed to flush metrics batch:', error);
    }
  }
}
```

## Debugging and Troubleshooting

### Debug Mode

```javascript
// Debug mode with detailed logging
export default {
  async fetch(request, env, ctx) {
    const debugMode = env.DEBUG_MODE === 'true' || 
                     request.headers.get('x-debug') === 'true';
    
    if (debugMode) {
      console.log('Debug: Request details', {
        method: request.method,
        url: request.url,
        headers: Object.fromEntries(request.headers.entries()),
        cf: request.cf
      });
    }
    
    // ... rest of handler
  }
};
```

### Performance Profiling

```javascript
// Simple profiler for debugging
class Profiler {
  constructor() {
    this.timers = new Map();
  }
  
  start(name) {
    this.timers.set(name, Date.now());
  }
  
  end(name) {
    const start = this.timers.get(name);
    if (start) {
      const duration = Date.now() - start;
      console.log(`Profile: ${name} took ${duration}ms`);
      return duration;
    }
  }
  
  getReport() {
    const report = {};
    this.timers.forEach((start, name) => {
      report[name] = Date.now() - start;
    });
    return report;
  }
}
```

## Resources

- [Datadog API Documentation](https://docs.datadoghq.com/api/)
- [Cloudflare Workers Analytics](https://developers.cloudflare.com/workers/observability/analytics/)
- [GitHub Actions Monitoring](https://docs.github.com/en/actions/monitoring-and-troubleshooting-workflows)
- [Datadog Dashboard JSON Schema](https://docs.datadoghq.com/dashboards/graphing_json/)

---

**Note**: Ensure you configure appropriate retention policies and cost controls for metrics and logs to avoid unexpected charges.