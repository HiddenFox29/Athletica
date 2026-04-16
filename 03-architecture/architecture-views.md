# Основные представления архитектуры

В данном документе представлены основные архитектурные представления системы Athletica, позволяющие описать систему с разных точек зрения:

- функциональной;
- информационной;
- многозадачности (concurrency — параллельное выполнение);
- инфраструктурной;
- безопасности.

Эти представления дополняют базовую архитектуру и обеспечивают целостное понимание системы.

---

## 1. Функциональное представление

Функциональное представление описывает систему с точки зрения пользовательских и бизнес-функций, а также распределения ответственности между доменами.

Система Athletica построена на основе доменного разбиения (domain-driven decomposition — разделение по бизнес-доменам). Функциональное представление фокусируется прежде всего на доменах, а также на ключевом инфраструктурном компоненте событийного взаимодействия:

Система Athletica построена на основе доменного разбиения (domain-driven decomposition — разделение по бизнес-доменам) и включает следующие основные домены и инфраструктурные компоненты:

- Auth Domain — регистрация, аутентификация и управление доступом;
- Profile Domain — управление пользовательскими профилями;
- Training Domain — хранение и обработка тренировок;
- Social Domain — публикация и взаимодействие пользователей;
- Challenge and Gamification Domain — челленджи, достижения и мотивация;
- Recommendation Domain — персонализированные рекомендации;
- Notification Domain — уведомления пользователей;
- Analytics Domain — сбор и анализ данных;
- Commerce Integration Domain — интеграция с внешними источниками контента и промоакций;
- Device Integration Domain — приём и обработка данных от устройств;
- Event Broker (RabbitMQ — брокер сообщений) — событийное взаимодействие между доменами.

Другие инфраструктурные компоненты системы, такие как Databases, Object Storage, Cache и Monitoring / Logging / Tracing, подробно раскрываются в информационном, инфраструктурном и security представлениях и не детализируются в функциональном представлении как самостоятельные бизнес-домены.

### Основные функциональные потоки

1. Регистрация пользователя:
   Auth → Profile

2. Запись тренировки:
   Device Integration → Training → Event Broker (RabbitMQ) → Analytics / Recommendation

3. Публикация поста:
   Social → Training → Event Broker (RabbitMQ) → Notification / Analytics

4. Генерация рекомендаций:
   Analytics + Training + Commerce → Recommendation


### Схема основных функциональных потоков

```mermaid
flowchart TB
    Reg[Регистрация пользователя]
    Workout[Запись тренировки]
    Post[Публикация поста]
    Reco[Генерация рекомендаций]

    Reg --> Auth[Auth Domain]
    Auth --> Profile[Profile Domain]

    Workout --> DeviceIntegration[Device Integration Domain]
    DeviceIntegration --> Training[Training Domain]
    Training --> WorkoutEvent[Workout Event]
    WorkoutEvent --> RabbitMQ
    RabbitMQ --> Analytics[Analytics Domain]
    RabbitMQ --> Recommendation[Recommendation Domain]

    Post --> Social[Social Domain]
    Social --> Training
    Social --> PostEvent[Post Event]
    PostEvent --> RabbitMQ
    RabbitMQ --> Notification[Notification Domain]
    RabbitMQ --> Analytics

    Reco --> Analytics
    Reco --> Training
    Reco --> Commerce[Commerce Integration Domain]
    Analytics --> Recommendation
    Training --> Recommendation
    Commerce --> Recommendation
```

Транзакционные пользовательские действия завершаются внутри соответствующих доменов (например, Auth, Profile, Training), после чего инициируются асинхронные события. Обработка таких событий выполняется через Event Broker (RabbitMQ) и может приводить к параллельной обработке в нескольких доменах (например, Analytics и Recommendation).

### Связанные ADR и документы

- ADR-001 — Архитектурный стиль;
- ADR-002 — Декомпозиция на домены;
- ADR-003 — Интеграционный стиль;
- use-cases (01–05);
- base-architecture.md.

---

## 2. Информационное представление

Информационное представление описывает структуру данных и их распределение между доменами.

Примечание: в диаграммах используются обозначения внешних источников данных, которые не входят в границы системы.

Каждый домен владеет своими данными (data ownership — владение данными) и не имеет прямого доступа к базам данных других доменов (no cross-DB access).

### Основные сущности системы

- User — пользователь;
- Profile — профиль пользователя;
- Training — тренировка;
- Social Post — публикация;
- Challenge — челлендж;
- Recommendation — рекомендация;
- Notification — уведомление;
- Device Data — данные устройства;
- Commerce Content — внешний контент и промоакции.
- Media File — медиа-файл;
- Backup Object — резервная копия или объект хранения;
- Cache Entry — кэшированная запись;

### Распределение данных по доменам

