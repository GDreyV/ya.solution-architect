# Отказоустойчивость данных

## Цели

Для RTB-платформы критично обеспечить:
- минимальную потерю данных;
- быстрое восстановление после сбоя;
- непрерывность обработки bid requests;
- изоляцию отказов между сервисами.


## RPO и RTO

| Сервис | RPO | RTO | Обоснование |
|----|----|----|----|
| Bidding Service | ~0–1 минута | < 5 минут | Сервис stateless, основное состояние хранится в Redis и Kafka |
| Campaign Service | < 5 минут | < 15 минут | Потеря изменений кампаний нежелательна, но допустимо кратковременное восстановление |
| Statistics Service | < 1 минута | < 30 минут | Kafka позволяет повторно обработать события |
| Analytics Service | До нескольких минут | < 1 часа | Аналитика не находится в hot path |
| Billing Service | ~0 | < 15 минут | Финансовые операции требуют минимальной потери данных |


## Репликация и failover

| Компонент | Стратегия |
|----|----|
| PostgreSQL | Primary + standby replica |
| Redis | Redis replication / Redis Sentinel |
| Kafka | replication factor ≥ 3 |
| ClickHouse | replicated cluster |
| Bidding Service | Несколько stateless-инстансов за Load Balancer |

Failover должен быть автоматизирован для критичных компонентов.


## Резервное копирование

| Хранилище | Стратегия backup |
|----|----|
| PostgreSQL | Daily full backup + WAL archiving |
| ClickHouse | Periodic snapshot backup |
| Redis | RDB snapshot + AOF |
| Kafka | Replication + retention policy |

Backups должны:
- храниться отдельно от production-кластера;
- регулярно проверяться через restore test;
- иметь ограниченный retention lifecycle.


## Восстановление после сбоя

При отказе:
- Load Balancer исключает unhealthy Bidding instances;
- PostgreSQL replica становится новым primary;
- Kafka consumers продолжают обработку после восстановления;
- Redis кэш прогревается через cache warming;
- статистика и аналитика могут быть восстановлены через replay Kafka events.


## 12-Factor App

Сервисы проектируются с учётом принципов 12-factor applications:

| Принцип | Применение |
|----|----|
| Config | Конфигурация через environment variables |
| Stateless processes | Bidding Service не хранит локальное состояние |
| Backing services | Redis, Kafka и PostgreSQL подключаются как внешние ресурсы |
| Disposability | Быстрый restart и горизонтальное масштабирование |
| Logs | Логи выводятся в stdout/stderr и централизованно собираются |
| Dev/prod parity | Одинаковый deployment pattern для environments |


## Вывод

Отказоустойчивость достигается за счёт репликации, stateless runtime, event replay через Kafka и разделения сервисов по типу нагрузки.

Критичные данные (финансы и кампании) защищаются через transactional storage и replication, а event-driven компоненты восстанавливаются через Kafka retention и replay.