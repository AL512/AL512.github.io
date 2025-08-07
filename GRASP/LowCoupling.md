layout: default
title: Low Coupling: Искусство независимости
parent: GRASP
nav_order: 4
---
# Low Coupling: Искусство независимости

*Как проектировать связи, чтобы развод был безболезненным?*

Представьте два класса, сцепленных как сиамские близнецы. Изменили один – сломался второй. Добавили фичу в модуль А – завыли тесты модуля Z. Знакомо? **Это высокая связанность (High Coupling)** – архитектурная чума. **Low Coupling** – искусство проектирования *минимально необходимых, явных и устойчивых к изменениям* связей между объектами и модулями. **Это не про отсутствие связей, а про их качество.** Цель: сделать так, чтобы изменение в одной части системы вызывало минимальные (в идеале – нулевые) изменения в других. Чтобы "развести" компоненты можно было с минимальной болью. Готовьтесь резать невидимые нити зависимости в вашем C# коде – это основа гибкости и поддерживаемости.

**Суть принципа:**

**Взаимосвязь (Coupling)** - это показатель того, насколько сильно один элемент связан с другими элементами, обладает знаниями о них или полагается на них. Проектируйте классы так, чтобы они имели минимальные и слабые зависимости друг от друга. Изменения в одном классе не должны вызывать цунами правок в других.

## Почему это "Искусство независимости"?

**Представьте LEGO:**

**High Coupling** — детали склеены суперклеем. Чтобы заменить синюю деталь, вы ломаете всю конструкцию.

**Low Coupling** — детали соединяются аккуратными креплениями. Замена детали - дело 5 секунд.

**Код должен собираться как LEGO, а не как бетонный монолит!**

## Чем Опасна Высокая Связанность (High Coupling)?

```csharp
// Антипаттерн: Токсичная зависимость
public class OrderService {
    private readonly SqlServerDatabase _db; // Жесткая привязка к SQL
    private readonly PayPalGateway _paypal; // Привязка к PayPal
    private readonly SmtpEmailService _email; // Привязка к SMTP

    public void ProcessOrder(Order order) {
        _db.Save(order);                  // Изменение БД = взлом этого класса
        _paypal.Charge(order.Total);       // Смена платежки = катастрофа
        _email.Send(order.Customer.Email); // Переход на Telegram? Не смешите!
    }
}
```

**Последствия:**

1. **Эффект домино:** Изменение PayPal на Stripe требует правок в 20 местах кода.
2. **Тесты - это боль:** Невозможно протестировать OrderService без реальной БД и PayPal.
3. **Сложность эволюции:** Добавить новый способ оплаты - это переписать половину системы.

## Как Достичь Low Coupling:

### 1. Зависимость от Абстракций (DIP)
```csharp
public interface IPaymentGateway { void Charge(decimal amount); }
public interface IEmailService { void Send(string email); }

public class OrderService {
    private readonly IPaymentGateway _payment;
    private readonly IEmailService _email;

    // Конструктор принимает АБСТРАКЦИИ
    public OrderService(IPaymentGateway payment, IEmailService email) {
        _payment = payment;
        _email = email;
    }

    public void ProcessOrder(Order order) {
        _payment.Charge(order.Total); // Не важно, что под капотом: PayPal или Stripe!
        _email.Send(order.Customer.Email); // SMTP, Telegram, голубь-почтальон?
    }
}
```

**Фишки:**
- Реализации (PayPalGateway, SmtpEmailService) регистрируются в DI-контейнере.
- Замена платежки = новая реализация IPaymentGateway. Класс OrderService даже не узнает!

### 2. Закон Деметры ("Не разговаривай с незнакомцами")

**Допустимо:**
```csharp
order.Customer.Name; // Обращение к непосредственному свойству  
```

**Недопустимо:**
```csharp
order.Customer.Address.City.Street.Name.Length; // Цепочка вызовов - хрупкость!  
```

**Решение:**

```csharp
// В классе Customer:
public string GetStreetName() => Address.City.Street.Name; 

// Теперь безопасно:
order.Customer.GetStreetName();
```

### 3. Шаблон "Посредник" (Mediator)

```csharp
// Событие вместо прямого вызова
public class OrderCompletedEvent : INotification {
    public Order Order { get; }
}

// Обработчики где угодно в системе
public class EmailHandler : INotificationHandler<OrderCompletedEvent> {
    public Task Handle(OrderCompletedEvent e, CancellationToken token) {
        // Отправка email
    }
}

public class OrderService {
    private readonly IMediator _mediator;
    
    public void CompleteOrder(Order order) {
        // OrderService не знает о почте/логах/аналитике!
        _mediator.Publish(new OrderCompletedEvent { Order = order });
    }
}
```

