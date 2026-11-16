# РОЗДІЛ 2. ПРОЄКТУВАННЯ СИСТЕМИ

> Статус: в роботі / виконано базове архітектурне моделювання (етапи 2–3, 01.11.2026 – 20.12.2026).  
> Пов'язані інженерні артефакти: [Специфікація вимог SRS (ISO/IEC/IEEE 29148)](srs.md) | [ADR-001: Вибір брокера повідомлень RabbitMQ](adr/0001-choice-of-technology.md)

---

## 2.1. Контекстне та контейнерне моделювання системи (C4 Model)

Проєктування архітектури платформи виконано за ієрархічною методологією Саймона Брауна (`C4 Model`) із застосуванням підходу `Diagrams-as-Code` (`Mermaid.js`) та наскрізним трасуванням функціональних і нефункціональних вимог зі специфікації [SRS](srs.md).

<a id="diagram-level1"></a>
### 2.1.1. Контекстна діаграма системи (C4 Level 1: System Context Diagram)

На першому рівні деталізації (`System Context`) платформу подано як цілісну програмну систему в оточенні двох категорій користувачів та чотирьох типів зовнішніх систем.

```mermaid
flowchart TB
    cand["<b>Candidate / Job Seeker</b><br/>[Person]<br/>Кандидат або дослідник ІТ-ринку:<br/>пошук вакансій, закладки, аналітика, підписки"]
    admin["<b>System Administrator</b><br/>[Person]<br/>Адміністратор платформи:<br/>керування джерелами, словниками та моніторинг"]

    subgraph Scope["System Boundary: Job Aggregation & Analytics Platform"]
        sys["<b>Event-Driven Job Aggregation & Analytics Platform</b><br/>[Software System: Java 21 / Spring Boot / React]<br/>Автоматизований збір, нормалізація, нечітка дедуплікація,<br/>аналітика ринку праці та реактивні сповіщення"]
    end

    extRest["<b>External Job APIs</b><br/>[External System]<br/>Публічні портали вакансій із<br/>програмним інтерфейсом (JSON)"]
    extRss["<b>External RSS Job Feeds</b><br/>[External System]<br/>Стрічки оголошень ІТ-платформ<br/>у форматі RSS 2.0 / Atom XML"]
    extHtml["<b>External HTML Job Boards</b><br/>[External System]<br/>Веб-сайти вакансій із серверною<br/>HTML-розміткою та robots.txt"]
    tgApi["<b>Telegram Bot API</b><br/>[External System]<br/>Хмарний шлюз месенджера Telegram<br/>для доставки сповіщень"]

    cand -->|"Пошук, фільтрація, перегляд аналітики,<br/>керування підписками [HTTPS / REST / JSON, JWT]"| sys
    admin -->|"Конфігурація джерел, словників синонімів,<br/>перегляд метрик і логів [HTTPS / REST / JSON, RBAC]"| sys

    sys -->|"Опитування за розкладом із Retry / Circuit Breaker<br/>[HTTPS / REST / JSON]"| extRest
    sys -->|"Періодичне завантаження стрічок оголошень<br/>[HTTPS / RSS / XML]"| extRss
    sys -->|"Верифікація robots.txt та парсинг сторінок<br/>[HTTPS / HTML]"| extHtml
    sys -->|"Асинхронна доставка персоналізованих сповіщень<br/>[HTTPS POST / JSON]"| tgApi
    tgApi -.->|"Push-сповіщення у месенджері<br/>[MTProto / HTTPS]"| cand
```

#### Технічний опис вузлів та протоколів рівня System Context (Трасування до SRS)

