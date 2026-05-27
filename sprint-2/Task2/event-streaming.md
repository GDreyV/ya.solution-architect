# Потоковая обработка событий

## Цель

Kafka используется для отделения RTB hot path от записи статистики, аналитики и биллинговых операций. Bidding Service быстро публикует событие и не ждёт записи в downstream-хранилища.

---

## Топики Kafka

| Топик | События | Ключ партиционирования | Хранение |
|---|---|---|---|
| `auction-events` | результат аукциона, no-bid, timeout | `auction_id` | 7 дней |
| `impression-events` | показы рекламы | `campaign_id` | 7–14 дней |
| `click-events` | клики по рекламе | `campaign_id` | 14 дней |
| `billing-events` | события для списания бюджета | `advertiser_id` / `campaign_id` | 30 дней |
| `campaign-events` | изменения кампаний и ставок | `campaign_id` | 7 дней |
| `dead-letter-events` | некорректные или необработанные события | original key | 30 дней |

---

## Формат событий

Для production-сценария предпочтительнее **Avro + Schema Registry**.

Причины:
- строгая схема;
- контроль совместимости;
- компактный бинарный формат;
- безопасная эволюция контрактов.

JSON можно использовать на первом этапе для простоты отладки, но целевой вариант — Avro.

---

## Пример схемы события показа

```json
{
  "event_id": "evt-123",
  "event_type": "impression",
  "event_time": "2026-05-27T10:15:30Z",
  "auction_id": "auc-456",
  "campaign_id": "cmp-789",
  "creative_id": "crt-001",
  "advertiser_id": "adv-001",
  "price": 1.25,
  "user_id": "usr-123",
  "placement": {
    "site": "example.com",
    "width": 300,
    "height": 250
  }
}
```

## Группы потребителей
| Consumer group | Читает топики | Назначение |
|----|----|----|
| statistics-consumers | auction-events, impression-events, click-events | Запись статистики и агрегатов |
| billing-consumers | billing-events, impression-events | Списание бюджета и финансовые операции |
| analytics-consumers | auction-events, impression-events, click-events | Загрузка данных в ClickHouse |
| cache-updaters | campaign-events | Обновление Redis/read-модели |
| fraud-consumers | click-events | Проверка подозрительных кликов |

Политика хранения и обработки ошибок
- Retention задаётся по типу события: от 7 до 30 дней.
- Для критичных финансовых событий используется больший retention.
- Некорректные события отправляются в dead-letter-events.
- Consumers должны быть идемпотентными, так как Kafka допускает повторную доставку.
- Для контроля обработки используются метрики consumer lag, throughput и error rate.