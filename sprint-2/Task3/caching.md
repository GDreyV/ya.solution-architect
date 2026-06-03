# Кэширование

## Цель

Кэширование нужно для снижения latency в RTB-потоке и уменьшения количества read-запросов к PostgreSQL.

Bidding Service должен принимать решение преимущественно на основе кэшированной read-модели.

---

## Что кэшировать

| Данные | Где используются | TTL |
|---|---|---|
| Активные кампании | Bidding Service | 30–60 секунд |
| Ставки и bid rules | Bidding Service | 30–60 секунд |
| Targeting rules | Bidding Service | 1–5 минут |
| Креативы и metadata | Bidding / Delivery | 5–10 минут |
| BudgetSnapshot | Bidding Service | 5–15 секунд |
| Справочники | Bidding / Campaign | 10–30 минут |

Финальные финансовые операции не выполняются только на основании кэша. Кэш бюджета используется как быстрый snapshot для принятия решения, а фактические списания выполняются через Billing Service.

---

## Где разместить Redis

Используется `Redis Cluster`, расположенный рядом с Bidding Service в той же зоне/кластере.

```text
Bidding Service
  -> Redis Cluster
  -> Campaign DB / Billing DB fallback
```

Redis должен быть:
- реплицированным;
- доступным из нескольких инстансов Bidding Service;
- изолированным от аналитической нагрузки;
- покрытым мониторингом latency, memory usage и hit ratio.
- Инвалидация кэша

Основная стратегия — event-driven invalidation.

Campaign Service
  -> campaign.updated event
  -> Cache Updater
  -> Redis

Инвалидация выполняется при:
- изменении статуса кампании;
- изменении ставки;
- изменении targeting rules;
- остановке кампании;
- изменении бюджета.

Дополнительно используется TTL как safety net на случай потери события инвалидации.

## Прогрев кэша

Cache warming выполняется:
- при старте Bidding Service;
- перед подключением новой DSP;
- после деплоя;
- после массового обновления кампаний.

Процесс:
```text
Campaign Service / DB
  -> active campaigns export
  -> Cache Warmer
  -> Redis
```

В первую очередь прогреваются:
- активные кампании;
- кампании с высоким трафиком;
- ставки;
- targeting rules;
- BudgetSnapshot.

## Fallback

Если Redis недоступен:
- использовать локальный in-memory cache с коротким TTL;
- при отсутствии валидных данных вернуть 204 No Content;
- не выполнять долгие синхронные запросы в БД в рамках RTB hot path.