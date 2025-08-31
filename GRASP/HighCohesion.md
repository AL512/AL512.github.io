---
layout: default
title:  High Cohesion - Сила фокуса
parent: GRASP
nav_order: 5
---
#  High Cohesion: Сила фокуса

*Почему "класс-швейцарский нож" — антипаттерн №1?*

Класс, который умеет всё: и с базой работать, и PDF генерить, и письма отправлять, и цены считать. Удобно? Только до первого изменения! **"Божественный объект" или "Класс-швейцарский нож" – это антипаттерн, порожденный нарушением High Cohesion (Высокой логической связности).** Этот принцип гласит: **элементы внутри класса (методы, свойства) должны быть тесно связаны одной четкой ответственностью**. Класс должен делать что-то одно, но делать это отлично. Высокая связность делает классы понятнее, тестируемее, легче в поддержке и менее подверженными "эффекту бабочки" при изменениях.

**Суть принципа:**
**Высокая логическая связность (High Cohesion)** означает, что все элементы класса (методы, свойства) работают для достижения **единой цели**. Класс фокусируется на **одной задаче** и делает её идеально.

## Почему "Сила фокуса"?
Представьте команду:
- **High Cohesion:** Каждый специалист — эксперт в своей области.
- **Low Cohesion:** Каждый пытается делать всё. 

**Результат Low Cohesion:** Хаос, ошибки и проект, который невозможно масштабировать.

**"Класс-швейцарский нож" — это антипаттерн, где один объект пытается делать всё, и в итоге не делает ничего хорошо.**

## Анатомия Антипаттерна: "Класс-Швейцарский Нож" ("God Object")

```csharp
public class OrderManager // Класс-бог: 7 ответственностей в одном!  
{  
    // 1. Работа с БД  
    public void SaveToDatabase(Order order) { ... }  
    
    // 2. Расчеты  
    public decimal CalculateTotal(Order order) { ... }  
    
    // 3. Логика скидок  
    public void ApplyDiscount(Order order) { ... }  
    
    // 4. Валидация  
    public bool Validate(Order order) { ... }  
    
    // 5. Оплата  
    public void ProcessPayment(Order order) { ... }  
    
    // 6. Логистика  
    public void ScheduleDelivery(Order order) { ... }  
    
    // 7. Уведомления  
    public void SendConfirmationEmail(Order order) { ... }  
}  
```

### Чем это опасно?

1. **Эффект домино:** Изменение в логике email сломает обработку заказов.
2. **Кошмар тестирования:** Чтобы проверить расчет скидки, нужно имитировать БД, платежи и SMTP-сервер.
3. **Невозможность рефакторинга:** 1200 строк кода, где всё связано со всем.
4. **Паралич разработки:** Новый разработчик неделю разбирается, где что находится.

## 5 Признаков Низкой Связности (Low Cohesion)


1. Класс требует 10+ (условно) зависимостей в конструкторе (например: ILogger, IPdfGenerator, IEmailService, IPaymentValidator,...).
2. Методы не используют поля класса:

```csharp
public class ReportService {
    private readonly DbContext _db; // Используется только в SaveReport()
    
    // А этот метод вообще не использует _db!
    public PdfDocument GeneratePdf(Data data) { ... } 
}
```

3. Класс меняется по разным причинам:
- Изменили логику валидации → правки в GodService,
- Обновили PDF-библиотеку → снова правки в GodService.
4. Невозможно кратко описать класс: "Этот класс сохраняет пользователей, генерит отчёты, шлёт письма, валидирует карты..."
5. Более 500 строк кода (условно): Статистически, такой классов нарушают SRP.

## Как Достичь High Cohesion: 4 Тактики

1. Декомпозиция по Принципу SRP
**Было:**

```csharp
public class GodService { ... } // 1200 строк кода  
```

**Стало:**

```csharp
public class UserService { public void SaveUser(User user) }  
public class PdfReportService { public byte[] Generate(Data data) }  
public class EmailService { public void SendPromo(string email) }  
public class PaymentValidator { public bool Validate(Card card) }  
public class ImageProcessor { public Image Resize(Image img) }  
```

**Фишка:* Каждый класс решает одну задачу из предметной области.