| Домен / компонент | Основные сущности | Тип данных | Критичность | Репликация |
|------|------------------|-----------|-------------|------------|
| Auth Domain | User / Auth Data | транзакционные | высокая | синхронная / near real-time |
| Profile Domain | Profile | транзакционные | высокая | синхронная / near real-time |
| Training Domain | Training | транзакционные | высокая | синхронная / near real-time |
| Social Domain | Social Post | транзакционные | средняя | асинхронная |
| Recommendation Domain | Recommendation | производные | средняя | асинхронная |
| Analytics Domain | Analytics Data | аналитические | низкая | асинхронная |
| Commerce Integration Domain | Commerce Content | интеграционные | средняя | асинхронная |
| Notification Domain | Notification | транзакционные | средняя | асинхронная |
| Device Integration Domain | Device Data | интеграционные | средняя | асинхронная |
| Challenge and Gamification Domain | Challenge | транзакционные | средняя | асинхронная |
| PostgreSQL | транзакционные данные доменов | транзакционные | высокая | межрегиональная репликация |
| Object Storage (S3-compatible) | Media File / Backup Object | объектные | средняя | асинхронная / резервное копирование |
| Cache (кэш) | Cache Entry | кэшированные | низкая | не требуется |

### Принципы работы с данными

- каждый домен имеет собственное хранилище данных;
- обмен данными происходит через события;
- используется eventual consistency (согласованность с задержкой);
- критичные данные (Auth, Profile, Training) реплицируются между регионами;
- аналитические данные реплицируются асинхронно;
- данные синхронизируются через события, а не через прямой доступ.
- транзакционные данные хранятся в отдельных PostgreSQL (реляционная СУБД — система управления базами данных);
- медиа-данные, файлы и резервные копии хранятся в Object Storage;
- часто читаемые данные могут кэшироваться в Cache;

### Потоки данных


```mermaid
flowchart LR
    DeviceData --> DeviceIntegration
    DeviceIntegration --> Training
    Training --> RabbitMQ
    RabbitMQ --> Analytics
    RabbitMQ --> Recommendation
```

### Общая схема информационных потоков

```mermaid
flowchart LR
    UserData[User / Auth Data] --> Auth[Auth Domain]
    Auth --> Profile[Profile Domain]
    Auth --> PostgreSQL[PostgreSQL]
    Profile --> PostgreSQL

    DeviceData[Device Data] --> DeviceIntegration[Device Integration Domain]
    DeviceIntegration --> Training[Training Domain]
    Training --> PostgreSQL

    SocialPost[Social Post Data] --> Social[Social Domain]
    Social --> PostgreSQL
    Social --> ObjectStorage[Object Storage S3]
    Social --> RabbitMQ[RabbitMQ]
    Social --> Training

    Training --> RabbitMQ
    Training --> ObjectStorage

    RabbitMQ --> Analytics[Analytics Domain]
    RabbitMQ --> Recommendation[Recommendation Domain]
    RabbitMQ --> Notification[Notification Domain]

    Analytics --> PostgreSQL
    Analytics --> ObjectStorage
    Analytics --> Recommendation

    ExternalCommerce[External Commerce Systems / Promo Sources] --> Commerce[Commerce Integration Domain]
    Commerce --> PostgreSQL
    Commerce --> Recommendation

    Recommendation --> PostgreSQL
    Recommendation --> Cache[(Cache)]
    Profile --> Cache
```

### Связанные ADR и документы

- ADR-004 — Стратегия хранения данных;
- ADR-003 — Интеграционный стиль;
- base-architecture.md;
- use-cases (02, 03, 04).

---

## 3. Многозадачность (Concurrency)

Представление многозадачности описывает параллельное выполнение процессов и взаимодействие компонентов.

Система использует смешанную модель взаимодействия:

- синхронные запросы (REST API) — для пользовательских операций;
- асинхронные события — для междоменного взаимодействия.

### Основные принципы

- обработка пользовательских запросов выполняется синхронно;
- тяжёлые операции выполняются асинхронно через RabbitMQ (брокер сообщений);
- домены обрабатывают события независимо друг от друга;
- используется idempotency (идемпотентность — повторяемость без побочных эффектов);
- используется retry (повторные попытки выполнения);
- для ускорения чтения может использоваться Cache, не влияющий на транзакционную консистентность доменов;

### Поток выполнения

```mermaid
sequenceDiagram
    participant User
    participant NGINX
    participant APIGateway
    participant Training
    participant RabbitMQ
    participant Analytics

    User->>NGINX: HTTP request
    NGINX->>APIGateway: Route request
    APIGateway->>Training: Сохранение данных
    Training->>RabbitMQ: Публикация события
    RabbitMQ->>Analytics: Асинхронная обработка
```

### Параллельная обработка событий

```mermaid
flowchart LR
    Event[RabbitMQ брокер сообщений]
    Event --> Analytics[Analytics Domain]
    Event --> Recommendation[Recommendation Domain]
    Event --> Notification[Notification Domain]
```

### Параллелизм

- события обрабатываются параллельно;
- домены масштабируются независимо;
- используется горизонтальное масштабирование (horizontal scaling — масштабирование через добавление экземпляров).

