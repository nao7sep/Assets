# UI Design Specification

## Overview

MaybeVault uses Avalonia UI with the Fluent theme for a modern, cross-platform appearance. The design philosophy is **"fire-and-forget primary, search secondary"** - the main purpose is quick archiving, with browse/search as a supporting feature.

---

## Main Window Layout

```
┌─────────────────────────────────────────────────────────────────┐
│  MaybeVault                                       [─] [□] [×]   │
├────────┬────────────────────────────────────────────────────────┤
│        │                                                        │
│  📥    │                                                        │
│  Add   │              [ Current View Content ]                  │
│        │                                                        │
│  🔍    │                                                        │
│ Browse │                                                        │
│        │                                                        │
│  ⚙️    │                                                        │
│Settings│                                                        │
│        │                                                        │
├────────┴────────────────────────────────────────────────────────┤
│  Status: Ready                                      v1.0.0      │
└─────────────────────────────────────────────────────────────────┘
```

### Navigation

- Left sidebar with icon buttons
- Three main views: Add, Browse, Settings
- Default view: Add (most common action)
- Status bar for feedback

---

## Add Files View

The primary view for archiving files.

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│    ┌───────────────────────────────────────────────────────┐   │
│    │                                                       │   │
│    │                                                       │   │
│    │         📁  Drop files here to archive               │   │
│    │                                                       │   │
│    │              or click to browse                       │   │
│    │                                                       │   │
│    │                                                       │   │
│    └───────────────────────────────────────────────────────┘   │
│                                                                 │
│    ─────────────── Pending Files (3) ───────────────           │
│                                                                 │
│    ┌─────────────────────────────────────────────────────────┐ │
│    │ 📄 homework-math.docx           15.2 KB    [×]          │ │
│    │ 📷 image.jpg                    245 KB     [×]          │ │
│    │ ⚠️ photo.png (duplicate)         1.2 MB    [×] [View]   │ │
│    └─────────────────────────────────────────────────────────┘ │
│                                                                 │
│    Category:  [ 📚 Homework                    ▼]  [+]         │
│                                                                 │
│    Tags:      [math ×] [daughter ×]            [+ Add Tag]     │
│                                                                 │
│    Notes:     ┌─────────────────────────────────────────────┐  │
│               │ Multiplication practice worksheet            │  │
│               └─────────────────────────────────────────────┘  │
│                                                                 │
│                                   [Clear]  [Archive 2 Files]   │
│                                                                 │
│    ─────────────── Recently Archived ───────────────           │
│                                                                 │
│    📄 notes.txt             2 minutes ago     [Open]           │
│    📷 receipt.jpg           1 hour ago        [Open]           │
│    📄 document.docx         yesterday         [Open]           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Elements

#### Drop Zone
- Large, prominent area
- Accepts drag & drop
- Click to open file picker
- Visual feedback on hover/drag

#### Pending Files List
- Shows files queued for archiving
- Displays filename, size, status
- Duplicate warnings with ⚠️ icon
- Remove button for each file
- "View" button for duplicates (shows existing file details)

#### Category Selector
- Dropdown with all categories
- [+] button opens category manager
- Icon displayed with name

#### Tag Selector
- Selected tags shown as pills
- Click [×] to remove
- [+ Add Tag] opens tag picker/manager
- Multi-select support

#### Notes Field
- Optional text area
- Applied to all files in batch
- Placeholder text: "Add a note about these files..."

#### Action Buttons
- **Clear**: Remove all pending files
- **Archive N Files**: Main action button
- Count excludes skipped duplicates

#### Recently Archived
- Shows last N archived files
- Quick access to recently added items
- Click to open file location

---

## Browse View

