# 10 лучших практик создания кастомных метрик в Spring Boot (Micrometer)

Spring Boot использует **Micrometer** как фасад метрик (аналог SLF4J, но для метрик), поэтому все практики ниже работают с Prometheus, Datadog, CloudWatch и другими системами мониторинга.

## 1. Выбирайте правильный тип метрики

| Тип | Когда использовать | Пример |
| --- | --- | --- |
| `Counter` | Монотонно растущее значение | число заказов, ошибок |
| `Gauge` | Текущее значение, которое растёт и падает | размер очереди, активные сессии |
| `Timer` | Длительность + количество вызовов | время обработки запроса |
| `DistributionSummary` | Распределение не-временных величин | размер payload, сумма заказа |

```java
Counter.builder("orders.created")
       .description("Количество созданных заказов")
       .register(registry);
```

**Почему это лучшая практика:** неверный тип ломает аналитику. Если сделать Gauge из того, что должно быть Counter, вы потеряете данные между scrape-интервалами, а `rate()` в Prometheus не будет работать. Правильный тип даёт корректные запросы (`rate`, `histogram_quantile`) и точные алерты.

## 2. Придерживайтесь единой конвенции именования

Micrometer использует точечную нотацию (`orders.created`), а реестр сам преобразует имя под бэкенд (`orders_created_total` в Prometheus).

```java
// Хорошо
"orders.created"
"payment.processing.duration"

// Плохо
"OrdersCreated"
"orders_created_total_count"   // суффиксы добавит сам Prometheus
"my metric!"
```

Рекомендации: нижний регистр, точки как разделители, префикс домена (`payment.`, `inventory.`), без единиц измерения в имени для Timer (единицы добавляются автоматически, например `_seconds`).

**Почему это лучшая практика:** когда в системе 500+ метрик, предсказуемые имена позволяют искать их по префиксу, строить универсальные дашборды и не гадать, как называется метрика в соседнем сервисе.

## 3. Контролируйте кардинальность тегов (самое важное)

Каждая уникальная комбинация значений тегов создаёт **отдельный временной ряд**.

```java
// ОПАСНО: userId, orderId, email, URL с параметрами
Counter.builder("orders.created")
       .tag("userId", userId)        // миллионы рядов!
       .register(registry);

// ХОРОШО: ограниченное множество значений
Counter.builder("orders.created")
       .tag("type", order.getType())          // 5 значений
       .tag("channel", order.getChannel())    // web / mobile / api
       .register(registry);
```

**Почему это лучшая практика:** высокая кардинальность — главная причина падения Prometheus и огромных счетов в SaaS-мониторинге (Datadog тарифицирует по числу рядов). Правило: общее число комбинаций тегов на метрику — до сотен, не до тысяч. Идентификаторы пользователей нужно писать в логи и трейсы, а не в метрики.

## 4. Используйте общие теги (common tags) для всего приложения

```yaml
management:
  metrics:
    tags:
      application: ${spring.application.name}
      env: ${ENVIRONMENT:local}
      region: eu-central-1
```

Или программно:

```java
@Bean
MeterRegistryCustomizer<MeterRegistry> commonTags() {
    return registry -> registry.config().commonTags("application", "order-service");
}
```

**Почему это лучшая практика:** в Kubernetes с десятками сервисов и реплик без тегов `application` и `env` невозможно отфильтровать метрики по сервису или окружению. Единые теги делают дашборды переиспользуемыми и позволяют сравнивать prod и stage.

## 5. Измеряйте время через `Timer` и `@Timed`, а не вручную

```java
@Service
public class PaymentService {

    private final Timer paymentTimer;

    public PaymentService(MeterRegistry registry) {
        this.paymentTimer = Timer.builder("payment.processing.duration")
                .description("Время обработки платежа")
                .publishPercentileHistogram()
                .register(registry);
    }

    public void process(Payment p) {
        paymentTimer.record(() -> doProcess(p));
    }
}
```

Или декларативно (требует бин `TimedAspect`):

```java
@Bean
TimedAspect timedAspect(MeterRegistry registry) {
    return new TimedAspect(registry);
}

@Timed(value = "payment.processing.duration", percentiles = {0.5, 0.95, 0.99})
public void process(Payment p) { ... }
```

**Почему это лучшая практика:** ручной подсчёт через `System.currentTimeMillis()` ведёт к ошибкам (забытый `finally`, неверные единицы). `Timer` сразу даёт count, sum и max, а значит и среднее время, и частоту вызовов.

## 6. Используйте гистограммы и SLO вместо клиентских перцентилей

```java
Timer.builder("payment.processing.duration")
     .publishPercentileHistogram()
     .serviceLevelObjectives(
         Duration.ofMillis(100),
         Duration.ofMillis(500),
         Duration.ofSeconds(1))
     .register(registry);
```

Или в конфиге:

```yaml
management:
  metrics:
    distribution:
      percentiles-histogram:
        payment.processing.duration: true
      slo:
        payment.processing.duration: 100ms,500ms,1s
```

