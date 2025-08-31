---
layout: default
title:  Protected Variations - Код-антихрупкость
parent: GRASP
nav_order: 9
---
#  Protected Variations - Код-антихрупкость

*Как изолировать изменения, которые еще не случились?*

Изменения неизбежны. Новые требования, смена API, другой способ оплаты. **Protected Variations (Защищенные Вариации)** – это принцип проактивной защиты. **Выявите точки, подверженные вероятным изменениям (точки нестабильности), и изолируйте их от остальной системы с помощью стабильных интерфейсов.**                                                                                                                                                                     Это создает "антихрупкий" слой: когда изменение все-таки случится, его эффект будет локализован. Вы не предскажете будущее, но вы можете подготовить код к нему. Адаптеры, фасады, абстрактные фабрики, Dependency Injection – ваши главные инструменты для строительства "буферных зон".


**Суть принципа:**

**Защитите элементы системы от нестабильности, создавая стабильные интерфейсы вокруг точек вероятных изменений.**

Это не просто "защита от изменений", а **проактивное проектирование системы, которая становится прочнее от воздействий**.

**Почему "Антихрупкость"?**  
Термин Нассима Талеба:

- **Хрупкое:** Ломается от изменений (стекло).
- **Устойчивое:** Выдерживает изменения (камень).
- **Антихрупкое:** Становится **сильнее** от изменений (иммунная система).
 
 **Ваш код должен быть антихрупким — извлекать выгоду из неопределенности!**


## Антипаттерн: "Хрупкий Монолит"

```csharp
// Код, который сломается при первом же изменении требований
public class PaymentProcessor {
    public void ProcessPayment(Order order) {
        // Прямые зависимости на конкретные реализации = хрупкость
        var payPal = new PayPalGateway(); 
        payPal.Charge(order.Total);
        
        var db = new SqlServerDatabase();
        db.SavePayment(order.Id, order.Total);
        
        var email = new SmtpEmailService();
        email.SendReceipt(order.CustomerEmail);
    }
}
```


**Чем это опасно:**

1. **Смена платежки** (PayPal → Stripe) требует правок в ядре системы.
2. **Переход на NoSQL** (MongoDB) означает взлом метода оплаты.
3. **Добавление SMS-уведомлений** — еще больше усложняет метод.
4. **Тестирование невозможно** без реальных сервисов.

## Как Реализовать Protected Variations: 5 Тактик

### **1. Стабильные Интерфейсы (Abstraction Barrier)**

```csharp
// Стабильный контракт - это защита от изменений реализации
public interface IPaymentGateway {
    void Charge(decimal amount);
    string GenerateReceipt();
}

public interface IPaymentRepository {
    void SavePayment(int orderId, decimal amount);
}

public interface INotificationService {
    void SendReceipt(string email, string receiptText);
}

// Клиентский код зависит только от интерфейсов
public class PaymentProcessor {
    private readonly IPaymentGateway _payment;
    private readonly IPaymentRepository _repository;
    private readonly INotificationService _notification;

    public PaymentProcessor(IPaymentGateway payment, 
                          IPaymentRepository repository,
                          INotificationService notification) {
        _payment = payment;
        _repository = repository;
        _notification = notification;
    }

    public void ProcessPayment(Order order) {
        _payment.Charge(order.Total);
        _repository.SavePayment(order.Id, order.Total);
        
        var receipt = _payment.GenerateReceipt();
        _notification.SendReceipt(order.CustomerEmail, receipt);
    }
}
```


**Что это дает:**

- Замена PayPal на Stripe - новая реализация `IPaymentGateway`.
- Смена БД - новая реализация `IPaymentRepository`.
- Добавление SMS - новый `INotificationService` (Open/Closed Principle!).
- **Ядро системы (`PaymentProcessor`) не меняется!**

### 2. Адаптеры для Внешних Сервисов

```csharp
// Адаптер защищает от изменений в API внешнего сервиса
public class StripeAdapter : IPaymentGateway {
    private readonly StripeClient _stripe;
    
    public StripeAdapter(StripeClient stripe) {
        _stripe = stripe;
    }
    
    public void Charge(decimal amount) {
        // Преобразование нашей модели в модель Stripe
        var request = new StripeChargeRequest { Amount = amount * 100 };
        _stripe.Charge(request); // Если Stripe изменит API — правки только здесь!
    }
}
```


### 3. Паттерн "Стратегия" для Вариативного Поведения

```csharp
// Защита от появления новых алгоритмов
public interface IDiscountStrategy {
    decimal ApplyDiscount(decimal price);
}

public class RegularDiscount : IDiscountStrategy { ... }
public class PremiumDiscount : IDiscountStrategy { ... }
public class HolidayDiscount : IDiscountStrategy { ... }

public class OrderProcessor {
    private readonly IDiscountStrategy _discount;
    
    public OrderProcessor(IDiscountStrategy discount) {
        _discount = discount;
    }
    
    public decimal CalculateFinalPrice(decimal basePrice) {
        return _discount.ApplyDiscount(basePrice); // Полиморфизм вместо if/else
    }
}
```


### 4. Событийная Архитектура (Mediator/Event Bus)

