# 2.1. Матрица мастер-систем

| Сущность | Источники сейчас | Мастер-система через 3 месяца | Мастер-система через 12 месяцев | Связь с BIAN | Обоснование |
|---|---|---|---|---|---|
| Клиент | FU CRM, RB CRM | MDM Customer Hub | MDM Customer Hub | Party Reference Data Directory | Золотая запись и единый идентификатор |
| Продукт, тариф | Core, Cards, Loans двух банков | Согласованный справочник в MDM | MDM | Product Directory | Единые коды и версии справочников |
| Счёт | FU Core Banking, RB Core | Исходная Core-система до миграции | FU Core Banking | Current Account | Финансовые записи нельзя переносить одним шагом |
| Договор | Core и Loans двух банков | Исходная система договора | FU Core Banking или FU Loans по типу договора | Customer Agreement | Сохраняется авторитетность исходной записи в переходный период |
| Карта | FU Cards, RB Cards | Исходный карточный процессинг | FU Cards | Issued Device Administration | Статус карты меняет процессинг |
| Кредит | FU Loans, RB Loans | Исходная кредитная система | FU Loans | Loan | Кредитный договор ведётся операционной системой |
| Платёж, операция | Core и Cards двух банков | Система, исполнившая операцию | FU Core Banking или FU Cards | Payment Order, Financial Accounting | Нужны целостность и неизменность финансового факта |
| Обращение | FU CRM, RB CRM | CRM, где создано обращение | FU CRM | Customer Case Management | Переход без остановки обслуживания |
