# File Management

## Overview

BrainLog uses file-based storage with one JSON file per entry. This enables:
- Sync via OneDrive/Dropbox (avoids corruption from partial syncs)
- Version control (git-friendly)
- Easy backup and recovery
- External editing if needed

## Directory Structure

### Standard Mode
```
%APPDATA%/BrainLog/           (Windows)
~/Library/Application Support/BrainLog/  (macOS)
├── Entries/
│   ├── 2025-12-13-{guid}.json
│   ├── 2025-12-12-{guid}.json
│   └── ...
├── AutoSaves/
│   ├── 2025-12-13-14-30-45-{guid}.json
│   └── ...
├── usersettings.json
└── prompts.json
```

### Portable Mode
```
{ExeDirectory}/
├── BrainLog.exe
├── appsettings.json
├── portable.marker           ← Triggers portable mode
├── Entries/
├── AutoSaves/
├── usersettings.json
└── prompts.json
```

## File Watching

### FileSystemWatcher Setup

```csharp
public class FileWatchService : IFileWatchService, IDisposable
{
    private FileSystemWatcher _watcher;
    private bool _selfWriteInProgress;

    public void StartWatching(string entriesDirectory)
    {
        _watcher = new FileSystemWatcher(entriesDirectory, "*.json")
        {
            NotifyFilter = NotifyFilters.LastWrite | NotifyFilters.FileName,
            EnableRaisingEvents = true
        };

        _watcher.Changed += OnFileChanged;
        _watcher.Deleted += OnFileDeleted;
        _watcher.Created += OnFileCreated;
        _watcher.Renamed += OnFileRenamed;
    }
}
```

### Event Debouncing

File saves often trigger multiple events. Debounce with ~300ms delay:

```csharp
private readonly Dictionary<string, CancellationTokenSource> _debounceTokens = new();

private async void OnFileChanged(object sender, FileSystemEventArgs e)
{
    var filePath = e.FullPath;

    // Cancel previous debounce for this file
    if (_debounceTokens.TryGetValue(filePath, out var existingCts))
    {
        existingCts.Cancel();
    }

    var cts = new CancellationTokenSource();
    _debounceTokens[filePath] = cts;

    try
    {
        await Task.Delay(300, cts.Token);
        _debounceTokens.Remove(filePath);

        // Process the change
        FileChanged?.Invoke(this, new FileChangedEventArgs(filePath));
    }
    catch (TaskCanceledException)
    {
        // Debounced - ignore
    }
}
```

### Self-Write Suppression

Prevent reacting to our own saves:

```csharp
public async Task SaveEntryAsync(Entry entry)
{
    _selfWriteInProgress = true;
    try
    {
        await WriteEntryFileAsync(entry);
        await Task.Delay(500); // Allow watcher events to fire and be ignored
    }
    finally
    {
        _selfWriteInProgress = false;
    }
}

private void OnFileChanged(...)
{
    if (_selfWriteInProgress) return;
    // ... handle external change
}
```

## Conflict Detection

### Three-State Tracking

```csharp
public class EntryEditContext
{
    public Entry Entry { get; set; }              // Domain model
    public string OriginalContent { get; set; }   // Content when opened
    public string OriginalDraft { get; set; }     // Draft when opened
    public string FileContent { get; set; }       // Current file content
    public string FileDraft { get; set; }         // Current file draft
    public bool HasExternalChange { get; private set; }
    public bool HasConflict { get; private set; }

    public void OnFileChanged(EntryDto newFileData)
    {
        FileContent = string.Join(Environment.NewLine, newFileData.Content ?? []);
        FileDraft = string.Join(Environment.NewLine, newFileData.Draft ?? []);

        // Check if file changed from original
        HasExternalChange =
            FileContent != OriginalContent ||
            FileDraft != OriginalDraft;

        // Check if conflict exists (file differs from editor AND editor has changes)
        HasConflict = HasExternalChange && (
            Entry.Content != FileContent ||
            Entry.Draft != FileDraft);
    }
}
```

### Conflict UI Response

When `HasConflict` becomes true:
1. Set warning border color on Draft/Content text areas
2. Update status bar: "File modified externally - changes differ from your edits"
3. Save button shows warning icon (but remains enabled)

When `HasExternalChange` but not `HasConflict`:
1. Update status bar: "File modified externally - matches your edits"
2. No visual warning needed

## Entry File Operations

### Loading Entries on Startup

```csharp
public async Task<IReadOnlyList<Entry>> LoadAllEntriesAsync()
{
    var entries = new List<Entry>();
    var directory = GetEntriesDirectory();

    if (!Directory.Exists(directory))
    {
        Directory.CreateDirectory(directory);
        return entries;
    }

    foreach (var file in Directory.EnumerateFiles(directory, "*.json"))
    {
        try
        {
            var dto = await ReadEntryFileAsync(file);
            var entry = EntryMapper.ToDomain(dto, file);
            entries.Add(entry);
        }
        catch (Exception ex)
        {
            // Report corrupt file to UI
            CorruptFileDetected?.Invoke(this, new CorruptFileEventArgs(file, ex.Message));
        }
    }

    return entries.OrderByDescending(e => e.CreatedUtc).ToList();
}
```

### Saving Entry

