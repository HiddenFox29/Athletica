# Стоимость владения системой (TCO — Total Cost of Ownership)

Данный документ содержит оценку стоимости владения системой Athletica на
горизонте 1, 2 и 5 лет с учётом роста нагрузки, пользователей и данных.

Анализ основан на архитектуре системы (base-architecture.md,
architecture-views.md) и включает основные инфраструктурные компоненты:

- Compute (вычислительные ресурсы);
- PostgreSQL (базы данных);
- Object Storage (объектное хранилище, S3-compatible);
- Cache (кэш);
- regional NGINX (региональные ingress-слои, сеть и балансировка);
- regional API Gateway (региональные слои клиентских API);
- Monitoring / Logging / Tracing (наблюдаемость).

---

## 1. Используемые провайдеры

Для оценки выбраны:

### AWS (Amazon Web Services)

- EC2 (Compute): https://aws.amazon.com/ec2/pricing/
- RDS (Database): https://aws.amazon.com/rds/pricing/
- S3 (Object Storage): https://aws.amazon.com/s3/pricing/
- AWS RabbitMQ (Amazon MQ for
  RabbitMQ): https://aws.amazon.com/amazon-mq/pricing/

### Yandex Cloud

- Compute Cloud: https://yandex.cloud/ru/docs/compute/pricing
- Managed PostgreSQL: https://yandex.cloud/ru/docs/managed-postgresql/pricing
- Object Storage: https://yandex.cloud/ru/docs/storage/pricing
- Yandex Cloud Managed Service for
  RabbitMQ: https://yandex.cloud/ru/docs/managed-rabbitmq/pricing

Цены приведены ориентировочно на основе публичных тарифов (on-demand — по
требованию, без резервирования).

---

## 2. Предположения

### Рост пользователей:

- Год 1: 10 000 пользователей
- Год 2: 50 000 пользователей
- Год 5: 500 000 пользователей

### Нагрузка:

- постепенный рост количества запросов и данных (x5 ежегодно)

### Архитектура:

- multi-region (2 региона);
- в каждом регионе развёрнут собственный NGINX как ingress (входной контур) и
  балансировщик;
- в каждом регионе развёрнут собственный API Gateway как слой клиентских API;
- в каждом регионе предусмотрен резервный standby-контур (резервный контур в
  готовности), увеличивающий инфраструктурный overhead;
- database per service на PostgreSQL;
- использование RabbitMQ как брокера событий;
- использование Cache;
- использование Object Storage для медиа и файлов.

---

## 3. Оценка стоимости по компонентам

### 3.1 Compute (серверы)

AWS:

- EC2 t3.medium (~$30/мес за инстанс)

Yandex Cloud:

- ВМ среднего уровня (~$25–35/мес)

Примечание:
Compute включает доменные сервисы, региональные API Gateway и часть внутренней
инфраструктуры приложений. Стоимость compute увеличивается из-за дублирования
edge-слоя по регионам и необходимости иметь резерв мощности для
standby-контура.

---

### 3.2 Databases (PostgreSQL)

AWS:

- RDS (~$100/мес за инстанс)

Yandex Cloud:

- Managed PostgreSQL (~$90–120/мес)

---

### 3.3 Object Storage (S3)

AWS:

- ~$0.023 за GB/месяц

Yandex Cloud:

- ~$0.02–0.03 за GB/месяц

---

### 3.4 Cache (Redis или аналог)

AWS:

- ElastiCache (~$50–150/мес)

Yandex Cloud:

- Managed Redis (~$40–120/мес)

---

### 3.5 Regional NGINX (ingress и балансировка)

AWS:

- Load Balancer (~$20–50/мес)

Yandex Cloud:

- Балансировщик (~$15–40/мес)

Примечание:
В итоговой topology NGINX не является единым глобальным компонентом. В каждом
регионе используется собственный NGINX ingress-слой, а значит стоимость
edge-компонентов дублируется по регионам и должна учитывать standby-overhead.

---

### 3.6 Regional API Gateway (шлюз API)

AWS:

- может быть реализован через API Gateway или как отдельные сервисы на EC2 (~$
  20–100/мес в зависимости от нагрузки)

Yandex Cloud:

