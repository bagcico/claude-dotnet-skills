## ELK Stack Configuration

### Docker Compose (Production)

```yaml
services:
  elasticsearch:
    image: elasticsearch:9.2.3
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=true
      - ELASTIC_PASSWORD=${ELASTIC_PASSWORD}
      - "ES_JAVA_OPTS=-Xms2g -Xmx2g"
    volumes:
      - es_data:/usr/share/elasticsearch/data
    ports:
      - "127.0.0.1:9200:9200"
    restart: unless-stopped

  kibana:
    image: kibana:9.2.3
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
      - ELASTICSEARCH_USERNAME=kibana_system
      - ELASTICSEARCH_PASSWORD=${KIBANA_PASSWORD}
    ports:
      - "127.0.0.1:5601:5601"
    depends_on:
      - elasticsearch
    restart: unless-stopped

  grafana:
    image: grafana/grafana:12.3.1
    volumes:
      - grafana_data:/var/lib/grafana
    ports:
      - "127.0.0.1:3000:3000"
    restart: unless-stopped

volumes:
  es_data:
  grafana_data:
```

### Elasticsearch Index Template

Create an index template for better field mappings:

```json
{
  "index_patterns": ["app-*"],
  "template": {
    "settings": {
      "number_of_shards": 2,
      "number_of_replicas": 1,
      "index.lifecycle.name": "app-logs-policy"
    },
    "mappings": {
      "properties": {
        "@timestamp": { "type": "date" },
        "level": { "type": "keyword" },
        "message": { "type": "text" },
        "CorrelationId": { "type": "keyword" },
        "UserId": { "type": "keyword" },
        "RequestMethod": { "type": "keyword" },
        "RequestPath": { "type": "keyword" },
        "StatusCode": { "type": "integer" },
        "Elapsed": { "type": "float" },
        "Module": { "type": "keyword" },
        "MediatRRequest": { "type": "keyword" },
        "Application": { "type": "keyword" },
        "Environment": { "type": "keyword" },
        "MachineName": { "type": "keyword" },
        "EcsTaskId": { "type": "keyword" },
        "exception.type": { "type": "keyword" },
        "exception.message": { "type": "text" },
        "exception.stacktrace": { "type": "text" }
      }
    }
  }
}
```

### Index Lifecycle Management (ILM)

Automatically manage log retention:

```json
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "rollover": {
            "max_age": "7d",
            "max_size": "50gb"
          }
        }
      },
      "warm": {
        "min_age": "7d",
        "actions": {
          "shrink": { "number_of_shards": 1 },
          "forcemerge": { "max_num_segments": 1 }
        }
      },
      "delete": {
        "min_age": "30d",
        "actions": { "delete": {} }
      }
    }
  }
}
```

---

## Performance Logging

### Operation Timer Utility

```csharp
public static class LogTimer
{
    public static IDisposable Time(ILogger logger, string operationName,
        params (string Key, object Value)[] properties)
    {
        return new TimerScope(logger, operationName, properties);
    }

    private sealed class TimerScope : IDisposable
    {
        private readonly ILogger _logger;
        private readonly string _operationName;
        private readonly Stopwatch _stopwatch;
        private readonly IDisposable? _logScope;

        public TimerScope(ILogger logger, string operationName,
            (string Key, object Value)[] properties)
        {
            _logger = logger;
            _operationName = operationName;
            _stopwatch = Stopwatch.StartNew();

            var enrichers = properties
                .Select(p => new PropertyEnricher(p.Key, p.Value))
                .Cast<ILogEventEnricher>()
                .ToArray();

            _logScope = enrichers.Length > 0
                ? LogContext.Push(enrichers)
                : null;
        }

        public void Dispose()
        {
            _stopwatch.Stop();
            _logger.LogInformation(
                "Operation {OperationName} completed in {ElapsedMs}ms",
                _operationName, _stopwatch.ElapsedMilliseconds);
            _logScope?.Dispose();
        }
    }
}

// Usage
using (LogTimer.Time(logger, "BulkNotificationSend",
    ("BatchSize", notifications.Count),
    ("Channel", "push")))
{
    await notificationService.SendBulkAsync(notifications, cancellationToken);
}
// Output: Operation BulkNotificationSend completed in 342ms
//         Properties: { BatchSize: 150, Channel: "push" }
```

---

## Health Check Logging

Log health check results for monitoring:

```csharp
public sealed class HealthCheckLoggingPublisher(
    ILogger<HealthCheckLoggingPublisher> logger
) : IHealthCheckPublisher
{
    public Task PublishAsync(HealthReport report, CancellationToken cancellationToken)
    {
        var level = report.Status switch
        {
            HealthStatus.Healthy => LogEventLevel.Information,
            HealthStatus.Degraded => LogEventLevel.Warning,
            HealthStatus.Unhealthy => LogEventLevel.Error,
            _ => LogEventLevel.Warning
        };

        foreach (var entry in report.Entries)
        {
            logger.Write(level,
                "Health check {HealthCheckName}: {Status} ({Duration}ms) - {Description}",
                entry.Key,
                entry.Value.Status,
                entry.Value.Duration.TotalMilliseconds,
                entry.Value.Description);
        }

        return Task.CompletedTask;
    }
}

// Registration
builder.Services.AddSingleton<IHealthCheckPublisher, HealthCheckLoggingPublisher>();
builder.Services.Configure<HealthCheckPublisherOptions>(options =>
{
    options.Period = TimeSpan.FromSeconds(30);
});
```
