layout: default
title:  Indirection: Магия посредников
parent: GRAPS
nav_order: 8
---
#  Indirection: Магия посредников

*Почему MediatR и Message Bus – ваши лучшие друзья?*

Два объекта вынуждены взаимодействовать, но прямая связь между ними создает кошмарную связанность (High Coupling) или нарушает другие принципы. Что делать? **Indirection (Посредничество) – введение промежуточного объекта, который берет на себя взаимодействие.** Это как дипломат между враждующими странами. Посредник (например, Mediator, шина сообщений, абстрактная команда или событие) разрывает прямую связь, обеспечивая Low Coupling и открывая двери для гибкости: асинхронности, очередей, легкой замены реализаций, централизованной логики (логирование, валидация, транзакции). **MediatR, MassTransit, NServiceBus или даже простой интерфейс с реализацией – инструменты воплощения этой магии в C#.**

**Суть принципа:**

**Введите промежуточный объект (посредник), чтобы предотвратить прямую связь между другими компонентами.**

**Почему "Магия"?**  
Посредники работают как **дипломаты между враждующими странами** — они позволяют классам общаться, не зная друг о друге. Это создает уровень абстракции, который:

- **Снижает связанность (Low Coupling)**
- **Делает систему гибкой к изменениям**
- **Централизует управление сложными взаимодействиями**

## Проблема: Прямые Связи - Хрупкий Код

```csharp
// Прямая связь - хрупкий код
public class OrderService
{
    private readonly EmailService _emailService;
    private readonly SmsService _smsService;
    private readonly AnalyticsService _analyticsService;

    public void CompleteOrder(Order order)
    {
        // Прямые вызовы - высокое зацепление
        _emailService.SendConfirmation(order);
        _smsService.SendNotification(order);
        _analyticsService.TrackCompletion(order);
        
        // Добавление новой функциональности требует изменения этого класса
    }
}
```


**Проблемы:**

1. **Жесткие зависимости:** OrderService знает о всех сервисах
2. **Сложность тестирования:** Требуются моки всех зависимостей
3. **Трудности расширения:** Добавление нового действия - изменение кода


## Решение: Посредники на Помощь

### 1. Медиатор (MediatR) — Локальный Посредник

```csharp
// Событие вместо прямого вызова
public class OrderCompletedEvent : INotification
{
    public Order Order { get; set; }
}

// Обработчики событий
public class EmailHandler : INotificationHandler<OrderCompletedEvent>
{
    public Task Handle(OrderCompletedEvent notification, CancellationToken token)
    {
        // Отправка email
        return Task.CompletedTask;
    }
}

public class AnalyticsHandler : INotificationHandler<OrderCompletedEvent>
{
    public Task Handle(OrderCompletedEvent notification, CancellationToken token)
    {
        // Отслеживание аналитики
        return Task.CompletedTask;
    }
}

// Сервис использует медиатор
public class OrderService
{
    private readonly IMediator _mediator;

    public OrderService(IMediator mediator) => _mediator = mediator;

    public async Task CompleteOrder(Order order)
    {
        // Единственная зависимость - медиатор
        await _mediator.Publish(new OrderCompletedEvent { Order = order });
    }
}
```


### 2. Шина Сообщений (Message Bus) — Распределенный Посредник

```csharp
// Интеграционное событие
public class OrderCompletedIntegrationEvent
{
    public Guid OrderId { get; set; }
    public decimal Amount { get; set; }
}

// Публикация события
public class OrderService
{
    private readonly IMessageBus _bus;

    public async Task CompleteOrder(Order order)
    {
        await _bus.Publish(new OrderCompletedIntegrationEvent
        {
            OrderId = order.Id,
            Amount = order.Total
        });
    }
}


// Подписчики в разных микросервисах
public class EmailService : IHandler<OrderCompletedIntegrationEvent>
{
    public async Task Handle(OrderCompletedIntegrationEvent @event)
    {
        // Отправка email
    }
}

public class AnalyticsService : IHandler<OrderCompletedIntegrationEvent>
{
    public async Task Handle(OrderCompletedIntegrationEvent @event)
    {
        // Обновление аналитики
    }
}
```

### 3. Абстрактная Фабрика — Посредник для Создания Объектов

