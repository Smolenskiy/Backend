# Разбор контрольной 1, вариант 2

## Создание службы

Для корректного учета занятых участков поля необходимо реализовать службу, которая будет хранить эти данные.

### Модель данных

Можно организовать хранение информации об участках как минимум двумя разными способами:
1. Хранить информацию обо всех ячейках поля и при необходимости ее менять.
2. Хранить информацию только о занятых ячейках поля

Второй способ является более простым, поэтому выберем его. Для ячейки нам надо хранить следующую информацию: координаты, владелец, цвет.

Модель хранимых данных можно определить как:

```csharp
public class FieldOccupiedCell
{
    public String Coordinates { get; set; }
    public String Owner { get; set; }
    public String Color { get; set; }
}
```

Координаты ячейки можно хранить различными способами, например: 
* Парой координат (x, y)
* Единой координатой, собираемой из пары координат путем умножения одной из них на размерность поля (x * 10 + y)
* Конкатенированной строкой "x y"

В данном примере решения выберем 3-й способ.

### Реализация службы

Служба должна решать 2 задачи:
1. Выдавать информацию о текущих занятых участках
2. Добавлять новые занятые участки (если они свободны) или сообщать о том, что выбранные участки не могут быть добавлены т.к. заняты

Определим интерфейс службы как:

```csharp
public interface IFieldService
{
    /// <summary>
    /// Получаем список занятых участков поля
    /// </summary>
    List<FieldOccupiedCell> GetOccupiedCells();

    /// <summary>
    /// Пытаемся добавить новые участки поля.
    /// </summary>
    /// <param name="cells">Коллекция участков, которые необходимо занять.</param>
    /// <returns>true, если удалось успешно занять участки, false, если какой-то из указанных участков уже занят ранее</returns>
    Boolean AddOccupiedCells(List<FieldOccupiedCell> cells);
}
```

Саму службу реализуем следующим образом:

```csharp
public class FieldService : IFieldService
{
    /// <summary>
    /// Информация о занятых участках будет храниться в этом массиве.
    /// </summary>
    private readonly List<FieldOccupiedCell> storedCells = new List<FieldOccupiedCell>();

    /// <summary>
    /// Получаем список занятых участков поля
    /// </summary>
    public List<FieldOccupiedCell> GetOccupiedCells()
    {
        // Критическая секция - только один поток может взаимодействовать с массивом storedCells
        lock (this.storedCells)
        {
            // Т.к. только один поток может взаимодействовать с массивом - создадим копию.
            var copy = new List<FieldOccupiedCell>();
            for (var i = 0; i < this.storedCells.Count; i++)
            {
                copy.Add(this.storedCells[i]);
            }
            return copy;
        }
    }

    /// <summary>
    /// Пытаемся добавить новые участки поля.
    /// </summary>
    /// <param name="cells">Коллекция участков, которые необходимо занять.</param>
    /// <returns>true, если удалось успешно занять участки, false, если какой-то из указанных участков уже занят ранее</returns>
    public Boolean AddOccupiedCells(List<FieldOccupiedCell> cells)
    {
        // Критическая секция - только один поток может взаимодействовать с массивом storedCells
        lock (this.storedCells)
        {
            // Проверяем, не заняты ли уже участки
            for (var i = 0; i < cells.Count; i++)
            {
                for (var j = 0; j < this.storedCells.Count; j++)
                {
                    // Если уже хранится участок с такими координатами - ошибка
                    if (this.storedCells[j].Coordinates == cells[i].Coordinates)
                    {
                        return false;
                    }
                }
            }

            // Если дошли до этого места - значит ни одна из добавляемых ячеек не вызвала ошибку
            // Добавим их к нашему массиву storedCells
            for (var i = 0; i < cells.Count; i++)
            {
                this.storedCells.Add(cells[i]);
            }

            // Добавление прошло успешно
            return true;
        }

    }
}
```

После реализации службы необходимо ее зарегистрировать в `Startup.cs` (Метод `ConfigureServices`):

