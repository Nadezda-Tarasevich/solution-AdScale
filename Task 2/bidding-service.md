# Выделение сервиса ставок (Auction Service) из монолита AdScale

#### 

##### Контекст

Cервис отвечает исключительно за логику аукциона и расчёт ставок. Он не занимается записью событий, финансовыми транзакциями или генерацией контента. Все внешние зависимости (правила, бюджеты) запрашиваются через асинхронные события или кэшированные данные, а не через прямые вызовы других сервисов.

Описание границ сервиса:

* Приём и обработку bid requests
* Исполнение аукциона (выбор победителя)
* Определение цены и проверка бюджета (first/second price In-memory кэш)
* Подписка на изменения бюджетов из Kafka
* Публикация результатов аукциона в Kafka
* Чтение правил и лимитов из кэша (Redis).
* Хранение истории аукционов, ставки, результаты (Auction DB, PostgreSQL (выделенная))

##### 

##### Зависимости от других сервисов



|Зависимость|Тип|Направление|Описание|
|-|-|-|-|
|Ad Server → Auction|Синхронная (gRPC)|Ad Server вызывает Auction|Передаёт кандидатов, получает победителя|
|Auction → Finance|Асинхронная (Kafka)|Finance публикует → Auction читает|Аuction НЕ вызывает Finance синхронно — использует in-memory кэш, обновляемый через Kafka|
|Auction → Delivery|Синхронная (gRPC)|Auction вызывает Delivery|Передаёт победителя для формирования разметки|
|Auction → Statistics|Асинхронная (Kafka)|Auction публикует → Statistics читает|Публикация результатов аукциона для статистики|

Правила:

* Auction НЕ вызывает Finance синхронно — использует in-memory кэш, обновляемый через Kafka
* Auction НЕ имеет прямого доступа к БД других сервисов
* Auction НЕ зависит от состояния других сервисов для принятия решения



##### Методы API (gRPC)

|метод|тип|назначение|вызывающий|критический путь|таймаут|
|-|-|-|-|-|-|
|PlaceBid|Unary|Основной аукцион — принять решение о показе|Ad Server|Да (каждый запрос)|100-200мс|
|GetAuctionStatus|Unary|Получить статус завершённого аукциона|Ad Server (fallback), Мониторинг|Нет (редко)|500 мс|
|AuctionResultEvent<br />|unary|Публикует результат|Bidding Service|Нет (async)|3 сек|
|BudgetChangedEvent — Kafka, асинхронный (потребляется от Finance)|Unary|Публикует факт изменения бюджета и сверяет|Bidding Service|Нет (async)|3 сек|
|SyncCampaigns|Server streaming|Синхронизация кэша кампаний|Bidding Service (сам себе)|Нет (фоновый)|30 сек (долгий)|
|NotifyWin|Unary|Уведомить выигравшего DSP/рекламодателя|Bidding Service (внутренний)|Нет (async)|1 сек|



Unary: Request → Response

Server streaming: Request → stream Response



###### Привет метода PlaceBid

service AuctionService {

&#x20;      rpc PlaceBid(PlaceBidRequest) returns (PlaceBidResponse);

&#x20;   rpc GetAuctionStatus(GetAuctionStatusRequest) returns (GetAuctionStatusResponse);

}



message PlaceBidRequest {

&#x20;   string request\_id = 1;          // Уникальный ID запроса (для идемпотентности)

&#x20;   string user\_id = 2;

&#x20;   string session\_id = 3;

&#x20;   repeated Candidate candidates = 4;  // Кандидаты от Ad Server

&#x20;   string ad\_slot\_id = 5;

&#x20;   string device\_type = 6;

&#x20;   string geo\_country = 7;

&#x20;   int64 timestamp = 8;

}



message Candidate {

&#x20;   string campaign\_id = 1;

&#x20;   double bid\_amount = 2;

&#x20;   string ad\_format = 3;

&#x20;   map<string, string> targeting\_data = 4;

}



message PlaceBidResponse {

&#x20;   string auction\_id = 1;

&#x20;   bool won = 2;

&#x20;   string campaign\_id = 3;

&#x20;   double win\_price = 4;

&#x20;   double processing\_time\_ms = 5;

&#x20;   string error = 6;

}





#### Модель данных Bidding Service (Auction Service)

|**Таблица**|**Назначение**|**Ключевые поля**|**Индексы**|
|-|-|-|-|
|campaigns|Хранит информацию о рекламных кампаниях. Данные кэшируются в In-Memory Cache при старте сервиса и обновляются через Kafka.|campaign\_id, budget\_remaining, targeting\_rules|status, advertiser, dates|
|auctions|Запись каждого проведённого аукциона. Пишется после обработки каждого bid request.|auction\_id, request\_id, campaign\_id, win\_price|created, campaign, user, request|
|bid\_history|Детали всех ставок в аукционе|id, auction\_id, campaign\_id, bid\_amount, rank|auction, campaign|
|budget\_transactions|Аудит изменений бюджета|id, campaign\_id, amount, idempotency\_key|campaign, auction, type|
|auction\_rules|Бизнес-правила, применяемые при проведении аукционов.|rule\_id, rule\_type, rule\_config|type, enabled|
|cache\_state|Мониторинг состояния In-Memory Cache Auction Service.|instance\_id, cache\_type, hit\_rate|instance|
|idempotency\_keys|Идемпотентность фин. операций: гарантирует, что финансовая операция выполняется только один раз|idempotency\_key, status, campaign\_id|status, campaign|



