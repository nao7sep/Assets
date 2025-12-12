# MaybeVault Application Specification

## Overview

MaybeVault is an Avalonia UI MVVM desktop application for archiving files that users "probably won't need again" but want to keep just in case. It provides a low-friction way to store files with minimal metadata.

## Repository Structure

```
MaybeVault/
├── src/
│   ├── MaybeVault/                          # Avalonia UI application
│   │   ├── MaybeVault.csproj
│   │   ├── App.axaml
│   │   ├── App.axaml.cs
│   │   ├── Program.cs
│   │   ├── ViewLocator.cs
│   │   ├── Views/
│   │   │   ├── MainWindow.axaml
│   │   │   ├── MainWindow.axaml.cs
│   │   │   ├── AddFilesView.axaml
│   │   │   ├── AddFilesView.axaml.cs
│   │   │   ├── BrowseView.axaml
│   │   │   ├── BrowseView.axaml.cs
│   │   │   ├── SettingsView.axaml
│   │   │   ├── SettingsView.axaml.cs
│   │   │   └── Dialogs/
│   │   │       ├── DuplicateWarningDialog.axaml
│   │   │       ├── TagManagerDialog.axaml
│   │   │       └── CategoryManagerDialog.axaml
│   │   ├── Controls/
│   │   │   ├── TagSelector.axaml
│   │   │   ├── CategoryDropdown.axaml
│   │   │   ├── FileDropZone.axaml
│   │   │   └── RecentFilesList.axaml
│   │   ├── Converters/
│   │   │   ├── FileSizeConverter.cs
│   │   │   ├── RelativeTimeConverter.cs
│   │   │   └── TagIdToTagConverter.cs
│   │   └── Assets/
│   │       └── Icons/
│   └── MaybeVaultCore/                      # Business logic library
│       ├── MaybeVaultCore.csproj
│       ├── Models/
│       │   ├── VaultFile.cs
│       │   ├── VaultSettings.cs
│       │   ├── AddFileOptions.cs
│       │   ├── AddFileResult.cs
│       │   └── DuplicateCheckResult.cs
│       ├── Services/
│       │   ├── IVaultService.cs
│       │   ├── VaultService.cs
│       │   ├── ISettingsService.cs
│       │   ├── SettingsService.cs
│       │   └── IDuplicateDetector.cs
│       ├── Storage/
│       │   ├── IVaultStorage.cs
│       │   ├── VaultStorage.cs
│       │   ├── VaultMetadataStore.cs
│       │   └── VaultHashIndex.cs
│       ├── ViewModels/
│       │   ├── ViewModelBase.cs
│       │   ├── MainWindowViewModel.cs
│       │   ├── AddFilesViewModel.cs
│       │   ├── BrowseViewModel.cs
│       │   ├── SettingsViewModel.cs
│       │   ├── TagManagerViewModel.cs
│       │   └── CategoryManagerViewModel.cs
│       └── Queries/
│           ├── IVaultQuery.cs
│           └── VaultQuery.cs
├── tests/
│   └── MaybeVaultTests/
│       ├── MaybeVaultTests.csproj
│       ├── Services/
│       └── Storage/
└── MaybeVault.sln
```

## Namespace Structure

```csharp
// MaybeVaultCore project
MaybeVault.Core.Models
MaybeVault.Core.Services
MaybeVault.Core.Storage
MaybeVault.Core.ViewModels
MaybeVault.Core.Queries

// MaybeVault project (UI)
MaybeVault.Views
MaybeVault.Controls
MaybeVault.Converters
```

## Dependencies

### MaybeVaultCore

```xml
<ItemGroup>
    <PackageReference Include="CommunityToolkit.Mvvm" Version="8.4.*" />
    <ProjectReference Include="..\..\..\Nekote\src\Nekote\Nekote.csproj" />
    <!-- Or use NuGet package reference when Nekote is published:
    <PackageReference Include="Nekote" Version="0.1.0" /> -->
</ItemGroup>
```

### MaybeVault (UI)

```xml
<ItemGroup>
    <PackageReference Include="Avalonia" Version="11.*" />
    <PackageReference Include="Avalonia.Desktop" Version="11.*" />
    <PackageReference Include="Avalonia.Themes.Fluent" Version="11.*" />
    <PackageReference Include="Avalonia.Fonts.Inter" Version="11.*" />
    <PackageReference Include="Avalonia.Diagnostics" Version="11.*" Condition="'$(Configuration)' == 'Debug'" />
    <ProjectReference Include="..\MaybeVaultCore\MaybeVaultCore.csproj" />
</ItemGroup>
```

---

## Core Services

### IVaultService

The main facade for all vault operations.