```csharp
public void ConfigureServices(IServiceCollection services)
{
    // Add framework services.
    services.AddMvc();
    services.AddSingleton<IFieldService, FieldService>();
}
```

## Реализация контроллера и представлений

### Создание контроллера

Добавим контроллер `FieldController` и подключим в него службу для хранения занятых участков:

```csharp
public class FieldController : Controller
{
    private readonly IFieldService fieldService;

    public FieldController(IFieldService fieldService)
    {
        this.fieldService = fieldService;
    }
}
```

### Первый action контроллера - отображение формы с полем.

Начнем реализацию контроллера с экшена `Index`, первой задачей которого будет являться отображение поля
Для начала добавим новый метод `IActionResult Index()` без параметров, который будет заниматься обработкой GET-запросов экшена `Index`.
Чтобы понять, какие участки поля заняты - получим эту информацию из службы `IFieldService` и передадим ее в представление (например через `ViewBag`):

```csharp
public IActionResult Index()
{
    this.ViewBag.OccupiedCells = this.fieldService.GetOccupiedCells();
    return this.View();
}
```

### Index.cshtml

Теперь скопируем представление `Mockups/Field.cshtml` в `Field/Index.cshtml`. Его модель мы пока не проектировали, поэтому оставим ее `dynamic`, но при этом разберем, как и какие участки нам надо изменить в связи с их занятостью.
В представлении `Mockups/FieldMarked.cshtml` можно увидеть примеры стилей для занятых участков из которых можно выделить следующие варианты:

Вариант 1 - Незанятый участок:

```html
<div class="col">
    <button class="btn btn-default"></button>
</div>
```

Вариант 2 - Занятый участок:

```html
<div class="col">
    <button class="btn btn-default" style="background-color: blue; color: white;">A</button>
</div>
```

Как видим, для занятого поля у кнопки устанавливаются:
* Текст, соответствующий владельцу
* Цвет текста - белый
* Цвет фона, соответствующий цвету участка

Поле строится двумя вложенными циклами (по строкам и столбцам). В теле внутреннего цикла нам необходимо определить, является ли участок занятым, и если да - оформить его соответствующе. Для этого проанализируем массив занятых ячеек (полученных от службы), на предмет наличия в нем этой информации:

```html
<form class="form-horizontal">
    @for (var row = 0; row < 10; row++)
    {
        <div class="row">
            @for (var col = 0; col < 10; col++)
            {
                String coordinates = row + " " + col;
                FieldOccupiedCell cell = null;
                for (var i = 0; i < ViewBag.OccupiedCells.Count; i++)
                {
                    if (ViewBag.OccupiedCells[i].Coordinates == coordinates)
                    {
                        cell = ViewBag.OccupiedCells[i];
                    }
                }
                if (cell != null)
                {
                    var color = cell.Color.ToLowerInvariant();
                    var owner = cell.Owner;
                    <div class="col">
                        <button class="btn btn-default" style="background-color: @color; color: white;">@owner</button>
                    </div>
                }
                else
                {
                    <div class="col">
                        <button class="btn btn-default"></button>
                    </div>
                }
            }
        </div>
    }
</form>
```

Для того, чтобы форма корректно отправлялась в нужны экшн необходимо установить у нее соответствующий атрибут `asp-action`:

```html
<form asp-action="Index" class="form-horizontal">
```

### Модель представления для Field/Index и определение нажатой кнопки

Для определения нажатой кнопки необходимо ей присвоить имя и значение, по которому мы сможем ее отличить от других. В нашем случае таким значением могут быть координаты ячейки (которые как раз представлены строкой):

```html
<button name="SelectedCell" class="btn btn-default" value="@coordinates"></button>
```

Теперь определим, какая модель представления нам нужна.  
Пользователь может выбрать несколько участков и мы можем, например, хранить массив содержащий координаты таких участков.

Определим модель представления как:

```csharp
public class FieldIndexViewModel
{
    public List<String> Selected { get; set; } = new List<String>();
}
```

Установим его как модель представления для `Field/Index.cshtml`:

```html
@model BackendTest1.Models.FieldIndexViewModel
```

И добавим метод `IActionResult Index(String selectedCell, FieldIndexViewModel model)` в контроллер для обработки выбора участка:

```csharp
[HttpPost]
[ValidateAntiForgeryToken]
public IActionResult Index(String selectedCell, FieldIndexViewModel model)
{
    // Снова передадим массив занятых ячеек
    this.ViewBag.OccupiedCells = this.fieldService.GetOccupiedCells();

    int index = -1;
    for (var i = 0; i < model.Selected.Count; i++)
    {
        // Т.к. мы будем добавлять/удалять элементы - сбросим ModelState для соответствующих полей
        this.ModelState.Remove("Selected[" + i + "]");

        if (model.Selected[i] == selectedCell)
        {
            // Участок был ранее выбран
            index = i;
        }
    }

    if (index == -1)
    {
        // Участок свободен - добавляем в список выбранных
        model.Selected.Add(selectedCell);
    }
    else
    {
        // Участок был выбран ранее - удаляем
        model.Selected.RemoveAt(index);
    }

    return this.View(model);
}
```

Помимо этого теперь необходимо передать модель в представление и в GET-версии:

```csharp
public IActionResult Index()
{
    this.ViewBag.OccupiedCells = this.fieldService.GetOccupiedCells();
    return this.View(new FieldIndexViewModel());
}
```

А также необходимо выделить выбранные участки на поле, для этого обратимся к представлению `Mockups/FieldSelected.cshtml` и выделим 3-й вариант отображения участка:

Вариант 3 - Выбранный участок:

```html
<div class="col">
    <button class="btn btn-default" style="border: 1px solid black"></button>
</div>
```

Помимо этого можно также выделить 4-й вариант, совмещающий 2 и 3:

Вариант 4 - Занятый и выбранный участок:

```html
<div class="col">
    <button class="btn btn-default" style="border: 1px solid black; background-color: blue; color: white;">A</button>
</div>
```

В результате итоговый вывод участка может принять следующий вид:

```html
String coordinates = row + " " + col;
FieldOccupiedCell cell = null;
for (var i = 0; i < ViewBag.OccupiedCells.Count; i++)
{
    if (ViewBag.OccupiedCells[i].Coordinates == coordinates)
    {
        cell = ViewBag.OccupiedCells[i];
    }
}
Boolean isSelected = false;
for (var i = 0; i < Model.Selected.Count; i++)
{
    if (Model.Selected[i] == coordinates)
    {
        isSelected = true;
    }
}
if (cell != null)
{
    var color = cell.Color.ToLowerInvariant();
    var owner = cell.Owner;
    if (isSelected)
    {
        <div class="col">
                <button name="SelectedCell" class="btn btn-default" value="@coordinates" style="border: 1px solid black; background-color: @color; color: white;">@owner</button>
        </div>
    }
    else
    {
        <div class="col">
                <button name="SelectedCell" class="btn btn-default" value="@coordinates" style="background-color: @color; color: white;">@owner</button>
        </div>
    }
}
else
{
    if (isSelected)
    {
        <div class="col">
                <button name="SelectedCell" class="btn btn-default" value="@coordinates" style="border: 1px solid black;"></button>
        </div>
    }
    else
    {
        <div class="col">
                <button name="SelectedCell" class="btn btn-default" value="@coordinates"></button>
        </div>
    }
}
```

Для того, чтобы модель представления корректно передавалась обратно в контроллер при отправке формы - добавить массив скрытых полей:

```html
@for (var i = 0; i < Model.Selected.Count; i++)
{
    <input type="hidden" asp-for="@Model.Selected[i]"/>
}
```

### Переход на следующую страницу

Для начала добавим вывод кнопки перехода на следующую страницу, если выбран хотя бы один участок:

```html
<form asp-action="Index" class="form-horizontal">
    @*Код вывода поля*@
    @if (Model.Selected.Count > 0)
    {
        <div class="form-group">
            <div class="col-md-10 col-md-offset-2">
                <button type="submit" name="SelectedCell" value="Build" class="btn btn-primary">Build</button>
            </div>
        </div>
    }
</form>
```

