

Readme · MD
iu5-observability-lab
Учебный стенд наблюдаемости (observability): Flask-приложение с OpenTelemetry отдаёт метрики, трейсы и логи, а набор сервисов собирает и визуализирует их.

Архитектура
Сервис	Контейнер	Образ	Назначение
app	iu5devops-app	iu5devops/app (собирается локально)	Flask-приложение: метрики, трейсы, логи
prometheus	iu5devops-prometheus	prom/prometheus:v2.55.0	Сбор метрик с /metrics
jaeger	iu5devops-jaeger	jaegertracing/all-in-one:1.64.0	Хранение и просмотр трейсов (OTLP gRPC :4317)
loki	iu5devops-loki	grafana/loki:main-672fb08	Хранение логов
promtail	iu5devops-promtail	grafana/promtail:main-672fb08	Отправка логов приложения в Loki
grafana	iu5devops-grafana	grafana/grafana:11.4.0	Визуализация (источники данных настроены через provisioning)
Поток данных:

app ──метрики (/metrics)──► Prometheus ──┐
app ──трейсы (OTLP :4317)──► Jaeger ─────┼──► Grafana
app ──логи (app/log/flask.log)──► Promtail ──► Loki ──┘
Структура проекта
.
├── app
│   ├── log
│   │   └── .gitkeep
│   ├── .gitignore
│   ├── app.py
│   ├── Dockerfile
│   └── requirements.txt
├── grafana
│   └── grafana.yml
├── prometheus
│   └── prometheus.yml
├── promtail
│   └── promtail.yaml
├── .gitignore
├── docker-compose.yaml
└── README.md
Адреса сервисов
Сервис	Адрес на хосте
Приложение	http://localhost:8080
Grafana	http://localhost:4000
Prometheus	http://localhost:9090
Jaeger UI	http://localhost:16686
Loki API	http://localhost:3100
Порт приложения внутри контейнера — 5000, на хост он проброшен как 8080. Promtail портов наружу не публикует. Порт OTLP 4317 Jaeger доступен только внутри сети Compose (iu5devops-jaeger:4317).

Эндпоинты приложения
GET /rolldice — бросок кубика, увеличивает счётчик request_counter, создаёт спаны do_roll и do_important_job
GET /metrics — метрики в формате Prometheus
bash
curl http://localhost:8080/rolldice
curl http://localhost:8080/metrics
В app.py используется randint(1, 7), поэтому примерно в 1 из 7 запросов /rolldice возвращает ошибку 500. Это удобно для демонстрации ошибок в метриках и логах.

Переменные окружения приложения
Переменная	Значение в compose	По умолчанию в коде
APP_SERVICE_NAME	iu5devops-app	my-python-service
TRACE_ENDPOINT	http://iu5devops-jaeger:4317	http://localhost:4317
APP_HOST_NAME	не задана	0.0.0.0
APP_PORT	не задана	5000
В Jaeger сервис будет называться iu5devops-app (из APP_SERVICE_NAME).

Быстрый старт
bash
git clone https://github.com/OptimusSound/iu5-observability-lab.git
cd iu5-observability-lab
Docker Compose v2
bash
docker compose up -d --build
docker compose ps
docker compose logs app
docker compose down
docker-compose v1 (например, 1.25.0 на DevNet VM)
bash
sudo docker-compose up -d --build
sudo docker-compose ps
sudo docker-compose logs app
sudo docker-compose down
Какая версия установлена:

bash
docker compose version     # v2
docker-compose --version   # v1
Проверка работы
Сделайте несколько запросов:
bash
   for i in $(seq 1 20); do curl -s http://localhost:8080/rolldice; echo; done
Prometheus → Status → Targets: цель app должна быть UP. В Graph можно ввести request_counter_total.
Jaeger → сервис iu5devops-app → Find Traces.
Grafana → Explore → источник Loki: логи приложения.
Совместимость с разными версиями Compose
docker-compose.yaml приведён к формату, который понимает старый docker-compose 1.25.0, поэтому он работает и в v1, и в v2.

Изменение	Причина
Первая строка version: "3.3"	Compose 1.25 поддерживает форматы до 3.7; без version v1 не запускается
Нет name: у networks и volumes	Поле не поддерживается старым форматом (Additional properties are not allowed ('name' was unexpected))
depends_on в виде простого списка	Условия condition: service_healthy недоступны в формате 3.x
Что меняется из-за отсутствия name:
Compose добавляет префикс проекта (имя папки) к сети и томам:

Ресурс	В файле	Реальное имя (пример)
сеть	iu5devops	iu5-observability-lab_iu5devops
том логов	app-log	iu5-observability-lab_app-log
том Grafana	grafana-storage	iu5-observability-lab_grafana-storage
том Prometheus	prometheus-data	iu5-observability-lab_prometheus-data
Сервисы обращаются друг к другу по именам контейнеров (iu5devops-jaeger и т. д.), поэтому конфигурации Prometheus, Promtail и Grafana менять не нужно. Реальные имена:

