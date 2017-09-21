# Prometheus+Grafana+Alertmanager+Exporters in docker

A monitoring solution for Docker hosts and containers with [Prometheus](https://prometheus.io/), [Grafana](http://grafana.org/), [cAdvisor](https://github.com/google/cadvisor), 
[NodeExporter](https://github.com/prometheus/node_exporter) and alerting with [AlertManager](https://github.com/prometheus/alertmanager).

## Install

Clone this repository on your Docker host, cd into directory and run compose up:

* `$ git clone ssh://git@github.com/e.lebed/monitor.git` 
* `$ cd monitor`
* `$ docker-compose up -d`

Containers:

* Prometheus (metrics database) `http://<host-ip>:9090`
* AlertManager (alerts management) `http://<host-ip>:9093`
* Grafana (visualize metrics) `http://<host-ip>:3000`
* NodeExporter (host metrics collector)
* cAdvisor (containers metrics collector)
* NginxExporter (nginx metrics collector)

While Grafana supports authentication, the Prometheus and AlertManager services have no such feature. 
You can remove the ports mapping from the docker-compose file and use NGINX as a reverse proxy providing basic authentication for Prometheus and AlertManager.

## Setup Grafana

Navigate to `http://<host-ip>:3000` and login with user ***admin*** password ***admin***. You can change the password from Grafana UI or 
 by modifying the [user.config] file.

From the Grafana menu, choose ***Data Sources*** and click on ***Add Data Source***. 
Use the following values to add the Prometheus container as data source:

* Name: Prometheus
* Type: Prometheus
* Url: http://prometheus:9090
* Access: proxy

Now you can import the dashboard temples from the [grafana] directory. 
From the Grafana menu, choose ***Dashboards*** and click on ***Import***.

## Define alerts

I've setup three alerts configuration files:

* Monitoring services alerts [targets.rules]
* Docker Host alerts [hosts.rules]
* Docker Containers alerts [containers.rules]

You can modify the alert rules and reload them by making a HTTP POST call to Prometheus:

```
curl -X POST http://<host-ip>:9090/-/reload
```

***Monitoring services alerts***

Trigger an alert if any of the monitoring targets (node-exporter and cAdvisor) are down for more than 30 seconds:

```yaml
ALERT monitor_service_down
  IF up == 0
  FOR 30s
  LABELS { severity = "critical" }
  ANNOTATIONS {
      summary = "Monitor service non-operational",
      description = "{{ $labels.instance }} service is down.",
  }
```

***Docker Host alerts***

Trigger an alert if the Docker host CPU is under high load:

```yaml
ALERT high_cpu_usage
  IF (100 - (avg by (instance) (irate(node_cpu{mode="idle"}[5m])) * 100)) > 75
  FOR 2m
  LABELS { severity="critical" }
  ANNOTATIONS {
    summary = "{{$labels.instance}}: High CPU usage detected",
    description = "{{$labels.instance}}: CPU usage is above 75% (current value is: {{ $value }})"
  }
```

Trigger an alert if the Docker host LA > 3.5:                         

```yaml
ALERT high_LA
  IF node_load5 > 3.5
  FOR 2m
  LABELS { severity = "warning" }
  ANNOTATIONS {
      summary = "{{$labels.instance}}: High LA detected",
      description = "Host {{ $labels.instance }} is under high load, the avg load 5m is at {{ $value}}",
  }
```

Trigger an alert if the Docker host memory usage is above 75%:

```yaml
ALERT high_memory_load
  IF (sum(node_memory_MemTotal) - sum(node_memory_MemFree + node_memory_Buffers + node_memory_Cached) ) / sum(node_memory_MemTotal) * 100 > 75
  FOR 2m
  LABELS { severity = "warning" }
  ANNOTATIONS {
      summary = "{{ $labels.instance }} memory is almost full",
      description = "Host {{ $labels.instance }} memory usage is {{ humanize $value}}%",
  }
```

Trigger an alert if swap usage is above 75%:

```yaml
ALERT high_swap_usage
  IF (((node_memory_SwapTotal - node_memory_SwapFree) / node_memory_SwapTotal) * 100) > 75
  FOR 2m
  LABELS { severity="warning" }
  ANNOTATIONS {
    summary = "{{$labels.instance}}: High swap usage detected",
    description = "{{$labels.instance}}: Swap usage is above 75% (current value is: {{ $value }})"
  }
```

Trigger an alert if the Docker host root disk usage is is above 75%:

```yaml
ALERT high_root_disk_load
  IF ((node_filesystem_size{fstype="ext4",mountpoint=~"/|/rootfs"} - node_filesystem_free{fstype="ext4",mountpoint=~"/|/rootfs"} ) / node_filesystem_size{fstype="ext4",mountpoint=~"/|/rootfs"} * 100) > 75
  FOR 2m
  LABELS { severity = "warning" }
  ANNOTATIONS {
      summary = "{{ $labels.instance }} root disk is almost full",
      description = "Host {{ $labels.instance }} root disk usage is {{ humanize $value}}%",
  }
```

Trigger an alert if the Hetzner storage disk usage is is above 75%:

```yaml
ALERT high_hetzner_storage_load
  IF ((node_filesystem_size{fstype="cifs",mountpoint=~"/mnt/.+"} - node_filesystem_free{fstype="cifs",mountpoint=~"/mnt/.+"} ) / node_filesystem_size{fstype="cifs",mountpoint=~"/mnt/.+"} * 100) > 75
  FOR 2m
  LABELS { severity = "warning" }
  ANNOTATIONS {
    summary = "{{$labels.instance}}: mounted storage is almost full",
    description = "Host {{$labels.instance}} mounted storage usage is {{ humanize $value}}%"
  }
```

***Docker Containers alerts***

Trigger an alert if a container is down for more than 30 seconds (example):

```yaml
ALERT satis
  IF absent(container_memory_usage_bytes{name="satis"})
  FOR 30s
  LABELS { severity = "critical" }
  ANNOTATIONS {
      summary= "CONTAINER '{{ $labels.name }}' down",
      description= "CONTAINER '{{ $labels.name }}' on '{{ $labels.host }}' is down for more than 30 seconds."
  }
```

Trigger an alert if a container is using more than 10% of total CPU cores more than 5 minutes:

```yaml
ALERT high_cpu_usage_on_container
  IF sum(rate(container_cpu_usage_seconds_total{name=~".+"}[1m])) by (name,host) * 100 > 10
  FOR 2m
  LABELS { severity = "warning" }
  ANNOTATIONS {
      summary = "High CPU usage: CONTAINER '{{ $labels.name }}' on '{{ $labels.host }}'",
      description = "{{ $labels.name }} is using a LOT of CPU. CPU usage is {{ humanize $value}}%.",
  }
```

Trigger an alert if a container GRAYLOG is using more than 1,2GB of RAM for more than 5 minutes:

```yaml
ALERT graylog_eating_memory
  IF sum(container_memory_rss{name=~"graylog"}) by (name) > 1200000000
  FOR 5m
  LABELS { severity = "warning" }
  ANNOTATIONS {
      summary = "High memory usage: CONTAINER '{{ $labels.name }}' on '{{ $labels.instance }}'",
      description = "{{ $labels.name }} is eating up a LOT of memory. Memory consumption of {{ $labels.name }} is at {{ humanize $value}}.",
  }
```
Other alert rules under development.

## Setup alerting

The AlertManager service is responsible for handling alerts sent by Prometheus server. 
AlertManager can send notifications via email, Pushover, Slack, HipChat or any other system that exposes a webhook interface. 
A complete list of integrations can be found [here](https://prometheus.io/docs/alerting/configuration).

You can view and silence notifications by accessing `http://<host-ip>:9093`.

The notification receivers can be configured in [alertmanager/config.yml] file.

To receive alerts via Slack you need to make a custom integration by choose ***incoming web hooks*** in your Slack team app page. 
You can find more details on setting up Slack integration [here](http://www.robustperception.io/using-slack-with-the-alertmanager/).

Copy the Slack Webhook URL into the ***api_url*** field and specify a Slack ***channel***.

```yaml
route:
    receiver: 'slack'

receivers:
    - name: 'slack'
      slack_configs:
          - send_resolved: true
            text: "{{ .CommonAnnotations.description }}"
            username: 'Prometheus'
            channel: '#<channel>'
            api_url: 'https://hooks.slack.com/services/<webhook-id>'
```
