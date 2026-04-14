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
- Event Broker — событийное взаимодействие между доменами.

### Основные функциональные потоки

1. Регистрация пользователя:
   Auth → Profile

2. Запись тренировки:
   Device Integration → Training → Event Broker → Analytics / Recommendation

3. Публикация поста:
   Social → Training → Event Broker → Notification / Analytics

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
    WorkoutEvent --> Analytics[Analytics Domain]
    WorkoutEvent --> Recommendation[Recommendation Domain]

    Post --> Social[Social Domain]
    Social --> Training
    Social --> PostEvent[Post Event]
    PostEvent --> Notification[Notification Domain]
    PostEvent --> Analytics

    Reco --> Analytics
    Reco --> Training
    Reco --> Commerce[Commerce Integration Domain]
    Analytics --> Recommendation
    Training --> Recommendation
    Commerce --> Recommendation
```

Транзакционные пользовательские действия завершаются внутри соответствующих доменов (например, Auth, Profile, Training), после чего инициируются асинхронные события. Обработка таких событий выполняется через Event Broker и может приводить к параллельной обработке в нескольких доменах (например, Analytics и Recommendation).

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

### Распределение данных по доменам

| Домен | Основные сущности | Тип данных | Критичность | Репликация |
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

### Принципы работы с данными

- каждый домен имеет собственное хранилище данных;
- обмен данными происходит через события;
- используется eventual consistency (согласованность с задержкой);
- критичные данные (Auth, Profile, Training) реплицируются между регионами;
- аналитические данные реплицируются асинхронно;
- данные синхронизируются через события, а не через прямой доступ.

### Потоки данных


```mermaid
flowchart LR
    DeviceData --> DeviceIntegration
    DeviceIntegration --> Training
    Training --> EventBroker
    EventBroker --> Analytics
    EventBroker --> Recommendation
```

### Общая схема информационных потоков

```mermaid
flowchart LR
    UserData[User / Auth Data] --> Auth[Auth Domain]
    Auth --> Profile[Profile Domain]

    DeviceData[Device Data] --> DeviceIntegration[Device Integration Domain]
    DeviceIntegration --> Training[Training Domain]

    Training --> EventBroker[Event Broker]
    SocialPost[Social Post Data] --> Social[Social Domain]
    Social --> EventBroker

    EventBroker --> Analytics[Analytics Domain]
    EventBroker --> Recommendation[Recommendation Domain]

    ExternalCommerce[External Commerce Systems / Promo Sources] --> Commerce[Commerce Integration Domain]
    Commerce --> Recommendation

    Analytics --> Recommendation
    Training --> Recommendation
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
- тяжёлые операции выполняются асинхронно через Event Broker;
- домены обрабатывают события независимо друг от друга;
- используется idempotency (идемпотентность — повторяемость без побочных эффектов);
- используется retry (повторные попытки выполнения);

### Поток выполнения

```mermaid
sequenceDiagram
    participant User
    participant API
    participant Training
    participant Broker
    participant Analytics

    User->>API: Создание тренировки
    API->>Training: Сохранение данных
    Training->>Broker: Публикация события
    Broker->>Analytics: Асинхронная обработка
```

### Параллельная обработка событий

```mermaid
flowchart LR
    Event[Event Broker]
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

- Geo Routing — маршрутизация пользователей;
- API Gateway — точка входа в систему;
- доменные сервисы;
- Event Broker;
- базы данных;
- кэш;
- системы мониторинга и логирования.

### Принципы

- active-active между регионами;
- failover внутри региона;
- репликация данных между регионами;
- изоляция доменов;
- масштабируемость.

### Упрощённая схема

```mermaid
flowchart TB
    User --> Routing[Geo Routing]
    Device --> DeviceIngress[Device Integration Entry]

    Routing --> Region1
    Routing --> Region2

    subgraph Region1
        DC1[Primary DC]
        DC2[Standby DC]
        DC1 -. failover .-> DC2
    end

    subgraph Region2
        DC3[Primary DC]
        DC4[Standby DC]
        DC3 -. failover .-> DC4
    end

    DC1 --> Broker1[Event Broker]
    DC3 --> Broker2[Event Broker]

    Broker1 <-. sync .-> Broker2

    DC1 --> DB1[(Databases)]
    DC3 --> DB2[(Databases)]

    DB1 <-. replication .-> DB2

    DeviceIngress --> DC1
    DeviceIngress --> DC3
```

### Связанные ADR и документы

- ADR-001 — Архитектурный стиль;
- ADR-003 — Интеграционный стиль;
- ADR-007 — Multi-region и отказоустойчивость;
- base-architecture.md.

---

## 5. Представление безопасности

Представление безопасности описывает подходы к защите системы и данных.

### Основные принципы

- все внешние взаимодействия проходят через защищённые протоколы (HTTPS/TLS);
- запрещено использование небезопасных протоколов извне;
- используется централизованная аутентификация (Auth Domain);
- реализована авторизация (RBAC — ролевая модель);
- данные шифруются при передаче и хранении;
- выполняется валидация входных данных;
- применяется rate limiting (ограничение частоты запросов);
- используется защита от повторных запросов (idempotency).

### Разделение доступа

- пользовательский доступ — через API Gateway;
- внутренние сервисы — через защищённый контур;
- устройства — через Device Integration Domain.

### Защита данных

- критичные данные реплицируются;
- используется резервное копирование;
- ведётся аудит действий пользователей;
- логируются события безопасности.

### Угрозы и защита

- защита от brute force атак;
- защита от replay атак;
- защита от DoS;
- изоляция доменов.

### Границы доверия

```mermaid
flowchart TB
    User --> Gateway[API Gateway]
    Device --> DeviceIntegration

    Gateway --> Internal[Internal Services]
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
    end
```

### Связанные ADR и документы

- ADR-006 — Стратегия безопасности;
- base-architecture.md;
- use-cases (01–05).
