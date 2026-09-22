# Сервисы по capability

Источник: [Capability Map](capability-map.md), [ADR-001](../Task2/adr-001-architecture-style.md). Модули входят в один backend.

| Capability | Продукт / сервис | Владелец | Внешняя зависимость |
|---|---|---|---|
| Ассортимент, наличие | Модуль Catalog / Inventory | Backend-1 | Контент продавца |
| Оформление заказа | Orders + PostgreSQL outbox | Backend-1 | PostgreSQL |
| Оплата, выплаты, возврат | Payments + Paystack adapter | Backend-2 | Paystack |
| Доставка и статусы | Fulfillment + Sendbox adapter | Backend-2 | Sendbox |
| Проверка продавца, споры | Sellers / Support | Full-stack; операционный владелец CEO | Юрист, сотрудники поддержки |
| Согласия | Consent ledger | Full-stack; юрист | Юридическое заключение |
| Сообщения | Notifications worker | Full-stack | Termii |
| Покупательский интерфейс | PWA | Frontend | API, CDN |
| Мобильный доступ | Тонкая оболочка приложения | Mobile | Магазины приложений |
| Надёжность | CI/CD, Patroni, HAProxy, мониторинг | SRE | Layer3, HOSTAFRICA, Azure |

## Assumptions

CEO обеспечивает операционную поддержку и назначает ответственного за данные; эти роли находятся вне шести контракторов разработки.
