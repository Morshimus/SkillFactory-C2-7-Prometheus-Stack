# SkillFactory-C2-7-Prometheus-Stack

* [x] - :one: **Разверните Prometheus Stack через docker-compose, в котором будет:**
    - *Prometheus*
    - *Grafana*
    - *Node Exporter*
    - *Blackbox Exporter*
    - *AlertManager*
    - *Plus caddy, alertmanager-bot and caadvisor as bonus*

```yaml
version: '3.8'

name: "morsh-prom"

   
x-environment-admin: &Admin_ENV
   - ADMIN_USER=${ADMIN_USER}
   - ADMIN_PASSWORD=${ADMIN_PASSWORD}
   - ADMIN_PASSWORD_HASH=${ADMIN_PASSWORD_HASH}


networks:
  monitor-net:
    driver: bridge

volumes:
    prometheus_data: {}
    grafana_data: {}


x-logging: &logging
 driver: "json-file"
 options:
   max-size: "100m"
   max-file: "1"

services:

  prometheus:
    image: prom/prometheus:v2.42.0
    container_name: prometheus
    volumes:
      - ./prometheus:/etc/prometheus
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.console.libraries=/etc/prometheus/console_libraries'
      - '--web.console.templates=/etc/prometheus/consoles'
      - '--storage.tsdb.retention.time=200h'
      - '--web.enable-lifecycle'
    restart: unless-stopped
    expose:
      - 9090
    networks:
      - monitor-net
    logging: *logging
    labels:
      org.label-schema.group: "monitoring"

  alertmanager-bot:
    container_name: alertmanager-bot
    command:
      - --alertmanager.url=http://alertmanager:9093
      - --log.level=info
      - --store=bolt
      - --bolt.path=/data/bot.db
    environment:
      TELEGRAM_ADMIN: ${TELEGRAM_CHATID}
      TELEGRAM_TOKEN: ${TELEGRAM_TOKEN}
    image: metalmatze/alertmanager-bot:0.4.3
    ports:
      - "8082:8080"
    networks:
      - monitor-net
    restart: unless-stopped
    volumes:
      - ./alertmanager-bot:/data
    labels:
      org.label-schema.group: "monitoring"

  alertmanager:
    image: prom/alertmanager:v0.25.0
    container_name: alertmanager
    environment:
      - TELEGRAM_TOKEN=${TELEGRAM_TOKEN}
      - TELEGRAM_CHATID=${TELEGRAM_CHATID}
    volumes:
      - ./alertmanager:/etc/alertmanager
    command:
      - '--config.file=/etc/alertmanager/config.yml'
      - '--storage.path=/alertmanager'
    restart: unless-stopped
    expose:
      - 9093
    networks:
      - monitor-net
    logging: *logging
    labels:
      org.label-schema.group: "monitoring"

  nodeexporter:
    image: prom/node-exporter:v1.5.0
    container_name: nodeexporter
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.rootfs=/rootfs'
      - '--path.sysfs=/host/sys'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
    restart: unless-stopped
    expose:
      - 9100
    networks:
      - monitor-net
    logging: *logging
    labels:
      org.label-schema.group: "monitoring"

  blackbox-exporter:
    image: prom/blackbox-exporter
    container_name: blackbox-exporter
    expose:
     - 9115
    networks:
     - monitor-net
    restart: unless-stopped
    volumes:
     - ./blackbox:/config
    command: --config.file=/config/blackbox.yml
    labels:
      org.label-schema.group: "monitoring"

  cadvisor:
    image: gcr.io/cadvisor/cadvisor:v0.47.1
    container_name: cadvisor
    privileged: true
    devices:
      - /dev/kmsg:/dev/kmsg
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker:/var/lib/docker:ro
      #- /cgroup:/cgroup:ro #doesn't work on MacOS only for Linux
    restart: unless-stopped
    expose:
      - 8080
    networks:
      - monitor-net
    logging: *logging
    labels:
      org.label-schema.group: "monitoring"

  grafana:
    image: grafana/grafana:9.4.1
    container_name: grafana
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/provisioning/dashboards:/etc/grafana/provisioning/dashboards
      - ./grafana/provisioning/datasources:/etc/grafana/provisioning/datasources
    environment:
      - GF_SECURITY_ADMIN_USER=${ADMIN_USER}
      - GF_SECURITY_ADMIN_PASSWORD=${ADMIN_PASSWORD}
      - GF_USERS_ALLOW_SIGN_UP=false
    restart: unless-stopped
    expose:
      - 3000
    networks:
      - monitor-net
    logging: *logging
    labels:
      org.label-schema.group: "monitoring"

  pushgateway:
    image: prom/pushgateway:v1.5.1
    container_name: pushgateway
    restart: unless-stopped
    expose:
      - 9091
    networks:
      - monitor-net
    logging: *logging
    labels:
      org.label-schema.group: "monitoring"

  caddy:
    image: caddy:2.6.4
    container_name: caddy
    ports:
      - "3000:3000"
      - "8080:8080"
      - "9090:9090"
      - "9093:9093"
      - "9091:9091"
      - "9115:9115"
      - "8081:8081"
    volumes:
      - ./caddy:/etc/caddy
    environment: *Admin_ENV
    restart: unless-stopped
    networks:
      - monitor-net
    logging: *logging
    labels:
      org.label-schema.group: "monitoring"
```