For finding archived files.

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│    Search: [                                         🔍]       │
│                                                                 │
│    ┌─ Filters ────────────────────────────────────────────────┐│
│    │ Month:    [All      ▼]  Category: [All        ▼]        ││
│    │ Tags:     [Select tags...                    ]          ││
│    │                                      [Clear] [Search]   ││
│    └──────────────────────────────────────────────────────────┘│
│                                                                 │
│    ─────────────── Results (24 files) ───────────────          │
│                                                                 │
│    ▼ 2024-07 (12 files)                                        │
│      ┌─────────────────────────────────────────────────────┐   │
│      │ ☐ 📄 homework-math.docx     15.2 KB    Jul 16      │   │
│      │     📚 Homework  [math] [daughter]                  │   │
│      ├─────────────────────────────────────────────────────┤   │
│      │ ☑ 📷 image.jpg              245 KB     Jul 18      │   │
│      │     (uncategorized)                                 │   │
│      ├─────────────────────────────────────────────────────┤   │
│      │ ☐ 📷 image(2).jpg           312 KB     Jul 20      │   │
│      │     📷 Photos  [family]                             │   │
│      └─────────────────────────────────────────────────────┘   │
│                                                                 │
│    ▶ 2024-06 (8 files)                                         │
│    ▶ 2024-05 (4 files)                                         │
│                                                                 │
│    ───────────────────────────────────────────────────────     │
│    Selected: 1 file (245 KB)                                   │
│                       [Open File]  [Open Location]  [Copy To]  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Elements

#### Search Bar
- Full-text search across filename and notes
- Instant search (debounced)
- Clear button when text present

#### Filters Panel
- **Month**: Dropdown with "All" + available months
- **Category**: Dropdown with "All" + categories
- **Tags**: Multi-select tag picker
- **Clear**: Reset all filters
- **Search**: Execute search (also auto-searches)

#### Results List
- Grouped by month (collapsible)
- File info: icon, name, size, date
- Category and tags displayed below name
- Checkbox for multi-select
- Click to select, double-click to open

#### Action Bar
- Shows selection summary
- **Open File**: Opens in default application
- **Open Location**: Opens folder in file explorer
- **Copy To**: Copies file to chosen location

---

## Settings View

Configuration and statistics.

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│    ─────────────── Storage ───────────────                     │
│                                                                 │
│    Vault Location:                                              │
│    ┌─────────────────────────────────────────────────┐ [📁]    │
│    │ C:\Users\User\Documents\MaybeVault              │         │
│    └─────────────────────────────────────────────────┘         │
│                                              [Open Folder]      │
│                                                                 │
│    ─────────────── Behavior ───────────────                    │
│                                                                 │
│    [✓] Warn when adding duplicate files                        │
│    [✓] Copy files (uncheck to move)                            │
│    [ ] Delete source files after archiving                     │
│                                                                 │
│    Default Category: [ None                    ▼]              │
│    Recent files to show: [10    ]                              │
│                                                                 │
│    ─────────────── Tags & Categories ───────────────           │
│                                                                 │
│    [Manage Tags]        [Manage Categories]                    │
│                                                                 │
│    ─────────────── Statistics ───────────────                  │
│                                                                 │
│    ┌─────────────────────────────────────────────────────────┐ │
│    │  Total Files:     1,234                                 │ │
│    │  Total Size:      2.4 GB                                │ │
│    │  Months:          18 (Jan 2023 - Jul 2024)             │ │
│    │  Tags:            15                                    │ │
│    │  Categories:      8                                     │ │
│    └─────────────────────────────────────────────────────────┘ │
│                                              [Refresh Stats]   │
│                                                                 │
│    ─────────────── About ───────────────                       │
│                                                                 │
│    MaybeVault v1.0.0                                           │
│    Files you probably won't need again, safely stored.         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Dialogs

### Duplicate Warning Dialog

```
┌────────────────────────────────────────────────────┐
│ ⚠️ Duplicate File Detected                    [×]  │
├────────────────────────────────────────────────────┤
│                                                    │
│  "photo.png" is identical to an existing file:    │
│                                                    │
│  ┌──────────────────────────────────────────────┐ │
│  │  📷 vacation-sunset.png                      │ │
│  │  📁 2024-06                                  │ │
│  │  📅 Archived: June 15, 2024                  │ │
│  │  🏷️ Tags: vacation, family                   │ │
│  │  📝 "Sunset photo from beach trip"           │ │
│  └──────────────────────────────────────────────┘ │
│                                                    │
│  What would you like to do?                        │
│                                                    │
│  [Skip This File]   [Add Anyway]   [Open Existing] │
│                                                    │
└────────────────────────────────────────────────────┘
```

