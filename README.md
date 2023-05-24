# Домашнее задание к занятию "`Система мониторинга Prometheus. Часть 2`"
# `Островский Евгений`


### Инструкция по выполнению домашнего задания

   1. Сделайте `fork` данного репозитория к себе в Github и переименуйте его по названию или номеру занятия, например, https://github.com/имя-вашего-репозитория/git-hw или  https://github.com/имя-вашего-репозитория/7-1-ansible-hw).
   2. Выполните клонирование данного репозитория к себе на ПК с помощью команды `git clone`.
   3. Выполните домашнее задание и заполните у себя локально этот файл README.md:
      - впишите вверху название занятия и вашу фамилию и имя
      - в каждом задании добавьте решение в требуемом виде (текст/код/скриншоты/ссылка)
      - для корректного добавления скриншотов воспользуйтесь [инструкцией "Как вставить скриншот в шаблон с решением](https://github.com/netology-code/sys-pattern-homework/blob/main/screen-instruction.md)
      - при оформлении используйте возможности языка разметки md (коротко об этом можно посмотреть в [инструкции  по MarkDown](https://github.com/netology-code/sys-pattern-homework/blob/main/md-instruction.md))
   4. После завершения работы над домашним заданием сделайте коммит (`git commit -m "comment"`) и отправьте его на Github (`git push origin`);
   5. Для проверки домашнего задания преподавателем в личном кабинете прикрепите и отправьте ссылку на решение в виде md-файла в вашем Github.
   6. Любые вопросы по выполнению заданий спрашивайте в чате учебной группы и/или в разделе “Вопросы по заданию” в личном кабинете.
   
Желаем успехов в выполнении домашнего задания!
   
### Дополнительные материалы, которые могут быть полезны для выполнения задания

1. [Руководство по оформлению Markdown файлов](https://gist.github.com/Jekins/2bf2d0638163f1294637#Code)

---

### Задание 1

Создайте файл с правилом оповещения, как в лекции, и добавьте его в конфиг Prometheus.

Создаем файл
```
sudo nano /etc/prometheus/my-test.yml
```
```
groups: # Список групп
- name: my-test # Имя группы
  rules: # Список правил текущей группы
  - alert: InstanceDown # Название текущего правила
    expr: up == 0 # Логическое выражение
    for: 1m # Сколько ждать отбоя сработки перед отправкой оповещения
    labels: 
      severity: critical # Критичность события
    annotations: # Описание
      description: '{{ $labels.instance }} of job {{ $labels.job }} has been down for more than 1 minute.' # Полное описание алерта
      summary: Instance {{ $labels.instance }} down # Краткое описание алерта
```
Подключаем правило
```
sudo nano /etc/prometheus/prometheus.yml
```
в rule_files добавляем
```
- "my-test.yml"
```
Перезапускаем Prometheus
```
sudo systemctl restart prometheus
systemctl status prometheus
```

### Оповещения в Alartmanager
Редактируем
```
sudo nano /etc/prometheus/alertmanager.yml
```
```
global:
route:
  group_by: ['alertname'] # Параметр группировки оповещений — по имени
  group_wait: 30s # Сколько ждать восстановления, перед тем как отправить первое оповещение
  group_interval: 10m # Сколько ждать, перед тем как дослать оповещение о новых сработках по текущему алерту
  repeat_interval: 60m # Сколько ждать, перед тем как отправить повторное оповещение
  receiver: 'email' # Способ, которым будет доставляться текущее оповещение
receivers: # Настройка способов оповещения
- name: 'email'
  email_configs:
    - to: 'yourmailto@todomain.com'
      from: 'yourmailfrom@fromdomain.com'        
      smarthost: 'mailserver:25'
      auth_username: 'user'
      auth_identity: 'user'
      auth_password: 'paS$w0rd'
```

### Требования к результату
Погасите node exporter, стоящий на мониторинге, и прикрепите скриншот раздела оповещений Prometheus, где оповещение будет в статусе Pending

![Task1](https://github.com/joos-net/Prometheus2/blob/main/task1.png)


---

### Задание 2

Установите Alertmanager и интегрируйте его с Prometheus.

Качаем и извлекаем
```
wget https://github.com/prometheus/alertmanager/releases/download/v0.24.0/alertmanager-0.24.0.linux-amd64.tar.gz
tar -xvf alertmanager-*linux-amd64.tar.gz
```
Копируем необходимые файлы и папки
```
sudo cp ./alertmanager-*.linux-amd64/alertmanager /usr/local/bin 
sudo cp ./alertmanager-*.linux-amd64/amtool /usr/local/bin
sudo cp ./alertmanager-*.linux-amd64/alertmanager.yml /etc/prometheus
```
Передаем права
```
sudo chown -R prometheus:prometheus /etc/prometheus/alertmanager.yml
```
Создаем сервис
```
nano /etc/systemd/system/prometheus-alertmanager.service
```
```
[Unit]
Description=Alertmanager
Service After=network.target
[Service]
EnvironmentFile=-/etc/default/alertmanager
User=prometheus
Group=prometheus
Type=simple ExecStart=/usr/local/bin/alertmanager --config.file=/etc/prometheus/alertmanager.yml --storage.path=/var/lib/prometheus/alertmanager $ARGS
ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure
[Install]
WantedBy=multi-user.target
```
Прописываем автозапуск и запускаем
```
sudo systemctl enable prometheus-alertmanager
sudo systemctl start prometheus-alertmanager
sudo systemctl status prometheus-alertmanager
```
Добавляем в config Prometheus подключение к Alartmanager
```
sudo nano /etc/prometheus/prometheus.yml
```
```
alerting: 
alertmanagers:
- static_configs:
- targets: # Можно указать как targets: [‘localhost”9093’]
- localhost:9093
```
Перезапускаем Prometheus
```
sudo systemctl restart prometheus
systemctl status prometheus
```

### Требования к результату
Прикрепите скриншот Alerts из Prometheus, где правило оповещения будет в статусе Fireing, и скриншот из Alertmanager, где будет видно действующее правило оповещения

![Task21](https://github.com/joos-net/Prometheus2/blob/main/task21.png)
![Task22](https://github.com/joos-net/Prometheus2/blob/main/task22.png)


---

### Задание 3

Активируйте экспортёр метрик в Docker и подключите его к Prometheus.

Создаем файл
```
sudo nano /etc/docker/daemon.json
```
```
{ 
"metrics-addr" : "ip_нашего_сервера:9323", 
"experimental" : true 
}
```
Перезапускаем Docker
```
sudo systemctl restart docker && systemctl status docker
```
Результат по адресу - http://server_ip:9323/metrics

Ставим Docker на мониторинг
```
sudo nano /etc/prometheus/prometheus.yml
```
Добавляем
```
static_configs:
- targets: ['localhost:9090', 'localhost:9100', 'server_ip:9323']
```
Перезапускаем Prometheus
```
sudo systemctl restart prometheus
systemctl status prometheus
```

### Требования к результату
Приложите скриншот браузера с открытым эндпоинтом, а также скриншот списка таргетов из интерфейса Prometheus.*

![Task31](https://github.com/joos-net/Prometheus2/blob/main/task31.png)
![Task32](https://github.com/joos-net/Prometheus2/blob/main/task32.png)


---
## Дополнительные задания (со звездочкой*)

Эти задания дополнительные (не обязательные к выполнению) и никак не повлияют на получение вами зачета по этому домашнему заданию. Вы можете их выполнить, если хотите глубже и/или шире разобраться в материале.

### Задание 4

Создайте свой дашборд Grafana с различными метриками Docker и сервера, на котором он стоит.

 - В интерфейсе Grafana нажмите на «+» и выберите Dashboards
 - Нажмите + New dashboard > Add new panel
 - В выпадающем меню Metrics выберите: engine > engine_daemon_container_states_containers
 - Нажмите Apply и перейдите в интерфейс Dashboard
 - Сохраните Dashboard

### Требования к результату
Приложите скриншот, на котором будет дашборд Grafana с действующей метрикой
 
![Task4](https://github.com/joos-net/Prometheus2/blob/main/task4.png)