```csharp
// Защита от добавления новой логики обработки
public class OrderCompletedEvent : INotification {
    public Order Order { get; set; }
}

// Обработчики могут добавляться без изменения основного кода
public class EmailHandler : INotificationHandler<OrderCompletedEvent> {
    public Task Handle(OrderCompletedEvent e, CancellationToken token) {
        // Отправка email
    }
}

public class InventoryHandler : INotificationHandler<OrderCompletedEvent> {
    public Task Handle(OrderCompletedEvent e, CancellationToken token) {
        // Списание товаров
    }
}

// Основной сервис не знает о обработчиках
public class OrderService {
    private readonly IMediator _mediator;
    
    public void CompleteOrder(Order order) {
        // ... логика завершения заказа ...
        _mediator.Publish(new OrderCompletedEvent { Order = order }); // Точка стабильности
    }
}
```


### 5. Фасады для Сложных Подсистем

```csharp
// Фасад скрывает сложность и нестабильность подсистемы
public interface IReportGenerator {
    Report Generate(ReportRequest request);
}

public class PdfReportFacade : IReportGenerator {
    // Инкапсулирует работу с: PdfLib, TemplateEngine, ChartBuilder
    public Report Generate(ReportRequest request) {
        var pdf = new PdfLib(); // Нестабильная библиотека
        var template = _templateEngine.Load(request.TemplateId);
        // ... сложная логика ...
        return pdf.Render();
    }
}

// Клиентский код зависит только от фасада
var report = _reportGenerator.Generate(request); // Если PdfLib изменится — клиент не пострадает
```


## Как Определить Точки Нестабильности?

1. **Внешние зависимости:** Платежные системы, API, базы данных, файловые хранилища.
2. **Бизнес-правила:** Часто меняющиеся скидки, налоговые ставки, условия доставки.
3. **Инфраструктура:** Логирование, кэширование, отправка уведомлений.
4. **Вероятные расширения:** Новые типы пользователей, способы оплаты, форматы отчетов.

**Методика:**

- Спросите: "Что может измениться в ближайший год?"
- Проведите анализ: "Что чаще всего менялось в прошлом?"
- Используйте **"Инженерию предположений"**: оберните вероятные изменения в интерфейсы.

## Ошибки и Решения

**Ошибка 1: Преждевременная абстракция**

```csharp
// ПЛОХО: Создавать интерфейс для службы, которая никогда не меняется
public interface IUniqueService { ... } // Изменялась 0 раз за 5 лет
```

**Решение:** Вводите абстракции только для реально нестабильных компонентов.

**Ошибка 2: "Абстракция-пустышка"**

```csharp
// ПЛОХО: Интерфейс повторяет реализацию
public interface ILogger {
    void Log(string message); // А если нужно добавить уровень логирования?
}
```

**Решение:** Проектируйте интерфейсы с заделом на будущее:

```csharp
public interface ILogger {
    void Log(LogLevel level, string message, Exception ex = null);
}
```


**Ошибка 3: Нарушение Liskov Substitution**

```csharp
public class PayPalGateway : IPaymentGateway {
    public void Charge(decimal amount) {
        if (amount > 10000) throw new Exception("Limit exceeded"); // Сюрприз!
    }
}
```

**Решение:** Все реализации должны соблюдать контракт интерфейса.

## Protected Variations и Dependency Injection

DI-контейнер — идеальный инструмент для PV:

```csharp

// Регистрация зависимостей
services.AddScoped<IPaymentGateway, StripeAdapter>(); // Сегодня Stripe
services.AddScoped<IPaymentGateway, PayPalAdapter>(); // Завтра PayPal — поменяли тут и всё!

// Внедрение через конструктор
public class PaymentProcessor {
    public PaymentProcessor(IPaymentGateway payment) { ... } // Получает нужную реализацию
}
```


## **Как Тестировать Антихрупкость?**

1. **Юнит-тесты:** Подменяйте реализации на моки.
2. **Интеграционные тесты:** Проверяйте адаптеры с реальными сервисами.
3. **Тест на замену:** Замените реализацию и убедитесь, что система работает.
4. **Mutation testing:** Вносите изменения в код и проверяйте, падают ли тесты.


## Почему Это "Антихрупкость"?**

PV делает код:

- **Адаптивным:** Вместо "сопротивления изменениям" — "адаптация к изменениям".
- **Эволюционным:** Система становится лучше с каждым новым требованием.
- **Предприимчивым:** Вы можете экспериментировать с новыми технологиями без риска.

**"Хрупкий код ломается от изменений. Антихрупкий код — эволюционирует."**

## Заключение

**Protected Variations — это стратегический принцип, который превращает вашу систему в живой организм, способный не просто выживать при изменениях, а становиться сильнее от них.** Это высшая форма архитектурной зрелости — когда вы проектируете не только под сегодняшние требования, но и под завтрашние неизвестные вызовы.

**Главные итоги:**

1. **PV — это проактивная защита:** Вы не ждете изменений, а заранее создаете «буферные зоны» вокруг точек нестабильности.
2. **Интерфейсы — ваша лучшая броня:** Стабильные контракты защищают ядро системы от изменений в реализации.
3. **Антихрупкость > Устойчивости:** Хорошая система не просто не ломается — она эволюционирует с каждым изменением.