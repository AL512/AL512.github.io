---
layout: default
title: Принцип Открытости/Закрытости
parent: SOLID
nav_order: 2
---
# Принцип Открытости/Закрытости (Open/Closed Principle - OCP)

Мир программного обеспечения нестабилен. Требования меняются как погода. Как создать систему, которую не нужно 
перелопачивать целиком каждый раз, когда появляется новая фича или нужно немного подкорректировать поведение? 
Принцип **Открытости/Закрытости (OCP)** дает нам волшебный ключ: "Программные сущности (классы, модули, функции) должны 
быть открыты для расширения, но закрыты для модификации". Звучит почти как парадокс? Это и есть **магия абстракции и 
полиморфизма** в C#! Вместо того чтобы лезть в проверенный, работающий код и рисковать что-то сломать, OCP учит нас 
расширять поведение. Как? Через **наследование, композицию и, особенно, через интерфейсы**. Вы проектируете систему вокруг 
стабильных абстракций (интерфейсов или базовых классов), а новую функциональность добавляете, создавая новые реализации
этих абстракций. Это как добавлять новые модули к космическому кораблю, не перестраивая его каркас. Код становится 
**устойчивым к изменениям**, его основа остается нерушимой, а новые возможности интегрируются почти безболезненно. 
OCP – это принцип, который превращает ваш код из хрустальной вазы в живой, растущий организм.

**Суть принципа:**  
Программные сущности (классы, модули, функции) должны быть:
* **Открыты для расширения** (можно добавлять новую функциональность).
* **Закрыты для модификации** (существующий код не изменяется при добавлении новой функциональности).

**Почему это важно:**
1. **Стабильность системы:** Основная логика защищена от изменений.
2. **Снижение риска регрессии:** Новый функционал не ломает существующий.
3. **Упрощение тестирования:** Не нужно перетестировать старый код.
4. **Гибкость архитектуры:** Легко добавлять новые функции.
5. **Улучшение масштабируемости:** Система растет без переписывания

## Ключевые подходы к реализации OCP
### 1. Полиморфизм через наследование
**Пример:** Система расчета площади фигур 