```csharp
namespace MaybeVault.Core.Services;

public interface IVaultService
{
    // Initialization
    Task InitializeAsync(CancellationToken cancellationToken = default);
    bool IsInitialized { get; }

    // Core operations
    Task<AddFileResult> AddFileAsync(string sourcePath, AddFileOptions options, CancellationToken cancellationToken = default);
    Task<AddFileResult[]> AddFilesAsync(string[] sourcePaths, AddFileOptions options, CancellationToken cancellationToken = default);

    // Duplicate detection
    Task<DuplicateCheckResult> CheckDuplicateAsync(string filePath, CancellationToken cancellationToken = default);

    // Query
    IVaultQuery CreateQuery();
    Task<VaultFile[]> GetFilesAsync(IVaultQuery query, CancellationToken cancellationToken = default);
    Task<VaultFile?> GetFileByIdAsync(string id, CancellationToken cancellationToken = default);

    // Recent files
    Task<VaultFile[]> GetRecentFilesAsync(int count = 10, CancellationToken cancellationToken = default);

    // Tags & Categories
    Task<Tag[]> GetTagsAsync(CancellationToken cancellationToken = default);
    Task<Category[]> GetCategoriesAsync(CancellationToken cancellationToken = default);
    Task AddTagAsync(Tag tag, CancellationToken cancellationToken = default);
    Task UpdateTagAsync(Tag tag, CancellationToken cancellationToken = default);
    Task DeleteTagAsync(string tagId, CancellationToken cancellationToken = default);
    Task AddCategoryAsync(Category category, CancellationToken cancellationToken = default);
    Task UpdateCategoryAsync(Category category, CancellationToken cancellationToken = default);
    Task DeleteCategoryAsync(string categoryId, CancellationToken cancellationToken = default);

    // File access
    string GetStoredFilePath(VaultFile file);
    Task OpenFileLocationAsync(VaultFile file);
    Task OpenFileAsync(VaultFile file);

    // Statistics
    Task<VaultStatistics> GetStatisticsAsync(CancellationToken cancellationToken = default);
}
```

### ISettingsService

```csharp
namespace MaybeVault.Core.Services;

public interface ISettingsService
{
    VaultSettings Settings { get; }

    Task LoadAsync(CancellationToken cancellationToken = default);
    Task SaveAsync(CancellationToken cancellationToken = default);

    event EventHandler<SettingsChangedEventArgs>? SettingsChanged;
}
```

---

## ViewModels

### MainWindowViewModel

```csharp
namespace MaybeVault.Core.ViewModels;

public partial class MainWindowViewModel : ViewModelBase
{
    [ObservableProperty]
    private ViewModelBase _currentView;

    [ObservableProperty]
    private int _selectedNavigationIndex;

    public AddFilesViewModel AddFilesViewModel { get; }
    public BrowseViewModel BrowseViewModel { get; }
    public SettingsViewModel SettingsViewModel { get; }

    [RelayCommand]
    private void NavigateToAddFiles();

    [RelayCommand]
    private void NavigateToBrowse();

    [RelayCommand]
    private void NavigateToSettings();
}
```

### AddFilesViewModel

```csharp
namespace MaybeVault.Core.ViewModels;

public partial class AddFilesViewModel : ViewModelBase
{
    private readonly IVaultService _vaultService;

    [ObservableProperty]
    private ObservableCollection<PendingFile> _pendingFiles = new();

    [ObservableProperty]
    private Category? _selectedCategory;

    [ObservableProperty]
    private ObservableCollection<Tag> _selectedTags = new();

    [ObservableProperty]
    private string _notes = string.Empty;

    [ObservableProperty]
    private bool _isProcessing;

    [ObservableProperty]
    private ObservableCollection<VaultFile> _recentFiles = new();

    // Available options
    public ObservableCollection<Category> Categories { get; }
    public ObservableCollection<Tag> AvailableTags { get; }

    [RelayCommand]
    private async Task AddFilesFromPathsAsync(string[] paths);

    [RelayCommand]
    private async Task BrowseForFilesAsync();

    [RelayCommand]
    private async Task ArchiveAllAsync();

    [RelayCommand]
    private void RemovePendingFile(PendingFile file);

    [RelayCommand]
    private void ClearPendingFiles();

    [RelayCommand]
    private async Task OpenTagManagerAsync();

    [RelayCommand]
    private async Task OpenCategoryManagerAsync();
}

public class PendingFile : ObservableObject
{
    public string Path { get; init; }
    public string FileName { get; init; }
    public long Size { get; init; }
    public DuplicateCheckResult? DuplicateCheck { get; set; }
    public bool IsDuplicate => DuplicateCheck?.IsDuplicate ?? false;
    public bool IsChecking { get; set; }
    public bool ShouldSkip { get; set; }
}
```

### BrowseViewModel