**Инструменты:** MediatR, MassTransit.

### 4. Фасады для Сложных Подсистем

```csharp
// Вместо 10 зависимостей:
public class ReportGeneratorFacade : IReportGenerator {
    public Report Generate(ReportRequest request) {
        // Инкапсулирует работу с:
        // - DataFetcher
        // - TemplateEngine
        // - ChartBuilder
        // - FormatConverter
    }
}

// Клиентский код зависит ТОЛЬКО от фасада
public class ReportController {
    private readonly IReportGenerator _generator;
    // ...
}
```

### 5. Событийная Архитектура

```csharp
// Order и Inventory вообще не знают друг о друге!
public class Order {
    public void Complete() {
        DomainEvents.Raise(new OrderCompletedEvent(this));
    }
}

public class InventoryHandler {
    public void OnOrderCompleted(OrderCompletedEvent e) {
        // Списание товаров со склада
    }
}
```

## Как Измерить Coupling?

1. Статический анализ:
- Показатель Afferent Coupling (Ca): Сколько классов зависит от этого.
- Efferent Coupling (Ce): От скольких классов зависит этот.
- Instability = Ce / (Ca + Ce). Цель: 0.5-1.0.
2. Визуальный тест:
- Можете ли вы удалить класс, не получив 100 ошибок компиляции?
- Можете ли вы подменить реализацию за 5 минут?

## Когда Нарушать Low Coupling?

1. Внутри модуля: Классы одного модуля могут тесно взаимодействовать (но изолируйте модуль!).
2. DTO/Value-объекты: Order.Address.City — нормально, если Address неизменяем.
3. Высокопроизводительные участки: Прямые вызовы быстрее, чем через интерфейсы.

## Практика: 3 Шага к Low Coupling

Шаг 1: Замените new на интерфейсы
```csharp
- var payment = new PayPalGateway();
+ public OrderService(IPaymentGateway payment)
```

Шаг 2: Разрубите цепочки вызовов
```csharp
- order.Customer.Address.City.Street.Name;
+ order.Customer.GetStreetName();
```

Шаг 3: Внедрите MediatR
```csharp
// Было:
_emailService.Send(...);
_logger.Log(...);

// Стало:
_mediator.Publish(new OrderCompletedEvent);
```

## Почему Это "Искусство"?

Low Coupling — это **баланс**:
- Между **гибкостью и сложностью.**
- Между **абстракциями и производительностью.**
- Между **"разделяй и властвуй" и "не создай космический корабль".**

**Худший грех:** Слепая погоня за "нулевой связанностью", порождающая 10 абстракций для одной логики...

**Золотое правило:**

**Зависимости должны быть явными, минимальными и управляемыми.
Код должен пережить "развод" классов без дележа имущества через суд!**

## 3 Заповеди для Ежедневной Практики

1. "Интерфейс":
```csharp
// Вместо этого:  
var logger = new FileLogger("log.txt");  

// Делай так:  
public class MyService(ILogger logger) // <- Интерфейс + DI  
```

2. "Цепочкам — нет!":
```csharp
// Вместо этого:  
var street = order.Customer.Address.City.Street;  

// Делай так:  
var street = order.GetCustomerStreet(); // Инкапсуляция деталей  
```

3. "События вместо пыток":
```csharp
// Вместо этого:  
_paymentService.Charge();  
_emailService.Send();  
_logger.Log();  

// Делай так:  
_mediator.Publish(new OrderPaidEvent(order)); // Одно событие → 10 обработчиков  
```


## Заключение

**Low Coupling — это не просто принцип, это философия выживания в мире изменений.** Это искусство строить отношения между классами так, чтобы их "развод" проходил без войны за наследство, а замена "партнера" не требовала перекройки всей системы.

**Low Coupling** — это свобода:
- Менять технологии без боли,
- Тестировать без слез,
- Масштабировать без страха.


**4 дара Low Coupling:**
1. **Гибкость:** Замените одного класса на другой за 5 минут, а не 5 дней.
2. **Тестируемость:** Мокируйте зависимости и тестируйте изоляции — без танцев с бубном.
3. **Понятность:** Классы перестают быть "паутиной зависимостей". Архитектура читается как книга.
4. **Устойчивость:** Изменения в одном модуле не взрывают всю систему.

**"Зависите от интерфейсов, а не реализаций; общайтесь через контракты, а не детали; связывайтесь минимально, но осознанно".**

**Стройте системы, где классы общаются как джентльмены — на расстоянии вытянутой руки, с уважением к личным границам. И тогда ваш код перестанет "стрелять в ноги" при первом же изменении требований!**