Установим у кнопки имя `SelectedCell` и значение `Build` которые будем использовать для определение того, была ли нажата она (альтернативно можно использовать еще один параметр).

В POST-методе `Index` добавим в начало метода проверку на то, что была нажата именно эта кнопка, и если да - выполним переход на следующий этап:

```csharp
if (selectedCell == "Build")
{
    return this.View("Build");
}
```

Скопируем представление `Mockups/Build.cshtml` в `Field/Build.cshtml`.

Для данного представления нам потребуется в модели еще два свойства: владелец и цвет. Определим их в классе, унаследованном от `FieldIndexViewModel`:

```csharp
public class FieldBuildViewModel : FieldIndexViewModel
{
    [Required]
    public String Owner { get; set; }

    [Required]
    public String Color { get; set; }
}
```

Укажем этот класс в качестве модели представления `Field/Build.cshtml`:

```html
@model BackendTest1.Models.FieldBuildViewModel
```

Установим у формы необходимый экшн:
```html
<form asp-action="Build" class="form-horizontal">
```

Установим у элементов формы привязку к соответствующим свойствам модели:

```html
<label asp-for="Owner" class="col-md-2 control-label">Owner</label>
<div class="col-md-10">
    <input asp-for="Owner" class="form-control" />
</div>
```

```html
<label asp-for="Color" class="col-md-2 control-label">Color</label>
<div class="col-md-10">
    <select asp-for="Color" class="form-control">
        <option>Red</option>
        <option>Green</option>
        <option>Blue</option>
    </select>
</div>
```

Добавим массив скрытых полей для выбранных участков:

```html
@for (var i = 0; i < Model.Selected.Count; i++)
{
    <input type="hidden" asp-for="@Model.Selected[i]"/>
}
```

Добавим элемент для вывода ошибок:
```html
<div asp-validation-summary="All"></div>
```

И реализуем соответствующий метод в контроллере:

```csharp
[HttpPost]
[ValidateAntiForgeryToken]
public IActionResult Build(FieldBuildViewModel model)
{
    // Проверяем что модель заполнена корректно
    if (!this.ModelState.IsValid)
    {
        return this.View(model);
    }

    // Собираем массив выбранных участков
    var cells = new List<FieldOccupiedCell>();
    for (var i = 0; i < model.Selected.Count; i++)
    {
        cells.Add(new FieldOccupiedCell
        {
            Coordinates = model.Selected[i],
            Owner = model.Owner,
            Color = model.Color
        });
    }

    // Пытаемся добавить его с помощью службы.
    if (!this.fieldService.AddOccupiedCells(cells))
    {
        // Если не получилось - выводим ошибку
        this.ModelState.AddModelError("", "Some of selected cells are already occupied");
        return this.View(model);
    }

    // Если получилось - отображаем отчет
    return this.View("Report", model);
}
```

Не забыв модифицировать переход из метода `Index`, передав корректную модель:

```csharp
return this.View("Build", new FieldBuildViewModel { Selected = model.Selected });
```

### Вывод отчета

Копируем представление `Mockups/Report.cshtml` в `Field/Report.cshtml`.
Устанавливаем у него представление `FieldBuildViewModel` и выведем переданные данные:

```html
@model BackendTest1.Models.FieldBuildViewModel
@{
    ViewBag.Title = "Report";
}

<h2>Report</h2>

<dl class="dl-horizontal">
    <dt>Owner</dt>
    <dd>@Model.Owner</dd>
    <dt>Color</dt>
    <dd>@Model.Color</dd>
    <dt>Built on cells:</dt>
    @{
        var cellsList = "";
        for (var i = 0; i < Model.Selected.Count; i++)
        {
            if (i != 0)
            {
                cellsList += ", ";
            }
            cellsList += "(" + Model.Selected[i] + ")";
        }
    }
    <dd>@cellsList</dd>
</dl>
```