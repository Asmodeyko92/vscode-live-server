# Windows 11 Widgets: кнопка для показа содержимого рабочего стола

Ниже — учебный пример архитектуры и кода для mini app (widget host), где:

1. Есть кнопка мини‑приложения.
2. Под ней есть вторая кнопка **"Показать рабочий стол"**.
3. По нажатию во второй кнопке в боковой панели отображается список файлов/папок с рабочего стола.

> Важно: для Windows 11 Widgets используйте официальный SDK/ограничения платформы. Если прямой доступ к файловой системе ограничен, показывайте данные через ваш backend/API или через разрешённый bridge слоя приложения.

---

## Идея

- **View (XAML)**: две кнопки и `ListView`.
- **ViewModel (C#)**: команда `LoadDesktopCommand`.
- **Service**: получает содержимое папки Desktop (`Environment.SpecialFolder.DesktopDirectory`).

Так код будет проще поддерживать и тестировать.

---

## Пример XAML (панель)

```xml
<StackPanel Spacing="8" Padding="12">
    <Button Content="Мини‑приложение" />

    <!-- Кнопка под мини‑приложением -->
    <Button Content="Показать рабочий стол"
            Command="{Binding LoadDesktopCommand}" />

    <ListView ItemsSource="{Binding DesktopItems}">
        <ListView.ItemTemplate>
            <DataTemplate>
                <StackPanel Orientation="Horizontal" Spacing="8">
                    <TextBlock Text="{Binding Kind}" Width="64" />
                    <TextBlock Text="{Binding Name}" />
                </StackPanel>
            </DataTemplate>
        </ListView.ItemTemplate>
    </ListView>
</StackPanel>
```

---

## Модель данных

```csharp
public sealed class DesktopEntry
{
    public string Name { get; init; } = string.Empty;
    public string Kind { get; init; } = string.Empty; // File или Folder
    public string FullPath { get; init; } = string.Empty;
}
```

---

## Сервис чтения рабочего стола

```csharp
using System;
using System.Collections.Generic;
using System.IO;
using System.Linq;

public interface IDesktopService
{
    IReadOnlyList<DesktopEntry> GetDesktopItems();
}

public sealed class DesktopService : IDesktopService
{
    public IReadOnlyList<DesktopEntry> GetDesktopItems()
    {
        var desktopPath = Environment.GetFolderPath(Environment.SpecialFolder.DesktopDirectory);
        if (string.IsNullOrWhiteSpace(desktopPath) || !Directory.Exists(desktopPath))
            return Array.Empty<DesktopEntry>();

        var dirs = Directory.GetDirectories(desktopPath)
            .Select(path => new DesktopEntry
            {
                Name = Path.GetFileName(path),
                Kind = "Folder",
                FullPath = path
            });

        var files = Directory.GetFiles(desktopPath)
            .Select(path => new DesktopEntry
            {
                Name = Path.GetFileName(path),
                Kind = "File",
                FullPath = path
            });

        return dirs.Concat(files)
                   .OrderBy(x => x.Kind)
                   .ThenBy(x => x.Name)
                   .ToList();
    }
}
```

---

## ViewModel

```csharp
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;
using System.Collections.ObjectModel;

public partial class WidgetPanelViewModel : ObservableObject
{
    private readonly IDesktopService _desktopService;

    public ObservableCollection<DesktopEntry> DesktopItems { get; } = new();

    public WidgetPanelViewModel(IDesktopService desktopService)
    {
        _desktopService = desktopService;
    }

    [RelayCommand]
    private void LoadDesktop()
    {
        DesktopItems.Clear();

        var items = _desktopService.GetDesktopItems();
        foreach (var item in items)
            DesktopItems.Add(item);
    }
}
```

---

## Что здесь происходит (по шагам)

1. Пользователь нажимает кнопку **"Показать рабочий стол"**.
2. Вызывается команда `LoadDesktopCommand`.
3. Команда обращается к `DesktopService`.
4. `DesktopService` читает папку рабочего стола и формирует список `DesktopEntry`.
5. `DesktopItems` обновляется, а `ListView` автоматически перерисовывается через data binding.

---

## Практические улучшения

- Добавить иконки файлов (по расширению).
- Добавить фильтр (например, показывать только документы).
- Добавить обработку ошибок/доступа (показ сообщения пользователю).
- Сделать асинхронную загрузку (`Task`, `AsyncRelayCommand`) для больших папок.
