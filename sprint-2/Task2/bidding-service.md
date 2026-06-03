# Проектирование сервиса ставок

## 1. Назначение сервиса

Bidding Service отвечает за обработку bid request от DSP-партнёра и выбор рекламного объявления для показа пользователю.

Сервис находится в latency-critical path, поэтому его основная цель — быстро принять решение по ставке и вернуть bid response в пределах SLA.

Целевые требования:
- поддержка OpenRTB bid request / bid response;
- P95 latency ≤ 100 ms;
- отсутствие деградации при росте до 18 000 RPS;
- горизонтальное масштабирование;
- минимальная зависимость от медленных downstream-сервисов.

---

## 2. Границы сервиса

Bidding Service включает:
- приём и валидацию bid request;
- подбор подходящих кампаний;
- применение targeting rules;
- расчёт ставки;
- выбор победителя аукциона;
- формирование bid response;
- публикацию событий аукциона.

Bidding Service не отвечает за:
- управление рекламными кампаниями;
- пополнение баланса;
- построение отчётов;
- долгую аналитическую обработку;
- синхронную запись статистики показов и кликов.

---

## 3. API сервиса

### `POST /openrtb/bid`

Основной endpoint для DSP-интеграции.

**Request:**

```json
{
  "id": "request-123",
  "imp": [
    {
      "id": "imp-1",
      "banner": {
        "w": 300,
        "h": 250
      }
    }
  ],
  "site": {
    "domain": "example.com"
  },
  "user": {
    "id": "user-456"
  },
  "device": {
    "ip": "192.0.2.1",
    "ua": "Mozilla/5.0"
  }
}
```

**Response:**
```json
{
  "id": "request-123",
  "seatbid": [
    {
      "bid": [
        {
          "id": "bid-789",
          "impid": "imp-1",
          "price": 1.25,
          "adm": "<script>...</script>",
          "crid": "creative-123",
          "cid": "campaign-123"
        }
      ]
    }
  ]
}
```

Если подходящей рекламы нет, сервис возвращает 204 No Content.

## 4. Зависимости

|Зависимость | Тип | Назначение |
|:----|:----:|:----|
|Campaign Service | sync / cache-backed | Получение данных кампаний, ставок и targeting rules |
|Redis / Cache | sync | Быстрый доступ к активным кампаниям и правилам | 
|Kafka / Event Broker | async | Публикация auction, impression и click events |
|Billing Service | async / eventual consistency | Списание бюджета и финансовые события |
|Statistics Service | async | Обработка статистики показов и кликов |
|Observability Stack | async | Метрики latency, RPS, error rate, timeout rate |

В hot path сервис должен опираться преимущественно на кэш, а не на прямые обращения к основной PostgreSQL-базе.

## 5. Модель данных

Bidding Service не должен владеть полным master-data рекламных кампаний. Источником истины остаётся Campaign Service.

Внутри Bidding Service используется оптимизированное read-модель / cache view.

### Основные сущности

| Сущность | Назначение |
|----|----|
| CampaignView | Активная кампания, доступная для участия в аукционе |
| CreativeView | Креативы, доступные для показа |
| TargetingRuleView | Правила таргетинга |
| BidRuleView | Правила расчёта ставки |
| BudgetSnapshot | Снимок доступного бюджета |
| AuctionEvent | Событие результата аукциона |

Пример read-модели
```json
{
  "campaign_id": "campaign-123",
  "status": "ACTIVE",
  "bid": 1.25,
  "targeting": {
    "geo": ["US"],
    "device": ["mobile"],
    "banner_sizes": ["300x250"]
  },
  "creative_ids": ["creative-123"],
  "budget_available": true
}
```

## 6. Поток обработки bid request

```text
DSP Partner
  -> Bidding Service
  -> validate request
  -> read active campaigns from cache
  -> apply targeting rules
  -> calculate bid
  -> build bid response
  -> publish auction event
  -> return response
```

Синхронный путь должен быть минимальным. Запись событий и статистики выполняется асинхронно через брокер сообщений.

## 7. Основные архитектурные решения
Bidding Service выделяется как отдельный latency-critical компонент.
Для данных кампаний используется кэш/read-модель.
PostgreSQL не используется напрямую в hot path, кроме fallback-сценариев.
События аукциона публикуются асинхронно.
Финансовые и аналитические операции не блокируют bid response.
Сервис масштабируется горизонтально за счёт stateless runtime.