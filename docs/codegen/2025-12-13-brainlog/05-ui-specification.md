# UI Specification

## Layout Overview

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│ [AI: ▼OpenAI] [New Entry]                                              [─][□][×]│
├──────────────┬──────────────────────────────────────────────────────────────────┤
│              │  Title: [_________________________] [Generate]                   │
│   Entry      │                                                                  │
│   List       │  ┌─ Draft ────────────────┐  ┌─ Content ──────────────┐         │
│              │  │                        │  │                        │         │
│  ┌─────────┐ │  │                        │  │                        │         │
│  │ ○ Entry │ │  │                        │  │                        │         │
│  │   Title │ │  │                        │  │                        │         │
│  │   12/13 │ │  │                        │  │                        │         │
│  ├─────────┤ │  │                        │  │                        │         │
│  │ ● Entry │ │  │                        │  │                        │         │
│  │   Title │ │  └────────────────────────┘  └────────────────────────┘         │
│  │   12/12 │ │                                                                  │
│  ├─────────┤ │  [Copy Draft →]  [Prompt: ▼Fix Errors] [Run] [Manage Prompts]   │
│  │ ○ Entry │ │                                                                  │
│  │   Title │ │  ┌─ Analysis ─────────────────────────────────────────────────┐ │
│  │   12/11 │ │  │                                                            │ │
│  └─────────┘ │  │ (AI-generated explanations appear here)                    │ │
│              │  │                                                            │ │
│ Filter:      │  └────────────────────────────────────────────────────────────┘ │
│ ○ Active     │                                                                  │
│ ○ Archived   │  State: Editing          [Mark as Checked]         [Trash]      │
│              │                                                                  │
├──────────────┴──────────────────────────────────────────────────────────────────┤
│ Status: Ready │ Auto-saves: 5                                          [Save]  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

## Main Window Components

### Top Bar
| Component | Description |
|-----------|-------------|
| AI Dropdown | Global AI service selection (OpenAI/Gemini) |
| New Entry Button | Deselects list, shows blank fields for new entry |
| Window Controls | Minimize, Maximize, Close |

### Entry List Panel (Left)

| Component | Description |
|-----------|-------------|
| Entry List | Scrollable list of entries |
| Entry Item | Shows: State indicator (color), Date, Title |
| Filter Radio | "Active" (E/C/H) or "Archived" (A) |

**Entry Item Layout**:
```
┌─────────────────────┐
│ 2025-12-13 ●        │  ← Created date + State color dot (on right)
│                     │
│ Meeting Ideas...    │  ← Title (or truncated content) after blank line
└─────────────────────┘
```

**State Colors**:
| State | Color |
|-------|-------|
| Editing | Blue |
| Checked | Green |
| Handled | Orange |
| Archived | Gray |

### Editor Panel (Center-Right)

#### Title Section
| Component | Description |
|-----------|-------------|
| Title TextBox | Editable title field |
| Generate Button | Regenerate title via AI |
| Watermark | "Auto-generated on first save" when empty |

#### Text Areas
| Component | Description |
|-----------|-------------|
| Draft | Original/raw input text (nullable) |
| Content | Refined/final text |
| Analysis | AI-generated explanations (display only, not persisted) |

#### AI Controls Row
| Component | Description |
|-----------|-------------|
| Copy Draft → | Copy Draft text to Content field |
| Prompt Dropdown | Select from user's prompts |
| Run Button | Execute selected prompt |
| Manage Prompts | Open prompt management window |

#### State Controls Row
| Component | Description |
|-----------|-------------|
| State Display | Shows current state (read-only text, not a dropdown) |
| State Button | Context-dependent: "Mark as Checked", "Mark as Handled", "Archive", or "Revert to {Previous}" |
| Trash Button | Moves entry file to OS trash (visible for all entries, but primarily for archived ones) |

**State Button Logic**:
| Current State | Primary Button | Secondary Button (after cooldown) | Trash Button |
|---------------|----------------|-----------------------------------|---------------|
| Editing | Mark as Checked | - | Enabled |
| Checked | Mark as Handled | Revert to Editing | Enabled |
| Handled | (disabled until cooldown) | Revert to Checked | Enabled |
| Handled (after cooldown) | Archive | Revert to Checked | Enabled |
| Archived | Revert to Handled | - | Enabled |

### Status Bar (Bottom)

| Component | Description |
|-----------|-------------|
| Status Text | "Ready", "Saving...", conflict warnings |
| Auto-save Counter | "Auto-saves: N" (hidden when 0) |
| Save Button | Save current entry |

## Text Area Behavior

### Conflict Indication
When external change detected and differs from editor content:
- Border color changes to red/warning color
- Status bar shows: "File modified externally"

### Editing Locked Entries
When editing a Checked/Handled/Archived entry:
- Fields remain editable (no visual lock)
- Save button shows confirmation dialog

## Keyboard Shortcuts

