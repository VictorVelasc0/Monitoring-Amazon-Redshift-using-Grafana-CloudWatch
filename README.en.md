# Monitoring Amazon Redshift using Grafana & CloudWatch

> 🇪🇸 [Leer este artículo en español](README.md)

If you ever worked with a database or system, you have surely needed to know how it is behaving. Just like with a car, understanding the parameters that affect its behavior; such as performance, speed, or range, helps us know if something is wrong or if we have the resources to get from point A to point B.

I have gathered some monitoring ideas using tools like Amazon Managed Grafana and CloudWatch with Slack alerts, hoping this can contribute with ideas on how to monitor a data warehouse:

In this article we will cover:
- [A data warehouse without monitoring](#a-data-warehouse-without-monitoring)
- [Monitoring and observability](#monitoring-and-observability)
- [Monitoring tools in AWS](#monitoring-tools-in-aws)
- [CloudWatch with Grafana: where monitoring shines](#cloudwatch-with-grafana-where-monitoring-shines)
- [Data warehouse monitoring architecture with Grafana](#data-warehouse-monitoring-architecture-with-grafana)
- [Cluster CPU monitoring](#cluster-cpu-monitoring)
- [CPU alerts using Slack](#cpu-alerts-using-slack)
- [Conclusions](#conclusions)

# A data warehouse without monitoring
The first signs were almost imperceptible: a load that took a little longer than usual, a dbt DAG that stretched a few extra minutes. But as the business grew, those minutes turned into hours, and the hours into a problem that could no longer be ignored. Loads to the data warehouse began to spike, queries were left suspended indefinitely, and DAG execution times kept increasing. The requirement was clear: optimization was needed.

<p align="center">
  <img src="resources/bart.png" alt="Meme blind bart" width="450" />
</p>

Researching AWS documentation, I found some techniques that promised relief "sort keys, distribution styles, compression" and dbt documentation offered its own strategies through different materializations. However, one question persisted: do we really know where the problem is? Which processes are the most inefficient? At what times do they occur? Who is running them?
Diagnosing a sick cluster without metrics is like trying to operate on a patient blindfolded: you may have the best technique in the world, but if you don't know where it hurts, any intervention is a shot in the dark. It was clear that we had no visibility into the real ailments of the cluster, and applying blind optimizations would only lead to low-impact solutions, impossible to measure and even harder to justify.

# Monitoring and observability
According to AWS, monitoring is defined as "the process of collecting data and generating reports on different metrics that define the state of the system" (Amazon Web Services, n.d.). In practical terms, this means having a mechanism for obtaining and reporting key elements, something that in the world of modern microservices is essential: when an application is composed of multiple interdependent services, measuring the behavior of each one is the first step to understanding the health of the system as a whole.

Observability, on the other hand, goes a step further. While monitoring tells us what is happening, observability allows us to understand the reasons why, it is an investigative approach that uses the collected data to find the root cause of problems (Amazon Web Services, n.d.). In microservices architectures, this is especially relevant, as a failure can originate in one service and manifest in a completely different one.

<p align="center">
  <img src="resources/observability_vs_monitoring.png" alt="observability vs monitoring image"  width="750" />
</p>

Bringing these concepts to our case, monitoring should capture the key metrics that affect the performance of the data warehouse, query execution times, memory usage, connection concurrency, while observability would give us the analytical framework to interpret that data and select the most appropriate optimization techniques for each situation.

# Monitoring tools in AWS
AWS offers a robust ecosystem of monitoring tools, both native and managed, that provide visibility into the state of services at different levels. Here are the most relevant for a data architecture:

## CloudWatch
This is AWS's central monitoring and observability service. It collects metrics, logs, and events from account resources in real time and allows you to set up alarms that can trigger automatic actions via EventBridge, Lambda functions, or SNS notifications (Amazon Web Services, n.d.-a; Bhatt, 2023).
## X-Ray
AWS's distributed tracing service. It collects data from the services that make up an application and generates dependency maps that help identify bottlenecks, errors, and exceptions across microservices, making it especially valuable for diagnosing problems that span multiple components (Amazon Web Services, n.d.-b; Bhatt, 2023).
## CloudTrail
Provides a unified view of governance, auditing, and compliance for all events and API calls made within an AWS account. It allows you to track which resources were created or deleted, who did it, and when. All records are immutable, ensuring the integrity of the activity history.
## DataDog
Although not an AWS service, DataDog is one of the most widely adopted SaaS observability and security platforms in the industry. It allows you to build interactive dashboards that consolidate metrics, logs, and traces from all microservices and systems in an organization in one place (Datadog, n.d.).

## Amazon Managed Grafana
This is AWS's fully managed service based on open-source Grafana. It allows you to query, correlate, and visualize metrics, logs, and traces from multiple data sources, including CloudWatch, X-Ray, and third-party services, without having to manage the underlying infrastructure. It supports collaborative dashboard creation, alerts, and team-level access control (Amazon Web Services, n.d.-c).

# CloudWatch with Grafana: where monitoring shines
Each of the above tools solves a specific visibility problem, but in practice, a recurring challenge arises: the data is scattered. When starting to monitor the data warehouse, one of the first obstacles was precisely that: I needed to bring together in one place the status metrics of the Redshift cluster CPU, latency, active connections with internal data such as running queries and custom performance alerts.

<p align="center">
  <img src="resources/meme.png" alt="meme handshake cloudwatch and grafana" width="450"  />
</p>

The integration of Amazon Managed Grafana with CloudWatch as a data source, together with the Redshift connection as a data source, provide a solution with elegance. By connecting both services, it was possible to unify in a single dashboard both the metrics exposed by AWS and the Redshift system tables, and also add a Slack integration to receive real-time notifications of any performance anomaly. In the following sections, we will see exactly how this architecture was built.

# Data warehouse monitoring architecture with Grafana

The architecture relies on three essential elements working together:
**Amazon Managed Grafana** as the central tool for visualization and alerts,
**Amazon CloudWatch** as the source of cluster metrics, and the **native Redshift plugin for Grafana** to directly query the data warehouse system tables.

<p align="center">
  <img src="resources/architecture_monitoring.png" alt="architecture diagram" width="750"  />
</p>

## 1. Configuring data sources in Grafana

The first step is to connect Grafana with its two sources of information. The first is **CloudWatch**, which exposes infrastructure metrics from the Redshift cluster, such as `CPUUtilization`, `ReadThroughput`, or `DatabaseConnections` and is added directly as a native data source within Grafana (Grafana Labs, n.d.-a; Amazon Web Services, n.d.-d).

The second source is **Redshift** itself, connected via the official `grafana-redshift-datasource` plugin, which allows you to run SQL queries from Grafana directly against the cluster's system tables. For this connection to work, you must first define an *access policy* in AWS with the appropriate permissions authorizing Grafana to connect to the cluster (Grafana Labs, n.d.-b).

## 2. Thematic dashboards

With both sources connected, the next step is to centralize the data in specialized dashboards by topic. Each dashboard combines CloudWatch metrics and queries to Redshift system tables to provide a complete view of a specific aspect of the cluster. In my implementation, the following dashboards were defined:

- **CPU Usage**: tracking CPU usage per node over time.
- **Query Monitoring**: active queries, execution times, and wait queues.
- **Cluster Healthcheck**: overall cluster status, connections, and throughput.

<p align="center">
  <img src="resources/health.png" alt="health monitoring" width="750"  />
</p>

## 3. Alerts with Alertmanager and Slack notifications

The last component of the architecture is the alert system. Grafana includes **Alertmanager**, which allows you to define alert rules based on any metric or query available in the configured data sources.

To connect alerts with Slack, you must first create a Grafana application within the Slack workspace and obtain the corresponding webhook. With this, you add a new Slack type *contact point* in Grafana (Grafana Labs, n.d.-c). Then, you define a *notification policy* that filters alerts using the `dwh` label, ensuring that only data warehouse alerts reach that channel.

<p align="center">
  <img src="resources/contact.png" alt="contact point illustration" width="750" />
</p>

As a concrete example, an alert was configured to trigger when the leader node's CPU usage exceeds **80% for more than 30 minutes**. The key evaluation parameters are:

- **Evaluation interval**: every 30 minutes.
- **Confirmation period**: 10 minutes (the condition must be reached for that time before the alert is triggered).
- **Data source**: CloudWatch with the cluster's `CPUUtilization` metric.

To customize the format of messages sent to Slack, Grafana allows you to configure templates in the *Optional Slack settings* of the contact point. The title and body of the message are defined using Go template syntax, allowing you to include dynamic icons according to the alert severity (`Critical`, `Warning`, `Info`), the alert name, its current state, a description, additional details, and a direct link to the corresponding dashboard in Grafana. These description and detail values are completed in the **"Add details for your alert rule"** section when defining each rule.

<p align="center">
  <img src="resources/alertconf.png" alt="alert configuration" width="750" />
</p>

## Cluster CPU monitoring

The CPU dashboard is probably the most critical of the entire architecture; it is the first to check when an alert arrives and the one that allows the fastest action. It consists of three complementary visual elements.

The first is a **timeline graph** showing the `CPUUtilization` metric from CloudWatch broken down by cluster node, allowing you to identify not only if the CPU is high, but *on which node* the pressure is concentrated. The second is **gauges** showing the most recent CPU status per node, useful for an instant reading of the current moment without having to interpret the historical graph.

<p align="center">
  <img src="resources/cpu.png" alt="Cpu usage dashboard" width="750" />
</p>

The third element is a **real-time active queries table**, built on the Redshift system table `STV_RECENTS`, which records the queries running in the cluster at that moment (Amazon Web Services, n.d.-e). The query used is:

```sql
SELECT
    pid,
    db_name                          AS "database",
    status,
    duration / 60000000.0            AS "query_duration_minutes",
    user_name,
    query,
    starttime
FROM stv_recents
WHERE status = 'Running';
```

This table is the piece that turns the CPU dashboard into an actionable diagnostic tool: by crossing the CPU status per node with the queries running at that moment, it is possible to identify which processes are running out resources, which users or applications are generating them, and how long they have been running. With that information, the corrective action can be as immediate as canceling a heavy query that has been saturating the leader node for hours, or as strategic as prioritizing that process for an optimization round.

### Worst CPU Queries

Beyond the real-time state, it is equally valuable to understand which queries have historically been the most expensive in terms of CPU. For this, the **Worst CPU Queries** panel was built, which crosses three Redshift system tables:

- `STL_QUERY`: historical record of all queries executed in the cluster (Amazon Web Services, n.d.-f).
- `SVL_QUERY_METRICS`: view that exposes performance metrics of completed queries, including `cpu_time`, `cpu_skew`, `io_skew`, and row counts processed (Amazon Web Services, n.d.-g).
- `STL_ALERT_EVENT_LOG`: table that records performance alerts that Redshift raises internally for a query, including the event type and the corresponding solution suggestion (Amazon Web Services, n.d.-h).

<p align="center">
  <img src="resources/worst.png" alt="Worst CPU usage table"  width="950" />
</p>

The combination of these three sources is particularly powerful: it not only shows how much CPU a query consumed, but also directly includes Redshift's recommendation to solve the detected performance problem. The query used is:

```sql
SELECT 
    stq.pid,
    stq.query,
    u.usename                                   AS "username",
    LEFT(stq.querytxt, 500)                     AS "querytext",
    svq.cpu_skew,
    svq.io_skew,
    TRIM(SPLIT_PART(event, ':', 1))             AS "redshift_alert",
    TRIM(solution)                              AS "redshift_solution",
    svq.query_cpu_usage_percent,
    svq.query_cpu_time                          AS "query_cpu_time_seconds",
    svq.query_cpu_time / 60                     AS "query_cpu_time_minutes",
    svq.join_row_count                          AS "rows_joined",
    svq.scan_row_count,
    svq.query_execution_time,
    stq.starttime,
    stq.endtime
FROM stl_query stq
JOIN svl_query_metrics svq 
    ON stq.query = svq.query 
LEFT JOIN stl_alert_event_log ael 
    ON ael.query = stq.query
LEFT JOIN pg_user u 
    ON stq.userid = u.usesysid
WHERE svq.query_cpu_time IS NOT NULL 
  AND $__timeFilter(stq.starttime) 
ORDER BY svq.query_cpu_time DESC 
LIMIT 20;
```

## CPU alerts using Slack

With the CPU dashboard built, the alert system closes the loop: instead of having to manually check Grafana periodically, the alert acts as a sentinel that notifies the exact moment the cluster is under pressure.

The defined rule evaluates the `CPUUtilization` metric from CloudWatch on the leader node every 30 minutes and triggers if the value exceeds **80% for a confirmation period of 10 minutes**, this avoids false positives due to momentary spikes. When the condition is reached, Grafana sends a notification to the corresponding Slack channel with the severity level, problem description, and a direct link to the CPU dashboard to start the diagnosis.

<p align="center">
  <img src="resources/alert.png" alt="Grafana alert slack" width="450"/>
</p>

The response to an alert can follow two paths depending on what the dashboard reveals: if there is a user query running for hours and saturating the leader node, the most immediate action is to cancel it. If the problem is recurrent and associated with an internal process, such as a dbt model with inefficient materialization or a query without an adequate sort key, the alert becomes the starting point for an optimization effort based on real data.

## Conclusions

Implementing this monitoring architecture was, in many ways, like turning on the lights in a room that had been dark for a long time. The problems were not new, slow queries, DAGs that took longer and longer, the cluster responding slowly, all this happening without metrics, without visibility, without a place to look, any attempt at a solution was little more than a guess.

What changed with monitoring was not the cluster itself, but the ability to understand it. We identified dbt testing queries that consumed resources in production without anyone knowing. We discovered users whose queries ran for hours and saturated the leader node. We found recurring patterns that, now visible, became concrete candidates for optimization.

Monitoring a data warehouse is one of those topics that gets postponed indefinitely "there's always a more urgent feature, a higher priority pipeline", until the system collapses and urgency takes over. However, investing on visibility before the problem arises is exactly what allows the system to scale with the business instead of becoming a bottleneck.

If there is one thing this experience made clear to me, it is that a data warehouse without monitoring is not a system that works well: it is a system that has not yet failed enough for us to realize that we do not understand it.

# References
- Amazon Web Services. (n.d.). The difference between monitoring and observability. AWS. https://aws.amazon.com/compare/the-difference-between-monitoring-and-observability/
- Amazon Web Services. (n.d.-a). Amazon CloudWatch. AWS. https://aws.amazon.com/cloudwatch/
- Amazon Web Services. (n.d.-b). AWS X-Ray. AWS. https://aws.amazon.com/xray/
- Amazon Web Services. (n.d.-c). Amazon Managed Grafana. AWS. https://aws.amazon.com/grafana/
- Bhatt, S. (2023). AWS Certified Data Engineer – Associate [Online course]. Udemy. https://www.udemy.com/course/aws-data-engineer-associate/
- Datadog. (n.d.). The monitoring and security platform for cloud applications. https://www.datadoghq.com/
- Amazon Web Services. (n.d.-d). *Viewing cluster performance data — Amazon Redshift*.  
https://docs.aws.amazon.com/redshift/latest/mgmt/performance-metrics-perf.html
- Grafana Labs. (n.d.-a). *Amazon CloudWatch data source*.  
https://grafana.com/docs/grafana/latest/datasources/aws-cloudwatch/
- Grafana Labs. (n.d.-b). *Redshift data source plugin*.  
https://grafana.com/docs/plugins/grafana-redshift-datasource/latest/
- Grafana Labs. (n.d.-c). *Configure Slack contact point*.  
https://grafana.com/docs/grafana/latest/alerting/configure-notifications/manage-contact-points/integrations/configure-slack/
- Amazon Web Services. (n.d.-e). *STV_RECENTS — Amazon Redshift*.  
https://docs.aws.amazon.com/redshift/latest/dg/r_STV_RECENTS.html
- Amazon Web Services. (n.d.-f). *STL_QUERY — Amazon Redshift*.  
https://docs.aws.amazon.com/redshift/latest/dg/r_STL_QUERY.html
- Amazon Web Services. (n.d.-g). *SVL_QUERY_METRICS — Amazon Redshift*.  
https://docs.aws.amazon.com/redshift/latest/dg/r_SVL_QUERY_METRICS.html
- Amazon Web Services. (n.d.-h). *STL_ALERT_EVENT_LOG — Amazon Redshift*.  
https://docs.aws.amazon.com/redshift/latest/dg/r_STL_ALERT_EVENT_LOG.html
