# Prometheus & Grafana DevOps Lab

## Section 1: Architecture and Setup Understanding

### Question 1: Setting up the environment

- Node Exporter mounts the host's /proc, /sys, and / filesystems to the container to collect various metrics about the host system.

  - The "/" is mounted to to collect details(metrics) about the host's filesystem such as the disk usage, inode usage, and other filesystem metrics.
  - The "/proc" contains valuable information about the host's processes and system resources. These include cpu usage, memory usage, and other system metrics.
  - The "/sys" has inforrmation on the host's physical hardware and kernel, from the "/sys" node monitor can collect metrics such as heardware health, network device statistics, power stats , cpu information, block device statistics and more.

By mounting these filesystems to the container, node exporter maintains isolation and modulariy while still being able to collect metrics from the host system, this approach also provides access to the host's resources in a controlled manner especially as they are mounted as read-only.

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