| Shortcut (Windows) | Shortcut (macOS) | Action |
|--------------------|------------------|--------|
| Ctrl+S | Cmd+S | Save |
| Ctrl+N | Cmd+N | New Entry |
| Ctrl+W | Cmd+W | Close Window |
| Ctrl+Shift+C | Cmd+Shift+C | Copy Draft to Content |
| Ctrl+Enter | Cmd+Enter | Run Selected Prompt |
| Tab | Tab | Navigate between text areas |
| Escape | Escape | Cancel AI operation (when in progress) |

## Dialogs

### Unsaved Changes Dialog
**Trigger**: Closing window or selecting different entry with unsaved changes

```
┌─────────────────────────────────────────┐
│  Unsaved Changes                        │
├─────────────────────────────────────────┤
│                                         │
│  You have unsaved changes. What would   │
│  you like to do?                        │
│                                         │
│  [Save and Continue] [Discard Changes] [Cancel] │
└─────────────────────────────────────────┘
```

### Save Confirmation Dialog (Non-Editing State)
**Trigger**: Saving when state is Checked/Handled/Archived (if enabled in settings)

```
┌─────────────────────────────────────────┐
│  Confirm Save                           │
├─────────────────────────────────────────┤
│                                         │
│  This entry is marked as "Checked".     │
│  Saving will modify a finalized entry.  │
│                                         │
│  Are you sure you want to continue?     │
│                                         │
│  □ Don't show this again                │
│                                         │
│           [Save Anyway] [Cancel]        │
└─────────────────────────────────────────┘
```

### AI Processing Dialog
**Trigger**: Running any AI operation

```
┌─────────────────────────────────────────┐
│  Processing                             │
├─────────────────────────────────────────┤
│                                         │
│        ◌ Running AI prompt...           │
│                                         │
│              [Cancel]                   │
└─────────────────────────────────────────┘
```

**On Error**:
```
┌─────────────────────────────────────────┐
│  Error                                  │
├─────────────────────────────────────────┤
│                                         │
│  AI request failed:                     │
│  "Connection timeout after 60 seconds"  │
│                                         │
│              [Close]                    │
└─────────────────────────────────────────┘
```

### File Deleted Dialog
**Trigger**: FileWatcher detects current entry's file was deleted

```
┌─────────────────────────────────────────┐
│  File Deleted                           │
├─────────────────────────────────────────┤
│                                         │
│  This entry's file has been deleted     │
│  externally.                            │
│                                         │
│  You can save to restore it, or close   │
│  to discard your current edits.         │
│                                         │
│              [OK]                       │
└─────────────────────────────────────────┘
```

### Corrupt File Dialog
**Trigger**: Failed to parse entry JSON file

```
┌─────────────────────────────────────────┐
│  Corrupt File                           │
├─────────────────────────────────────────┤
│                                         │
│  Failed to load entry:                  │
│                                         │
│  Path: C:\...\2025-12-13-abc123.json    │
│  Error: Missing required field 'state'  │
│                                         │
│  This file will be skipped.             │
│                                         │
│              [OK]                       │
└─────────────────────────────────────────┘
```

### Success Notification
**Trigger**: Save completed successfully
**Behavior**: Toast-style notification, auto-dismisses after configured seconds

```
┌─────────────────────────────┐
│  ✓ Entry saved              │
└─────────────────────────────┘
```

## Prompt Management Window

```
┌─────────────────────────────────────────────────────────────────┐
│  Manage Prompts                                          [×]   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────┐   Name:                               │
│  │ ► Fix Transcription │   [Fix Transcription Errors_____]     │
│  │   Improve Clarity   │                                       │
│  │   Summarize         │   System Prompt:                      │
│  │                     │   ┌─────────────────────────────────┐ │
│  │                     │   │ You are a text editor assistant │ │
│  │                     │   │ ...                             │ │
│  │                     │   └─────────────────────────────────┘ │
│  │                     │                                       │
│  │                     │   User Prompt:                        │
│  │                     │   ┌─────────────────────────────────┐ │
│  │                     │   │ Please fix errors in {DRAFT}... │ │
│  │                     │   │ Return JSON with CONTENT and... │ │
│  │                     │   └─────────────────────────────────┘ │
│  └─────────────────────┘                                       │
│                                                                 │
│  [Add] [Delete] [▲ Up] [▼ Down]           [Save] [Cancel]      │
└─────────────────────────────────────────────────────────────────┘
```

## New Entry Flow

1. Click "New Entry" button (or Ctrl/Cmd+N)
2. If unsaved changes exist: Show unsaved changes dialog
3. Deselect all items in entry list
4. Clear all fields (Title, Draft, Content, Analysis)
5. State shows "Editing" (but entry doesn't exist yet)
6. User types in Draft and/or Content
7. User clicks Save
8. Entry created with new GUID, added to list, selected
9. Title generation triggered in background

## Responsive Considerations

- Minimum window width: 900px
- Text areas should have reasonable minimum heights
- Analysis area can collapse/expand if needed
- On narrow screens: Consider scrolling within editor panel
