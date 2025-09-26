# Prometheus Metrics Demo

A comprehensive guide and demo for understanding and implementing Prometheus metrics in your applications.

## Table of Contents
1. [Introduction to Prometheus](#introduction-to-prometheus)
   - [Key Components](#key-components)
   - [Metric Collection Process](#metric-collection-process)
2. [Metrics](#metrics)
   - [Types of Metrics](#types-of-metrics)
3. [The 4 Golden Signals](#the-4-golden-signals)
4. [Querying and Alerting](#querying-and-alerting)
   - [PromQL Examples](#promql-examples)
   - [Alerting Rules](#alerting-rules)
5. [Best Practices](#best-practices)
   - [Label Cardinality](#label-cardinality)
   - [Scrape Interval](#scrape-interval)
   - [Retention Period](#retention-period)
   - [Recording Rules](#recording-rules)
6. [Demo](/demo/README.md)

## Introduction to Prometheus
Prometheus is an open-source systems monitoring and alerting toolkit originally built at SoundCloud. Since its inception in 2012, many companies and organizations have adopted Prometheus, and the project has a very active developer and user community. It is now a standalone open source project and maintained by the Cloud Native Computing Foundation.

### Key Components
1. **Prometheus Server**: The main component that collects and stores metrics
2. **Exporters**: Programs that expose metrics in a format that Prometheus can scrape
3. **Service Discovery**: Automatically finds targets to scrape
4. **Pushgateway**: For pushing metrics (use sparingly)
5. **Alertmanager**: For handling alerts

### Metric Collection Process
1. Exporters expose metrics on HTTP endpoints
2. Prometheus scrapes these endpoints at regular intervals
3. Metrics are stored in Prometheus's time-series database
4. Data can be queried using PromQL (Prometheus Query Language)

## Metrics
Metrics are numerical measurements collected over time to monitor the health, performance, and behavior of systems and applications

### Types of metrics
Prometheus supports four main types of metrics, each designed for specific use cases. Understanding the differences is crucial for effective monitoring.

1. **Counter**
   - Represents a single numerical value that only ever goes up
   - Example: `http_requests_total`
   - Use for counting events
   - Practical example:
     ```promql
     # Total HTTP requests in the last 5 minutes
     increase(http_requests_total[5m])
     ```
   - Key characteristics:
     - Monotonic (only increases)
     - Resets on restart
     - Use `increase()` for rate calculations

2. **Gauge**
   - Represents a single numerical value that can go up and down
   - Example: `memory_usage_bytes`
   - Use for measuring current state
   - Practical example:
     ```promql
     # Memory usage percentage
     (memory_usage_bytes / memory_total_bytes) * 100
     ```
   - Key characteristics:
     - Can increase or decrease
     - Maintains current value
     - Use `rate()` for trend analysis

3. **Histogram**
   - Tracks how often values fall into specific ranges (buckets)
   - Good for visualizing data distribution
   - Example: `http_request_duration_seconds_bucket{le="0.1"}`
   - Use for measuring request durations or response sizes
   - Practical example:
     ```promql
     # 95th percentile request latency
     histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))
     ```
   - Practical use: Monitoring API response times across different buckets
     - 0-100ms bucket: Fast responses
     - 100-500ms bucket: Normal responses
     - 500ms+ bucket: Slow responses

4. **Summary**
   - Calculates percentiles over a time window
   - More memory-efficient than histograms
   - Example: `http_request_duration_seconds_summary{quantile="0.95"}`
   - Use for calculating exact quantiles and tracking service level objectives (SLOs)
   - Practical example:
     ```promql
     # 95th percentile response time
     http_request_duration_seconds_summary{quantile="0.95"}
     ```   
   - Practical use: Ensuring 95% of requests complete within 200ms
     - p95: 95% of requests complete within this time
     - p99: 99% of requests complete within this time

### Key differences between Histogram and Summary

| Aspect | Summary | Histogram |
|--------|---------|-----------|
| **Calculation Time** | Calculated when the metric is collected | Calculated when the query is run |
| **Flexibility** | Fixed quantiles (usually 0.5, 0.9, 0.95, 0.99) | Can calculate any quantile at query time |
| **Memory Usage** | More memory-efficient (stores pre-calculated quantiles) | Uses more memory (stores raw data in buckets) |
| **Accuracy** | Less accurate (quantiles are pre-calculated) | More accurate (can calculate exact quantiles) |

5. **Best Practices**
   - Use Histograms when:
     - You need detailed latency distributions
     - You want to calculate different quantiles
     - You have sufficient memory
   - Use Summaries when:
     - You need to track SLOs
     - Memory is constrained
     - You only need specific quantiles

## The 4 Golden Signals
The 4 Golden Signals are the most important metrics to monitor for any service. They provide a comprehensive view of service health and should be the foundation of your monitoring strategy.

1. **Latency**
   - Measures how long it takes to service requests
   - Critical for understanding user experience
   - Metrics: `http_request_duration_seconds`
   - Example Alert - using Histogram:
   ```promql
   # Alert if 95th percentile latency is above 1 second
   histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le)) > 1
   ```
   - Example Alert - using Summary:
   ```promql
   # Alert if 95th percentile latency is above 1 second
   http_request_duration_seconds_summary{quantile="0.95"} > 1
   ```

2. **Traffic**
   - Measures the demand on your service
   - Helps identify sudden spikes or drops
   - Metrics: `http_requests_total`
   - Example Alert:
   ```promql
   rate(http_requests_total[5m]) > 10000
   ```

3. **Errors**
   - Measures the rate of requests that fail
   - Essential for detecting service issues
   - Metrics: `http_requests_total{status=~"5.."}`
   - Example Alert:
   ```promql
   rate(http_requests_total{status=~"5.."}[5m]) > 0.01
   ```

4. **Saturation**
   - Measures how "full" your service is
   - Helps identify resource constraints
   - Metrics: `cpu_usage_ratio`, `memory_usage_bytes`
   - Example Alert:
   ```promql
   process_resident_memory_bytes > 0.8 * node_memory_MemTotal_bytes
   ```


## Querying and Alerting

### PromQL Examples
1. Basic Query
```promql
http_requests_total
```

2. Rate Calculation
```promql
rate(http_requests_total[5m])
```

3. Aggregation
```promql
sum(http_requests_total) by (status)
```

4. Complex Query
```promql
sum by (job) (rate(http_requests_total[5m])) > 100
```

### Alerting Rules
Example alert configuration:
```yaml
alert: HighRequestLatency
expr: histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le)) > 1
for: 5m
labels:
  severity: critical
annotations:
  summary: "High request latency detected"
  description: "95th percentile request latency is above 1 second"
```




## Best Practices

### Label Cardinality
**Problem**: Too many unique label combinations can cause:
   - Memory Usage: Each unique label combination creates a new time series
   - Query Performance: High cardinality metrics make queries slower
   - Storage: More time series require more storage
   - Alerting: Alerts can become less effective

**Solutions**:
   - Use aggregation:
     
     ```promql
     # Bad: Uses high cardinality labels (User IDs, Session IDs, Request IDs). 
     request_duration{user_id="123"}
     
     # Good: Uses aggregated metrics
     request_duration{user_type="premium"}
     
     # Aggregate by user type instead of individual users
     sum by (user_type) (request_duration)
     ```
   - Limit label values:
     - Use enums instead of free-form strings
   - Use relabeling:
     ```yaml
     relabel_configs:
       # Drop high cardinality labels during scraping
       - source_labels: [user_id]
         target_label: __tmp_user_id
         action: labeldrop
       # Group by environment instead of individual hosts
       - source_labels: [instance]
         target_label: environment
         regex: "^(.*?)-(.*?)-(.*?)"
         replacement: "$1-$2"
     ```

### Scrape Interval
- **Problem**: Too frequent scraping can overload targets
- **Solution**: Adjust scrape interval based on metric volatility

### Retention Period
- **Problem**: Running out of disk space
- **Solution**: Configure appropriate retention period based on needs

### Recording Rules
- **Problem**: Slow queries or high load on Prometheus
- **Solution**: Use recording rules to pre-compute complex queries
    - Benefits:
      - Pre-compute complex queries to optimize performance
      - Reduce load on the Prometheus server
      - Make queries more efficient and faster
    - Considerations:
      - Only use for frequently used, expensive queries
      - Be mindful of storage impact
      - Use appropriate retention periods for pre-computed metrics
    - Example:
      ```yaml
      groups:
      - name: example
        rules:
        - record: job:http_inprogress_requests:sum
          expr: sum(http_inprogress_requests) by (job)
        - record: job:http_request_duration_seconds:avg
          expr: sum(rate(http_request_duration_seconds_sum[5m])) by (job)
            / sum(rate(http_request_duration_seconds_count[5m])) by (job)
      ```

