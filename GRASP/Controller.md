layout: default
title: Controller: Дирижер системных операций
parent: GRASP
nav_order: 3
---
# Controller: Дирижер системных операций

*Кто должен управлять потоком? Почему ваш API-контроллер не должен знать про БД?*

Ваш API-контроллер получил запрос. Что дальше? Прямо в нем открыть соединение с БД, выполнить запрос, смапить сущность на DTO и отдать ответ? **Стоп, расстрельная команда для архитектуры!** Контроллер в MVC/WebAPI – это *точка входа, а не мозг операции*. *Принцип Controller* вводит ключевую фигуру – *координатора* (не путать с MVC Controller!). Это первый объект за границей UI/API, которому делегируется выполнение сценария пользователя. **Его задача – управлять потоком, координировать работу "экспертов" (сервисов, доменных объектов), но НЕ выполнять их работу самому!** Узнайте, как создать настоящего "дирижера" в C#, который сохранит ваш API-контроллер тонким, а бизнес-логику – независимой от фреймворков и баз данных.

**Суть принципа:**
Назначьте ответственность за обработку системного события (запроса пользователя или внешнего триггера) классу, который:
- Представляет всю систему или подсистему (Facade Controller).
- Инкапсулирует сценарий использования (Use Case Controller).

**Controller не выполняет работу сам!** Он координирует экспертов (сервисы, доменные объекты), управляя потоком выполнения.

## Почему это "Дирижер"?

Представьте оркестр:
- **Музыканты (эксперты):** Владеют инструментами (логика домена).
- **Дирижер (Controller):** Не играет сам, но знает партитуру (сценарий), задает темп и указывает, когда и кому вступать.

**Без Controller'а:**
- API-контроллеры превращаются в "божественные объекты",
- Бизнес-логика прилипает к слою UI/API,
- Система становится монолитной и не тестируемой.

## Антипаттерн: "Толстый контроллер"

```csharp
[ApiController]
public class OrderController : ControllerBase // НЕПРАВИЛЬНО!
{
    [HttpPost("create")]
    public async Task<IActionResult> CreateOrder(OrderDto dto)
    {
        // Антипаттерн: Контроллер делает ВСЁ
        var user = await _context.Users.FindAsync(dto.UserId);  // Прямой доступ к БД
        if (user == null) return NotFound();
        
        var order = new Order(); 
        // Маппинг DTO -> Entity (должен быть в сервисе)
        order.Total = dto.Items.Sum(i => i.Price * i.Quantity); // Логика расчета
        
        if (order.Total > 1000 && !user.IsPremium)              // Бизнес-правило
            return BadRequest("Премиум-требование");

        _context.Orders.Add(order);                             // Сохранение
        await _context.SaveChangesAsync();
        
        await _emailService.SendConfirmation(order);            // Уведомление
        return Ok(order.Id);
    }
}
```

**Проблемы:**
1. **Нарушение SRP:** 5+ ответственностей в одном методе.
2. **High Coupling:** Контроллер знает про БД, email-сервис, бизнес-правила.
3. **Нетестируемость:** Требует поднятия всего ASP.NET Core + БД для юнит-теста.
4. **Смешивание слоев:** UI/API + бизнес-логика + инфраструктура.

 ## Решение: Настоящий Controller в GRASP
 
 ```csharp
 // Use Case Controller (сценарий "Создание заказа")
public class CreateOrderController // Не путать с MVC Controller!
{
    private readonly IOrderRepository _repo;
    private readonly IEmailService _email;
    private readonly IUserValidator _validator;

    public CreateOrderController(
        IOrderRepository repo, 
        IEmailService email,
        IUserValidator validator) // DI зависимостей
    {
        _repo = repo;
        _email = email;
        _validator = validator;
    }

    // Метод обработки сценария (публичный API)
    public async Task<OrderResult> Execute(OrderRequest request)
    {
        // 1. Координация шагов (дирижирование)
        var user = await _repo.GetUser(request.UserId);  
        
        // 2. Делегирование экспертам
        _validator.Validate(user);                         // Бизнес-правила
        
        var order = Order.Create(user, request.Items);     // Information Expert!
        
        // 3. Управление транзакцией
        await _repo.Save(order);
        
        // 4. Триггер сторонних действий
        await _email.SendOrderConfirmation(order);
        
        return new OrderResult(order.Id);
    }
}
 ```
 