- может быть реализован как отдельные сервисы (на Compute Cloud) (~$20–80/мес)

Примечание:
В итоговой architecture topology API Gateway не рассматривается как единый
глобальный слой. В каждом регионе развёрнут собственный API Gateway, поэтому
его стоимость либо считается отдельно, либо включается в региональный
compute-слой как часть edge-дублирования.

---

### 3.7 RabbitMQ (брокер сообщений)

AWS:

- Amazon MQ for RabbitMQ (~$50–150/мес)

Yandex Cloud:

- Managed Service for RabbitMQ (~$40–120/мес)

---

## 4. Итоговая стоимость

### Год 1 (MVP)

- Compute: $160
- PostgreSQL: $120
- Object Storage: $50
- Cache: $50
- Regional NGINX: $40
- Regional API Gateway: $40
- RabbitMQ: $50
- Monitoring: $50

Итого:
~ $560 / месяц
~ $6 720 / год

Примечание:
Оценка учитывает regional edge duplication (дублирование edge-слоя по регионам)
и минимальный standby overhead (резерв мощности для standby-контура).

---

### Год 2

- Compute: $380
- PostgreSQL: $240
- Object Storage: $150
- Cache: $100
- Regional NGINX: $90
- Regional API Gateway: $90
- RabbitMQ: $100
- Monitoring: $100

Итого:
~ $1 250 / месяц
~ $15 000 / год

Примечание:
На этом этапе увеличиваются не только доменные compute-ресурсы, но и стоимость
региональных edge-слоёв из-за роста нагрузки и необходимости сохранять резерв
для failover.

---

### Год 5

- Compute: $1 500
- PostgreSQL: $600
- Object Storage: $500
- Cache: $200
- Regional NGINX: $220
- Regional API Gateway: $220
- RabbitMQ: $200
- Monitoring: $200

Итого:
~ $3 640 / месяц
~ $43 680 / год

Примечание:
При высокой нагрузке стоимость regional edge-слоёв и standby-overhead
становится заметной частью TCO, поэтому multi-region архитектура требует
отдельного учёта входного контура и API-слоя.

---

## 5. Сравнение AWS vs Yandex Cloud

| Компонент            | AWS                    | Yandex Cloud                    |
|----------------------|------------------------|---------------------------------|
| Compute              | выше                   | ниже или сопоставимо            |
| PostgreSQL           | выше                   | немного дешевле                 |
| Object Storage       | сопоставимо            | сопоставимо                     |
| Cache                | выше                   | немного дешевле                 |
| RabbitMQ             | выше                   | немного дешевле или сопоставимо |
| Regional NGINX       | выше                   | дешевле                         |
| Regional API Gateway | выше (managed сервисы) | ниже (self-hosted)              |

Вывод:

- Yandex Cloud может быть дешевле на ранних этапах;
- AWS предоставляет более развитую экосистему;
- выбор зависит от требований к масштабированию и географии.

---

## 6. Связь с архитектурой

Стоимость напрямую зависит от архитектурных решений:

- microservices увеличивают количество compute-инстансов;
- event-driven требует RabbitMQ и дополнительной инфраструктуры;
- multi-region увеличивает стоимость почти в 2 раза;
- regional NGINX и regional API Gateway создают edge duplication (дублирование
  входного edge-слоя) по регионам;
- standby-контуры увеличивают инфраструктурный overhead даже при отсутствии
  постоянного основного трафика;
- использование Object Storage снижает нагрузку на PostgreSQL;
- использование Cache снижает нагрузку на compute и PostgreSQL;

Связанные документы:

- base-architecture.md;
- architecture-views.md;
- ADR-001, ADR-003, ADR-004, ADR-007;
- risk-analysis.md.

---

## 7. Вывод

Стоимость владения системой растёт не только пропорционально нагрузке, но и
из-за multi-region topology, regional edge duplication и standby-overhead. При
этом стоимость остаётся управляемой и архитектурно оправданной.

Выбранная архитектура обеспечивает баланс между:

- масштабируемостью;
- надёжностью;
- стоимостью.

При необходимости возможна оптимизация за счёт:

- резервирования ресурсов;
- использования более дешёвых инстансов;
- оптимизации хранения данных.
