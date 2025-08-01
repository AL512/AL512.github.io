# Принцип Единственной Ответственности (Single Responsibility Principle - SRP)

Представьте мастера на все руки – он может и гвоздь забить, и проводку починить, и стену покрасить. 
Но попробуйте поручить ему построить небоскреб... Хаос неизбежен. Так и в коде. 
**Принцип Единственной Ответственности (SRP)** провозглашает священное правило: "*Один класс – одна причина для изменения*". 
Это **основа всего здания SOLID**. Когда класс берет на себя слишком много, он становится монолитом: его сложно понять,
невозможно изменить без риска сломать что-то еще, а тестирование превращается в кошмар. 
SRP учит нас искусству **фокуса и декомпозиции**. Он говорит: "Раздели бремя!". Создай классы, каждый из которых 
отвечает за одну четко очерченную задачу или аспект поведения. В мире C# это означает создание классов с ясной, 
**легко формулируемой целью**. Такой код становится подобен отлаженному механизму, где каждая шестеренка знает свое дело.
Он легче читается, проще тестируется (ведь тестируешь одну маленькую функциональность), и изменения в одной части 
системы перестают вызывать землетрясения в других. SRP – это первый шаг к превращению хаоса в порядок, 
к управляемой сложности. 

**Суть принципа:**  
Класс должен иметь **только одну причину для изменения** — то есть отвечать за одну конкретную задачу или функциональность. 
Каждый компонент системы должен фокусироваться на выполнении единственной цели.

**Почему это важно:**
1. Упрощение тестирования
2. Повышение читаемости кода
3. Снижение связанности компонентов
4. Упрощение рефакторинга
5. Улучшение повторного использования кода

## Примеры нарушений SRP и их исправления
### Пример 1: Класс с двумя ответственностями
**Нарушение:** Класс объединяет бизнес-логику и работу с данными.

```csharp
public class ProductService
{
    public void AddProduct(Product product)
    {
        // Проверка бизнес-правил
        if (product.Price <= 0)
            throw new ArgumentException("Invalid price");
        
        // Логика сохранения в БД
        using var context = new AppDbContext();
        context.Products.Add(product);
        context.SaveChanges();
        
        // Отправка уведомления
        EmailService.SendEmail("admin@example.com", "New product added");
    }
}
```

**Проблемы:**
* Изменение правил валидации потребует правки класса
* Смена способа сохранения (например, на файловое хранилище) затронет этот же класс
* Обновление механизма нотификации повлияет на тот же код

**Исправленная версия (соблюдение SRP):**
```csharp
// Отвечает только за бизнес-логику
public class ProductValidator
{
    public void Validate(Product product)
    {
        if (product.Price <= 0)
            throw new ArgumentException("Invalid price");
        
        if (string.IsNullOrEmpty(product.Name))
            throw new ArgumentException("Name required");
    }
}

// Отвечает только за работу с хранилищем
public class ProductRepository
{
    public void Save(Product product)
    {
        using var context = new AppDbContext();
        context.Products.Add(product);
        context.SaveChanges();
    }
}

// Отвечает только за коммуникации
public class NotificationService
{
    public void SendProductAddedNotification(Product product)
    {
        EmailService.SendEmail("admin@example.com", $"New product: {product.Name}");
    }
}

// Координирует процессы
public class ProductService
{
    private readonly ProductValidator _validator;
    private readonly ProductRepository _repository;
    private readonly NotificationService _notification;

    public ProductService(
        ProductValidator validator,
        ProductRepository repository,
        NotificationService notification)
    {
        _validator = validator;
        _repository = repository;
        _notification = notification;
    }

    public void AddProduct(Product product)
    {
        _validator.Validate(product);
        _repository.Save(product);
        _notification.SendProductAddedNotification(product);
    }
}
```

### Пример 2: Класс-обработчик с множеством функций
**Нарушение:**
```csharp
public class FileProcessor
{
    public void ProcessFile(string path)
    {
        // 1. Чтение файла
        var content = File.ReadAllText(path);
        
        // 2. Преобразование данных
        var transformed = TransformContent(content);
        
        // 3. Валидация данных
        if (!Validate(transformed))
            throw new Exception("Invalid data");
        
        // 4. Сохранение в БД
        SaveToDatabase(transformed);
        
        // 5. Архивирование файла
        ArchiveFile(path);
    }
    
    private string TransformContent(string content) { /* ... */ }
    private bool Validate(string data) { /* ... */ }
    private void SaveToDatabase(string data) { /* ... */ }
    private void ArchiveFile(string path) { /* ... */ }
}
```

**Исправленная версия:**
```csharp
public class FileReader
{
    public string Read(string path) => File.ReadAllText(path);
}

public class ContentTransformer
{
    public string Transform(string content) { /* ... */ }
}

public class DataValidator
{
    public bool IsValid(string data) { /* ... */ }
}

public class DatabaseSaver
{
    public void Save(string data) { /* ... */ }
}

public class FileArchiver
{
    public void Archive(string path) { /* ... */ }
}

public class FileProcessor
{
    private readonly FileReader _reader;
    private readonly ContentTransformer _transformer;
    private readonly DataValidator _validator;
    private readonly DatabaseSaver _saver;
    private readonly FileArchiver _archiver;

    public FileProcessor(
        FileReader reader,
        ContentTransformer transformer,
        DataValidator validator,
        DatabaseSaver saver,
        FileArchiver archiver)
    {
        _reader = reader;
        _transformer = transformer;
        _validator = validator;
        _saver = saver;
        _archiver = archiver;
    }

    public void ProcessFile(string path)
    {
        var content = _reader.Read(path);
        var transformed = _transformer.Transform(content);
        
        if (!_validator.IsValid(transformed))
            throw new Exception("Invalid data");
        
        _saver.Save(transformed);
        _archiver.Archive(path);
    }
}
```

**Ключевые индикаторы нарушения SRP**
1. Класс имеет более 5-7 методов
2. Методы класса выполняют несвязанные операции
3. При изменении требований приходится менять один класс в разных местах
4. Класс имеет зависимости от множества внешних систем
5. Тесты для класса требуют сложных setup-ов

**Преимущества соблюдения SRP**
1. **Упрощение отладки:** Ошибки локализуются в конкретном компоненте
2. **Гибкость:** Замена реализации не затрагивает другие компоненты
3. **Тестируемость:** Компоненты тестируются изолированно
4. **Читаемость:** Каждый класс имеет четкое назначение
5. **Безопасность изменений:** Риск сломать существующую функциональность минимален

**Важно:** SRP не означает, что каждый класс должен иметь только один метод. 
Речь идет о единой ответственности, которая может реализовываться несколькими связанными методами.

## Заключение
Применяя SRP, Вы превращаете монолитный хаос в **архитектуру из четко очерченных "атомов" функциональности**. 
Каждый класс становится надежным, предсказуемым строительным блоком. 
Ваш код перестает быть хрупким: изменения локализованы, тесты пишутся за минуты, а понимание структуры приложения 
становится интуитивно понятным. SRP – это не просто "разделяй и властвуй", это фундамент для 
**устойчивой, адаптируемой и поддерживаемой кодовой базы** в любом масштабе.  
Помните: **фокус – это сила**.


[Оглавление](/README.md)         -> [Принцип Открытости/Закрытости (Open/Closed Principle - OCP)](/SOLID/OCP.md)