### Связанные ADR и документы

- ADR-003 — Интеграционный стиль;
- base-architecture.md;
- use-cases (02, 03, 04).

---

## 4. Инфраструктурное представление

Инфраструктурное представление описывает физическое размещение системы.

Система построена по multi-region модели:

- два географически независимых региона;
- в каждом регионе:
  - основной ЦОД;
  - резервный ЦОД (standby).

### Основные компоненты инфраструктуры

- Geo Routing / DNS layer — межрегиональная маршрутизация пользователей;
- NGINX — ingress (входной контур), reverse proxy (обратный прокси) и балансировщик нагрузки;
- API Gateway — слой публикации и маршрутизации клиентских API (mobile и web);
- Device Integration Domain — обработка интеграционного трафика устройств и партнёров;
- доменные сервисы;
- RabbitMQ (брокер сообщений);
- PostgreSQL;
- Object Storage (S3-compatible);
- Cache;
- Monitoring / Logging / Tracing.

### Принципы

- active-active между регионами;
- failover внутри региона;
- репликация данных между регионами;
- изоляция доменов;
- масштабируемость.

### Упрощённая схема

```mermaid
flowchart TB
    User --> Routing[Geo Routing / DNS]
    Device --> Routing

    Routing --> NGINX1[NGINX Region 1]
    Routing --> NGINX2[NGINX Region 2]

    NGINX1 --> APIGW1[API Gateway Region 1]
    NGINX2 --> APIGW2[API Gateway Region 2]

    NGINX1 --> DeviceIntegration1[Device Integration Domain]
    NGINX2 --> DeviceIntegration2[Device Integration Domain]

    subgraph Region1[Region 1]
        DC1[Primary DC]
        DC2[Standby DC]
        DC1 -. failover .-> DC2
        RabbitMQ1[RabbitMQ]
        DB1[PostgreSQL]
        S31[Object Storage S3]
        Cache1[Cache]
        Obs1[Monitoring / Logging / Tracing]
    end

    subgraph Region2[Region 2]
        DC3[Primary DC]
        DC4[Standby DC]
        DC3 -. failover .-> DC4
        RabbitMQ2[RabbitMQ]
        DB2[PostgreSQL]
        S32[Object Storage S3]
        Cache2[Cache]
        Obs2[Monitoring / Logging / Tracing]
    end

    RabbitMQ1 <-. sync .-> RabbitMQ2
    DB1 <-. replication .-> DB2
    S31 <-. backup / replication .-> S32
```

### Связанные ADR и документы

- ADR-001 — Архитектурный стиль;
- ADR-003 — Интеграционный стиль;
- ADR-007 — Multi-region и отказоустойчивость;
- base-architecture.md.
- base-architecture.md (разделы по Databases, Object Storage, Cache и Monitoring / Logging / Tracing).

---

## 5. Представление безопасности

Представление безопасности описывает подходы к защите системы и данных.

### Основные принципы

- все внешние взаимодействия проходят через защищённые протоколы (HTTPS/TLS);
- запрещено использование небезопасных протоколов извне;
- используется централизованная аутентификация (Auth Domain);
- реализована авторизация (RBAC — ролевая модель);
- данные шифруются при передаче и хранении;
- доступ к PostgreSQL, Object Storage и Cache ограничивается внутренним доверенным контуром;
- доступ к объектам в Object Storage предоставляется только через контролируемые механизмы доступа;
- выполняется валидация входных данных;
- применяется rate limiting (ограничение частоты запросов);
- используется защита от повторных запросов (idempotency).

### Разделение доступа

- пользовательский доступ — через NGINX и API Gateway;
- внутренние сервисы — через защищённый контур;
- устройства — через Device Integration Domain.

### Защита данных

- критичные данные реплицируются;
- используется резервное копирование;
- ведётся аудит действий пользователей;
- логируются события безопасности.
- доступ к инфраструктурным компонентам хранения и наблюдаемости изолирован от внешнего контура.

### Угрозы и защита

- защита от brute force атак;
- защита от replay атак;
- защита от DoS;
- изоляция доменов.

### Границы доверия

```mermaid
flowchart TB
    User --> Gateway[NGINX]
    Gateway --> APIGateway[API Gateway]
    Device --> Gateway
    Gateway --> DeviceIntegration[Device Integration Domain]

    Gateway --> Internal[Internal Trusted Zone]
    DeviceIntegration --> Internal

    subgraph Internal
        Auth
        Profile
        Training
        Social
        Recommendation
        Notification
        Analytics
        Commerce
        Challenge
        DB[PostgreSQL]
        S3[Object Storage]
        Cache[Cache]
        Obs[Monitoring / Logging / Tracing]
    end
```

### Связанные ADR и документы

- ADR-006 — Стратегия безопасности;
- base-architecture.md;
- use-cases (01–05).
- base-architecture.md (разделы по инфраструктурным компонентам и trust boundaries).
