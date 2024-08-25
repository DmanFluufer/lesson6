## Установка и настройка Alertmanager

1. Скачиваем архив

   ```shell
   wget https://github.com/prometheus/alertmanager/releases/download/v0.27.0/alertmanager-0.27.0.linux-amd64.tar.gz
   ```

2. Создаем пользователя и нужные каталоги, настраиваем для них владельцев 

   ```shell
   useradd --no-create-home --shell /bin/false alertmanager
   mkdir /etc/alertmanager
   mkdir /var/lib/alertmanager
   chown alertmanager:alertmanager /etc/alertmanager
   chown alertmanager:alertmanager /var/lib/alertmanager
   ```

3. Создаём файл конфигурации `alertmanager.yml`

   ```yaml
   global:
     resolve_timeout: 5m
     smtp_smarthost: 'smtp.mysmtp.com:587'
     smtp_from: 'alerts@mysmtp.com'
     smtp_auth_username: 'myuser'
     smtp_auth_password: 'mypassword'
   
   route:
     group_by: ['alertname', 'severity']
     repeat_interval: 1h
     receiver: 'telegram'
   
     routes:
       - receiver: 'email'
         match:
           alertname: 'HostDown'
         group_wait: 30s
         group_interval: 5m
   
   receivers:
     - name: telegram
       telegram_configs:
       - chat_id: <chat_id>
         parse_mode: 'Markdown'
         bot_token: '<token>'
   
     - name: email
       email_configs:
         - to: 'support@mysmtp.com'
           send_resolved: true
   ```

4. Меняем владельцев у бинарников

   ```shell
   chown alertmanager:alertmanager /usr/local/bin/alertmanager
   chown alertmanager:alertmanager /usr/local/bin/amtool
   ```

5. Создаём systemd сервис и запускаем Alertmanager

   ```shell
   $ vim /etc/systemd/system/alertmanager.service
   [Unit]
   Description=Alertmanager
   Wants=network-online.target
   After=network-online.target
   
   [Service]
   User=alertmanager
   Group=alertmanager
   Type=simple
   ExecStart=/usr/local/bin/alertmanager \
     --config.file=/etc/alertmanager/alertmanager.yml \
     --storage.path=/var/lib/alertmanager
   
   [Install]
   WantedBy=multi-user.target
   
   $ systemctl daemon-reload
   $ systemctl start alertmanager
   $ systemctl status alertmanager
   ```

   

## Настройка Prometheus

1. Добавил следующие строчки в текущую конфигурацию

   ```yaml
   alerting:
     alertmanagers:
       - static_configs:
           - targets:
             - 'localhost:9093'
   
   rule_files:
     - "/etc/prometheus/alert.rules"
   ```

2. Создадим файл правил

   ```yaml
   groups:
   - name: disk_alerts
     rules:
     - alert: DiskSpaceUsageHigh
       expr: (node_filesystem_avail_bytes{mountpoint="/usr/protei/data"} / node_filesystem_size_bytes{mountpoint="/usr/protei/data"}) * 100 < 10
       for: 5m
       labels:
         severity: warning
       annotations:
         summary: "High disk utilization on {{ $labels.instance }}"
   
   - name: host_availability
     rules:
     - alert: HostDown
       expr: up{job="pcrf_node_exporter_centos", instance="192.168.110.120:9100"} == 0
       for: 5m
       labels:
         severity: critical
       annotations:
         summary: "Host {{ $labels.instance }} is down"
         description: "The host {{ $labels.instance }} is not reachable for more than 5 minutes."
   ```
