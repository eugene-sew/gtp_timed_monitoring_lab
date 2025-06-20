# Prometheus & Grafana DevOps Lab

## Section 1: Architecture and Setup Understanding

### Question 1: Setting up the environment

- Node Exporter mounts the host's /proc, /sys, and / filesystems to the container to collect various metrics about the host system. Once the services are running (as shown in `docker-compose ps.png` in the Screenshots section below), Node Exporter provides these metrics. Prometheus can then be configured to scrape these metrics, and its target status page will confirm if Node Exporter is UP (see `node-exporter-up-confirmation.png` in the Screenshots section).

  - The "/" is mounted to collect details(metrics) about the host's filesystem such as disk usage (an example visualization can be seen in `disk_usage.png` in the Screenshots section), inode usage, and other filesystem metrics.
  - The "/proc" contains valuable information about the host's processes and system resources. These include CPU usage, memory usage (an example visualization can be seen in `memory_uage.png` in the Screenshots section), and other system metrics.
  - The "/sys" has inforrmation on the host's physical hardware and kernel, from the "/sys" node monitor can collect metrics such as heardware health, network device statistics, power stats , cpu information, block device statistics and more.

By mounting these filesystems to the container, node exporter maintains isolation and modulariy while still being able to collect metrics from the host system, this approach also provides access to the host's resources in a controlled manner especially as they are mounted as read-only. For visualization, Grafana is set up to connect to Prometheus as a data source (see `datasource_config.png` in the Screenshots section).

### Question 2: Network Security Implications

- The docker network "prometheus-grafana" is a bridge network that allows the containers to communicate with each other within the same host especially using the container name as the hostname. With the current port exposure configuration, as the ports are currently exposed to the internet, unauthorized users can access sensitive metrics, APIs, or the Grafana dashboard and even eploit misconfigured services to gain access to the host system fully.

- For deployment in a production environment,I would add a reverse proxy with HTTPs in front of the containers to limit access to the containers and expose only the necessary services to the public. Docker's "--internal" flag can be used to limit the visibility of the containers to the host system so the custom network is only accessible internally. For my security groups, I will add only port 3000 to allow traffic from my IP address only.

### Question 3: Data Persistence Strategy

- Prometheus uses both a bind mount for configuration (./prometheus.yml) and a named volume for data (prometheus_data:/prometheus), while Grafana only uses a named volume (grafana_data:/var/lib/grafana) for persistent storage. This approach allows Prometheus configuration to be easily edited on the host while both services maintain their critical data (metrics and dashboards) across container restarts. Without these volume configurations, you would lose all historical metrics data in Prometheus and all custom dashboards, users, and data sources in Grafana every time the containers restart.

## Section 2: Metrics and Query Understanding

### Question 4: PromQL Logic Breakdown

- node_time_seconds represents the current system time.
- node_boot_time_seconds represents when the system last booted.

- These are both in Unix timestamp format. Subtracting boot time from current time gives uptime in seconds, but this method can be affected by clock synchronization issues, making node_uptime_seconds a more reliable alternative for direct uptime monitoring as the metric is calculated based on the system's actual uptime.

### Question 5: Memory Metrics Deep Dive

- Linux distinguishes between MemFree as completely unused memory and MemAvailable as memory readily available to applications, including reclaimable cache/buffers Using node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes provides accurate memory usage monitoring because it accounts for Linux's aggressive caching strategy, preventing false high-memory alerts that would occur with MemFree

### Question 6: Filesystem Query Analysis

- **The Query**
  ```promql
  1 - (node_filesystem_avail_bytes{mountpoint="/"} /
       node_filesystem_size_bytes{mountpoint="/"})
  ```

This query calculates filesystem usage by dividing available bytes by total bytes to get the fraction of free space, then subtracts from 1 to convert it to usage percentage (e.g., 0.2 free becomes 0.8 or 80% used).

Monitoring multiple mount points using this query could lead to false alerts. This is because it doesn't account for temporary filesystems such as `tmpfs` or `devtmpfs`, which are used for temporary storage and don't represent actual storage usage.

To avoid this issue, I would filter out temporary filesystems using `fstype!~"tmpfs|devtmpfs|overlay"` to exclude virtual filesystems that don't represent actual storage usage.

This filtering would make my equation:

```promql
1 - (
    node_filesystem_avail_bytes{
        mountpoint="/",
        fstype!~"tmpfs|devtmpfs|overlay|aufs|squashfs|proc|sysfs"
    } /
    node_filesystem_size_bytes{
        mountpoint="/",
        fstype!~"tmpfs|devtmpfs|overlay|aufs|squashfs|proc|sysfs"
    }
)
```

## Section 3: Visualization and Dashboard Design

### Question 7: Visualization Type Justification