### Tag Manager Dialog

```
┌────────────────────────────────────────────────────┐
│  Manage Tags                                  [×]  │
├────────────────────────────────────────────────────┤
│                                                    │
│  ┌──────────────────────────────────────────────┐ │
│  │ [■] math                              [✏️][🗑️] │ │
│  │ [■] daughter                          [✏️][🗑️] │ │
│  │ [■] homework                          [✏️][🗑️] │ │
│  │ [■] receipts                          [✏️][🗑️] │ │
│  │ [■] photos                            [✏️][🗑️] │ │
│  └──────────────────────────────────────────────┘ │
│                                                    │
│  ─────────────── Add New Tag ───────────────      │
│                                                    │
│  Name:  [                    ]                     │
│  Color: [■ #3B82F6  ▼]                            │
│                                                    │
│                              [Cancel]   [Add Tag]  │
│                                                    │
└────────────────────────────────────────────────────┘
```

### Category Manager Dialog

```
┌────────────────────────────────────────────────────┐
│  Manage Categories                            [×]  │
├────────────────────────────────────────────────────┤
│                                                    │
│  ┌──────────────────────────────────────────────┐ │
│  │ [↑][↓] 📚 Homework                    [✏️][🗑️] │ │
│  │ [↑][↓] 🧾 Receipts                    [✏️][🗑️] │ │
│  │ [↑][↓] 📷 Photos                      [✏️][🗑️] │ │
│  │ [↑][↓] 📝 Notes                       [✏️][🗑️] │ │
│  └──────────────────────────────────────────────┘ │
│                                                    │
│  ─────────────── Add New Category ───────────────  │
│                                                    │
│  Name:  [                    ]                     │
│  Icon:  [📁 ▼]  (click to select emoji)           │
│                                                    │
│                           [Cancel]  [Add Category] │
│                                                    │
└────────────────────────────────────────────────────┘
```

---

## Color Palette

Using Fluent design colors:

| Usage | Color | Hex |
|-------|-------|-----|
| Primary | Blue | `#0078D4` |
| Success | Green | `#107C10` |
| Warning | Orange | `#FF8C00` |
| Error | Red | `#E81123` |
| Background | Light Gray | `#F3F3F3` |
| Card Background | White | `#FFFFFF` |
| Text Primary | Dark Gray | `#323130` |
| Text Secondary | Medium Gray | `#605E5C` |
| Border | Light Border | `#EDEBE9` |

### Tag Colors (Presets)

```
Blue:    #3B82F6
Green:   #10B981
Yellow:  #F59E0B
Red:     #EF4444
Purple:  #8B5CF6
Pink:    #EC4899
Indigo:  #6366F1
Teal:    #14B8A6
```

---

## Icons

Using Fluent UI System Icons or similar:

| Action | Icon |
|--------|------|
| Add/Archive | `folder_add` |
| Browse/Search | `search` |
| Settings | `settings` |
| File | `document` |
| Image | `image` |
| Folder | `folder` |
| Delete | `delete` |
| Edit | `edit` |
| Open | `open` |
| Copy | `copy` |
| Warning | `warning` |
| Tag | `tag` |
| Category | `folder_open` |

---

## Responsive Behavior

### Minimum Window Size
- Width: 800px
- Height: 600px

### Resize Behavior
- Drop zone grows/shrinks with window
- File lists use available vertical space
- Sidebar remains fixed width
- Dialogs centered and fixed size

---

## Accessibility

### Keyboard Navigation
- Tab through all interactive elements
- Enter to activate buttons
- Escape to close dialogs
- Arrow keys in lists

### Screen Reader Support
- All buttons have accessible names
- Status updates announced
- File list items properly labeled

### Visual
- Sufficient color contrast (WCAG AA)
- Focus indicators visible
- No color-only information (icons + text)

---

## Animations

Subtle animations for feedback:

| Action | Animation |
|--------|-----------|
| Drop zone hover | Border color fade |
| File drop | Brief highlight |
| Button press | Scale down slightly |
| Dialog open | Fade in + slight scale |
| List item add | Slide in from top |
| List item remove | Fade out |

Keep animations short (150-300ms) and respect system "reduce motion" settings.