```csharp
// Базовый класс, закрытый для модификации
public abstract class Shape
{
    public abstract double Area();
}

// Расширение без изменения базового класса
public class Circle : Shape
{
    public double Radius { get; set; }
    public override double Area() => Math.PI * Radius * Radius;
}

public class Rectangle : Shape
{
    public double Width { get; set; }
    public double Height { get; set; }
    public override double Area() => Width * Height;
}

// Новый тип фигуры без изменения существующего кода
public class Triangle : Shape
{
    public double Base { get; set; }
    public double Height { get; set; }
    public override double Area() => 0.5 * Base * Height;
}

// Класс для расчетов не требует изменений при добавлении новых фигур
public class AreaCalculator
{
    public double TotalArea(IEnumerable<Shape> shapes) 
        => shapes.Sum(shape => shape.Area());
}
```
### 2. Использование интерфейсов
**Пример:** Система обработки платежей
```csharp
// Интерфейс, закрытый для модификации
public interface IPaymentProcessor
{
    void ProcessPayment(decimal amount);
}

// Реализации, которые можно расширять
public class CreditCardProcessor : IPaymentProcessor
{
    public void ProcessPayment(decimal amount)
        => Console.WriteLine($"Processing credit card payment: {amount}");
}

public class PayPalProcessor : IPaymentProcessor
{
    public void ProcessPayment(decimal amount)
        => Console.WriteLine($"Processing PayPal payment: {amount}");
}

// Новая платежная система без изменения основного кода
public class CryptoProcessor : IPaymentProcessor
{
    public void ProcessPayment(decimal amount)
        => Console.WriteLine($"Processing crypto payment: {amount}");
}

// Класс обработки заказов не изменяется при добавлении новых процессоров
public class OrderService
{
    private readonly IPaymentProcessor _processor;

    public OrderService(IPaymentProcessor processor) => _processor = processor;

    public void ProcessOrder(Order order)
    {
        // Логика оформления заказа
        _processor.ProcessPayment(order.TotalAmount);
    }
}
```
### 3. Паттерн "Стратегия"
**Пример:** Система фильтрации данных
```csharp
public interface IFilterStrategy<T>
{
    IEnumerable<T> Apply(IEnumerable<T> items);
}

public class NameFilter : IFilterStrategy<Product>
{
    private readonly string _name;
    public NameFilter(string name) => _name = name;
    
    public IEnumerable<Product> Apply(IEnumerable<Product> items)
        => items.Where(p => p.Name.Contains(_name));
}

public class PriceRangeFilter : IFilterStrategy<Product>
{
    private readonly decimal _min;
    private readonly decimal _max;
    public PriceRangeFilter(decimal min, decimal max) => (_min, _max) = (min, max);
    
    public IEnumerable<Product> Apply(IEnumerable<Product> items)
        => items.Where(p => p.Price >= _min && p.Price <= _max);
}

// Новая стратегия фильтрации без изменения основного кода
public class StockFilter : IFilterStrategy<Product>
{
    private readonly int _minStock;
    public StockFilter(int minStock) => _minStock = minStock;
    
    public IEnumerable<Product> Apply(IEnumerable<Product> items)
        => items.Where(p => p.Stock >= _minStock);
}

// Основной сервис не изменяется при добавлении новых фильтров
public class ProductFilter
{
    public IEnumerable<Product> Filter(
        IEnumerable<Product> products,
        IFilterStrategy<Product> strategy)
        => strategy.Apply(products);
}
```
### 4. Декораторы
```csharp
public interface IDataService
{
    string GetData();
}

public class BasicDataService : IDataService
{
    public string GetData() => "Core data";
}

// Декоратор добавляет функциональность без изменения основного класса
public class LoggingDataService : IDataService
{
    private readonly IDataService _inner;
    private readonly ILogger _logger;

    public LoggingDataService(IDataService inner, ILogger logger)
        => (_inner, _logger) = (inner, logger);

    public string GetData()
    {
        _logger.Log("Fetching data...");
        var data = _inner.GetData();
        _logger.Log($"Data received: {data}");
        return data;
    }
}

// Еще один декоратор
public class CachingDataService : IDataService
{
    private readonly IDataService _inner;
    private string _cache;

    public CachingDataService(IDataService inner) => _inner = inner;

    public string GetData() => _cache ??= _inner.GetData();
}

// Использование
var service = new CachingDataService(
    new LoggingDataService(
        new BasicDataService(),
        new ConsoleLogger()
    )
);
```
### Нарушения OCP и их исправление
**Типичное нарушение:**
```csharp
public class ReportGenerator
{
    public void GenerateReport(string reportType)
    {
        if (reportType == "PDF")
        {
            // Генерация PDF
        }
        else if (reportType == "Excel")
        {
            // Генерация Excel
        }
        // При добавлении нового типа приходится изменять класс
    }
}
```
**Исправленная версия:**
```csharp
public interface IReportGenerator
{
    void Generate();
}

public class PdfReportGenerator : IReportGenerator
{
    public void Generate() => // Генерация PDF
}

public class ExcelReportGenerator : IReportGenerator
{
    public void Generate() => // Генерация Excel
}

public class ReportService
{
    private readonly IReportGenerator _generator;
    
    public ReportService(IReportGenerator generator)
        => _generator = generator;
    
    public void GenerateReport() => _generator.Generate();
}

// Добавление нового типа без изменения существующего кода
public class CsvReportGenerator : IReportGenerator
{
    public void Generate() => // Генерация CSV
}
```
## Заключение
OCP – это ваша главная защита от бесконечного рефакторинга. Следуя ему, вы создаете систему, которая **эволюционирует 
через расширение, а не через постоянные переделки**. Абстракции становятся стабильным фундаментом, а новое поведение 
безопасно "подключается" через полиморфизм и композицию. Ваш код обретает иммунитет к изменениям требований. В мире C# 
OCP напрямую ведет к использованию стратегий, плагинов и других паттернов, делающих приложение **по-настоящему гибким и 
будущеустойчивым**.  
**Расширяйте, не ломая!**