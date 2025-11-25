# 🔍 Мониторинг доступности сайтов leads.su api.leads.su tracker.leads.su

Система мониторинга доступности веб-сайтов на базе Prometheus, Grafana и Blackbox Exporter.

##  Задание

- В docker развернуть prometheus & grafana
- Настроить мониторинг доменов (не чаще чем 1 раз в минуту): 
    - leads.su 
    -api.leads.su 
    - tracker.leads.su
- Настроить графики в grafana
- Сделать триггеры на недоступность

## Quickstart

### 1. Запуск системы мониторинга

```
# Запустить все сервисы
docker-compose up -d

# Проверить статус
docker-compose ps

# Просмотр логов
docker-compose logs -f
```

### 2. Доступ к интерфейсам

| Сервис | URL | Credentials |
|--------|-----|-------------|
| **Grafana** | http://localhost:3000 | admin / admin |
| **Prometheus** | http://localhost:9090 | - |
| **Alertmanager** | http://localhost:9093 | - |
| **Blackbox Exporter** | http://localhost:9115 | - |

### 3. Просмотр дашбордов

1. url: http://localhost:3000
2. login/pass: admin/admin
3. go to  **Dashboards** → **Leads.su Monitoring**

## Метрики и графики

### Основные метрики

1. **Статус доступности** (`probe_success`)
   - 1 = сайт доступен
   - 0 = сайт недоступен

2. **Время отклика** (`probe_duration_seconds`)
   - Общее время ответа сервера

3. **HTTP статус коды** (`probe_http_status_code`)
   - 200, 301, 404, 500 и т.д.

4. **SSL сертификаты** (`probe_ssl_earliest_cert_expiry`)
   - Время до истечения сертификата

5. **Детализация запроса** (`probe_http_duration_seconds`)
   - DNS lookup
   - TCP connect
   - TLS handshake
   - Processing
   - Transfer

### Графики в Grafana

Dashboard включает:

1. **Статус панели** - текущее состояние каждого сайта (UP/DOWN)
2. **График доступности** - история доступности за период
3. **Время отклика** - график времени ответа
4. **HTTP коды** - отслеживание статус кодов
5. **Uptime метрики** - процент доступности за 24 часа
6. **Breakdown по фазам** - детализация времени запроса
7. **Таблица статусов** - сводная таблица
8. **SSL сертификаты** - время до истечения

## Алерты и триггеры

### Настроенные алерты

#### Критические (Critical)

1. **WebsiteDown**
   - Условие: `probe_success == 0`
   - Время: 2 минуты
   - Действие: Немедленное уведомление

2. **SSLCertificateExpired**
   - Условие: Сертификат истек
   - Время: 1 минута
   - Действие: Критическое уведомление

3. **BlackboxExporterDown**
   - Условие: Blackbox Exporter недоступен
   - Время: 1 минута
   - Действие: Система мониторинга не работает

#### Предупреждения (Warning)

1. **WebsiteSlowResponse**
   - Условие: Время ответа > 5 секунд
   - Время: 3 минуты
   - Действие: Предупреждение о производительности

2. **SSLCertificateExpiringSoon**
   - Условие: Сертификат истекает через < 30 дней
   - Время: 1 час
   - Действие: Напоминание обновить

3. **HTTPStatusCodeError**
   - Условие: HTTP код >= 400
   - Время: 2 минуты
   - Действие: Ошибка на сайте

4. **LowUptime**
   - Условие: Uptime < 99% за 24 часа
   - Время: 5 минут
   - Действие: Низкая доступность

##  Настройка

### Изменение интервала проверки
 `prometheus/prometheus.yml`:

```yaml
global:
  scrape_interval: 60s  # Изменить интервал (минимум 15s)
```

### Добавление новых доменов

 `prometheus/prometheus.yml`

```yaml
- job_name: 'new-domain-check'
  metrics_path: /probe
  params:
    module: [http_2xx]
  static_configs:
    - targets:
        - https://newdomain.com
      labels:
        domain: 'newdomain.com'
  relabel_configs:
    - source_labels: [__address__]
      target_label: __param_target
    - source_labels: [__param_target]
      target_label: instance
    - target_label: __address__
      replacement: blackbox:9115
```

```
docker-compose restart prometheus
```

### Настройка email уведомлений

`alertmanager/alertmanager.yml`:

```yaml
global:
  smtp_smarthost: 'smtp.gmail.com:587'
  smtp_from: 'alerts@example.com'
  smtp_auth_username: 'your-email@gmail.com'
  smtp_auth_password: 'your-app-password'

receivers:
  - name: 'email-alerts'
    email_configs:
      - to: 'ops-team@example.com'
        headers:
          Subject: '🚨 Alert: {{ .GroupLabels.alertname }}'
```

## Примеры запросов Prometheus

### Проверка доступности

```promql
# Текущий статус всех сайтов
probe_success{job=~".*leads.*"}

# Сайты, которые недоступны
probe_success{job=~".*leads.*"} == 0

# Uptime за последние 24 часа
avg_over_time(probe_success[24h]) * 100
```

### Производительность

```promql
# Среднее время ответа за последний час
avg_over_time(probe_duration_seconds{job=~".*leads.*"}[1h])

# Максимальное время ответа
max_over_time(probe_duration_seconds{job=~".*leads.*"}[1h])

# Время DNS lookup
probe_http_duration_seconds{phase="resolve"}
```

### SSL сертификаты

```promql
# Дни до истечения сертификата
(probe_ssl_earliest_cert_expiry - time()) / 86400

# Сертификаты, истекающие менее чем через 30 дней
(probe_ssl_earliest_cert_expiry - time()) / 86400 < 30
```

## Тестирование

### Ручная проверка через Blackbox Exporter

```
# Проверка leads.su
curl "http://localhost:9115/probe?target=https://leads.su&module=http_2xx"

# Проверка с метриками
curl "http://localhost:9115/probe?target=https://leads.su&module=http_2xx" | grep probe_
```

### Проверка метрик в Prometheus

```
# Получить все метрики для leads.su
curl "http://localhost:9090/api/v1/query?query=probe_success{instance=\"https://leads.su\"}"
```

### Тест алертов

```
# Просмотр активных алертов
curl http://localhost:9090/api/v1/alerts | jq .

# Просмотр правил
curl http://localhost:9090/api/v1/rules | jq .
```


