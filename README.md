# prometheus-adv-it

### Prometheus Agent называется Exporter

Node Exporter -> Linux monitoring
Windows Exporter
MySQL Server Exporter
Apache Exporter
NVIDIA GPU Exporter
cAdvisor -> Мониторинг контейнеров (docker, k8s)

### У каждого exporter есть своя web-страница мониторинга

Обычно это порт 9100
http://192.168.8.13:9100 - информация об экспортере
http://192.168.8.13:9100/metrics - метрики

### Типы метрик
Counter - счетчик, который только увеличивается
Gauge - метрика которая увеличивается и уменьшается
Histogram - распределение величин метрики по группам (промежутки)
Summary - Percentile/Quantile из Histogram

### Prometheus Server
http://192.168.8.12:9090 - стандартный порт api сервера

### Тестовая конфигурация (2 linux + 2 win servers)
Client Linux (Node Exporter)
http://10.0.0.1:9100/metrics


Client Linux (Node Exporter + JMeter Exporter)
http://10.0.0.2:9100/metrics
http://10.0.0.2:9270/metrics

Client Windows (Windows Exporter)
http://10.0.0.3:9182/metrics

Client Windows (Windows Exporter)
http://10.0.0.4:9182/metrics

### Управляющий сервер
Prometheus Server
http://10.0.0.0:9090

### Prometheus Alert Manager
Отправляет алерты куда угодно (email, discord, telegram, sms, WebHook и куча всего другого)

### PromQL
node_os_info or windows_os_info
rate(node_network_receive_bytes_total[1m])

### Установка Prometheus из архива
```bash
mkdir prometheus
wget https://github.com/prometheus/prometheus/releases/download/v2.54.0-rc.0/prometheus-2.54.0-rc.0.linux-amd64.tar.gz

tar xvfz prometheus-2.54.0-rc.0.linux-amd64.tar.gz
cd prometheus-2.54.0-rc.0.linux-amd64/
# Удаляем ненужный мусор
rm LICENSE 
rm NOTICE 
rm promtool
rm rf console*
rm -rf data/

# Переносим исполняемый файл
mv prometheus /usr/bin

# Переносим конфиг приложения в /etc/prometheus/
mkdir /etc/prometheus
mv prometheus.yml /etc/prometheus/

# Создаем отдельную УЗ и наделяем правами все содержимое /etc/prometheus/
useradd -rs /bin/false prometheus
chown -R prometheus:prometheus /etc/prometheus/
mkdir /etc/prometheus/data

# Теперь нужно поместить Prometheus в автозагрузку
nano /etc/systemd/system/prometheus.service
# Содержание:
[Unit]
Description=Prometheus Server
After=network.target

[Service]
User=prometheus
Group=prometheus
Type=simple
Restart=on-failure
ExecStart=/usr/bin/prometheus \
  --config.file         /etc/prometheus/prometheus.yml \
  --storage.tsdb.path   /etc/prometheus/data

[Install]
WantedBy=multi-user.target

# Перезапускаем конфигурацию служб, чтобы наш файл подхватить
systemctl daemon-reload
systemctl start prometheus
systemctl status prometheus

systemctl enable prometheus
```