- Visualisation of Uptime: I chose Stat Panel for uptime (an example is shown in `up_time.png` in the Screenshots section) because of its ability to display a single value, which makes it ideal for rapidly determining recent reboots by providing the current system runtime in an easy-to-understand manner.

- RAM Visualisation: Since trend analysis was required, I used Time Series to visualise RAM usage and total RAM. By superimposing various metrics, it makes it easier to spot memory leaks or capacity planning needs.

- Disc Usage Visualisation: I chose to use Gauge for disc usage because of its percentage-based visual representation, which gives operators instant threshold awareness with colour-coded zones (red at 80–100%) that instantly convey storage criticality.

### Question 8:

- The 80% threshold for disk usage is a common standard designed to prevent performance degradation and provide a buffer for temporary files and usage spikes. Industry standards suggest tailored thresholds based on system type: 70–75% for database servers due to their write-heavy nature, 85–90% for web servers which are read-heavy, and 80–85% for application servers with balanced workloads.

- To implement multi-level thresholds that provide actionable insights, I would configure a monitoring system, such as Prometheus, to trigger alerts at different usage levels. Informational alerts would be set at 70% usage to monitor trends, warning alerts at 85% to prompt cleanup actions, and critical alerts at 95% to demand immediate intervention. This can be achieved by defining Prometheus rules with severity labels like `info`, `warning`, and `critical`. Additionally, thresholds would be customized based on the server type (e.g., database or web servers) to account for varying workloads. Predictive alerts leveraging historical trends would enable proactive measures, ensuring reliability and optimal performance.

### Question 9:

- In Grafana, variables like $job dynamically populate values from data sources, enabling filtering by substituting selected values into queries (an example of variable configuration can be seen in `variable_configuration.png` in the Screenshots section). When multiple values are selected, they are expanded into regex-like expressions (e.g., job=~"(job1|job2)"). Poorly implemented variables, such as those with invalid, empty, or excessive values, can cause query errors, slow performance, or unexpected results.

- To test variable robustness in Grafana, I would validate that variable queries return expected values across all scenarios, simulate edge cases such as no values or special characters to check for errors, monitor performance to assess the impact of multiple or complex selections, verify that dependent panels respond correctly to variable changes, and implement fallback mechanisms like default values

## Section 4: Production and Scalability Considerations

### Question 10: Resource Planning and Scaling

For 100 servers, I'd estimate needing 2-4 CPU cores, 4-8 GB of RAM, and about 40 GB of storage. I expect the first bottleneck would be disk I/O, which I'd solve with fast SSDs, followed by high cardinality issues that I would address by scaling up or using a tool like Thanos.

### Question 11: High Availability Design

I would design an HA architecture by running two identical Prometheus instances, a three-node Alertmanager cluster, and two Grafana instances connected to a shared database. This adds complexity, but I'd make that trade-off for reliability, using regular snapshots and infrastructure-as-code for disaster recovery.

### Question 12: Security Hardening Analysis

To harden this setup, I would fix five key vulnerabilities by placing services behind a reverse proxy with TLS and auth, using a secrets manager like Vault, and configuring RBAC in Grafana. I would also encrypt internal traffic with mTLS and restrict access to the node-exporter endpoint via firewall rules.

## Section 5: Troubleshooting and Operations

### Question 13: Debugging Methodology

When a target is "DOWN," my first step is to check the error in the Prometheus UI, then I'd use `curl` from the Prometheus host to test connectivity. If that works, I'd check if the exporter service is running on the target and finally double-check for typos in my `prometheus.yml` config.

### Question 14: Performance Optimization

To optimize performance, my primary strategy would be to use Prometheus recording rules to pre-calculate expensive queries and speed up dashboards. I would also limit variable options in Grafana and monitor the monitoring system itself with a separate, dedicated "meta-monitor" Prometheus instance.

### Question 15: Capacity Planning Scenario

To manage storage growth, I would define a short retention period (e.g., 30 days) on my primary Prometheus for high-resolution operational data. For long-term trends, I would downsample key metrics and send them to a cheaper storage solution like Thanos using `remote_write`.

## Screenshots

![Datasource Configuration](screenshots/datasource_config.png)
*Datasource Configuration in Grafana*

![Disk Usage Gauge](screenshots/disk_usage.png)
*Disk Usage visualization in Grafana*

![Docker Compose PS Output](screenshots/docker-compose%20ps.png)
*Output of `docker-compose ps` showing running containers*

![Memory Usage Time Series](screenshots/memory_uage.png)
*Memory Usage visualization in Grafana*

![Node Exporter Up Confirmation](screenshots/node-exporter-up-confirmation.png)
*Prometheus target page showing Node Exporter as UP*

![Uptime Stat Panel](screenshots/up_time.png)
*Uptime visualization in Grafana*

![Variable Configuration](screenshots/variable_configuration.png)
*Grafana dashboard variable configuration*