```csharp
namespace MaybeVault.Core.ViewModels;

public partial class BrowseViewModel : ViewModelBase
{
    private readonly IVaultService _vaultService;

    [ObservableProperty]
    private string _searchText = string.Empty;

    [ObservableProperty]
    private ObservableCollection<string> _availableMonths = new();

    [ObservableProperty]
    private string? _selectedMonth;  // null = all

    [ObservableProperty]
    private Category? _filterCategory;

    [ObservableProperty]
    private ObservableCollection<Tag> _filterTags = new();

    [ObservableProperty]
    private ObservableCollection<VaultFile> _results = new();

    [ObservableProperty]
    private VaultFile? _selectedFile;

    [ObservableProperty]
    private bool _isSearching;

    [RelayCommand]
    private async Task SearchAsync();

    [RelayCommand]
    private async Task OpenFileAsync(VaultFile file);

    [RelayCommand]
    private async Task OpenFileLocationAsync(VaultFile file);

    [RelayCommand]
    private async Task CopyFileAsync(VaultFile file);

    [RelayCommand]
    private void ClearFilters();
}
```

### SettingsViewModel

```csharp
namespace MaybeVault.Core.ViewModels;

public partial class SettingsViewModel : ViewModelBase
{
    private readonly ISettingsService _settingsService;
    private readonly IVaultService _vaultService;

    [ObservableProperty]
    private string _storagePath = string.Empty;

    [ObservableProperty]
    private VaultStatistics? _statistics;

    [RelayCommand]
    private async Task BrowseStoragePathAsync();

    [RelayCommand]
    private async Task SaveSettingsAsync();

    [RelayCommand]
    private async Task RefreshStatisticsAsync();

    [RelayCommand]
    private async Task OpenStorageFolderAsync();
}
```

### TagManagerViewModel

```csharp
namespace MaybeVault.Core.ViewModels;

public partial class TagManagerViewModel : ViewModelBase
{
    [ObservableProperty]
    private ObservableCollection<Tag> _tags = new();

    [ObservableProperty]
    private Tag? _selectedTag;

    [ObservableProperty]
    private string _newTagName = string.Empty;

    [ObservableProperty]
    private string _newTagColor = "#3B82F6";

    [RelayCommand]
    private async Task AddTagAsync();

    [RelayCommand]
    private async Task UpdateTagAsync();

    [RelayCommand]
    private async Task DeleteTagAsync(Tag tag);
}
```

### CategoryManagerViewModel

```csharp
namespace MaybeVault.Core.ViewModels;

public partial class CategoryManagerViewModel : ViewModelBase
{
    [ObservableProperty]
    private ObservableCollection<Category> _categories = new();

    [ObservableProperty]
    private Category? _selectedCategory;

    [ObservableProperty]
    private string _newCategoryName = string.Empty;

    [ObservableProperty]
    private string _newCategoryIcon = "📁";

    [RelayCommand]
    private async Task AddCategoryAsync();

    [RelayCommand]
    private async Task UpdateCategoryAsync();

    [RelayCommand]
    private async Task DeleteCategoryAsync(Category category);

    [RelayCommand]
    private void MoveUp(Category category);

    [RelayCommand]
    private void MoveDown(Category category);
}
```

---

## Application Workflow

### Startup Flow

```
1. App.axaml.cs OnFrameworkInitializationCompleted
2. Load settings (ISettingsService.LoadAsync)
3. Initialize vault (IVaultService.InitializeAsync)
   - Verify storage path exists
   - Load tag/category stores
   - Build hash index if needed
4. Create MainWindowViewModel
5. Show MainWindow with AddFilesView as default
```

### Add Files Flow

```
1. User drops files or clicks browse
2. For each file:
   a. Compute SHA256 hash
   b. Check against hash index
   c. Show duplicate warning if found
3. User selects category and tags
4. User clicks "Archive"
5. For each file (not skipped):
   a. Determine month directory (YYYY-MM)
   b. Resolve filename conflicts
   c. Copy file to vault
   d. Create/update metadata JSON
   e. Update hash index
6. Show success, update recent files list
```

### Browse/Search Flow

```
1. User enters search text and/or selects filters
2. Build query from filters
3. Execute query across monthly metadata files
4. Display results grouped by month
5. User can:
   - Open file in default app
   - Open file location in explorer
   - Copy file to a new location
```

---

## Supported Platforms

| Platform | Status |
|----------|--------|
| Windows 11+ (Windows 10 22H2+) | Primary target |
| macOS 13+ (Ventura or later) | Supported |
| Linux (Ubuntu 22.04+) | Supported |

## Build Configurations

- **Debug**: Includes Avalonia DevTools, verbose logging
- **Release**: Optimized, no devtools, minimal logging

## Target Framework

```xml
<TargetFramework>net10.0</TargetFramework>
```

For platform-specific builds (if needed):
```xml
<TargetFrameworks>net10.0-windows;net10.0</TargetFrameworks>
```