### Как использовать в API:

```csharp
[ApiController]
public class OrderApiController : ControllerBase // ТОЛЬКО ВХОД/ВЫХОД
{
    private readonly CreateOrderController _controller;

    public OrderApiController(CreateOrderController controller) 
        => _controller = controller;

    [HttpPost("create")]
    public async Task<IActionResult> CreateOrder(OrderDto dto)
    {
        try
        {
            var result = await _controller.Execute(dto.ToRequest());
            return Ok(result);
        }
        catch (ValidationException ex) // Обработка ошибок
        {
            return BadRequest(ex.Message);
        }
    }
}
```

### Ключевые характеристики Controller'а

1. **Один контроллер - это один сценарий:**
- CreateOrderController, CancelOrderController, GenerateReportController.
- Соответствует High Cohesion.
2. **Не знает о:**
- HTTP, JSON, Cookies (это забота API-контроллера),
- БД, внешних сервисах (их абстракции внедряются через DI).
3. **Только координирует:**
- Вызывает методы экспертов (Order.Create()),
- Управляет транзакционностью,
- Обрабатывает ошибки уровня сценария.
4. **Легко тестируется:**

```csharp
[Test]
public async Task CreateOrder_ValidRequest_ReturnsOrderId()
{
    // Arrange
    var repoMock = new Mock<IOrderRepository>();
    var controller = new CreateOrderController(repoMock.Object, ...);
    
    // Act
    var result = await controller.Execute(testRequest);
    
    // Assert
    Assert.IsNotNull(result.OrderId);
    repoMock.Verify(r => r.Save(It.IsAny<Order>()), Times.Once);
}
```

## Ошибки и решения

**Ошибка:** Controller превращается в "богатый сервис".
**Решение:** Переместить бизнес-логику в доменные объекты (Information Expert).

**Ошибка:** Контроллер знает об инфраструктуре.
**Решение:** Внедрять только интерфейсы (например, IEmailService, а не SmtpEmailService).

**Ошибка:** Один контроллер на весь модуль.
**Решение:** Разбить на мелкие Use Case-контроллеры.

## Почему это "Контролируемый Хаос"?

**Хаос:** Множество объектов (заказ, пользователь, email, БД).
**Контроль:** Controller знает последовательность, но не детали.

**Controller — это "анти-бог":**
- Он не всесилен,
- Не лезет в чужие обязанности,
- Его сила — в знании порядка действий.

**Без него:**
- Код превращается в "свалку сценариев",
- Изменение потока работы требует перелопачивания 10+ классов,
- Система сопротивляется изменениям.

**С ним:**
- Вы получаете точку входа для сценария,
- Которая изолирована от UI и инфраструктуры,
- И готова к эволюции.

## Заключение

Принцип **Controller** в GRASP — это не просто паттерн, а **архитектурное спасение** от хаоса в сложных сценариях. Он превращает бессистемное нагромождение логики в четко дирижируемый оркестр, где каждый инструмент знает свою партию.

**Главные истины:**

1. **Контроллер - это НЕ Божественный объект:**
Он не лезет в БД, не рассчитывает скидки и не шлет письма. Его сила — в координации, а не в выполнении.

*«Дирижер не играет на скрипке — он заставляет скрипку звучать в нужный момент».*

2. **Сценарий — Единица Работы:**
Один контроллер, один сценарий (CreateOrder, CancelBooking). Никаких монстров с 20 методами!

3. **Слои — Неприкосновенны:**
- **UI/API-контроллер:** Только формат данных (HTTP/JSON).
- **GRASP Controller:** Только поток шагов (координация).
- **Эксперты (Order, User):** Только бизнес-логика.

4. **Тестируемость - это Свобода:**
Проверяйте сценарий без поднятия веб-сервера и БД:
```csharp
var controller = new CreateOrderController(mockRepo, mockEmail, mockValidator);
var result = await controller.Execute(testRequest); // Чистый юнит-тест!
```