2. Группировка по Контексту
Если методы связаны общим контекстом, но решают подзадачи — объедините их в класс:

```csharp
// Не размазываем логику отчётов по 5 классам!
public class FinancialReportService {
    // Высокая связность: все методы вокруг финансовых отчётов
    public Report GenerateQuarterlyReport() { ... }
    public Report GenerateTaxReport() { ... }
    public void ExportToExcel(Report report) { ... }
    public void PrintReport(Report report) { ... }
}
```

3. Изоляция Сложной Логики
Вынесите алгоритмы в отдельные классы:

```csharp
// Было:
public class OrderProcessor {
    public void Process(Order order) {
        // 50 строк валидации
        // 30 строк расчета цены
        // 20 строк применения скидок
    }
}

// Стало:
public class OrderProcessor {
    private readonly IPriceCalculator _calculator;
    public void Process(Order order) {
        _calculator.Calculate(order); // Делегируем!
    }
}

public class PriceCalculator : IPriceCalculator {
    public void Calculate(Order order) { ... } // Сфокусирован ТОЛЬКО на расчетах
}
```

4. Применение Domain-Driven Design (DDD)
Создавайте классы, отражающие термины предметной области:

```csharp
// Агрегат "Заказ" с высокой связностью
public class Order {
    // Состояние и поведение в одном месте
    public void AddItem(Product product, int quantity) { ... }
    public void ApplyDiscount(Coupon coupon) { ... }
    public void ShipTo(Address address) { ... }
    public decimal CalculateTotal() { ... }
}
```

## Как Измерить Cohesion? Метрики и Практика
1. **LCOM (Lack of Cohesion in Methods):**
- Показывает, сколько методов класса не используют его поля.
- Идеал: LCOM = 0 (все методы работают с полями класса).
- Инструменты: NDepend, Visual Studio Code Metrics.
2. Правило 30 секунд:
Попросите коллегу объяснить, что делает класс. Если он не справится за 30 сек — у класса Low Cohesion.
3. Тест "Удаления метода":
Если можно удалить метод из класса, и это не повлияет на работу других методов — это признак Low Cohesion.

## Когда Нарушать High Cohesion?

1. Классы-утилиты:

```csharp
public static class MathUtils {
    public static double CalculateDistance(Point a, Point b) { ... }
    public static double ConvertToRadians(double degrees) { ... }
}
```

*Обоснование:* Группировка математических функций — тематическая связность.

2. Фасады:

```csharp
public class OrderFacade {
    // Объединяет вызовы для упрощения клиентского кода
    public void PlaceOrder(Order order) {
        _validator.Validate(order);
        _payment.Process(order);
        _inventory.ReserveItems(order);
        _notification.SendConfirmation(order);
    }
}
```

## High Cohesion и Low Coupling - лучшые друзья

Эти принципы дополняют друг друга:

- **High Cohesion** объндиняет зависимостей внутри класса,
- **Low Coupling** уменьшает зависимости между классами.

"Класс должен знать мало о других (Low Coupling) и быть сфокусированным на одной задачи (High Cohesion)".

## Почему "Швейцарский Нож" — Антипаттерн?

Потому что:

**"Мастер на все руки = мастер НИ В ЧЕМ"**

- **Класс-специалист (High Cohesion):**
	- Легко тестировать,
	- Просто изменять,
	- Легко понять.
- **Класс-швейцарский нож (Low Cohesion):**
	- Хрупкий как стекло,
	- Непонятный как древний свиток,
	- Неизменяемый как гора.

**Разделяйте ответственности — и ваш код обретёт силу фокуса!**

## Заключение

**High Cohesion — это принцип, который превращает хаотичную "свалку методов" в элегантную систему атомарных и понятных компонентов.** Это архитектурная дисциплина, заставляющая каждый класс заниматься своим делом — и делать это блестяще.

1. Класс должен решать ОДНУ задачу: Не "работать с заказами, PDF и email", а "управлять жизненным циклом заказа".
2. Сила в фокусе: Высокая связность = легкое тестирование, простое понимание и безопасные изменения.
3. "Швейцарский нож" — антипаттерн: 1000-строчные классы-боги ломаются при любом изменении требований.