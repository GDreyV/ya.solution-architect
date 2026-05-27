# API Gateway для DSP-интеграции

## 1. Назначение

API Gateway принимает внешний RTB-трафик от новой DSP-площадки и маршрутизирует bid requests в Bidding Service.

Gateway должен защищать RTB-сервис от перегрузки, контролировать latency и обеспечивать базовую безопасность внешнего API.

Выбранный вариант: **NGINX + Lua**.

---

## 2. Маршрутизация

Основной endpoint:

```http
POST /openrtb/v2/bid
```

Маршрутизация:
```text
DSP Partner
  -> API Gateway
  -> Bidding Service instances
```

Gateway выполняет:
- TLS termination;
- routing по URL;
- балансировку между инстансами Bidding Service;
- настройку коротких timeout для RTB-запросов.

Пример timeout policy:
```text
gateway timeout: 80–100 ms;
upstream connect timeout: 5–10 ms;
upstream response timeout: 50–70 ms.
```

## 3. Rate limiting

Для защиты от перегрузки используется rate limiting на уровне DSP-партнёра.

Ограничения задаются по:
- API key / partner id;
- IP allowlist;
- RPS;
- burst limit.

Пример политики:
```text
partner=new-dsp
limit=18_000 RPS
burst=10%
```

При превышении лимита Gateway возвращает:
```http
429 Too Many Requests
```

## 4. Аутентификация

Для DSP-интеграции используется:
- API key или HMAC-подпись запроса;
- IP allowlist для доверенных адресов DSP;
- TLS для защиты канала.

Минимальный вариант для первого этапа:
- X-API-Key;
- allowlist IP-адресов DSP.

Более строгий вариант:
- X-Partner-Id;
- X-Signature;
- X-Timestamp;
- проверка подписи через HMAC.

## 5. Circuit Breaker

Gateway должен ограничивать количество запросов к Bidding Service при деградации upstream.

Circuit breaker срабатывает при:
- росте 5xx;
- росте timeout;
- превышении P95 latency;
- недоступности части инстансов Bidding Service.

Поведение:
- временно размыкать трафик на проблемный upstream;
- быстро возвращать 204 No Content или 503 Service Unavailable;
- не накапливать долгие ожидающие запросы;
- периодически выполнять health checks.

Для RTB предпочтительнее быстро вернуть no-bid, чем ждать и нарушить SLA.

## 6. Monitoring

Gateway должен собирать технические метрики:

| Метрика  | Назначение |
|----|----|
| RPS по DSP | Контроль входящей нагрузки |
| P50/P95/P99 latency | Контроль SLA |
| 2xx / 4xx / 5xx rate | Контроль ошибок |
| Timeout rate	Обнаружение деградации | Bidding Service |
| 429 rate | Контроль rate limiting |
| Upstream health | Доступность инстансов Bidding Service |

Метрики отправляются в Prometheus, визуализируются в Grafana, по критичным отклонениям настраиваются alert rules.

## 7. Основной поток
```text
DSP Partner
  -> API Gateway
  -> authenticate request
  -> apply rate limit
  -> route to Bidding Service
  -> monitor latency/errors
  -> return bid response / no-bid
```