* [x] - :two: **Соберите метрики с ~~https://.skillfactory.ru~~  https://apps.skillfactory.ru через Blackbox, соберите метрики с вашего сервера через Node Exporter.**
> Сделано, как доказательсто предоставляю скриншот с probe.
 ![image](https://ams03pap004files.storage.live.com/y4mQf28aHQms9mQv8Iy5JEgFyt-BaEHvlfypChUb7jgB0Wf0HMJtFEdkjHofimL_hxhFz2h-jlwFraYKE3_CqrBOivFwcjGGxdc8FoA5Ab1_hSu9qKTJ0MSXawIBX4p-HPDIteJCNAvUBgXM0Y5JAOgoLFCezcCSUceG3pddXNB_fcxkwdQLJKtk6t3Vk3C502F?encodeFailures=1&width=1917&height=603)
 
* [x] - :three: **Создайте dashboard в Grafana, в котором будут отображены следующие метрики:**

На вашем сервере (или локальной машине):

  - время работы (Uptime); 
  - нагрузка на процессор (CPU) в %;
  - использование памяти (RAM) в %;
  - использование диска в %.
 
 ![image](https://ams03pap004files.storage.live.com/y4mqDQBaKhWdU_Z5HPnOlOKrTWG9Iuef8-1FX-o8IfuZ4AIdVc_PmzpuxLgfZXy8VqaiyqKfCggqSx4uVNhZ5_nBoYWuuJhGlWRUG8RoymuPIZq5KpfzRu4vJEC4VOTsAnKa61IWDB9TbBGkEU40GFpDvI9Vv80tayoixqT01bLfYNQLqrFrIvtHfY-QzzwLSxd?encodeFailures=1&width=1676&height=801)
 ![image](https://ams03pap004files.storage.live.com/y4mfgyTuTyUVuLEwsq5-upzIwXMjc9d7_b2iphduxr7fjTflwwU-JFu0TZ3XVdi6woOyrSCITQqA-kC4AzuTgsP-eNA5EymrbzEnFtqmbKf8cSdPIAl0jVtn8e6BtCbawKu7Zo8f4J8r6rYNIYbtz1K9PvDGm03XbFPqOaSPcqKcMOe2aWGTfIBJoGQN8WQyyeK?encodeFailures=1&width=1720&height=801)

  *PS у меня Windows - в Linux место диска будет нормальным, для Windows данный не подходит.*

На ~~lms.skillfactory.ru~~ apps.skillfactory.ru:

  - возвращаемый статус-код;
  - задержка ответа сайта;
  - срок действия сертификата.

  ![image](https://ams03pap004files.storage.live.com/y4m7WnOFg-bqhYl9xXu4d1GQ-4vuW26b8nqnjSw-7S6opOo2Yf9Neqd7_DxMK-iSaQ_ift3eM2hyf130zUeZoXCqYq_ow-khey4Tgcsqsgyq8rnW4s8IoEh8m8eyjsUlJ-xJOcB2mNFZHr2ZgDHBKUdCvL6euyHSnnwjAl-qstC5cmFxXbpvH03-PDZijNXh6Oq?encodeFailures=1&width=1628&height=801)
  ![image](https://ams03pap004files.storage.live.com/y4mYkze3ygIQJUfvBr50bGq0tKt-45Pj7Aeg4V4NuGxZUYe1Dntv0Jo_1-JH6RZh4otjVuU3XrJIvZWY2h7S0iIduPnQ9lJUf_AU7-19SM42yL1tnK8vmzJTwoAlZou8vtTBnBIewbgXTWqw9OyPif85n1Hf7kwyKIQsfPaJiQy1Daioi13wSkBrRTa7bShNI26?encodeFailures=1&width=1758&height=362)
  ![image](https://ams03pap004files.storage.live.com/y4mrQ5Vprg3P5EZYWrnTO6FNkzUu5rwMsoF91GFuwO9fNY4wn_C-dWMWFdJ_x1eryZfhVlYhQmwX_OpI346aQsjSUNKTDEcC2F9FW2gq6YwtzTWRluox2bINdwHf40na4jXkbK7yWjBABPIrOnqanNAp2kZNAwOY_Wy9MlDodfm4rLK5YLwHtw161ar175hFiDd?encodeFailures=1&width=1272&height=332)


* [x] - :four: **Добавьте алерты в AlertManager на следующие события:**

    - изменился статус-код сайта ~~lms~~apps.skillfactory.ru;
    - задержка превышает 5 секунд ~~lms~~apps.skillfactory.ru;
    - сервер перезагрузился (через метрику Uptime).

> Сделано, в качестве теста я перезагрузил свой Laptop :smile:

```yaml
groups:
- name: targets
  rules:
  - alert: monitor_service_down
    expr: up == 0
    for: 30s
    labels:
      severity: critical
    annotations:
      summary: "Monitor service non-operational"
      description: "Service {{ $labels.instance }} is down."

- name: host
  rules:
  - alert: high_cpu_load
    expr: node_load1 > 1.5
    for: 30s
    labels:
      severity: warning
    annotations:
      summary: "Server under high load"
      description: "Docker host is under high load, the avg load 1m is at {{ $value}}. Reported by instance {{ $labels.instance }} of job {{ $labels.job }}."

  - alert: high_memory_load
    expr: (sum(node_memory_MemTotal_bytes) - sum(node_memory_MemFree_bytes + node_memory_Buffers_bytes + node_memory_Cached_bytes) ) / sum(node_memory_MemTotal_bytes) * 100 > 85
    for: 30s
    labels:
      severity: warning
    annotations:
      summary: "Server memory is almost full"
      description: "Docker host memory usage is {{ humanize $value}}%. Reported by instance {{ $labels.instance }} of job {{ $labels.job }}."

  - alert: high_storage_load
    expr: (node_filesystem_size_bytes{fstype="aufs"} - node_filesystem_free_bytes{fstype="aufs"}) / node_filesystem_size_bytes{fstype="aufs"}  * 100 > 85
    for: 30s
    labels:
      severity: warning
    annotations:
      summary: "Server storage is almost full"
      description: "Docker host storage usage is {{ humanize $value}}%. Reported by instance {{ $labels.instance }} of job {{ $labels.job }}."

  - alert: server_was_rebooted
    expr: node_time_seconds - node_boot_time_seconds  < 300
    for: 30s
    labels:
      severity: warning
    annotations:
      summary: "Server was rebooted"
      description: "Server was rebooted. Uptime is less than 5 minutes"

- name: apps.skillfactory.ru
  rules:
  - alert: skillfactory_not_200_reply
    expr: probe_http_status_code{instance="https://apps.skillfactory.ru/"} != 200
    for: 30s
    labels:
      severity: critical
    annotations:
      summary: "Skillfactory lms down"
      description: "Skillfactory lms web app is not replying for more than 30 seconds."

  - alert: skillfactory_reply_delay_greater_than_5_seconds 
    expr: probe_http_duration_seconds{instance="https://apps.skillfactory.ru/"} > 5
    for: 30s
    labels:
      severity: warning
    annotations:
      summary: "Skillfactory lms delay greater than 5 seconds"
      description: "Skillfactory lms common reply delay more than 5 seconds last 30 seconds"


```

![image](https://ams03pap004files.storage.live.com/y4m9PcWBqyo5SkU_LJAPwB7Vj4t4hNP9v1NzzVixI30QzfaKDq9_OeIN3ik9FU3AhS34KDrftZO3BNjMHDNaSQ9tFhShhZXSsuatNFBM3p7PKWIEu-0VZBpoD2zH9HULnCbMLjSGmFhiiYB-Qq5EUXjJ7Uj1G6F8HngPgbmVlhz4HqBBhKqVxLIkVNtjdeL-3Rq?encodeFailures=1&width=1046&height=801)
