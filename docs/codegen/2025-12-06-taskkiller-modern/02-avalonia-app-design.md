# Avalonia App Design

Architecture and implementation guide for a modern Avalonia UI application that provides CRUD operations for taskKiller data.

## Table of Contents

1. [Project Structure](#project-structure)
2. [Solution Setup](#solution-setup)
3. [Data Layer (TaskKillerLib)](#data-layer-taskkillerlib)
4. [UI Layer (TaskKillerApp)](#ui-layer-taskkillerapp)
5. [MVVM Implementation](#mvvm-implementation)
6. [Key Features](#key-features)
7. [Styling](#styling)
8. [Cross-Platform Considerations](#cross-platform-considerations)

---

## Project Structure

```
TaskKillerModern/
├── TaskKillerModern.sln
├── src/
│   ├── TaskKillerLib/                    # Class library
│   │   ├── TaskKillerLib.csproj
│   │   ├── Models/
│   │   │   ├── TaskDto.cs
│   │   │   ├── TaskState.cs
│   │   │   ├── NoteDto.cs
│   │   │   ├── NotePart.cs
│   │   │   ├── NotePartType.cs
│   │   │   └── FileAttachmentDto.cs
│   │   ├── Storage/
│   │   │   ├── ITaskRepository.cs
│   │   │   ├── FileSystemTaskRepository.cs
│   │   │   ├── TaskFileReader.cs
│   │   │   ├── TaskFileWriter.cs
│   │   │   └── AttachmentManager.cs
│   │   ├── Parsing/
│   │   │   ├── TextEscaping.cs
│   │   │   ├── ParagraphParser.cs
│   │   │   ├── KeyValueParser.cs
│   │   │   └── NoteContentParser.cs
│   │   └── Utilities/
│   │       └── GuidExtensions.cs
│   │
│   └── TaskKillerApp/                    # Avalonia UI app
│       ├── TaskKillerApp.csproj
│       ├── App.axaml
│       ├── App.axaml.cs
│       ├── ViewModels/
│       │   ├── ViewModelBase.cs
│       │   ├── MainWindowViewModel.cs
│       │   ├── TaskListViewModel.cs
│       │   ├── TaskViewModel.cs
│       │   ├── NoteViewModel.cs
│       │   └── NotePartViewModel.cs
│       ├── Views/
│       │   ├── MainWindow.axaml
│       │   ├── MainWindow.axaml.cs
│       │   ├── TaskListView.axaml
│       │   ├── NoteListView.axaml
│       │   ├── NoteContentView.axaml
│       │   └── Dialogs/
│       │       ├── TaskEditDialog.axaml
│       │       └── NoteEditDialog.axaml
│       ├── Services/
│       │   ├── IDialogService.cs
│       │   └── DialogService.cs
│       ├── Converters/
│       │   ├── TaskStateToColorConverter.cs
│       │   └── BoolToVisibilityConverter.cs
│       └── Styles/
│           └── Styles.axaml
│
└── tests/
    └── TaskKillerLib.Tests/
        ├── TaskKillerLib.Tests.csproj
        ├── TextEscapingTests.cs
        ├── ParagraphParserTests.cs
        └── NoteContentParserTests.cs
```

---

## Solution Setup

### TaskKillerLib.csproj

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
    <LangVersion>latest</LangVersion>
  </PropertyGroup>
</Project>
```

### TaskKillerApp.csproj

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>WinExe</OutputType>
    <TargetFramework>net8.0</TargetFramework>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
    <BuiltInComInteropSupport>true</BuiltInComInteropSupport>
    <ApplicationManifest>app.manifest</ApplicationManifest>
    <AvaloniaUseCompiledBindingsByDefault>true</AvaloniaUseCompiledBindingsByDefault>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Avalonia" Version="11.2.*" />
    <PackageReference Include="Avalonia.Desktop" Version="11.2.*" />
    <PackageReference Include="Avalonia.Themes.Fluent" Version="11.2.*" />
    <PackageReference Include="Avalonia.Fonts.Inter" Version="11.2.*" />
    <PackageReference Include="Avalonia.Diagnostics" Version="11.2.*" Condition="'$(Configuration)' == 'Debug'" />
    <PackageReference Include="CommunityToolkit.Mvvm" Version="8.*" />
  </ItemGroup>

  <ItemGroup>
    <ProjectReference Include="..\TaskKillerLib\TaskKillerLib.csproj" />
  </ItemGroup>
</Project>
```

---

## Data Layer (TaskKillerLib)

### ITaskRepository Interface

```csharp
public interface ITaskRepository
{
    // Configuration
    string CurrentTaskListPath { get; }
    void SetTaskListPath(string path);

    // Task operations
    Task<IReadOnlyList<TaskDto>> GetActiveTasksAsync(CancellationToken ct = default);
    Task<IReadOnlyList<TaskDto>> GetCompletedTasksAsync(CancellationToken ct = default);
    Task<TaskDto?> GetTaskAsync(Guid id, CancellationToken ct = default);
    Task<TaskDto> CreateTaskAsync(string content, TaskState state, CancellationToken ct = default);
    Task UpdateTaskAsync(TaskDto task, CancellationToken ct = default);
    Task DeleteTaskAsync(Guid id, CancellationToken ct = default);

    // State transitions
    Task SetTaskStateAsync(Guid id, TaskState newState, CancellationToken ct = default);
    Task MoveTaskAsync(Guid id, int newIndex, CancellationToken ct = default);

    // Note operations
    Task<NoteDto> AddNoteAsync(Guid taskId, string content, CancellationToken ct = default);
    Task UpdateNoteAsync(Guid taskId, NoteDto note, CancellationToken ct = default);
    Task DeleteNoteAsync(Guid taskId, Guid noteId, CancellationToken ct = default);

    // Attachment operations
    Task<IReadOnlyList<FileAttachmentDto>> GetAttachmentsAsync(CancellationToken ct = default);
    Task<IReadOnlyList<FileAttachmentDto>> GetAttachmentsForParentAsync(Guid parentId, CancellationToken ct = default);
    Task<FileAttachmentDto> AttachFileAsync(string sourcePath, Guid? parentId, CancellationToken ct = default);
    string GetAttachmentFullPath(FileAttachmentDto attachment);

    // Task list discovery
    Task<IReadOnlyList<string>> DiscoverTaskListsAsync(string rootPath, CancellationToken ct = default);
}
```

---

## UI Layer (TaskKillerApp)

### App.axaml

```xml
<Application xmlns="https://github.com/avaloniaui"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             x:Class="TaskKillerApp.App"
             RequestedThemeVariant="Default">
    <Application.Styles>
        <FluentTheme />
        <StyleInclude Source="avares://TaskKillerApp/Styles/Styles.axaml" />
    </Application.Styles>
</Application>
```

### MainWindow.axaml

```xml
<Window xmlns="https://github.com/avaloniaui"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:vm="using:TaskKillerApp.ViewModels"
        xmlns:views="using:TaskKillerApp.Views"
        x:Class="TaskKillerApp.Views.MainWindow"
        x:DataType="vm:MainWindowViewModel"
        Title="{Binding Title}"
        Width="1200" Height="800">

    <Grid ColumnDefinitions="*, 3, *">
        <!-- Task List -->
        <views:TaskListView Grid.Column="0"
                            DataContext="{Binding TaskList}" />

        <GridSplitter Grid.Column="1"
                      ResizeDirection="Columns" />

        <!-- Notes for Selected Task -->
        <views:NoteListView Grid.Column="2"
                            DataContext="{Binding SelectedTask}" />
    </Grid>
</Window>
```

---

## MVVM Implementation

### ViewModelBase

```csharp
using CommunityToolkit.Mvvm.ComponentModel;

namespace TaskKillerApp.ViewModels;

public abstract class ViewModelBase : ObservableObject
{
}
```

### MainWindowViewModel

```csharp
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;
using TaskKillerLib.Storage;

namespace TaskKillerApp.ViewModels;

public partial class MainWindowViewModel : ViewModelBase
{
    private readonly ITaskRepository _repository;
    private readonly IDialogService _dialogService;

    [ObservableProperty]
    private string _title = "taskKiller Modern";

    [ObservableProperty]
    private TaskListViewModel _taskList;

    [ObservableProperty]
    private TaskViewModel? _selectedTask;

    public MainWindowViewModel(ITaskRepository repository, IDialogService dialogService)
    {
        _repository = repository;
        _dialogService = dialogService;
        _taskList = new TaskListViewModel(repository, dialogService);

        _taskList.PropertyChanged += (s, e) =>
        {
            if (e.PropertyName == nameof(TaskListViewModel.SelectedTask))
                SelectedTask = _taskList.SelectedTask;
        };
    }

    [RelayCommand]
    private async Task OpenTaskListAsync()
    {
        var path = await _dialogService.ShowFolderPickerAsync();
        if (path is not null)
        {
            _repository.SetTaskListPath(path);
            await TaskList.LoadAsync();
            Title = $"taskKiller Modern - {Path.GetFileName(path)}";
        }
    }
}
```

### TaskViewModel

```csharp
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;
using TaskKillerLib.Models;

namespace TaskKillerApp.ViewModels;

public partial class TaskViewModel : ViewModelBase
{
    private readonly TaskDto _dto;
    private readonly ITaskRepository _repository;

    public Guid Guid => _dto.Guid;
    public DateTime CreationUtc => _dto.CreationUtc;

    [ObservableProperty]
    private string _content;

    [ObservableProperty]
    private TaskState _state;

    [ObservableProperty]
    private bool _isSpecial;

    public ObservableCollection<NoteViewModel> Notes { get; } = new();

    public TaskViewModel(TaskDto dto, ITaskRepository repository)
    {
        _dto = dto;
        _repository = repository;
        _content = dto.Content;
        _state = dto.State;
        _isSpecial = dto.IsSpecial;

        foreach (var note in dto.Notes)
            Notes.Add(new NoteViewModel(note, this, repository));
    }

    [RelayCommand]
    private async Task MarkAsDoneAsync()
    {
        await _repository.SetTaskStateAsync(Guid, TaskState.Done);
        State = TaskState.Done;
    }

    [RelayCommand]
    private async Task CancelAsync()
    {
        await _repository.SetTaskStateAsync(Guid, TaskState.Cancelled);
        State = TaskState.Cancelled;
    }

    [RelayCommand]
    private async Task SetStateAsync(TaskState newState)
    {
        await _repository.SetTaskStateAsync(Guid, newState);
        State = newState;
    }

    public TaskDto ToDto() => _dto with
    {
        Content = Content,
        State = State,
        Notes = Notes.Select(n => n.ToDto()).ToList()
    };
}
```

### NoteViewModel with AI Section Support

```csharp
using CommunityToolkit.Mvvm.ComponentModel;
using TaskKillerLib.Models;
using TaskKillerLib.Parsing;

namespace TaskKillerApp.ViewModels;

public partial class NoteViewModel : ViewModelBase
{
    private readonly NoteDto _dto;
    private readonly TaskViewModel _parent;
    private readonly ITaskRepository _repository;

    public Guid Guid => _dto.Guid;
    public DateTime CreationUtc => _dto.CreationUtc;

    [ObservableProperty]
    private string _content;

    // Parsed parts for rendering
    public ObservableCollection<NotePartViewModel> Parts { get; } = new();

    public NoteViewModel(NoteDto dto, TaskViewModel parent, ITaskRepository repository)
    {
        _dto = dto;
        _parent = parent;
        _repository = repository;
        _content = dto.Content;

        RefreshParts();
    }

    partial void OnContentChanged(string value)
    {
        RefreshParts();
    }

    private void RefreshParts()
    {
        Parts.Clear();
        var parsed = NoteContentParser.Parse(Content);
        foreach (var part in parsed)
            Parts.Add(new NotePartViewModel(part));
    }

    public NoteDto ToDto() => _dto with { Content = Content };
}

public class NotePartViewModel : ViewModelBase
{
    public NotePartType Type { get; }
    public string Content { get; }
    public bool IsAiResponse => Type == NotePartType.AiResponse;

    public NotePartViewModel(NotePart part)
    {
        Type = part.Type;
        Content = part.Content;
    }
}
```

---

## Key Features

### Task List View

```xml
<UserControl xmlns="https://github.com/avaloniaui"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             xmlns:vm="using:TaskKillerApp.ViewModels"
             x:Class="TaskKillerApp.Views.TaskListView"
             x:DataType="vm:TaskListViewModel">

    <DockPanel>
        <!-- Toolbar -->
        <StackPanel DockPanel.Dock="Top" Orientation="Horizontal" Spacing="4" Margin="8">
            <Button Content="New Task" Command="{Binding CreateTaskCommand}" />
            <Button Content="↑" Command="{Binding MoveUpCommand}" />
            <Button Content="↓" Command="{Binding MoveDownCommand}" />
            <Separator />
            <Button Content="Done" Command="{Binding SelectedTask.MarkAsDoneCommand}" />
            <Button Content="Cancel" Command="{Binding SelectedTask.CancelCommand}" />
        </StackPanel>

        <!-- Task List -->
        <ListBox ItemsSource="{Binding Tasks}"
                 SelectedItem="{Binding SelectedTask}"
                 SelectionMode="Single">
            <ListBox.ItemTemplate>
                <DataTemplate x:DataType="vm:TaskViewModel">
                    <Border Padding="8"
                            Classes.special="{Binding IsSpecial}">
                        <StackPanel>
                            <TextBlock Text="{Binding Content}"
                                       TextWrapping="Wrap"
                                       FontWeight="SemiBold" />
                            <StackPanel Orientation="Horizontal" Spacing="8" Margin="0,4,0,0">
                                <TextBlock Text="{Binding State}"
                                           Classes.later="{Binding State, Converter={StaticResource StateConverter}, ConverterParameter=Later}"
                                           Classes.soon="{Binding State, Converter={StaticResource StateConverter}, ConverterParameter=Soon}"
                                           Classes.now="{Binding State, Converter={StaticResource StateConverter}, ConverterParameter=Now}" />
                                <TextBlock Text="{Binding Notes.Count, StringFormat='({0} notes)'}"
                                           Opacity="0.6" />
                            </StackPanel>
                        </StackPanel>
                    </Border>
                </DataTemplate>
            </ListBox.ItemTemplate>
        </ListBox>
    </DockPanel>
</UserControl>
```

### Note Content View with AI Sections

```xml
<UserControl xmlns="https://github.com/avaloniaui"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             xmlns:vm="using:TaskKillerApp.ViewModels"
             x:Class="TaskKillerApp.Views.NoteContentView"
             x:DataType="vm:NoteViewModel">

    <ItemsControl ItemsSource="{Binding Parts}">
        <ItemsControl.ItemTemplate>
            <DataTemplate x:DataType="vm:NotePartViewModel">
                <Border Classes.aiSection="{Binding IsAiResponse}"
                        Margin="0,4">
                    <SelectableTextBlock Text="{Binding Content}"
                                         TextWrapping="Wrap" />
                </Border>
            </DataTemplate>
        </ItemsControl.ItemTemplate>
    </ItemsControl>
</UserControl>
```

### Drag-and-Drop for Attachments

```csharp
// In MainWindow.axaml.cs
private void OnDragOver(object? sender, DragEventArgs e)
{
    e.DragEffects = e.Data.Contains(DataFormats.Files)
        ? DragDropEffects.Copy
        : DragDropEffects.None;
}

private async void OnDrop(object? sender, DragEventArgs e)
{
    if (!e.Data.Contains(DataFormats.Files))
        return;

    var files = e.Data.GetFiles();
    if (files is null)
        return;

    var vm = DataContext as MainWindowViewModel;
    if (vm is null)
        return;

    // Determine parent based on what's focused/selected
    Guid? parentGuid = null;
    if (vm.SelectedTask is not null)
        parentGuid = vm.SelectedTask.Guid;

    foreach (var file in files)
    {
        if (file.Path.IsFile)
        {
            await vm.AttachFileAsync(file.Path.LocalPath, parentGuid);
        }
    }
}
```

---

## Styling

### Styles.axaml

```xml
<Styles xmlns="https://github.com/avaloniaui"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">

    <!-- Task state colors -->
    <Style Selector="TextBlock.later">
        <Setter Property="Foreground" Value="Gray" />
    </Style>
    <Style Selector="TextBlock.soon">
        <Setter Property="Foreground" Value="DodgerBlue" />
        <Setter Property="FontWeight" Value="SemiBold" />
    </Style>
    <Style Selector="TextBlock.now">
        <Setter Property="Foreground" Value="OrangeRed" />
        <Setter Property="FontWeight" Value="Bold" />
    </Style>

    <!-- Special (imported) task highlighting -->
    <Style Selector="Border.special">
        <Setter Property="Background" Value="#FFFEF3" />
        <Setter Property="BorderBrush" Value="#FFE066" />
        <Setter Property="BorderThickness" Value="0,0,0,2" />
    </Style>

    <!-- AI response section styling -->
    <Style Selector="Border.aiSection">
        <Setter Property="Background" Value="#F0F7FF" />
        <Setter Property="BorderBrush" Value="#4A90D9" />
        <Setter Property="BorderThickness" Value="3,0,0,0" />
        <Setter Property="Padding" Value="12,8" />
        <Setter Property="CornerRadius" Value="0,4,4,0" />
    </Style>

    <!-- Note list item -->
    <Style Selector="ListBoxItem">
        <Setter Property="Padding" Value="8" />
    </Style>
</Styles>
```

---

## Cross-Platform Considerations

### File Paths

```csharp
// Always use Path.Combine for cross-platform compatibility
var taskPath = Path.Combine(taskListPath, "Tasks", $"{guid:D}.txt");

// Normalize directory separators when reading from files
var normalizedPath = relativePath.Replace('\\', Path.DirectorySeparatorChar);
```

### Line Endings

```csharp
// Write with CRLF for taskKiller compatibility
var content = BuildFileContent();
await File.WriteAllTextAsync(path, content.Replace("\n", "\r\n"), Encoding.UTF8);

// Read tolerating both
var text = await File.ReadAllTextAsync(path, Encoding.UTF8);
// Parser handles both \r\n and \n via TrimEnd('\r')
```

### Encoding

```csharp
// Always use UTF-8
var encoding = new UTF8Encoding(encoderShouldEmitUTF8Identifier: false);
await File.WriteAllTextAsync(path, content, encoding);

// Handle BOM on read (File.ReadAllText does this automatically)
```

### File System Watching (Optional)

```csharp
public class FileWatchingTaskRepository : FileSystemTaskRepository
{
    private FileSystemWatcher? _watcher;

    public event EventHandler? TasksChanged;

    public override void SetTaskListPath(string path)
    {
        base.SetTaskListPath(path);

        _watcher?.Dispose();
        _watcher = new FileSystemWatcher(Path.Combine(path, "Tasks"))
        {
            Filter = "*.txt",
            NotifyFilter = NotifyFilters.LastWrite | NotifyFilters.FileName
        };

        _watcher.Changed += (s, e) => TasksChanged?.Invoke(this, EventArgs.Empty);
        _watcher.Created += (s, e) => TasksChanged?.Invoke(this, EventArgs.Empty);
        _watcher.Deleted += (s, e) => TasksChanged?.Invoke(this, EventArgs.Empty);
        _watcher.EnableRaisingEvents = true;
    }
}
```

---

## Summary

| Component | Technology | Purpose |
|-----------|------------|---------|
| TaskKillerLib | .NET 8 Class Library | Data access, file parsing, business logic |
| TaskKillerApp | Avalonia 11 | Cross-platform UI |
| MVVM | CommunityToolkit.Mvvm | Property binding, commands |
| Styling | Fluent Theme + Custom | Consistent look, state indicators |

### Key Implementation Notes

1. **Repository pattern** abstracts file system access
2. **Async/await** throughout for responsive UI
3. **ObservableCollection** for automatic UI updates
4. **AI sections** rendered with distinct styling via `NotePartViewModel`
5. **Drag-drop** for file attachments
6. **Cross-platform** file handling (paths, line endings, encoding)