```csharp
public async Task SaveEntryAsync(Entry entry, bool isNew)
{
    var dto = EntryMapper.ToDto(entry);
    var fileName = $"{entry.CreatedUtc:yyyy-MM-dd}-{entry.Id}.json";
    var filePath = Path.Combine(GetEntriesDirectory(), fileName);

    var options = new JsonSerializerOptions
    {
        WriteIndented = true,
        DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull
    };

    _selfWriteInProgress = true;
    try
    {
        var json = JsonSerializer.Serialize(dto, options);
        await File.WriteAllTextAsync(filePath, json);
    }
    finally
    {
        await Task.Delay(500);
        _selfWriteInProgress = false;
    }
}
```

### Deleting Entry (to Trash)

```csharp
public void DeleteEntry(Entry entry)
{
    var fileName = $"{entry.CreatedUtc:yyyy-MM-dd}-{entry.Id}.json";
    var filePath = Path.Combine(GetEntriesDirectory(), fileName);

    if (File.Exists(filePath))
    {
        MoveToTrash(filePath);
    }
}

private void MoveToTrash(string filePath)
{
    // Use platform-specific trash API
    // Windows: SHFileOperation or Microsoft.VisualBasic.FileIO
    // macOS: NSFileManager.trashItem

    if (OperatingSystem.IsWindows())
    {
        Microsoft.VisualBasic.FileIO.FileSystem.DeleteFile(
            filePath,
            Microsoft.VisualBasic.FileIO.UIOption.OnlyErrorDialogs,
            Microsoft.VisualBasic.FileIO.RecycleOption.SendToRecycleBin);
    }
    else if (OperatingSystem.IsMacOS())
    {
        // Use NSFileManager via P/Invoke or third-party library
        MacOsTrash.MoveToTrash(filePath);
    }
}
```

## Auto-Save

### Auto-Save Service

```csharp
public class AutoSaveService : IAutoSaveService, IDisposable
{
    private readonly Timer _timer;
    private EntryEditContext? _currentContext;
    private string _lastSavedHash;

    public int AutoSaveCount { get; private set; }

    public void Start(EntryEditContext context)
    {
        _currentContext = context;
        _lastSavedHash = ComputeHash(context);
        AutoSaveCount = 0;

        _timer.Change(
            TimeSpan.FromSeconds(_options.IntervalSeconds),
            TimeSpan.FromSeconds(_options.IntervalSeconds));
    }

    private async void OnTimerElapsed(object? state)
    {
        if (_currentContext is null) return;

        var currentHash = ComputeHash(_currentContext);
        if (currentHash == _lastSavedHash) return; // No changes

        await SaveAutoBackupAsync(_currentContext);
        _lastSavedHash = currentHash;
        AutoSaveCount++;

        await CleanupOldBackupsAsync();
    }
}
```

### Auto-Save File Naming

```
{AutoSaveDir}/{yyyy-MM-dd-HH-mm-ss}-{entryGuid}.json
```

Example: `2025-12-13-14-30-45-a1b2c3d4-e5f6-7890-abcd-ef1234567890.json`

### Auto-Save Cleanup

```csharp
private async Task CleanupOldBackupsAsync()
{
    var directory = GetAutoSaveDirectory();
    var files = Directory.GetFiles(directory, "*.json")
        .OrderByDescending(f => f)
        .ToList();

    if (files.Count <= _options.MaxBackups) return;

    var toDelete = files.Skip(_options.MaxBackups);
    foreach (var file in toDelete)
    {
        try
        {
            File.Delete(file);
        }
        catch
        {
            // Best effort - log and continue
        }
    }
}
```

## Archive Cooldown Timer

### Timer Implementation

```csharp
public class ArchiveCooldownService
{
    private const int CheckIntervalSeconds = 180; // 3 minutes
    private readonly Timer _timer;

    public event EventHandler<Entry>? CooldownCompleted;

    public void StartMonitoring()
    {
        _timer = new Timer(
            CheckCooldowns,
            null,
            TimeSpan.FromSeconds(CheckIntervalSeconds),
            TimeSpan.FromSeconds(CheckIntervalSeconds));
    }

    private void CheckCooldowns(object? state)
    {
        var cooldownMinutes = _options.ArchiveCooldownMinutes;

        foreach (var entry in _handledEntries)
        {
            if (entry.HandledUtc is null) continue;

            var elapsed = DateTime.UtcNow - entry.HandledUtc.Value;
            if (elapsed.TotalMinutes >= cooldownMinutes)
            {
                CooldownCompleted?.Invoke(this, entry);
            }
        }
    }
}
```

### UI Response to Cooldown

When cooldown completes for currently displayed entry:
- Archive button becomes visible/enabled (without user interaction)
- Use Dispatcher to update UI from timer thread

## Portable Mode Detection

```csharp
public static class StorageLocationResolver
{
    public static string GetDataDirectory()
    {
        var exeDir = Path.GetDirectoryName(Assembly.GetExecutingAssembly().Location)!;
        var portableMarker = Path.Combine(exeDir, "portable.marker");

        // Check explicit setting first
        var settings = LoadAppSettings();
        if (settings?.Storage?.PortableMode == true)
        {
            return exeDir;
        }

        // Check for marker file
        if (File.Exists(portableMarker))
        {
            return exeDir;
        }

        // Standard location
        if (OperatingSystem.IsWindows())
        {
            return Path.Combine(
                Environment.GetFolderPath(Environment.SpecialFolder.ApplicationData),
                "BrainLog");
        }
        else if (OperatingSystem.IsMacOS())
        {
            return Path.Combine(
                Environment.GetFolderPath(Environment.SpecialFolder.UserProfile),
                "Library", "Application Support", "BrainLog");
        }

        throw new PlatformNotSupportedException();
    }
}
```
