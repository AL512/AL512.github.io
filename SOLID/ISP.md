---
layout: default
title: Принцип Разделения Интерфейса
parent: SOLID
nav_order: 4
---
# Принцип Разделения Интерфейса (Interface Segregation Principle - ISP)

Представьте себе универсальный пульт управления: со 100 кнопками, половина из которых Вам никогда не понадобится, 
но которые мешаются под рукой и усложняют понимание. Так работают "жирные" интерфейсы в коде. **Принцип Разделения 
Интерфейса (ISP)** провозглашает: "*Клиенты не должны зависеть от методов, которые они не используют*". Или иначе: 
**лучше много специализированных интерфейсов, чем один универсальный "монстр"**. В C# это означает, что мы должны 
проектировать интерфейсы точно под нужды их клиентов. Если классу нужна только возможность Save(), зачем ему навязывать 
еще Load(), Update() и Delete(), если он их не использует? ISP борется с:
* **Избыточными зависимостями:** Класс вынужден зависеть (и возможно, даже реализовывать пустые методы!) от того, что ему не нужно.
* **Хрупкостью:** Изменение в неиспользуемом методе большого интерфейса заставляет пересобирать и потенциально ломать всех его клиентов, даже тех, кто этот метод не вызывает.
* **Размыванием ответственности:** Интерфейс теряет фокус.

Решение? *Дробить!* Создавайте компактные, сфокусированные интерфейсы (ISaveable, ILoadable, IDeletable). 
Класс может реализовывать несколько таких интерфейсов, если ему нужно несколько возможностей. 
В результате код становится **чище, менее связанным, проще для понимания и тестирования**. 
ISP – это искусство создания интерфейсов, которые говорят на языке клиента, не навязывая лишнего. Готовы избавиться от 
"пульта с ненужными кнопками" в вашем коде?

**Суть принципа:**
Клиенты не должны зависеть от интерфейсов, которые они не используют. Вместо создания "толстых" интерфейсов, 
следует проектировать множество специализированных интерфейсов.

**Почему это важно:**
1. Избегает принудительной реализации ненужных методов.
2. Уменьшает связность компонентов.
3. Предотвращает загрязнение кода пустыми реализациями.
4. Повышает гибкость системы.
5. Упрощает тестирование.


## Глубокий анализ проблемы "толстых интерфейсов"
**Типичные симптомы нарушения ISP (на мой взгляд):**
1. Пустые реализации методов (throw new NotImplementedException).
2. Интерфейсы с >5-7 методами.
3. Клиенты используют только подмножество методов интерфейса.
4. Изменения в интерфейсе затрагивают не связанные классы.
5. Тесты требуют заглушек для неиспользуемых методов.

**Последствия нарушений:**
1. Хрупкие архитектуры.
2. Трудности при рефакторинге.
3. Ненужные зависимости.
4. Нарушение принципа единственной ответственности.