1. **Candidate / Job Seeker (`[Person]`):** основний користувач системи, який звертається до платформи за протоколом `HTTPS` (`REST / JSON`) із безстановою автентифікацією `JWT` ([SRS-NFR-SEC-01](srs.md#srs-nfr-sec-01)). Отримує доступ до повнотекстового та фасетного пошуку вакансій ([SRS-FUN-07](srs.md#srs-fun-07)), закладок і статусів відгуків ([SRS-FUN-08](srs.md#srs-fun-08)), а також кешованих дашбордів ринкової аналітики ([SRS-FUN-09](srs.md#srs-fun-09)).
2. **System Administrator (`[Person]`):** привілейований користувач із роллю `ROLE_ADMIN` ([SRS-NFR-SEC-01](srs.md#srs-nfr-sec-01)), який через захищений `HTTPS / REST API` керує параметрами конекторів, довідником синонімів, порогом дедуплікації та контролює журнали помилок і телеметрію ([SRS-FUN-10](srs.md#srs-fun-10)).
3. **External Job APIs / RSS Feeds / HTML Job Boards (`[External Systems]`):** гетерогенні зовнішні постачальники даних про вакансії. Взаємодія здійснюється в режимі ізольованого опитування за протоколом `HTTPS` (`JSON`, `XML`, `HTML`) відповідно до вимог [SRS-FUN-01](srs.md#srs-fun-01), з перевіркою директив `robots.txt` ([SRS-FUN-02](srs.md#srs-fun-02)) та обгортанням викликів у декоратори `Retry` і `Circuit Breaker` ([SRS-NFR-REL-01](srs.md#srs-nfr-rel-01)).
4. **Telegram Bot API (`[External System]`):** зовнішній шлюз доставки, до якого платформа надсилає `HTTPS POST` запити з `JSON`-тілом при виявленні вакансій, релевантних підпискам користувача ([SRS-FUN-10](srs.md#srs-fun-10), [SRS-NFR-PERF-03](srs.md#srs-nfr-perf-03)).

---

<a id="diagram-level2"></a>
### 2.1.2. Контейнерна діаграма системи (C4 Level 2: Container Diagram)

На другому рівні моделі `C4` (`Container Diagram`) розкрито внутрішню топологію розгортання платформи у середовищі `Docker` ([SRS-NFR-MAINT-01](srs.md#srs-nfr-maint-01)), що складається з клієнтського застосунку, вхідного шлюзу, чотирьох спеціалізованих серверних сервісів, реляційної СУБД, In-Memory кешу, брокера повідомлень та контуру спостережуваності.

```mermaid
flowchart TB
    cand["<b>Candidate / Job Seeker</b><br/>[Person]"]
    admin["<b>System Administrator</b><br/>[Person]"]

    subgraph Platform["System Boundary: Event-Driven Job Platform (Docker Compose Network)"]
        spa["<b>Web Application (SPA)</b><br/>[Container: React 18, TypeScript, Vite]<br/>Інтерфейс пошуку вакансій, закладок,<br/>аналітичних графіків та адмін-панелі"]
        
        gw["<b>API Gateway / Reverse Proxy</b><br/>[Container: Nginx + Bucket4j Filter]<br/>TLS Termination, маршрутизація запитів,<br/>Rate Limiting (100 req/min/IP)"]

        apiSvc["<b>API Service</b><br/>[Container: Java 21, Spring Boot 3, Spring Security]<br/>REST API, JWT/RBAC, фасетний пошук,<br/>агрегація аналітики, закладки та експорт CSV"]

        ingestSvc["<b>Ingestion Service</b><br/>[Container: Java 21, Spring Boot 3, Resilience4j, Jsoup]<br/>Планувальник збору, конектори Strategy,<br/>перевірка robots.txt, гешування SHA-256"]

        normWorker["<b>Normalization & Dedup Worker</b><br/>[Container: Java 21, Spring Boot 3, Commons Text]<br/>Нормалізація валют і синонімів, двофазна<br/>дедуплікація (Jaro-Winkler, Levenshtein)"]

        notifSvc["<b>Notification Service</b><br/>[Container: Java 21, Spring Boot 3, TelegramBots]<br/>Зіставлення нових вакансій із підписками<br/>та контрольована відправка у Telegram"]

        mq[("<b>Message Broker</b><br/>[Container: RabbitMQ 3.13]<br/>Обмінники подій, черги воркерів<br/>та Dead Letter Queues (DLQ)")]

        db[("<b>Primary Database</b><br/>[Container: PostgreSQL 16 + Flyway]<br/>Канонічні вакансії, посилання на джерела,<br/>словники синонімів, користувачі, індекси pg_trgm")]

        cache[("<b>In-Memory Cache</b><br/>[Container: Redis 7]<br/>Кеш ринкової аналітики (Cache-Aside),<br/>курсів валют, словників та ключів ідемпотентності")]

        obs["<b>Observability Stack</b><br/>[Container: Prometheus + Grafana]<br/>Збір технічних метрик Micrometer,<br/>моніторинг SLA, черг та Circuit Breaker"]
    end

    extSources["<b>External Job Sources</b><br/>[External Systems: REST / RSS / HTML]"]
    tgApi["<b>Telegram Bot API</b><br/>[External System]"]

    cand -->|"Завантаження SPA та виклики API<br/>[HTTPS / REST / JSON, JWT]"| gw
    admin -->|"Адміністрування та перегляд дашбордів<br/>[HTTPS / REST / JSON]"| gw
    gw -->|"Статичні ресурси та клієнтський роутинг<br/>[HTTPS]"| spa
    spa -->|"Асинхронні запити даних<br/>[HTTPS / REST / JSON, Bearer JWT]"| gw
    gw -->|"Проксіювання верифікованих запитів<br/>[HTTP/1.1 / REST / JSON]"| apiSvc

    ingestSvc -->|"Опитування джерел із Retry & Circuit Breaker<br/>[HTTPS / JSON, XML, HTML]"| extSources
    ingestSvc -->|"Перевірка дублікатів content_hash<br/>[RESP / TCP]"| cache
    ingestSvc -->|"Публікація RawVacancyCollectedEvent<br/>[AMQP 0-9-1 / TCP (Publisher Confirms)]"| mq

    mq -->|"Доставка сирих подій (prefetch=10) та DLQ<br/>[AMQP 0-9-1 / TCP]"| normWorker
    normWorker -->|"Читання словників синонімів та курсів валют<br/>[RESP / TCP]"| cache
    normWorker -->|"Блокування кандидатів (pg_trgm) та запис вакансій<br/>[JDBC / TCP (SQL)]"| db
    normWorker -->|"Публікація VacancyConsolidatedEvent<br/>[AMQP 0-9-1 / TCP]"| mq

    mq -->|"Доставка консолідованих подій<br/>[AMQP 0-9-1 / TCP]"| notifSvc
    notifSvc -->|"Вибірка активних підписок користувачів<br/>[JDBC / TCP (SQL)]"| db
    notifSvc -->|"Відправка сповіщень з Rate Limiting<br/>[HTTPS POST / JSON]"| tgApi

    apiSvc -->|"Пошукові вибірки, CRUD закладок та підписок<br/>[JDBC / TCP (SQL)]"| db
    apiSvc -->|"Читання/запис агрегатів аналітики (Cache-Aside)<br/>[RESP / TCP]"| cache

    obs -->|"Періодичний збір метрик (/actuator/prometheus)<br/>[HTTP / TCP]"| apiSvc
    obs -->|"Періодичний збір метрик (/actuator/prometheus)<br/>[HTTP / TCP]"| ingestSvc
    obs -->|"Періодичний збір метрик (/actuator/prometheus)<br/>[HTTP / TCP]"| normWorker
    obs -->|"Періодичний збір метрик (/actuator/prometheus)<br/>[HTTP / TCP]"| notifSvc
```

#### Технічний опис контейнерів та наскрізне трасування вимог

1. **Web Application (`[Container: React 18, TypeScript, Vite]`):** клієнтський односторінковий застосунок (`SPA`), який візуалізує фасетний пошук вакансій, список першоджерел кожної консолідованої вакансії ([SRS-FUN-06](srs.md#srs-fun-06)), особистий кабінет закладок ([SRS-FUN-08](srs.md#srs-fun-08)) та графіки ринкової аналітики ([SRS-FUN-09](srs.md#srs-fun-09)).
2. **API Gateway / Reverse Proxy (`[Container: Nginx + Bucket4j]`):** єдина точка входу з термінацією `TLS` (`HTTPS`), яка забезпечує захист периметра та обмеження інтенсивності запитів `Rate Limiting` (`100 req/min` на IP) згідно з вимогою безпеки [SRS-NFR-SEC-02](srs.md#srs-nfr-sec-02).
3. **Ingestion Service (`[Container: Java 21, Spring Boot 3, Resilience4j, Jsoup]`):** автономний сервіс збору даних, що реалізує функціональні вимоги [SRS-FUN-01](srs.md#srs-fun-01), [SRS-FUN-02](srs.md#srs-fun-02) та [SRS-FUN-03](srs.md#srs-fun-03). Захищає вихідні `HTTPS`-з'єднання автоматами `Circuit Breaker` і `Retry` відповідно до [SRS-NFR-REL-01](srs.md#srs-nfr-rel-01) та публікує події за протоколом `AMQP 0-9-1 / TCP`.
4. **Message Broker (`[Container: RabbitMQ 3.13]`):** центральний вузол асинхронної комунікації, обраний та обґрунтований в архітектурному рішенні [ADR-001](adr/0001-choice-of-technology.md). Забезпечує буферизацію потоків між збором, нормалізацією та сповіщеннями ([SRS-NFR-PERF-03](srs.md#srs-nfr-perf-03)), а також маршрутизацію збійних повідомлень у `Dead Letter Queues` ([SRS-NFR-REL-02](srs.md#srs-nfr-rel-02)).
5. **Normalization & Dedup Worker (`[Container: Java 21, Spring Boot 3, Apache Commons Text]`):** обчислювальний воркер, що реалізує вимоги уніфікації валют ([SRS-FUN-04](srs.md#srs-fun-04)), словникової класифікації атрибутів ([SRS-FUN-05](srs.md#srs-fun-05)) та двофазної нечіткої дедуплікації ([SRS-FUN-06](srs.md#srs-fun-06)) із цільовою точністю `F1 >= 0.88` ([SRS-NFR-QUAL-01](srs.md#srs-nfr-qual-01)).
6. **API Service (`[Container: Java 21, Spring Boot 3, Spring Security]`):** обслуговує синхронні запити користувачів ([SRS-FUN-07](srs.md#srs-fun-07), [SRS-FUN-08](srs.md#srs-fun-08), [SRS-FUN-09](srs.md#srs-fun-09)), забезпечує верифікацію токенів `JWT` та ролей `RBAC` ([SRS-NFR-SEC-01](srs.md#srs-nfr-sec-01)) і витримує цільову затримку `p95 <= 200 мс` при `300 RPS` ([SRS-NFR-PERF-01](srs.md#srs-nfr-perf-01)).
7. **Notification Service (`[Container: Java 21, Spring Boot 3, TelegramBots]`):** споживає події `VacancyConsolidatedEvent` з `RabbitMQ` за протоколом `AMQP 0-9-1 / TCP`, виконує матчинг підписок у `PostgreSQL` (`JDBC / TCP`) та доставляє сповіщення через `Telegram Bot API` ([SRS-FUN-10](srs.md#srs-fun-10), [SRS-NFR-PERF-03](srs.md#srs-nfr-perf-03)).
8. **Primary Database (`[Container: PostgreSQL 16 + Flyway]`):** реляційне сховище канонічних вакансій, джерел `VacancySourceLink`, словників синонімів та підписок. Використовує індекси `B-Tree`, `GIN` та розширення триграмного пошуку `pg_trgm` для виконання вимог [SRS-FUN-06](srs.md#srs-fun-06) і [SRS-NFR-PERF-01](srs.md#srs-nfr-perf-01).
9. **In-Memory Cache (`[Container: Redis 7]`):** високошвидкісне сховище, що взаємодіє з сервісами за протоколом `RESP / TCP` і забезпечує кешування ринкової аналітики за патерном `Cache-Aside` (`p95 <= 50 мс`, [SRS-NFR-PERF-02](srs.md#srs-nfr-perf-02)), зберігання довідників синонімів і ключів ідемпотентності подій ([SRS-NFR-REL-02](srs.md#srs-nfr-rel-02)).
10. **Observability Stack (`[Container: Prometheus + Grafana]`):** інфраструктура телеметрії, що опитує ендпоінти `/actuator/prometheus` усіх Java-контейнерів за протоколом `HTTP / TCP` ([SRS-NFR-MAINT-01](srs.md#srs-nfr-maint-01)).

---

<a id="diagram-level3"></a>
## 2.2. Компонентне моделювання критичного сервісу системи (C4 Model: Component)

Для декомпозиції на третьому рівні моделі `C4` (`Component Diagram`) обрано найбільш алгоритмічно навантажений контейнер бекенду — **Normalization & Dedup Worker** (`Java 21 / Spring Boot 3`). Архітектуру всередині контейнера побудовано за принципами **Clean Architecture** та **SOLID (Dependency Inversion Principle)**: доменні сервіси ізольовані від деталей брокера повідомлень і СУБД через інвертовані інтерфейсні порти (`Inbound / Outbound Ports`).

```mermaid
flowchart TB
    mqIn[("<b>RabbitMQ Broker</b><br/>[External Container: Queue vacancy.raw.queue]")]
    redisExt[("<b>Redis 7 Cache</b><br/>[External Container]")]
    pgExt[("<b>PostgreSQL 16 DB</b><br/>[External Container]")]
    mqOut[("<b>RabbitMQ Broker</b><br/>[External Container: Exchange vacancy.events / DLX]")]
    promExt["<b>Prometheus Server</b><br/>[External Container]"]

    subgraph WorkerBoundary["Container Boundary: Normalization & Dedup Worker (Spring Boot 3 / Java 21)"]
        
        subgraph InboundLayer["Inbound Layer (Presentation / Message Consumers & Guards)"]
            listener["<b>RawVacancyEventListener</b><br/>[Component: @RabbitListener Adapter]<br/>Прийом повідомлень із черги, керування<br/>Manual ACK / NACK"]
            guard["<b>EventValidationAndIdempotencyGuard</b><br/>[Component: Interceptor / Guard]<br/>Валідація DTO (Jakarta Validation) та перевірка<br/>ключа ідемпотентності event_id / content_hash"]
            errHandler["<b>GlobalWorkerExceptionHandler</b><br/>[Component: AOP Error Handler]<br/>Класифікація помилок (Retryable vs Non-Retryable)<br/>та маршрутизація у DLQ після 3 спроб"]
        end

        subgraph CoreLayer["Core Application & Domain Layer (Use Cases & Domain Engines)"]
            orchestrator["<b>VacancyProcessingOrchestrator</b><br/>[Component: Application Use Case Service]<br/>Координація конвеєра нормалізації,<br/>дедуплікації та збереження (@Transactional)"]
            
            salEngine["<b>SalaryNormalizationEngine</b><br/>[Component: Domain Service]<br/>Парсинг зарплатних вилок та уніфікація<br/>у базову валюту USD (salary_min, salary_max)"]
            
            synEngine["<b>SynonymExtractionEngine</b><br/>[Component: Domain Service]<br/>Екстракція технологій, грейду, локації<br/>та формату роботи за SynonymDictionary"]
            
            dedupEngine["<b>TwoPhaseDeduplicationEngine</b><br/>[Component: Domain Service]<br/>Фаза 1: формування критеріїв блокування;<br/>Фаза 2: розрахунок Jaro-Winkler + Levenshtein"]
            
            metricsComp["<b>WorkerTelemetryCollector</b><br/>[Component: Micrometer Aspect]<br/>Реєстрація таймерів обробки, лічильників<br/>дублікатів та розподілу similarity_score"]
        end

        subgraph OutboundLayer["Outbound Layer (Infrastructure Adapters & Repositories)"]
            cacheAdapter["<b>DictionaryAndIdempotencyCacheAdapter</b><br/>[Component: Spring Data Redis Adapter]<br/>Отримання словників синонімів, курсів валют<br/>та атомарна фіксація ключів SETNX"]
            
            jpaAdapter["<b>VacancyPersistenceAdapter</b><br/>[Component: Spring Data JPA Repository]<br/>Пошук кандидатів за індексом pg_trgm (30 діб),<br/>збереження NormalizedVacancy та VacancySourceLink"]
            
            eventPub["<b>ConsolidatedEventPublisher</b><br/>[Component: Spring AMQP Outbound Adapter]<br/>Публікація VacancyConsolidatedEvent<br/>та відправка збійних подій у DLX"]
        end
    end

    mqIn -->|"Доставка RawVacancyCollectedEvent<br/>[AMQP 0-9-1 / TCP]"| listener
    listener -->|"Передача сирого DTO на перевірку<br/>[Method Call / Interceptor]"| guard
    listener -.->|"Перехоплення винятків обробки<br/>[Spring AOP Advice]"| errHandler
    errHandler -->|"Публікація отруйного повідомлення у DLQ<br/>[Interface Call -> Outbound Port]"| eventPub

    guard -->|"Перевірка та блокування event_id<br/>[Interface Call -> CachePort]"| cacheAdapter
    guard -->|"Делегування валідної події в ядро<br/>[Interface Call -> UseCasePort]"| orchestrator

    orchestrator -->|"1. Конвертація валюти та вилки зарплати<br/>[Spring DI / Domain Call]"| salEngine
    orchestrator -->|"2. Класифікація тегів, рівня та локації<br/>[Spring DI / Domain Call]"| synEngine
    orchestrator -->|"3. Пошук та оцінка нечітких дублікатів<br/>[Spring DI / Domain Call]"| dedupEngine
    orchestrator -->|"4. Збереження канонічної вакансії / джерела<br/>[Interface Call -> VacancyRepositoryPort (DIP)]"| jpaAdapter
    orchestrator -->|"5. Емісія події про нову вакансію<br/>[Interface Call -> EventPublisherPort (DIP)]"| eventPub
    orchestrator -.->|"Запис метрик тривалості та результату<br/>[Micrometer API / AOP]"| metricsComp

    salEngine -->|"Запит актуального курсу валют<br/>[Interface Call -> CachePort (DIP)]"| cacheAdapter
    synEngine -->|"Запит кешованого SynonymDictionary<br/>[Interface Call -> CachePort (DIP)]"| cacheAdapter
    dedupEngine -->|"Вибірка блоку кандидатів за 30 діб<br/>[Interface Call -> VacancyRepositoryPort (DIP)]"| jpaAdapter

    cacheAdapter -->|"Команди GET / SETNX / HGETALL<br/>[RESP / TCP]"| redisExt
    jpaAdapter -->|"Параметризовані SQL та pg_trgm запити<br/>[JDBC / TCP]"| pgExt
    eventPub -->|"Публікація подій та DLQ-повідомлень<br/>[AMQP 0-9-1 / TCP]"| mqOut
    promExt -->|"Scrape метрик /actuator/prometheus<br/>[HTTP / TCP]"| metricsComp
```

### Технічний опис компонентів та трасування архітектурних рішень

1. **RawVacancyEventListener (`[Inbound Adapter]`):** слухач черги `vacancy.raw.queue` ([ADR-001](adr/0001-choice-of-technology.md)), який приймає `JSON`-повідомлення `RawVacancyCollectedEvent` ([SRS-FUN-03](srs.md#srs-fun-03)) і керує ручним підтвердженням `Manual ACK` після завершення транзакції.
2. **EventValidationAndIdempotencyGuard (`[Inbound Guard]`):** виконує структурну валідацію вхідного DTO (`Jakarta Bean Validation`) та звертається до порту кешу для атомарної перевірки ключа `event_id` (`Redis SETNX`). Реалізує патерн `Idempotent Consumer` згідно з вимогою надійності [SRS-NFR-REL-02](srs.md#srs-nfr-rel-02).
3. **GlobalWorkerExceptionHandler (`[Inbound Error Handler]`):** централізований обробник помилок (`@RabbitListenerErrorHandler`), який відстежує кількість спроб опрацювання (`x-death` / `RetryContext`) та після `3` невдалих спроб маршрутизує повідомлення через `ConsolidatedEventPublisher` до обмінника `Dead Letter Exchange` (`vacancy.dlx.exchange`), гарантуючи `Zero Message Loss` ([SRS-NFR-REL-02](srs.md#srs-nfr-rel-02), [ADR-001](adr/0001-choice-of-technology.md)).
4. **VacancyProcessingOrchestrator (`[Application Use Case]`):** координатор сценарію обробки вакансії. Взаємодіє з інфраструктурою виключно через абстрактні інтерфейси (`VacancyRepositoryPort`, `CachePort`, `EventPublisherPort`), що забезпечує дотримання принципу інверсії залежностей (`SOLID DIP`) та дозволяє тестувати доменне ядро ізольовано з `Mockito` ([SRS-NFR-MAINT-01](srs.md#srs-nfr-maint-01)).
5. **SalaryNormalizationEngine (`[Domain Service]`):** реалізує вимогу [SRS-FUN-04](srs.md#srs-fun-04) — розпізнає формати запису зарплати (вилки, погодинні та місячні ставки, валюти `USD`, `EUR`, `UAH`) і перераховує їх у `salary_min` та `salary_max` у `USD`.
6. **SynonymExtractionEngine (`[Domain Service]`):** реалізує вимогу [SRS-FUN-05](srs.md#srs-fun-05) — здійснює токенізацію заголовка й опису та нормалізує технології, грейд кандидата, місто і формат роботи за довідником `SynonymDictionary`.
7. **TwoPhaseDeduplicationEngine (`[Domain Service]`):** реалізує ключову вимогу [SRS-FUN-06](srs.md#srs-fun-06) та забезпечує метрики якості дедуплікації `Precision >= 0.90`, `Recall >= 0.85`, `F1 >= 0.88` ([SRS-NFR-QUAL-01](srs.md#srs-nfr-qual-01)). На першій фазі запитує через `VacancyRepositoryPort` вузький набір кандидатів за останні `30 діб`, на другій — обчислює зважену суму схожості `Jaro-Winkler` та нормалізованої відстані `Levenshtein`.
8. **WorkerTelemetryCollector (`[Observability Component]`):** публікує технічні метрики швидкодії конвеєра (`vacancy_normalization_duration_seconds`, `vacancy_dedup_matches_total`) через `Micrometer` для моніторингу виконання вимоги [SRS-NFR-PERF-03](srs.md#srs-nfr-perf-03) та [SRS-NFR-MAINT-01](srs.md#srs-nfr-maint-01).
9. **VacancyPersistenceAdapter, DictionaryAndIdempotencyCacheAdapter, ConsolidatedEventPublisher (`[Outbound Infrastructure Adapters]`):** конкретні імплементації вихідних портів доменного ядра для роботи з `PostgreSQL` (`Spring Data JPA`), `Redis` (`Spring Data Redis`) та `RabbitMQ` (`Spring AMQP`).

---

## 2.3. Документування архітектурних рішень (Architecture Decision Records)

Для фіксації інженерних компромісів (`Architectural Trade-offs`), прийнятих під час проєктування розподіленої платформи, у репозиторії розгорнуто реєстр архітектурних рішень у каталозі [`docs/adr/`](adr/) за шаблоном Майкла Нігардза.

Таблиця 2.1 – Реєстр архітектурних рішень (ADR Log)

| ID рішення | Назва документа | Статус | Пов'язані вимоги SRS | Ключовий компроміс (Trade-off Summary) |
| :--- | :--- | :--- | :--- | :--- |
| **[ADR-001](adr/0001-choice-of-technology.md)** | [Вибір брокера повідомлень RabbitMQ для організації подієво-орієнтованого конвеєра обробки вакансій](adr/0001-choice-of-technology.md) | `Approved` | [SRS-FUN-03](srs.md#srs-fun-03), [SRS-FUN-10](srs.md#srs-fun-10), [SRS-NFR-PERF-03](srs.md#srs-nfr-perf-03), [SRS-NFR-REL-02](srs.md#srs-nfr-rel-02) | Обрано `RabbitMQ` замість `Apache Kafka` та БД-черги: отримано нативну підтримку поштучного `ACK/NACK`, маршрутизації у `DLQ`, балансування `prefetchCount` для нерівномірної дедуплікації та зниження витрат `RAM` на `> 1.2 ГБ` ціною відмови від довгострокового `Event Replay` на рівні брокера та необхідності реалізації `Idempotent Consumer`. |