bash
docker network ls
docker volume ls
В Compose v2 явные имена можно вернуть через name:, но в v1 (1.25) это вызовет ошибку.

Зависимости приложения
app/requirements.txt:

Flask==3.0.0
setuptools<81
opentelemetry-sdk==1.21.0
opentelemetry-exporter-prometheus==0.42b0
opentelemetry-exporter-otlp-proto-grpc==1.21.0
opentelemetry-instrumentation==0.42b0
opentelemetry-instrumentation-flask==0.42b0
opentelemetry-instrumentation-logging==0.42b0
setuptools<81 — в Python 3.12 setuptools не ставится по умолчанию, а opentelemetry-instrumentation импортирует pkg_resources, который удалён в новых версиях setuptools. Предупреждение pkg_resources is deprecated в логах безвредно.
opentelemetry-exporter-prometheus==0.42b0 — версия подобрана под линейку SDK 1.21.0.
Решение частых проблем
Симптом	Решение
unknown shorthand flag: 'd' in -d	Нет плагина Compose v2. Используйте docker-compose (с дефисом)
The Compose file is invalid ... 'name' was unexpected	Удалите name: из networks и volumes, поставьте version: "3.3"
ModuleNotFoundError: No module named 'pkg_resources'	Добавьте setuptools<81 в requirements.txt, пересоберите образ
Образ не обновился после правки requirements.txt	docker-compose build --no-cache app
permission denied на сокете Docker	Запускайте через sudo или добавьте пользователя в группу docker
container name ... is already in use	docker-compose down или docker rm -f <имя контейнера>, затем up -d
/rolldice иногда возвращает 500	Так задумано: randint(1, 7) может вернуть 7
RuntimeError: can't start new thread (при сборке или запуске app)	См. раздел ниже
error adding seccomp filter rule for syscall clone3: permission denied	См. раздел ниже
Старый runc на виртуалке (Ubuntu 20.04, DevNet VM)
Если при сборке или запуске контейнера появляются ошибки can't start new thread или error adding seccomp filter rule for syscall clone3, причина в устаревших runc и containerd: на DevNet VM стоит runc 1.0.0~rc10 и containerd 1.3.3, которые не работают с новым seccomp-профилем Docker и новыми образами. Смена базового образа не помогает, нужно обновить компоненты:

bash
sudo apt update
sudo apt install --only-upgrade runc containerd libseccomp2
sudo systemctl restart containerd docker
runc --version        # должно быть 1.1.x
sudo docker-compose build --no-cache app
sudo docker-compose up -d
Работа с Git на нескольких устройствах
bash
git pull                      # перед началом работы
git add .
git commit -m "Описание изменений"
git push                      # после завершения
Логи (app/log/*.log), .env и кэш Python в репозиторий не попадают благодаря .gitignore; папка app/log создаётся за счёт .gitkeep.



ИТОГ
На свежей виртуалке (DevNet VM, Ubuntu 20.04) порядок такой.

## 1. Обновите runc и containerd (делается один раз)

Это самый важный шаг: без него сборка и запуск падают с `clone3` или `can't start new thread`.

```bash
sudo apt update
sudo apt install --only-upgrade runc containerd libseccomp2
sudo systemctl restart containerd docker
runc --version        # должно быть 1.1.x
```

## 2. Проверьте, что есть git и docker-compose

```bash
git --version
docker-compose --version    # на DevNet VM обычно 1.25.0
```

Если `docker-compose` нет: `sudo apt install -y docker-compose`.

## 3. Клонируйте проект

```bash
git clone https://github.com/OptimusSound/iu5-observability-lab.git
cd iu5-observability-lab
```

Если репозиторий приватный, вместо пароля вводите Personal Access Token.

## 4. Запустите

```bash
sudo docker-compose up -d --build
sudo docker-compose ps
```

Все шесть контейнеров должны быть в статусе `Up`. Версия `3.3` в `docker-compose.yaml`, `setuptools<81` в `requirements.txt` и правки совместимости уже лежат в репозитории, повторять их не нужно.

## 5. Проверьте работу

```bash
for i in $(seq 1 20); do curl -s http://localhost:8080/rolldice; echo; done
```

Затем откройте Prometheus (`:9090`), Jaeger (`:16686`) и Grafana (`:4000`).

## Если что-то пошло не так

- Конфликт имён контейнеров: `sudo docker-compose down`, затем снова `up -d --build`
- Ошибка `clone3` или `can't start new thread`: вернитесь к шагу 1
- Подробности по всем ошибкам есть в разделе «Решение частых проблем» в `README.md`

Пока шаг 1 не выполнен, остальные шаги будут давать те же ошибки, что и раньше.