**Почему это лучшая практика:** перцентили, посчитанные на клиенте (`.publishPercentiles(0.99)`), **нельзя агрегировать** между репликами: среднее p99 по 10 подам не равно p99 всей системы. Гистограммы передают бакеты, и Prometheus считает корректный перцентиль по всему кластеру через `histogram_quantile`. Бакеты SLO напрямую отвечают на вопрос «какой процент запросов уложился в 500 мс».

## 7. Регистрируйте метрики один раз и кэшируйте

```java
// Плохо: создаётся/ищется при каждом вызове
public void create(Order o) {
    registry.counter("orders.created", "type", o.getType()).increment();
}

// Хорошо: заранее для известных значений
private final Map<OrderType, Counter> counters = new EnumMap<>(OrderType.class);

public OrderService(MeterRegistry registry) {
    for (OrderType t : OrderType.values()) {
        counters.put(t, Counter.builder("orders.created")
                .tag("type", t.name().toLowerCase())
                .register(registry));
    }
}

public void create(Order o) {
    counters.get(o.getType()).increment();
}
```

**Почему это лучшая практика:** `registry.counter(...)` каждый раз вычисляет ключ, ищет в `ConcurrentHashMap` и аллоцирует теги. На горячем пути (тысячи RPS) это заметные накладные расходы и давление на GC. Кэширование убирает этот оверхед, а предзаданные значения тегов защищают от неожиданной кардинальности.

## 8. Для Gauge используйте живой объект и не применяйте его для счётчиков

```java
// Хорошо: Micrometer держит WeakReference на объект
Gauge.builder("orders.queue.size", queue, Queue::size)
     .description("Размер очереди заказов")
     .register(registry);

// Плохо: примитив или локальная переменная — gauge «замрёт» или исчезнет
int size = queue.size();
Gauge.builder("orders.queue.size", () -> size);   // всегда одно значение
```

Для потокобезопасных значений:

```java
AtomicInteger activeSessions = registry.gauge("sessions.active", new AtomicInteger(0));
activeSessions.incrementAndGet();
```

**Почему это лучшая практика:** типичная ошибка новичков — Gauge, который показывает `NaN` из-за сборки мусора объекта или всегда одно и то же значение. Правильная привязка к живому объекту гарантирует актуальные данные и отсутствие утечек памяти.

## 9. Измеряйте бизнес-метрики и метрики ошибок, а не только технические

Помимо JVM и HTTP (Spring Boot даёт их из коробки), добавляйте метрики, отражающие ценность продукта:

```java
Counter.builder("orders.completed")
       .tag("payment_method", method)
       .register(registry).increment();

Counter.builder("orders.failed")
       .tag("reason", "insufficient_funds")   // ограниченный справочник причин
       .register(registry).increment();

DistributionSummary.builder("orders.amount")
       .baseUnit("eur")
       .publishPercentileHistogram()
       .register(registry).record(order.getTotal());
```

**Почему это лучшая практика:** CPU и память могут быть в норме, а выручка при этом падает из-за бага в платёжном шлюзе. Бизнес-метрики ловят проблемы, которые инфраструктурный мониторинг не видит, и позволяют настраивать алерты вида «конверсия упала на 30% за 10 минут». Метод RED (Rate, Errors, Duration) для каждой ключевой операции — хороший минимум.

## 10. Тестируйте метрики и защищайте эндпоинт actuator

**Тест с `SimpleMeterRegistry`:**

```java
@Test
void shouldIncrementCounterOnOrderCreation() {
    MeterRegistry registry = new SimpleMeterRegistry();
    OrderService service = new OrderService(registry);

    service.create(new Order(OrderType.STANDARD));

    assertThat(registry.get("orders.created").tag("type", "standard")
            .counter().count()).isEqualTo(1.0);
}
```

**Конфигурация и безопасность:**

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,prometheus
  endpoint:
    prometheus:
      enabled: true
  server:
    port: 9091            # отдельный порт для actuator, закрытый от внешнего мира
```

Дополнительно можно фильтровать ненужные метрики через `MeterFilter` (например, `MeterFilter.deny()` или ограничение числа значений тега через `maximumAllowableTags`):

```java
@Bean
MeterFilter limitUriTags() {
    return MeterFilter.maximumAllowableTags("http.server.requests", "uri", 100,
            MeterFilter.deny());
}
```

**Почему это лучшая практика:** метрики — это контракт с дашбордами и алертами. Если рефакторинг молча переименовал или сломал метрику, вы узнаете об этом во время инцидента. Тесты предотвращают такие «слепые зоны». А открытый наружу `/actuator/prometheus` раскрывает внутреннюю структуру системы, поэтому его нужно изолировать.

## Итог: как практики помогают вместе

| Практика | Главная выгода |
| --- | --- |
| 1, 5, 6 | Корректные данные и точные перцентили |
| 2, 4 | Единообразие, переиспользуемые дашборды |
| 3, 7, 10 (фильтры) | Стабильность и низкая стоимость мониторинга |
| 8 | Достоверные значения Gauge |
| 9 | Видимость бизнес-проблем, а не только инфраструктурных |
| 10 | Надёжность и безопасность самих метрик |

**Минимальный чек-лист перед merge:**

- Правильный ли тип метрики?
- Имя соответствует конвенции?
- В тегах нет идентификаторов?
- Настроены common tags?
- Для latency используются гистограммы?
- Метрика покрыта тестом?