## Подробные примеры с решениями
**Пример 1: Система управления устройствами (нарушение ISP)**
```csharp
public interface IMultiFunctionDevice
{
    void Print(Document document);
    void Scan(Document document);
    void Fax(Document document);
    void Email(Document document);
}

public class BasicPrinter : IMultiFunctionDevice
{
    public void Print(Document document) { /* Реализация */ }
    
    // Пустые реализации - нарушение ISP
    public void Scan(Document document) 
        => throw new NotImplementedException();
    
    public void Fax(Document document) 
        => throw new NotImplementedException();
    
    public void Email(Document document) 
        => throw new NotImplementedException();
}
```
**Решение через ISP:**
```csharp
public interface IPrinter
{
    void Print(Document document);
}

public interface IScanner
{
    void Scan(Document document);
}

public interface IFax
{
    void Fax(Document document);
}

public interface IEmailSender
{
    void Email(Document document);
}

// Для базового принтера
public class BasicPrinter : IPrinter
{
    public void Print(Document document) { /* ... */ }
}

// Для многофункционального устройства
public class OfficeMachine : IPrinter, IScanner, IFax, IEmailSender
{
    public void Print(Document document) { /* ... */ }
    public void Scan(Document document) { /* ... */ }
    public void Fax(Document document) { /* ... */ }
    public void Email(Document document) { /* ... */ }
}

// Клиенты используют только нужные интерфейсы
public class PrintService
{
    private readonly IPrinter _printer;
    
    public PrintService(IPrinter printer) => _printer = printer;
    
    public void Execute(Document doc) => _printer.Print(doc);
}
```
**Пример 2: Система управления пользователями (реальный кейс)**
**Проблемный интерфейс:**
```csharp
public interface IUserService
{
    void CreateUser(User user);
    void UpdateUser(User user);
    void DeleteUser(int id);
    void LockUser(int id);
    void UnlockUser(int id);
    void ResetPassword(int id);
    void AssignRole(int userId, string role);
    void GenerateReport(UserFilter filter);
}
```
**Решение через ISP:**
```csharp
public interface IUserCRUD
{
    void CreateUser(User user);
    void UpdateUser(User user);
    void DeleteUser(int id);
}

public interface IUserSecurity
{
    void LockUser(int id);
    void UnlockUser(int id);
    void ResetPassword(int id);
}

public interface IUserRoles
{
    void AssignRole(int userId, string role);
}

public interface IUserReporting
{
    void GenerateReport(UserFilter filter);
}

// Реализация для разных клиентов
public class AdminPanelService : IUserCRUD, IUserSecurity, IUserRoles, IUserReporting
{
    // Все методы
}

public class MobileAppService : IUserCRUD, IUserSecurity
{
    // Только CRUD + Security
}
```
## Техники применения ISP
1. Интерфейсы с реализацией по умолчанию
```csharp
public interface IOrderProcessor
{
    void Validate(Order order);
    void Process(Order order);
    
    // Новый метод с реализацией по умолчанию
    string GenerateReceipt(Order order) 
        => $"Receipt for order #{order.Id}";
}

public class BasicOrderProcessor : IOrderProcessor
{
    public void Validate(Order order) { /* ... */ }
    public void Process(Order order) { /* ... */ }
    
    // Не требуется переопределять GenerateReceipt
}
```
2. Комбинирование интерфейсов
```csharp
public interface ISaveable
{
    void Save();
}

public interface ILoadable
{
    void Load();
}

public interface IPersistable : ISaveable, ILoadable {}

public class Document : IPersistable
{
    public void Save() => Console.WriteLine("Saving...");
    public void Load() => Console.WriteLine("Loading...");
}

// Клиент, которому нужно только сохранение
public class AutoSaver
{
    private readonly ISaveable _saveable;
    
    public AutoSaver(ISaveable saveable) => _saveable = saveable;
    
    public void Execute() => _saveable.Save();
}
```
3. Адаптеры для легаси-кода
```csharp
public class LegacySystem
{
    public void ExecuteCommand(string cmd) { /* ... */ }
    public string ReadOutput() { /* ... */ }
    public void Connect() { /* ... */ }
    public void Disconnect() { /* ... */ }
}

// Адаптер, реализующий только необходимый интерфейс
public class LegacyAdapter : ICommandExecutor
{
    private readonly LegacySystem _legacy;

    public LegacyAdapter(LegacySystem legacy) => _legacy = legacy;

    public void Execute(string command)
    {
        _legacy.Connect();
        _legacy.ExecuteCommand(command);
        _legacy.Disconnect();
    }
}
```
## Реальные примеры ISP
1. IAsyncDisposable и IDisposable
```csharp
public class ResourceManager : IAsyncDisposable, IDisposable
{
    public void Dispose() { /* Синхронное освобождение */ }
    
    public ValueTask DisposeAsync() 
    {
        /* Асинхронное освобождение */
        return ValueTask.CompletedTask;
    }
}

// Клиенты используют только нужный интерфейс
public class SyncClient : IDisposable
{
    private readonly IDisposable _resource;
    public SyncClient(IDisposable resource) => _resource = resource;
}

public class AsyncClient : IAsyncDisposable
{
    private readonly IAsyncDisposable _resource;
    public AsyncClient(IAsyncDisposable resource) => _resource = resource;
}
```
2. Минимальные API в ASP.NET Core
```csharp
// Интерфейс с одним методом
public interface IEndpointHandler
{
    Task Handle(HttpContext context);
}

// Регистрация
app.MapGet("/api/users", new UsersHandler().Handle);
app.MapPost("/api/orders", new OrdersHandler().Handle);
```
3. System.Linq
```csharp
public static class EnumerableExtensions
{
    // Разделение на специализированные методы
    public static IEnumerable<T> Where<T>(this IEnumerable<T> source, ...);
    public static IEnumerable<R> Select<T, R>(this IEnumerable<T> source, ...);
    public static IEnumerable<T> Take<T>(this IEnumerable<T> source, ...);
}
```
## Практические рекомендации
1. **Правило 5 методов:** Если интерфейс содержит >5 методов - проверьте на нарушение ISP.
2. **Анализ клиентов:** Создавайте интерфейсы на основе потребностей клиентов.
3. **Композиция интерфейсов:**
```csharp
public interface IAdvancedPrinter : IPrinter, IScanner {}
```
4. **Избегайте "интерфейсного загрязнения":** Не создавайте интерфейсы с единственной реализацией. 

## Ошибки при применении ISP
**Чрезмерное разделение:**
```csharp
// Антипаттерн: слишком мелкие интерфейсы
public interface IIdSetter { void SetId(int id); }
public interface IIdGetter { int GetId(); }
public interface INameSetter { void SetName(string name); }
```
**Решение: Группировать связанные методы**
```csharp
public interface IIdentifiable
{
    int Id { get; set; }
    string Name { get; set; }
}
```
## Паттерны проектирования для ISP
1. Адаптер (Adapter):
```csharp
public class LegacyPrinterAdapter : IPrinter
{
    private readonly LegacyPrinter _legacy;
    public LegacyPrinterAdapter(LegacyPrinter legacy) => _legacy = legacy;
    public void Print(Document doc) => _legacy.PrintDocument(doc);
}
```
2. Фасад (Facade):
```csharp
public class OrderFacade : IOrderProcessing
{
    private readonly InventoryService _inventory;
    private readonly PaymentService _payment;
    private readonly ShippingService _shipping;
    
    public void Process(Order order)
    {
        _inventory.Reserve(order);
        _payment.Charge(order);
        _shipping.Schedule(order);
    }
}
```
3. Команда (Command):
```csharp
public interface ICommand
{
    void Execute();
}

public class PrintCommand : ICommand { /* ... */ }
public class ScanCommand : ICommand { /* ... */ }
```
## Заключение 
ISP освобождает ваши классы от тирании ненужных обязательств. Создавая **узкоспециализированные, клиентоориентированные 
интерфейсы**, Вы устраняете вредные зависимости, уменьшаете связность и резко повышаете ясность кода. Классы реализуют 
только то, что **им действительно нужно**, становясь легче и понятнее. Тестировать такие классы – одно удовольствие. 
В C# ISP напрямую способствует созданию чистых, выразительных контрактов между компонентами, минимизируя побочные 
эффекты от изменений.  
**Не навязывай лишнего – разделяй интерфейсы!**