# Стоимость владения системой (TCO — Total Cost of Ownership)

Данный документ содержит оценку стоимости владения системой Athletica на горизонте 1, 2 и 5 лет с учётом роста нагрузки, пользователей и данных.

Анализ основан на архитектуре системы (base-architecture.md, architecture-views.md) и включает основные инфраструктурные компоненты:
- Compute (вычислительные ресурсы);
- PostgreSQL (базы данных);
- Object Storage (объектное хранилище, S3-compatible);
- Cache (кэш);
- Network и NGINX Load Balancer (сеть и балансировка);
- API Gateway (шлюз API);
- Monitoring / Logging / Tracing (наблюдаемость).

---

## 1. Используемые провайдеры

Для оценки выбраны:

### AWS (Amazon Web Services)
- EC2 (Compute): https://aws.amazon.com/ec2/pricing/
- RDS (Database): https://aws.amazon.com/rds/pricing/
- S3 (Object Storage): https://aws.amazon.com/s3/pricing/
- AWS RabbitMQ (Amazon MQ for RabbitMQ): https://aws.amazon.com/amazon-mq/pricing/

### Yandex Cloud
- Compute Cloud: https://yandex.cloud/ru/docs/compute/pricing
- Managed PostgreSQL: https://yandex.cloud/ru/docs/managed-postgresql/pricing
- Object Storage: https://yandex.cloud/ru/docs/storage/pricing
- Yandex Cloud Managed Service for RabbitMQ: https://yandex.cloud/ru/docs/managed-rabbitmq/pricing

Цены приведены ориентировочно на основе публичных тарифов (on-demand — по требованию, без резервирования).

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
- database per service на PostgreSQL;
- использование RabbitMQ как брокера событий;
- использование Cache;
- использование Object Storage для медиа и файлов;
- использование NGINX как ingress (входной контур) и балансировщика;
- использование API Gateway как слоя клиентских API.

---

## 3. Оценка стоимости по компонентам

### 3.1 Compute (серверы)

AWS:
- EC2 t3.medium (~$30/мес за инстанс)

Yandex Cloud:
- ВМ среднего уровня (~$25–35/мес)

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

### 3.5 Network и NGINX (ingress и балансировка)

AWS:
- Load Balancer (~$20–50/мес)

Yandex Cloud:
- Балансировщик (~$15–40/мес)

---

### 3.6 API Gateway (шлюз API)

AWS:
- может быть реализован через API Gateway или как отдельные сервисы на EC2 (~$20–100/мес в зависимости от нагрузки)

Yandex Cloud:
- может быть реализован как отдельные сервисы (на Compute Cloud) (~$20–80/мес)

Примечание:
API Gateway может быть реализован как логический слой внутри compute-инстансов, поэтому его стоимость частично входит в Compute.

---

### 3.7 RabbitMQ (брокер сообщений)

AWS:
- Amazon MQ for RabbitMQ (~$50–150/мес)

Yandex Cloud:
- Managed Service for RabbitMQ (~$40–120/мес)

---

## 4. Итоговая стоимость

### Год 1 (MVP)

- Compute: $120
- DB: $100
- Storage: $50
- Cache: $50
- Network: $30
- Monitoring: $50

Итого:
~ $400 / месяц
(включая API Gateway как часть compute-слоя)
~ $4 800 / год
(включая API Gateway как часть compute-слоя)

---

### Год 2

- Compute: $300
- DB: $200
- Storage: $150
- Cache: $100
- Network: $80
- Monitoring: $100

Итого:
~ $930 / месяц
(включая API Gateway как часть compute-слоя)
~ $11 160 / год
(включая API Gateway как часть compute-слоя)

---

### Год 5

- Compute: $1200
- DB: $500
- Storage: $500
- Cache: $200
- Network: $200
- Monitoring: $200

Итого:
~ $2 800 / месяц
(включая API Gateway как часть compute-слоя)
~ $33 600 / год
(включая API Gateway как часть compute-слоя)

---

## 5. Сравнение AWS vs Yandex Cloud

| Компонент | AWS | Yandex Cloud |
|----------|-----|--------------|
| Compute | выше | ниже или сопоставимо |
| PostgreSQL | выше | немного дешевле |
| Object Storage | сопоставимо | сопоставимо |
| Cache | выше | немного дешевле |
| RabbitMQ | выше | немного дешевле или сопоставимо |
| Network + NGINX | выше | дешевле |
| API Gateway | выше (managed сервисы) | ниже (self-hosted) |

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
- использование Object Storage снижает нагрузку на PostgreSQL;
- использование Cache снижает нагрузку на compute и PostgreSQL;
- выделение API Gateway как отдельного слоя повышает управляемость API, но увеличивает потребление compute-ресурсов;

Связанные документы:
- base-architecture.md;
- architecture-views.md;
- ADR-001, ADR-003, ADR-004, ADR-007;
- risk-analysis.md.

---

## 7. Вывод

Стоимость владения системой растёт пропорционально нагрузке, но остаётся управляемой.

Выбранная архитектура обеспечивает баланс между:
- масштабируемостью;
- надёжностью;
- стоимостью.

При необходимости возможна оптимизация за счёт:
- резервирования ресурсов;
- использования более дешёвых инстансов;
- оптимизации хранения данных.
