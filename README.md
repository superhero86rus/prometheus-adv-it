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
sudo nano /etc/systemd/system/prometheus.service
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

# Автоматическая установка: chmod +x install.sh и выполнить его
```

### Node Exporter

```bash
# Добавляем новые сервера под мониторинг, модифицируя prometheus.yml
global:
  scrape_interval: 10s 

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: "ubuntu-servers"
    static_configs:
      - targets:
          - 192.168.88.29:9100
          - 192.168.88.30:9100

# Установка Node Exporter
wget https://github.com/prometheus/node_exporter/releases/download/v1.8.2/node_exporter-1.8.2.linux-amd64.tar.gz
tar xvfz node_exporter-1.8.2.linux-amd64.tar.gz
cd node_exporter-1.8.2.linux-amd64
mv node_exporter /usr/bin

# -rs Системный пользователь, без логина
useradd -rs /bin/false node_exporter
chown node_exporter:node_exporter /usr/bin/node_exporter

sudo nano /etc/systemd/system/node_exporter.service
# Содержание:
[Unit]
Description=Prometheus Node Exporter
After=network.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
Restart=on-failure
ExecStart=/usr/bin/node_exporter

[Install]
WantedBy=multi-user.target

# Включаем
systemctl daemon-reload
systemctl start node_exporter
systemctl status node_exporter
systemctl enable node_exporter

# Автоматическая установка - node_exporter.sh
scp /home/superhero86/Documents/source/prometheus-adv-it/node_exporter.sh root@ubuntuserver2:~
ssh root@ubuntuserver2
chmod +x node_exporter.sh
./node_exporter.sh
```

### Windows Exporter

```bash
# По умолчанию, Windows Exporter экспортирует данные на порту 9182
# Правим конфиг
global:
  scrape_interval: 10s 

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: "ubuntu-servers"
    static_configs:
      - targets:
          - 192.168.88.29:9100
          - 192.168.88.30:9100
  
  - job_name: "windows-servers"
    static_configs:
      - targets:
          - 192.168.88.25:9182

# sudo systemctl restart prometheus
# https://github.com/prometheus-community/windows_exporter/releases/download/v0.27.1/windows_exporter-0.27.1-amd64.msi
# http://192.168.88.25:9182/metrics
# Автоматическая установка - InstallWindowsExporter.ps1
```

### Grafana

```bash
# Все важные конфиг файлы хранятся в /etc/grafana
# Prometheus Datasource настраивается в /etc/grafana/provisioning/datasources/prometheus.yml
# Инсталяционная директория графаны - /usr/share/grafana
# Данные и дашборды хранятся в /var/lib/grafana/grafana.db
# Grafana работает на порту 3000, admin/admin

# Установка из репозитория
sudo apt-get install -y apt-transport-https software-properties-common wget
sudo mkdir -p /etc/apt/keyrings/
wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null
echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list
sudo apt-get update

# Автоматическая установка
# sudo apt-get install grafana

# Но мы скачаем и установим вручную
sudo apt-get install -y adduser libfontconfig1 musl
wget https://dl.grafana.com/oss/release/grafana_11.1.4_amd64.deb
sudo dpkg -i grafana_11.1.4_amd64.deb
sudo /bin/systemctl daemon-reload
sudo /bin/systemctl enable grafana-server
systemctl start grafana-server

# Автоматическая установка скриптом install_grafana.sh
scp /home/superhero86/Documents/source/prometheus-adv-it/node_exporter.sh root@ubuntuserver2:~
```