```csharp
public interface IPaymentGatewayFactory
{
    IPaymentGateway Create(string gatewayType);
}

public class PaymentGatewayFactory : IPaymentGatewayFactory
{
    private readonly IServiceProvider _serviceProvider;

    public PaymentGatewayFactory(IServiceProvider serviceProvider)
    {
        _serviceProvider = serviceProvider;
    }

    public IPaymentGateway Create(string gatewayType)
    {
        return gatewayType switch
        {
            "PayPal" => _serviceProvider.GetService<PayPalGateway>(),
            "Stripe" => _serviceProvider.GetService<StripeGateway>(),
            _ => throw new ArgumentException("Invalid gateway type")
        };
    }
}

// Использование
public class PaymentService
{
    private readonly IPaymentGatewayFactory _factory;

    public void ProcessPayment(PaymentRequest request)
    {
        var gateway = _factory.Create(request.GatewayType);
        gateway.Process(request);
    }
}
```

## Преимущества Indirection

|**Преимущество**|**Пример реализации**|**Выигрыш**|
|---|---|---|
|**Снижение связанности**|MediatR|Классы не знают друг о друге|
|**Централизация логики**|Message Bus|Управление потоком в одном месте|
|**Легкое тестирование**|Абстрактная фабрика|Mock одного посредника вместо множества сервисов|
|**Гибкость расширения**|Обработчики событий|Новый функционал - новый обработчик|

## Паттерны Indirection 

1. **Adapter** — преобразует интерфейс класса в другой интерфейс:

```csharp
public interface ILogger
{
    void Log(string message);
}

public class ThirdPartyLoggerAdapter : ILogger
{
    private readonly ThirdPartyLogger _thirdPartyLogger;

    public void Log(string message)
    {
        _thirdPartyLogger.WriteMessage(message);
    }
}
```


2. **Facade** — предоставляет простой интерфейс к сложной системе:

```csharp
public class OrderProcessingFacade
{
    private readonly InventoryService _inventory;
    private readonly PaymentService _payment;
    private readonly ShippingService _shipping;

    public async Task ProcessOrder(Order order)
    {
        await _inventory.ReserveItems(order);
        await _payment.ProcessPayment(order);
        await _shipping.ScheduleDelivery(order);
    }
}
```

3. **Proxy** — контролирует доступ к объекту:

```csharp
public class SecurePaymentProxy : IPaymentService
{
    private readonly RealPaymentService _realService;
    private readonly IAuthorizationService _auth;

    public async Task ProcessPayment(PaymentRequest request)
    {
        if (_auth.IsAuthorized(request.UserId))
            await _realService.ProcessPayment(request);
        else
            throw new UnauthorizedAccessException();
    }
}
```



## Когда Использовать Indirection

**Хорошие случаи:**

- Сложные взаимодействия между компонентами
- Необходимость динамического выбора реализации
- Распределенные системы и микросервисы
- Часто меняющиеся бизнес-правила

**Когда избегать:**

- Простые сценарии с минимальными изменениями
- Критичные к производительности участки кода
- Когда избыточная сложность не оправдана

## Best Practices для Разработчиков

1. **Используйте DI для управления зависимостями:**


```csharp
// Регистрация в DI контейнере
services.AddMediatR(typeof(Startup));
services.AddSingleton<IMessageBus, RabbitMQBus>();
services.AddScoped<IPaymentGatewayFactory, PaymentGatewayFactory>();
```


2. **Применяйте принцип "инверсии зависимостей":**

```csharp

// Вместо конкретных реализаций
public class OrderService(IMediator mediator, IMessageBus bus) 
{
    // Зависимости от абстракций
}
```

3. **Используйте атрибуты для автоматической регистрации:**

```csharp

[Handler("order.completed")]
public class OrderCompletedHandler : IHandler<OrderCompletedEvent>
{
    // Обработчик автоматически обнаруживается и регистрируется
}
```

## Заключение

**Магия, Которую Стоит Использовать**

**Indirection** — это мощный инструмент в арсенале архитектора, который позволяет:

- **Управлять сложностью** через абстракции
- **Создавать гибкие системы**, готовые к изменениям
- **Упрощать тестирование** за счет снижения связанности

**Главный принцип:**

Не общайтесь напрямую — используйте посредников для важных разговоров между компонентами"

**Инструменты C#:**

- **MediatR** для внутрипроцессной коммуникации
- **MassTransit/RabbitMQ/Kafka** для распределенной системы
- **DI контейнер** для управления зависимостями

**Важно:** Как и любая магия, Indirection требует разумного использования. Слишком много посредников могут создать избыточную сложность, но правильное применение делает архитектуру чистой и гибкой.