# Принцип Подстановки Лисков (Liskov Substitution Principle - LSP)

Наследование – мощный инструмент C#, но им так легко злоупотребить! Что если ваш "утенок", наследующий от "птицы", 
вдруг окажется... игрушечной ракетой и не сможет летать? **Принцип Подстановки Лисков (LSP)** устанавливает строгие правила 
"*семейной игры*": "*Объекты должны быть заменяемыми экземплярами своих базовых типов без нарушения правильности программы*". 
Проще говоря: если у вас есть код, работающий с Bird, то вы должны **безопасно подставить** туда любой его подкласс 
(например, Duck или Eagle), и программа не должна сломаться или выдать неожиданный результат. Это не просто про 
синтаксис наследования, это про **поведенческую совместимость**! 

LSP борется с коварными ошибками, возникающими когда подклаcс:
* Ослабляет предусловия (требует меньше, чем базовый класс).
* Усиливает постусловия (гарантирует меньше, чем базовый класс)
* Искажает инварианты (нарушает внутренние правила базового класса).
* Выбрасывает неожиданные исключения.

Соблюдение LSP – это залог **надежности иерархий наследования** и полиморфного кода в C#. 
Это договор между базовым классом и его потомками, гарантирующий, что клиентский код может доверять абстракции. 
LSP превращает наследование из опасной игры в предсказуемый механизм построения гибких систем.

**Формальное определение:**
"*Если для каждого объекта o1 типа S существует объект o2 типа T, такой, что для всех программ P, определенных в 
терминах T, поведение P не изменяется при подстановке o1 вместо o2, тогда S является подтипом T.*" — Барбара Лисков, 1987

**Проще говоря:**
Объекты производных классов должны быть способны заменять объекты базовых классов без нарушения работы программы. 
Наследники должны дополнять, а не изменять поведение родителя.

## Ключевые аспекты LSP
1. Контрактное поведение:
    * Предусловия не могут быть усилены в подклассе.
    * Постусловия не могут быть ослаблены в подклассе.
    * Инварианты базового класса должны сохраняться.
2. Типичные нарушения:
    * Выбрасывание новых исключений в наследниках.
    * Возврат значений другого типа.
    * Изменение состояния базового класса недопустимым образом.
    * Требование дополнительных условий для работы.

## Подробные примеры с анализом
**Пример 1:** Классический пример с прямоугольником и квадратом
**Нарушение LSP:**
```csharp
public class Rectangle
{
    public virtual int Width { get; set; }
    public virtual int Height { get; set; }
    
    public int Area => Width * Height;
}

public class Square : Rectangle
{
    public override int Width
    {
        set { base.Width = value; base.Height = value; }
    }
    
    public override int Height
    {
        set { base.Width = value; base.Height = value; }
    }
}

// Тест, демонстрирующий проблему
public void TestRectangleArea(Rectangle rect)
{
    rect.Width = 5;
    rect.Height = 4;
    Console.WriteLine(rect.Area); // Ожидается 20, но для Square будет 16
}
```
**Проблема:** Квадрат изменяет ожидаемое поведение сеттеров, нарушая контракт базового класса.
**Решение через LSP:**
```csharp
public interface IShape
{
    int Area { get; }
}

public class Rectangle : IShape
{
    public int Width { get; set; }
    public int Height { get; set; }
    public int Area => Width * Height;
}

public class Square : IShape
{
    public int Side { get; set; }
    public int Area => Side * Side;
}
```

**Пример 2:** Система обработки заказов
**Нарушение LSP:**
```csharp
public abstract class Order
{
    public virtual void Process()
    {
        // Базовая обработка
    }
    
    public virtual bool Validate()
    {
        return true; // Базовая валидация
    }
}

public class DiscountOrder : Order
{
    public override void Process()
    {
        if (!ValidateDiscount())
            throw new InvalidOperationException("Discount not valid");
        
        base.Process();
    }
    
    private bool ValidateDiscount() { /* ... */ }
}

// Клиентский код
public void ProcessOrder(Order order)
{
    order.Validate();
    order.Process(); // Может выбросить исключение для DiscountOrder
}
```
**Проблема:** Клиентский код не ожидает исключений от базового метода Process().
**Решение через LSP:**
```csharp
public abstract class Order
{
    protected abstract void ValidateCore();
    
    public void Validate()
    {
        // Общая логика валидации
        ValidateCore();
    }
    
    public abstract void Process();
}

public class DiscountOrder : Order
{
    protected override void ValidateCore()
    {
        // Специфичная валидация скидки
    }
    
    public override void Process()
    {
        // Обработка со скидкой
    }
}
```
## Правила проектирования по LSP
1. Не изменяйте ожидаемое поведение:
    * Переопределенные методы должны выполнять ту же логическую функцию.
    * Результаты должны быть совместимы по типам.
2. Не вводите новые исключения:
    * Подклассы не должны генерировать исключения, неизвестные базовому классу.
3. Соблюдайте инварианты:
    * Состояние объекта должно оставаться валидным после любого метода.
4. Поддерживайте исторические ограничения:
    * Новые правила не должны запрещать то, что разрешалось базовым классом.

## Паттерны проектирования для соблюдения LSP
1. Шаблонный метод (Template Method):
```csharp
public abstract class DataProcessor
{
    // Неизменяемый алгоритм
    public void Process()
    {
        LoadData();
        TransformData();
        SaveResult();
    }
    
    protected abstract void LoadData();
    protected abstract void TransformData();
    protected virtual void SaveResult() 
        => Console.WriteLine("Default saving");
}

public class CsvProcessor : DataProcessor
{
    protected override void LoadData() { /* ... */ }
    protected override void TransformData() { /* ... */ }
}
```
2. Стратегия (Strategy):
```csharp
public interface IExportStrategy
{
   void Export(Data data);
}

public class PdfExport : IExportStrategy { /* ... */ }
public class ExcelExport : IExportStrategy { /* ... */ }

public class ReportGenerator
{
private readonly IExportStrategy _exporter;

    public ReportGenerator(IExportStrategy exporter) 
        => _exporter = exporter;
    
    public void Generate(Data data) 
        => _exporter.Export(data);
}
```
3. Декоратор (Decorator):
```csharp
public interface INotifier
{
    void Send(string message);
}

public class EmailNotifier : INotifier { /* ... */ }

public class SmsDecorator : INotifier
{
    private readonly INotifier _inner;
    
    public SmsDecorator(INotifier inner) 
        => _inner = inner;
    
    public void Send(string message)
    {
        _inner.Send(message);
        Console.WriteLine($"SMS: {message}");
    }
}
```
## Последствия нарушения LSP
**Методика "Тест подстановки":**
1. **Хрупкость кода:** Изменения в подклассах ломают клиентский код.
2. **Непредсказуемость:** Поведение системы становится неочевидным.
3. **Сложность тестирования:** Требуются специальные тесты для каждого подкласса.
4. **Нарушение OCP:** Приходится модифицировать существующий код для поддержки новых подклассов.

## Заключение 
LSP – это страховой полис для вашего полиморфизма. Соблюдение этого принципа гарантирует, что **иерархии наследования 
работают как часы**, а обещания базовых типов не нарушаются их потомками. Клиентский код получает право полагаться на 
абстракции без страха неприятных сюрпризов. Это принцип, который предотвращает коварные баги, вызванные некорректным 
переопределением, и обеспечивает **логическую целостность и надежность** вашей объектной модели в C#. 
**Наследуйте ответственно, подставляйте безопасно!**


[Оглавление](/README.md)         -> [Принцип Разделения Интерфейса (Interface Segregation Principle - ISP))](/SOLID/ISP.md)