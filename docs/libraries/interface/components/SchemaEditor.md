# SchemaEditor (IL Component)

**Layer:** Interaction Layer (IL)
**Domain:** Schema Registry (Vertical 5.2)
**Status:** Draft
**Last Updated:** 2025-10-25

---

## Table of Contents

1. [Overview](#overview)
2. [Component Interface](#component-interface)
3. [Architecture](#architecture)
4. [UI Components](#ui-components)
5. [Editor Features](#editor-features)
6. [Visual Wireframes](#visual-wireframes)
7. [States & Lifecycle](#states--lifecycle)
8. [Multi-Domain Examples](#multi-domain-examples)
9. [User Interactions](#user-interactions)
10. [Accessibility](#accessibility)
11. [Performance Optimizations](#performance-optimizations)
12. [Integration Examples](#integration-examples)
13. [Related Components](#related-components)
14. [References](#references)

---

## Overview

### Purpose

The **SchemaEditor** is a sophisticated code editor component specifically designed for creating, modifying, and validating JSON schemas within a Schema Registry. It provides a Monaco Editor-based interface with syntax highlighting, auto-completion, real-time validation, compatibility checking, and version management capabilities. This component transforms schema authoring from a risky, error-prone process into a guided, validated, and collaborative experience.

### Key Capabilities

- **Monaco Editor Integration**: Full-featured code editor with VS Code-like experience
- **JSON Schema Validation**: Real-time syntax and semantic validation with error highlighting
- **Compatibility Checking**: Inline detection of breaking changes vs. additive changes
- **Version Bump Suggestions**: Automatic recommendation of MAJOR/MINOR/PATCH based on changes
- **Side-by-Side Diff**: Compare current schema with previous versions
- **Auto-completion**: Context-aware suggestions for JSON Schema keywords
- **Format Preservation**: Maintains formatting preferences (indent, spacing)
- **Undo/Redo Support**: Full history with granular undo/redo
- **Dark/Light Theme**: Adapts to user preferences
- **Keyboard Shortcuts**: Power-user workflows (Cmd+S to save, Cmd+K to format, etc.)

### Problem Statement

Organizations struggle with schema evolution because:

1. **Breaking Changes Go Undetected**: Developers remove required fields without realizing impact
2. **No Validation Feedback**: Schemas are published with syntax errors or invalid constraints
3. **Version Confusion**: Unclear whether a change is MAJOR, MINOR, or PATCH
4. **Collaboration Friction**: Multiple developers editing schemas without conflict detection
5. **Migration Planning**: No visibility into how schema changes affect existing data
6. **Documentation Gaps**: Schema changes lack context about why they were made

### Solution

The SchemaEditor provides:

- **Pre-Publish Validation**: Catch errors before they break consumers
- **Change Impact Analysis**: Visual indicators of breaking vs. non-breaking changes
- **Guided Versioning**: Automatic version bump recommendations based on SemVer rules
- **Diff View**: Side-by-side comparison with previous versions
- **Confidence Scoring**: Flag risky changes that need review
- **Audit Trail Integration**: All edits are logged with author, timestamp, and reason

This transforms schema management from ad-hoc file editing to a governed, traceable process.

---

## Component Interface

### Primary Props

```typescript
interface SchemaEditorProps {
  // ============================================================
  // Schema Configuration
  // ============================================================

  /**
   * Unique identifier for the schema being edited
   * @example 'payment_event_v3', 'patient_record_v2'
   */
  schemaId: string

  /**
   * Current version of the schema (semantic version)
   * @example '2.1.0', '1.0.0-beta.1'
   */
  currentVersion: string

  /**
   * The JSON schema content being edited
   * Can be string (JSON) or parsed object
   */
  initialSchema: string | object

  /**
   * Schema language/format
   * @default 'jsonschema'
   */
  schemaFormat?: 'jsonschema' | 'avro' | 'protobuf' | 'graphql'

  /**
   * Previous version of the schema (for diff view)
   * Optional - if provided, enables side-by-side comparison
   */
  previousSchema?: string | object

  /**
   * Previous version number (for diff view header)
   */
  previousVersion?: string

  // ============================================================
  // Editor Configuration
  // ============================================================

  /**
   * Editor theme
   * @default 'vs-dark'
   */
  theme?: 'vs' | 'vs-dark' | 'hc-black' | 'hc-light'

  /**
   * Enable/disable specific editor features
   */
  features?: {
    autoCompletion?: boolean        // Default: true
    syntaxValidation?: boolean      // Default: true
    compatibilityCheck?: boolean    // Default: true
    diffView?: boolean              // Default: true
    formatOnPaste?: boolean         // Default: true
    wordWrap?: boolean              // Default: true
    minimap?: boolean               // Default: true
    lineNumbers?: boolean           // Default: true
    folding?: boolean               // Default: true
    bracketMatching?: boolean       // Default: true
  }

  /**
   * Indentation settings
   */
  indentation?: {
    size?: number                   // Default: 2
    useTabs?: boolean               // Default: false
  }

  /**
   * Editor height (CSS value)
   * @default '600px'
   */
  height?: string

  /**
   * Read-only mode (for viewing schemas)
   * @default false
   */
  readOnly?: boolean

  /**
   * Enable keyboard shortcuts
   * @default true
   */
  enableShortcuts?: boolean

  // ============================================================
  // Validation Configuration
  // ============================================================

  /**
   * Validation rules to apply
   */
  validationRules?: {
    enforceRequired?: boolean       // Require 'required' fields
    enforceDescriptions?: boolean   // Require field descriptions
    maxDepth?: number               // Max nesting depth (default: 10)
    allowAdditionalProperties?: boolean  // Default: true
    customRules?: ValidationRule[]  // Custom validation logic
  }

  /**
   * Compatibility check strictness
   * @default 'strict'
   */
  compatibilityMode?: 'strict' | 'moderate' | 'lenient'

  /**
   * Auto-save interval (milliseconds)
   * Set to 0 to disable auto-save
   * @default 30000 (30 seconds)
   */
  autoSaveInterval?: number

  /**
   * Show validation errors inline
   * @default true
   */
  inlineValidation?: boolean

  /**
   * Show compatibility warnings inline
   * @default true
   */
  inlineCompatibility?: boolean

  // ============================================================
  // Version Management
  // ============================================================

  /**
   * Show version bump suggestions
   * @default true
   */
  showVersionSuggestions?: boolean

  /**
   * Allow user to override suggested version bump
   * @default true
   */
  allowVersionOverride?: boolean

  /**
   * Versioning strategy
   * @default 'semver'
   */
  versioningStrategy?: 'semver' | 'calver' | 'custom'

  /**
   * Pre-release identifier (e.g., 'alpha', 'beta', 'rc')
   * If set, versions will include this (e.g., 2.0.0-beta.1)
   */
  preReleaseTag?: string

  // ============================================================
  // Callbacks
  // ============================================================

  /**
   * Called when schema content changes
   * @param schema - Updated schema content
   * @param isValid - Whether schema passes validation
   */
  onChange?: (schema: string, isValid: boolean) => void

  /**
   * Called when user clicks "Publish" button
   * @param schema - Final schema content
   * @param versionBump - Suggested version bump (MAJOR/MINOR/PATCH)
   * @param breakingChanges - Array of breaking changes detected
   */
  onPublish?: (
    schema: string,
    versionBump: VersionBump,
    breakingChanges: BreakingChange[]
  ) => Promise<void>

  /**
   * Called when user clicks "Cancel" button
   */
  onCancel?: () => void

  /**
   * Called when user requests format/prettify
   * @param schema - Schema before formatting
   * @returns Formatted schema
   */
  onFormat?: (schema: string) => string

  /**
   * Called when validation errors are detected
   * @param errors - Array of validation errors
   */
  onValidationError?: (errors: ValidationError[]) => void

  /**
   * Called when compatibility check completes
   * @param report - Compatibility analysis report
   */
  onCompatibilityCheck?: (report: CompatibilityReport) => void

  /**
   * Called when user toggles diff view
   * @param isVisible - Whether diff view is now visible
   */
  onDiffToggle?: (isVisible: boolean) => void

  /**
   * Called on auto-save
   * @param schema - Schema being auto-saved
   */
  onAutoSave?: (schema: string) => void

  // ============================================================
  // UI Options
  // ============================================================

  /**
   * Show toolbar with actions (save, format, undo, redo)
   * @default true
   */
  showToolbar?: boolean

  /**
   * Show footer with status (character count, errors, version)
   * @default true
   */
  showFooter?: boolean

  /**
   * Show version badge in header
   * @default true
   */
  showVersionBadge?: boolean

  /**
   * Show compatibility badge in header
   * @default true
   */
  showCompatibilityBadge?: boolean

  /**
   * Show diff toggle button
   * @default true (if previousSchema provided)
   */
  showDiffToggle?: boolean

  /**
   * Custom toolbar actions
   * Allows adding domain-specific actions to toolbar
   */
  customActions?: CustomAction[]

  /**
   * Loading state (external loading indicator)
   * @default false
   */
  isLoading?: boolean

  /**
   * Error message to display (external error)
   */
  errorMessage?: string

  /**
   * Success message to display (e.g., after publish)
   */
  successMessage?: string

  // ============================================================
  // Advanced Options
  // ============================================================

  /**
   * Enable collaborative editing (real-time multi-user)
   * Requires WebSocket connection
   * @default false
   */
  enableCollaboration?: boolean

  /**
   * Collaboration session ID
   * Required if enableCollaboration=true
   */
  collaborationSessionId?: string

  /**
   * Current user info (for collaboration)
   */
  currentUser?: {
    id: string
    name: string
    avatar?: string
    color?: string  // Cursor color
  }

  /**
   * Schema templates/snippets
   * Pre-defined schema patterns users can insert
   */
  templates?: SchemaTemplate[]

  /**
   * Enable AI-powered suggestions
   * @default false
   */
  enableAISuggestions?: boolean

  /**
   * Custom Monaco editor options
   * Allows fine-grained control over editor behavior
   */
  monacoOptions?: monaco.editor.IStandaloneEditorConstructionOptions

  /**
   * Debug mode (show internal state, logs)
   * @default false
   */
  debugMode?: boolean
}

// ============================================================
// Supporting Types
// ============================================================

interface ValidationError {
  line: number
  column: number
  message: string
  severity: 'error' | 'warning' | 'info'
  code?: string
  suggestion?: string
}

interface BreakingChange {
  type: 'field_removed' | 'type_changed' | 'constraint_added' | 'enum_reduced'
  path: string  // JSONPath to affected field
  description: string
  impact: 'high' | 'medium' | 'low'
  affectedRecords?: number  // Estimated number of records impacted
}

interface CompatibilityReport {
  compatible: boolean
  breakingChanges: BreakingChange[]
  additiveChanges: AdditiveChange[]
  suggestedVersionBump: VersionBump
  confidence: number  // 0-1 score
  migrationRequired: boolean
  estimatedMigrationTime?: string  // Human-readable (e.g., "2 minutes")
}

interface AdditiveChange {
  type: 'field_added' | 'description_added' | 'example_added' | 'constraint_relaxed'
  path: string
  description: string
}

type VersionBump = 'MAJOR' | 'MINOR' | 'PATCH' | 'NONE'

interface CustomAction {
  id: string
  label: string
  icon?: React.ReactNode
  onClick: () => void
  disabled?: boolean
  tooltip?: string
}

interface SchemaTemplate {
  id: string
  name: string
  description: string
  schema: string
  category?: string
}

interface ValidationRule {
  id: string
  name: string
  validate: (schema: object) => ValidationError[]
}

// ============================================================
// Component State
// ============================================================

interface SchemaEditorState {
  // Editor state
  schema: string
  isValid: boolean
  isDirty: boolean  // Has unsaved changes
  cursorPosition: { line: number; column: number }

  // Validation state
  validationErrors: ValidationError[]
  validationInProgress: boolean

  // Compatibility state
  compatibilityReport: CompatibilityReport | null
  compatibilityCheckInProgress: boolean

  // Version state
  suggestedVersionBump: VersionBump
  suggestedNextVersion: string

  // UI state
  showDiffView: boolean
  activePanel: 'editor' | 'diff' | 'preview'

  // Collaboration state
  connectedUsers: CollaborationUser[]
  remoteCursors: RemoteCursor[]

  // Status
  lastSaved: Date | null
  autoSaveEnabled: boolean
  publishing: boolean
  publishError: string | null
  publishSuccess: boolean
}

interface CollaborationUser {
  id: string
  name: string
  avatar?: string
  color: string
  cursorPosition?: { line: number; column: number }
}

interface RemoteCursor {
  userId: string
  position: { line: number; column: number }
  selection?: { start: { line: number; column: number }; end: { line: number; column: number } }
}
```

---

## Architecture

### Component Hierarchy

```
SchemaEditor
├── EditorHeader
│   ├── SchemaBadge (schemaId, currentVersion)
│   ├── CompatibilityBadge (breakingChanges count)
│   └── DiffToggle (show/hide diff view)
├── EditorToolbar
│   ├── UndoRedoButtons
│   ├── FormatButton
│   ├── SearchButton
│   ├── TemplatesDropdown
│   └── CustomActions
├── EditorContainer
│   ├── MonacoEditor (main editing area)
│   │   ├── SyntaxHighlighting
│   │   ├── AutoCompletion
│   │   ├── InlineValidation (error markers)
│   │   └── InlineCompatibility (warning markers)
│   └── DiffEditor (side-by-side comparison)
│       ├── OriginalPane (previousSchema)
│       └── ModifiedPane (currentSchema)
├── ValidationPanel
│   ├── ErrorList (filterable)
│   ├── WarningList
│   └── InfoList
├── CompatibilityPanel
│   ├── BreakingChangesList
│   ├── AdditiveChangesList
│   └── VersionBumpSuggestion
└── EditorFooter
    ├── StatusBar
    │   ├── CursorPosition
    │   ├── CharacterCount
    │   ├── ValidationStatus
    │   └── AutoSaveIndicator
    └── ActionButtons
        ├── CancelButton
        ├── SaveDraftButton
        └── PublishButton
```

### Data Flow

```
User Types in Editor
    ↓
[Debounced onChange] (300ms delay)
    ↓
Parse JSON Schema
    ↓
    ├── Syntax Validation
    │   └── Show errors in editor + ValidationPanel
    ↓
    ├── Semantic Validation
    │   └── Check schema constraints, required fields
    ↓
    └── Compatibility Check (if previousSchema exists)
        ├── Detect Breaking Changes
        ├── Detect Additive Changes
        ├── Calculate Version Bump
        └── Update CompatibilityPanel

User Clicks "Publish"
    ↓
Validate Schema (final check)
    ↓
Run Compatibility Check
    ↓
Show Confirmation Dialog
    ├── Display suggested version bump
    ├── List breaking changes
    └── Request confirmation
    ↓
onPublish(schema, versionBump, breakingChanges)
    ↓
External system handles publication
```

### State Management

The SchemaEditor uses a combination of local state (React hooks) and Monaco Editor's internal state management:

1. **Local State**: Manages validation results, compatibility reports, UI toggles
2. **Monaco State**: Manages editor content, cursor position, undo/redo stack
3. **Synchronization**: Debounced sync between Monaco content and local validation state

---

## UI Components

### 1. EditorHeader

Displays schema metadata and quick actions.

**Elements:**
- **Schema Badge**: Shows schemaId and currentVersion
- **Compatibility Badge**: Color-coded indicator (green=compatible, red=breaking)
- **Diff Toggle**: Switch between edit mode and diff mode
- **Last Saved Indicator**: "Last saved 2 minutes ago" or "Unsaved changes"

**Visual Design:**
```
┌────────────────────────────────────────────────────────────┐
│ 📄 payment_event  v2.1.0          🟢 Compatible   [Diff] │
│ Last saved: 2 minutes ago                                  │
└────────────────────────────────────────────────────────────┘
```

### 2. EditorToolbar

Provides quick access to common actions.

**Elements:**
- **Undo/Redo**: Standard editing controls
- **Format**: Prettify JSON (auto-indent, sort keys)
- **Search**: Find/replace within schema
- **Templates**: Insert pre-defined schema patterns
- **Custom Actions**: Domain-specific actions (e.g., "Test with sample data")

**Visual Design:**
```
┌────────────────────────────────────────────────────────────┐
│ ↶ ↷ | 🎨 Format | 🔍 Search | 📋 Templates | 🔧 More     │
└────────────────────────────────────────────────────────────┘
```

### 3. MonacoEditor

The main editing area using Microsoft Monaco Editor.

**Features:**
- Syntax highlighting for JSON
- Auto-completion for JSON Schema keywords ($schema, properties, required, etc.)
- Bracket matching and auto-closing
- Multi-cursor editing
- Minimap for large schemas
- Error/warning squiggles (inline validation)

**Visual Design:**
```
┌────────────────────────────────────────────────────────────┐
│ 1  {                                                       │
│ 2    "$schema": "http://json-schema.org/draft-07/schema#",│
│ 3    "type": "object",                                     │
│ 4    "properties": {                                       │
│ 5      "amount": {                                         │
│ 6        "type": "number",                                 │
│ 7        "minimum": 0        ← ⚠️ Constraint added (MINOR)│
│ 8      },                                                  │
│ 9      "currency": {                                       │
│10        "type": "string",                                 │
│11        "enum": ["USD", "EUR"]  ← 🔴 Enum reduced (MAJOR)│
│12      }                                                   │
│13    },                                                    │
│14    "required": ["amount", "currency"]                    │
│15  }                                                       │
└────────────────────────────────────────────────────────────┘
```

### 4. DiffEditor (Side-by-Side)

Shows previous version alongside current version with highlighted changes.

**Visual Design:**
```
┌──────────────────────┬──────────────────────────────────────┐
│ v2.0.0 (previous)    │ v2.1.0 (current)                     │
├──────────────────────┼──────────────────────────────────────┤
│ {                    │ {                                    │
│   "properties": {    │   "properties": {                    │
│     "amount": {      │     "amount": {                      │
│       "type": "number"│       "type": "number",             │
│                      │       "minimum": 0  ← GREEN (added) │
│     },               │     },                               │
│     "currency": {    │     "currency": {                    │
│       "enum": [      │       "enum": [                      │
│         "USD",       │         "USD",                       │
│         "EUR",       │         "EUR"                        │
│         "GBP"  ← RED │                  ← RED (removed)     │
│       ]              │       ]                              │
│     }                │     }                                │
│   }                  │   }                                  │
│ }                    │ }                                    │
└──────────────────────┴──────────────────────────────────────┘
```

### 5. ValidationPanel

Displays validation errors, warnings, and info messages.

**Visual Design:**
```
┌────────────────────────────────────────────────────────────┐
│ Validation Results (2 errors, 1 warning)                   │
├────────────────────────────────────────────────────────────┤
│ 🔴 Line 11: Property 'currency.enum' removed value 'GBP'   │
│    This is a BREAKING change affecting 234 records         │
│                                                            │
│ 🔴 Line 14: Missing 'description' field (required by policy)│
│                                                            │
│ ⚠️  Line 7: Adding constraint 'minimum' may break existing │
│    records with negative amounts (12 records found)        │
└────────────────────────────────────────────────────────────┘
```

### 6. CompatibilityPanel

Shows compatibility analysis and version bump suggestion.

**Visual Design:**
```
┌────────────────────────────────────────────────────────────┐
│ Compatibility Check                                        │
├────────────────────────────────────────────────────────────┤
│ Status: 🔴 BREAKING CHANGES DETECTED                       │
│                                                            │
│ Suggested Version Bump: MAJOR (2.1.0 → 3.0.0)             │
│                                                            │
│ Breaking Changes (1):                                      │
│  • currency.enum reduced (removed 'GBP')                   │
│    Impact: HIGH - affects 234 records (12% of total)      │
│                                                            │
│ Additive Changes (1):                                      │
│  • amount.minimum added (new constraint)                   │
│    Impact: MEDIUM - may invalidate 12 existing records    │
│                                                            │
│ Migration Required: YES                                    │
│ Estimated Time: ~5 minutes (246 records to update)        │
│                                                            │
│ [View Migration Plan] [Override Version]                   │
└────────────────────────────────────────────────────────────┘
```

### 7. EditorFooter

Status bar and action buttons.

**Visual Design:**
```
┌────────────────────────────────────────────────────────────┐
│ Ln 11, Col 8 | 456 chars | ✅ Valid | 💾 Auto-save ON     │
│                                                            │
│              [Cancel] [Save Draft] [Publish v3.0.0]       │
└────────────────────────────────────────────────────────────┘
```

---

## Editor Features

### 1. Auto-Completion

Provides context-aware suggestions as users type.

**Triggers:**
- Typing `"` inside `properties` object → suggests field names
- Typing `type` → suggests JSON Schema types (string, number, object, array, etc.)
- Typing `$` → suggests JSON Schema keywords ($schema, $ref, $id, etc.)
- Inside `properties` → suggests common schema keywords (type, description, enum, etc.)

**Example:**
```json
{
  "properties": {
    "email": {
      "type": "str|"  ← Cursor here
    }
  }
}
```

**Auto-completion dropdown shows:**
```
string     ← Primary suggestion
number
boolean
object
array
null
```

### 2. Real-Time Validation

Validates schema as user types (debounced 300ms).

**Validation Layers:**

1. **Syntax Validation**
   - Valid JSON
   - Proper bracket/quote matching
   - No trailing commas (strict JSON)

2. **Schema Validation**
   - Valid JSON Schema (Draft 7 / 2020-12)
   - Required keywords present ($schema, type)
   - No conflicting constraints (e.g., minimum > maximum)

3. **Policy Validation**
   - Organization-specific rules
   - Required descriptions on all fields
   - Maximum nesting depth
   - Naming conventions (e.g., snake_case for field names)

**Error Display:**
- Red squiggly underlines in editor
- Error messages in ValidationPanel
- Gutter icons (🔴 for errors, ⚠️ for warnings)
- Hover tooltips with suggested fixes

### 3. Compatibility Checking

Compares current schema with previous version to detect changes.

**Detection Algorithm:**

```typescript
function detectChanges(oldSchema: object, newSchema: object): CompatibilityReport {
  const breakingChanges: BreakingChange[] = []
  const additiveChanges: AdditiveChange[] = []

  // Breaking changes
  if (fieldRemoved(oldSchema, newSchema)) {
    breakingChanges.push({ type: 'field_removed', path: '$.properties.fieldName', ... })
  }
  if (typeChanged(oldSchema, newSchema)) {
    breakingChanges.push({ type: 'type_changed', path: '$.properties.amount.type', ... })
  }
  if (enumReduced(oldSchema, newSchema)) {
    breakingChanges.push({ type: 'enum_reduced', path: '$.properties.status.enum', ... })
  }
  if (constraintAdded(oldSchema, newSchema)) {
    // Adding 'required' field is breaking
    // Adding min/max constraints might be breaking (needs data analysis)
    breakingChanges.push({ type: 'constraint_added', path: '$.required', ... })
  }

  // Additive changes
  if (fieldAdded(oldSchema, newSchema)) {
    additiveChanges.push({ type: 'field_added', path: '$.properties.newField', ... })
  }
  if (descriptionAdded(oldSchema, newSchema)) {
    additiveChanges.push({ type: 'description_added', path: '$.properties.email.description', ... })
  }
  if (constraintRelaxed(oldSchema, newSchema)) {
    // Removing 'required' or relaxing min/max
    additiveChanges.push({ type: 'constraint_relaxed', path: '$.properties.age.minimum', ... })
  }

  // Determine version bump
  const suggestedVersionBump = breakingChanges.length > 0 ? 'MAJOR'
    : additiveChanges.length > 0 ? 'MINOR'
    : 'PATCH'

  return {
    compatible: breakingChanges.length === 0,
    breakingChanges,
    additiveChanges,
    suggestedVersionBump,
    confidence: calculateConfidence(breakingChanges, additiveChanges),
    migrationRequired: breakingChanges.length > 0
  }
}
```

**Visual Indicators:**

In the editor, breaking changes are highlighted with red inline decorations:
```json
{
  "properties": {
    "status": {
      "enum": ["active", "pending"]  ← 🔴 Removed 'cancelled' (BREAKING)
    }
  }
}
```

Additive changes are highlighted with green decorations:
```json
{
  "properties": {
    "email": {
      "type": "string",
      "format": "email"  ← 🟢 Added validation (ADDITIVE)
    }
  }
}
```

### 4. Version Bump Suggestions

Based on detected changes, suggest next version number.

**Rules (Semantic Versioning):**

- **MAJOR**: Breaking changes detected
  - Field removed
  - Type changed
  - Required field added
  - Enum values removed
  - Constraint made stricter

- **MINOR**: Additive changes only
  - New optional field added
  - Description added/updated
  - Enum values added
  - Constraint relaxed

- **PATCH**: Documentation/metadata only
  - Examples added
  - Comments updated
  - No schema structure changes

**Badge Display:**

In the footer, show suggested version:
```
Current: v2.1.0
Suggested: v3.0.0 (MAJOR - breaking changes detected)

[Override to v2.2.0] [Accept v3.0.0]
```

### 5. Keyboard Shortcuts

Power users can use shortcuts for common actions.

| Shortcut | Action |
|----------|--------|
| `Cmd+S` / `Ctrl+S` | Save draft |
| `Cmd+Enter` / `Ctrl+Enter` | Publish schema |
| `Cmd+K` / `Ctrl+K` | Format/prettify JSON |
| `Cmd+F` / `Ctrl+F` | Find in schema |
| `Cmd+H` / `Ctrl+H` | Find and replace |
| `Cmd+Z` / `Ctrl+Z` | Undo |
| `Cmd+Shift+Z` / `Ctrl+Y` | Redo |
| `Cmd+D` / `Ctrl+D` | Toggle diff view |
| `Cmd+/` / `Ctrl+/` | Toggle comment |
| `Alt+↑` / `Alt+↓` | Move line up/down |
| `Cmd+]` / `Ctrl+]` | Indent line |
| `Cmd+[` / `Ctrl+[` | Outdent line |

---

## Visual Wireframes

### Wireframe 1: Default Edit Mode (No Breaking Changes)

```
┌──────────────────────────────────────────────────────────────────────┐
│ SchemaEditor                                                          │
├──────────────────────────────────────────────────────────────────────┤
│ ┌────────────────────────────────────────────────────────────────┐   │
│ │ HEADER                                                          │   │
│ │ 📄 payment_event_schema  v2.1.0    🟢 Compatible   [Diff ▼]   │   │
│ │ Last saved: Just now                                           │   │
│ └────────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ ┌────────────────────────────────────────────────────────────────┐   │
│ │ TOOLBAR                                                         │   │
│ │ ↶ ↷ │ 🎨 Format │ 🔍 Search │ 📋 Templates │ + Custom         │   │
│ └────────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ ┌────────────────────────────────────────────────────────────────┐   │
│ │ EDITOR (Monaco)                                           Mini│   │
│ │  1  {                                                      map│   │
│ │  2    "$schema": "http://json-schema.org/draft-07/schema#",  │   │
│ │  3    "type": "object",                                       │   │
│ │  4    "title": "Payment Event",                               │   │
│ │  5    "description": "Schema for payment transactions",       │   │
│ │  6    "properties": {                                         │   │
│ │  7      "amount": {                                           │   │
│ │  8        "type": "number",                                   │   │
│ │  9        "description": "Payment amount",                    │   │
│ │ 10        "minimum": 0  ← 🟢 Added constraint (MINOR)         │   │
│ │ 11      },                                                    │   │
│ │ 12      "currency": {                                         │   │
│ │ 13        "type": "string",                                   │   │
│ │ 14        "enum": ["USD", "EUR", "GBP"]                       │   │
│ │ 15      },                                                    │   │
│ │ 16      "timestamp": {                                        │   │
│ │ 17        "type": "string",                                   │   │
│ │ 18        "format": "date-time"                               │   │
│ │ 19      }                                                     │   │
│ │ 20    },                                                      │   │
│ │ 21    "required": ["amount", "currency", "timestamp"]         │   │
│ │ 22  }                                                         │   │
│ │                                                               │   │
│ └────────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ ┌────────────────────────────────────────────────────────────────┐   │
│ │ COMPATIBILITY PANEL (Collapsible)                              │   │
│ │ ✅ Schema is compatible with v2.0.0                            │   │
│ │                                                                │   │
│ │ Suggested Version: v2.1.0 (MINOR)                              │   │
│ │                                                                │   │
│ │ Additive Changes (1):                                          │   │
│ │  🟢 Line 10: Added 'minimum' constraint to 'amount'            │   │
│ │     Impact: LOW - existing records already satisfy this        │   │
│ └────────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ ┌────────────────────────────────────────────────────────────────┐   │
│ │ FOOTER                                                          │   │
│ │ Ln 10, Col 18 │ 523 chars │ ✅ Valid │ 💾 Auto-saved 5s ago    │   │
│ │                                                                │   │
│ │                    [Cancel]  [Save Draft]  [Publish v2.1.0]   │   │
│ └────────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────┘
```

### Wireframe 2: Diff View (Breaking Changes Detected)

```
┌──────────────────────────────────────────────────────────────────────┐
│ SchemaEditor - Diff Mode                                              │
├──────────────────────────────────────────────────────────────────────┤
│ ┌────────────────────────────────────────────────────────────────┐   │
│ │ HEADER                                                          │   │
│ │ 📄 payment_event_schema  v2.1.0    🔴 Breaking   [Edit ▼]     │   │
│ │ Comparing: v2.0.0 ← → v2.1.0                                   │   │
│ └────────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ ┌────────────────────────────────────────────────────────────────┐   │
│ │ TOOLBAR                                                         │   │
│ │ [Previous Change] [Next Change] │ Filters: All ▼                │   │
│ └────────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ ┌───────────────────────────┬────────────────────────────────────┐   │
│ │ v2.0.0 (Previous)         │ v2.1.0 (Current)                   │   │
│ ├───────────────────────────┼────────────────────────────────────┤   │
│ │  1  {                     │  1  {                              │   │
│ │  2    "$schema": "...",   │  2    "$schema": "...",            │   │
│ │  3    "type": "object",   │  3    "type": "object",            │   │
│ │  4    "properties": {     │  4    "properties": {              │   │
│ │  5      "amount": {       │  5      "amount": {                │   │
│ │  6        "type": "number"│  6        "type": "number",        │   │
│ │  7                        │  7        "minimum": 0  ← GREEN    │   │
│ │  8      },                │  8      },                         │   │
│ │  9      "currency": {     │  9      "currency": {              │   │
│ │ 10        "enum": [       │ 10        "enum": [                │   │
│ │ 11          "USD",        │ 11          "USD",                 │   │
│ │ 12          "EUR",        │ 12          "EUR"                  │   │
│ │ 13          "GBP",  ← RED │ 13                   ← RED         │   │
│ │ 14          "JPY"   ← RED │ 14                   ← RED         │   │
│ │ 15        ]               │ 15        ]                        │   │
│ │ 16      }                 │ 16      }                          │   │
│ │ 17    },                  │ 17    },                           │   │
│ │ 18    "required": [...]   │ 18    "required": [...]            │   │
│ │ 19  }                     │ 19  }                              │   │
│ └───────────────────────────┴────────────────────────────────────┘   │
│                                                                       │
│ ┌────────────────────────────────────────────────────────────────┐   │
│ │ COMPATIBILITY PANEL                                             │   │
│ │ 🔴 BREAKING CHANGES DETECTED                                   │   │
│ │                                                                │   │
│ │ Suggested Version: v3.0.0 (MAJOR)                              │   │
│ │                                                                │   │
│ │ Breaking Changes (1):                                          │   │
│ │  🔴 Lines 13-14: Enum values removed from 'currency'           │   │
│ │     Removed: ["GBP", "JPY"]                                    │   │
│ │     Impact: HIGH - 234 records use these currencies            │   │
│ │     Affected: 12% of total records (1,950 total)               │   │
│ │                                                                │   │
│ │ Additive Changes (1):                                          │   │
│ │  🟢 Line 7: Added 'minimum' constraint to 'amount'             │   │
│ │     Impact: MEDIUM - 12 records have negative amounts          │   │
│ │                                                                │   │
│ │ Migration Required: YES                                        │   │
│ │ Estimated Time: ~5 minutes (246 records to migrate)            │   │
│ │                                                                │   │
│ │ [View Migration Plan] [Override to v2.1.0]                     │   │
│ └────────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ ┌────────────────────────────────────────────────────────────────┐   │
│ │ FOOTER                                                          │   │
│ │ 2 differences found │ 1 breaking, 1 additive                   │   │
│ │                                                                │   │
│ │                    [Cancel]  [Save Draft]  [Publish v3.0.0]   │   │
│ └────────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────┘
```

### Wireframe 3: Validation Errors State

```
┌──────────────────────────────────────────────────────────────────────┐
│ SchemaEditor                                                          │
├──────────────────────────────────────────────────────────────────────┤
│ ┌────────────────────────────────────────────────────────────────┐   │
│ │ HEADER                                                          │   │
│ │ 📄 patient_record_schema  v1.2.0    🔴 Invalid   [Diff]        │   │
│ │ Unsaved changes                                                │   │
│ └────────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ ┌────────────────────────────────────────────────────────────────┐   │
│ │ EDITOR                                                          │   │
│ │  1  {                                                           │   │
│ │  2    "$schema": "http://json-schema.org/draft-07/schema#",    │   │
│ │  3    "type": "object",                                         │   │
│ │  4    "properties": {                                           │   │
│ │  5      "patientId": {                                          │   │
│ │  6        "type": "number"  🔴 ← Missing description (policy)   │   │
│ │  7      },                                                      │   │
│ │  8      "dateOfBirth": {                                        │   │
│ │  9        "type": "string",                                     │   │
│ │ 10        "format": "dat"  🔴 ← Invalid format (should be 'date')│   │
│ │ 11      },                                                      │   │
│ │ 12      "medications": {                                        │   │
│ │ 13        "type": "array",                                      │   │
│ │ 14        "items": {                                            │   │
│ │ 15          "type": "object"                                    │   │
│ │ 16          "properties": {  🔴 ← Missing comma                 │   │
│ │ 17            "name": { "type": "string" },                     │   │
│ │ 18            "dosage": { "type": "number" }                    │   │
│ │ 19          }                                                   │   │
│ │ 20        }                                                     │   │
│ │ 21      }                                                       │   │
│ │ 22    },                                                        │   │
│ │ 23    "required": ["patientId", "dateOfBirth", "ssn"]          │   │
│ │ 24  }                     🔴 ← 'ssn' not defined in properties  │   │
│ └────────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ ┌────────────────────────────────────────────────────────────────┐   │
│ │ VALIDATION PANEL                                                │   │
│ │ ❌ 4 errors found - Cannot publish until resolved              │   │
│ │                                                                │   │
│ │ 🔴 Line 6: Missing required 'description' field                │   │
│ │    Policy violation: All fields must have descriptions         │   │
│ │    Quick fix: [Add description]                                │   │
│ │                                                                │   │
│ │ 🔴 Line 10: Invalid format value 'dat'                         │   │
│ │    Did you mean 'date'?                                        │   │
│ │    Quick fix: [Replace with 'date']                            │   │
│ │                                                                │   │
│ │ 🔴 Line 16: Syntax error - Missing comma                       │   │
│ │    Expected ',' after "type": "object"                         │   │
│ │    Quick fix: [Add comma]                                      │   │
│ │                                                                │   │
│ │ 🔴 Line 23: Field 'ssn' in 'required' not defined              │   │
│ │    Field must exist in 'properties'                            │   │
│ │    Quick fix: [Add ssn to properties] [Remove from required]   │   │
│ └────────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ ┌────────────────────────────────────────────────────────────────┐   │
│ │ FOOTER                                                          │   │
│ │ Ln 16, Col 21 │ 612 chars │ ❌ 4 errors │ 💾 Not saved         │   │
│ │                                                                │   │
│ │                    [Cancel]  [Save Draft]  [Publish] (disabled)│   │
│ └────────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────┘
```

### Wireframe 4: Collaborative Editing

```
┌──────────────────────────────────────────────────────────────────────┐
│ SchemaEditor - Collaborative Mode                                     │
├──────────────────────────────────────────────────────────────────────┤
│ ┌────────────────────────────────────────────────────────────────┐   │
│ │ HEADER                                                          │   │
│ │ 📄 order_schema  v3.0.0    🟢 Valid                            │   │
│ │ 👤 3 users editing: Alice (you), Bob, Carol                    │   │
│ └────────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ ┌────────────────────────────────────────────────────────────────┐   │
│ │ EDITOR                                                          │   │
│ │  1  {                                                           │   │
│ │  2    "$schema": "http://json-schema.org/draft-07/schema#",    │   │
│ │  3    "type": "object",                                         │   │
│ │  4    "properties": {                                           │   │
│ │  5      "orderId": {                                            │   │
│ │  6        "type": "string"                                      │   │
│ │  7      },                                                      │   │
│ │  8      "items": {  ← 🔵 Bob is editing here                   │   │
│ │  9        "type": "array",                                      │   │
│ │ 10        "items": {                                            │   │
│ │ 11          "type": "object",                                   │   │
│ │ 12          "properties": {                                     │   │
│ │ 13            "productId": { "type": "string" },                │   │
│ │ 14            "quantity": { "type": "number" }                  │   │
│ │ 15          }|  ← 🟣 Carol is editing here                      │   │
│ │ 16        }                                                     │   │
│ │ 17      }                                                       │   │
│ │ 18    }                                                         │   │
│ │ 19  }                                                           │   │
│ └────────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ ┌────────────────────────────────────────────────────────────────┐   │
│ │ ACTIVE USERS                                                    │   │
│ │ 👤 Alice Chen (you) - Line 5                                   │   │
│ │ 🔵 Bob Martinez - Line 8-10                                     │   │
│ │ 🟣 Carol Johnson - Line 15                                      │   │
│ └────────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ ┌────────────────────────────────────────────────────────────────┐   │
│ │ ACTIVITY LOG (Real-time)                                        │   │
│ │ Just now: Bob added 'minItems' constraint                       │   │
│ │ 2m ago: Carol updated 'quantity' description                    │   │
│ │ 5m ago: You added 'orderId' field                               │   │
│ └────────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ ┌────────────────────────────────────────────────────────────────┐   │
│ │ FOOTER                                                          │   │
│ │ Ln 5, Col 8 │ ✅ Valid │ 🔄 Synced 1s ago │ 👤 3 active        │   │
│ │                                                                │   │
│ │                    [Cancel]  [Save Draft]  [Publish v3.1.0]   │   │
│ └────────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────┘
```

---

## States & Lifecycle

### Component States

```typescript
type EditorState =
  | 'idle'                    // Editor loaded, waiting for user input
  | 'editing'                 // User is typing
  | 'validating'              // Running validation checks
  | 'checking_compatibility'  // Comparing with previous version
  | 'formatting'              // Auto-formatting JSON
  | 'saving_draft'            // Saving to local storage or backend
  | 'publishing'              // Publishing schema to registry
  | 'error'                   // Error occurred (validation, network, etc.)
  | 'success'                 // Successfully published
  | 'loading'                 // Loading initial schema or history
```

### State Transitions

```
┌─────────┐
│ Loading │ ← Initial state (fetching schema from registry)
└────┬────┘
     │
     ▼
┌─────────┐
│  Idle   │ ← Schema loaded, editor ready
└────┬────┘
     │
     │ User types
     ▼
┌─────────┐
│ Editing │ ← Content changed, validation pending
└────┬────┘
     │
     │ Debounce timer (300ms)
     ▼
┌─────────────┐
│ Validating  │ ← Running syntax + semantic validation
└────┬────────┘
     │
     ├─ Valid ──────────────────────────┐
     │                                  │
     │ Invalid                          ▼
     ▼                          ┌─────────────────────┐
┌─────────┐                    │ Checking            │
│  Error  │                    │ Compatibility       │
└────┬────┘                    └──────┬──────────────┘
     │                                │
     │ Fix error                      │ Complete
     ▼                                ▼
┌─────────┐                    ┌─────────┐
│ Editing │                    │  Idle   │
└─────────┘                    └────┬────┘
                                    │
                                    │ User clicks "Publish"
                                    ▼
                             ┌─────────────┐
                             │ Publishing  │
                             └──────┬──────┘
                                    │
                                    ├─ Success ─┐
                                    │           │
                                    │ Error     ▼
                                    ▼      ┌─────────┐
                             ┌─────────┐   │ Success │
                             │  Error  │   └─────────┘
                             └─────────┘
```

### State Indicators (UI)

Each state has visual feedback:

| State | Header Badge | Footer Status | Publish Button |
|-------|-------------|---------------|----------------|
| `idle` | 🟢 Valid | ✅ Ready to publish | Enabled |
| `editing` | 🟡 Validating... | 💾 Unsaved changes | Disabled |
| `validating` | 🟡 Validating... | 🔄 Validating... | Disabled |
| `checking_compatibility` | 🟡 Checking... | 🔄 Checking compatibility... | Disabled |
| `error` | 🔴 Invalid | ❌ 3 errors found | Disabled |
| `publishing` | 🟡 Publishing... | 🚀 Publishing v3.0.0... | Disabled (loading) |
| `success` | 🟢 Published | ✅ Published v3.0.0 | Hidden (show "Edit" instead) |

---

## Multi-Domain Examples

### 1. Finance: Transaction Schema

**Context:** A fintech company needs to version their transaction event schema as they add support for crypto payments.

**Initial Schema (v1.0.0):**
```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "title": "Transaction Event",
  "properties": {
    "transactionId": { "type": "string" },
    "amount": { "type": "number", "minimum": 0 },
    "currency": {
      "type": "string",
      "enum": ["USD", "EUR", "GBP"]
    },
    "paymentMethod": {
      "type": "string",
      "enum": ["card", "bank_transfer", "paypal"]
    },
    "timestamp": { "type": "string", "format": "date-time" }
  },
  "required": ["transactionId", "amount", "currency", "timestamp"]
}
```

**Updated Schema (v2.0.0) - Adding Crypto Support:**
```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "title": "Transaction Event",
  "properties": {
    "transactionId": { "type": "string" },
    "amount": { "type": "number", "minimum": 0 },
    "currency": {
      "type": "string",
      "enum": ["USD", "EUR", "GBP", "BTC", "ETH"]  // ← ADDITIVE
    },
    "paymentMethod": {
      "type": "string",
      "enum": ["card", "bank_transfer", "crypto_wallet"]  // ← BREAKING (removed 'paypal')
    },
    "timestamp": { "type": "string", "format": "date-time" },
    "blockchainTxHash": {  // ← ADDITIVE (new optional field)
      "type": "string",
      "pattern": "^0x[a-fA-F0-9]{64}$"
    }
  },
  "required": ["transactionId", "amount", "currency", "timestamp"]
}
```

**SchemaEditor Analysis:**
- **Breaking Change**: `paymentMethod` enum removed 'paypal' → MAJOR version bump required
- **Additive Changes**: New currencies, new optional field → Would be MINOR, but breaking change forces MAJOR
- **Suggested Version**: v2.0.0
- **Impact**: 1,234 transactions used 'paypal' → migration required

**User Workflow:**
1. Load v1.0.0 schema
2. Make changes (add crypto fields, update enums)
3. Editor detects breaking change (paypal removal)
4. Compatibility panel shows: "🔴 MAJOR version bump required - 1,234 records affected"
5. User reviews migration plan
6. Publishes as v2.0.0

### 2. Healthcare: Patient Record Schema

**Context:** A hospital system needs to add HIPAA-required audit fields to patient records.

**Initial Schema (v1.5.0):**
```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "title": "Patient Record",
  "properties": {
    "patientId": { "type": "string" },
    "firstName": { "type": "string" },
    "lastName": { "type": "string" },
    "dateOfBirth": { "type": "string", "format": "date" },
    "medications": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "name": { "type": "string" },
          "dosage": { "type": "string" }
        }
      }
    }
  },
  "required": ["patientId", "firstName", "lastName"]
}
```

**Updated Schema (v1.6.0) - Adding Audit Fields:**
```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "title": "Patient Record",
  "properties": {
    "patientId": { "type": "string" },
    "firstName": { "type": "string" },
    "lastName": { "type": "string" },
    "dateOfBirth": { "type": "string", "format": "date" },
    "medications": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "name": { "type": "string" },
          "dosage": { "type": "string" }
        }
      }
    },
    "auditLog": {  // ← ADDITIVE (new optional field)
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "action": { "type": "string" },
          "performedBy": { "type": "string" },
          "timestamp": { "type": "string", "format": "date-time" }
        },
        "required": ["action", "performedBy", "timestamp"]
      }
    }
  },
  "required": ["patientId", "firstName", "lastName"]
}
```

**SchemaEditor Analysis:**
- **No Breaking Changes**: All changes are additive
- **Suggested Version**: v1.6.0 (MINOR)
- **Impact**: No migration required, backward compatible

**User Workflow:**
1. Load v1.5.0 schema
2. Add `auditLog` field (optional)
3. Editor validates: ✅ No breaking changes
4. Compatibility panel: "🟢 Compatible - MINOR version bump"
5. Publishes as v1.6.0

### 3. Legal: Case Document Schema

**Context:** A law firm needs to enforce stricter validation on case numbers after discovering data quality issues.

**Initial Schema (v2.3.0):**
```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "title": "Legal Case Document",
  "properties": {
    "caseNumber": {
      "type": "string"  // ← No validation
    },
    "filingDate": { "type": "string", "format": "date" },
    "courtName": { "type": "string" },
    "plaintiffs": {
      "type": "array",
      "items": { "type": "string" }
    },
    "defendants": {
      "type": "array",
      "items": { "type": "string" }
    }
  },
  "required": ["caseNumber", "filingDate"]
}
```

**Updated Schema (v3.0.0) - Stricter Validation:**
```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "title": "Legal Case Document",
  "properties": {
    "caseNumber": {
      "type": "string",
      "pattern": "^[A-Z]{2}-[0-9]{4}-[0-9]{6}$"  // ← BREAKING (new constraint)
    },
    "filingDate": { "type": "string", "format": "date" },
    "courtName": {
      "type": "string",
      "minLength": 1  // ← BREAKING (was optional, now required non-empty)
    },
    "plaintiffs": {
      "type": "array",
      "items": { "type": "string" },
      "minItems": 1  // ← BREAKING (must have at least one plaintiff)
    },
    "defendants": {
      "type": "array",
      "items": { "type": "string" },
      "minItems": 1  // ← BREAKING (must have at least one defendant)
    }
  },
  "required": ["caseNumber", "filingDate", "courtName"]  // ← BREAKING (courtName now required)
}
```

**SchemaEditor Analysis:**
- **Breaking Changes**:
  - `caseNumber` pattern added (567 existing records don't match)
  - `courtName` now required (89 records missing this field)
  - `minItems` constraints added (12 records have empty arrays)
- **Suggested Version**: v3.0.0 (MAJOR)
- **Impact**: 668 records require migration

**User Workflow:**
1. Load v2.3.0 schema
2. Add validation constraints
3. Editor flags multiple breaking changes
4. Compatibility panel shows: "🔴 MAJOR - 668 records need migration"
5. User clicks "View Migration Plan" to see data transformation steps
6. Publishes as v3.0.0 with migration script

### 4. Research (RSRCH - Utilitario): Founder Fact Schema

**Context:** RSRCH team needs to track additional metadata for fact sources.

**Initial Schema (v1.0.0):**
```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "title": "Founder Fact",
  "properties": {
    "factId": { "type": "string" },
    "claim": { "type": "string" },
    "sources": {
      "type": "object",
      "properties": {
        "sourceType": { "type": "string", "enum": ["web_article", "podcast", "tweet"] },
        "measurements": { "type": "array", "items": { "type": "number" } }
      }
    },
    "conductedBy": { "type": "string" }
  },
  "required": ["experimentId", "hypothesis"]
}
```

**Updated Schema (v1.1.0) - Adding Reproducibility Metadata:**
```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "title": "Experiment Result",
  "properties": {
    "experimentId": { "type": "string" },
    "hypothesis": { "type": "string" },
    "results": {
      "type": "object",
      "properties": {
        "outcome": { "type": "string", "enum": ["success", "failure", "inconclusive"] },
        "measurements": { "type": "array", "items": { "type": "number" } }
      }
    },
    "conductedBy": { "type": "string" },
    "reproducibility": {  // ← ADDITIVE
      "type": "object",
      "properties": {
        "randomSeed": { "type": "integer" },
        "environmentVersion": { "type": "string" },
        "dependencies": {
          "type": "array",
          "items": {
            "type": "object",
            "properties": {
              "package": { "type": "string" },
              "version": { "type": "string" }
            }
          }
        }
      }
    }
  },
  "required": ["experimentId", "hypothesis"]
}
```

**SchemaEditor Analysis:**
- **No Breaking Changes**: All new fields are optional
- **Suggested Version**: v1.1.0 (MINOR)
- **Impact**: Fully backward compatible

### 5. E-commerce: Product Schema

**Context:** An e-commerce platform needs to support product variants (size, color).

**Initial Schema (v2.0.0):**
```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "title": "Product",
  "properties": {
    "productId": { "type": "string" },
    "name": { "type": "string" },
    "price": { "type": "number", "minimum": 0 },
    "category": { "type": "string" },
    "inStock": { "type": "boolean" }
  },
  "required": ["productId", "name", "price"]
}
```

**Updated Schema (v2.1.0) - Adding Variants:**
```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "title": "Product",
  "properties": {
    "productId": { "type": "string" },
    "name": { "type": "string" },
    "price": { "type": "number", "minimum": 0 },
    "category": { "type": "string" },
    "inStock": { "type": "boolean" },
    "variants": {  // ← ADDITIVE
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "sku": { "type": "string" },
          "attributes": {
            "type": "object",
            "additionalProperties": { "type": "string" }
          },
          "priceAdjustment": { "type": "number" },
          "stock": { "type": "integer", "minimum": 0 }
        },
        "required": ["sku"]
      }
    }
  },
  "required": ["productId", "name", "price"]
}
```

**SchemaEditor Analysis:**
- **No Breaking Changes**: Variants are optional
- **Suggested Version**: v2.1.0 (MINOR)

### 6. SaaS: API Event Schema

**Context:** A SaaS platform needs to deprecate old event types.

**Initial Schema (v3.5.0):**
```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "title": "API Event",
  "properties": {
    "eventId": { "type": "string" },
    "eventType": {
      "type": "string",
      "enum": ["user.created", "user.updated", "user.deleted", "user.login", "user.logout"]
    },
    "payload": { "type": "object" },
    "timestamp": { "type": "string", "format": "date-time" }
  },
  "required": ["eventId", "eventType", "timestamp"]
}
```

**Updated Schema (v4.0.0) - Deprecating Events:**
```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "title": "API Event",
  "properties": {
    "eventId": { "type": "string" },
    "eventType": {
      "type": "string",
      "enum": ["user.created", "user.updated", "user.deleted", "session.started", "session.ended"]
      // ← BREAKING: Removed "user.login", "user.logout"
      // ← ADDITIVE: Added "session.started", "session.ended"
    },
    "payload": { "type": "object" },
    "timestamp": { "type": "string", "format": "date-time" }
  },
  "required": ["eventId", "eventType", "timestamp"]
}
```

**SchemaEditor Analysis:**
- **Breaking Changes**: Removed event types
- **Additive Changes**: New event types
- **Suggested Version**: v4.0.0 (MAJOR)

### 7. Insurance: Claim Schema

**Context:** An insurance company needs to add fraud detection scores.

**Initial Schema (v1.8.0):**
```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "title": "Insurance Claim",
  "properties": {
    "claimId": { "type": "string" },
    "policyNumber": { "type": "string" },
    "claimAmount": { "type": "number", "minimum": 0 },
    "incidentDate": { "type": "string", "format": "date" },
    "status": {
      "type": "string",
      "enum": ["pending", "approved", "denied"]
    }
  },
  "required": ["claimId", "policyNumber", "claimAmount"]
}
```

**Updated Schema (v1.9.0) - Adding Fraud Detection:**
```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "title": "Insurance Claim",
  "properties": {
    "claimId": { "type": "string" },
    "policyNumber": { "type": "string" },
    "claimAmount": { "type": "number", "minimum": 0 },
    "incidentDate": { "type": "string", "format": "date" },
    "status": {
      "type": "string",
      "enum": ["pending", "approved", "denied", "under_review"]  // ← ADDITIVE
    },
    "fraudScore": {  // ← ADDITIVE
      "type": "number",
      "minimum": 0,
      "maximum": 1,
      "description": "ML-generated fraud likelihood (0=unlikely, 1=very likely)"
    },
    "flaggedReasons": {  // ← ADDITIVE
      "type": "array",
      "items": { "type": "string" }
    }
  },
  "required": ["claimId", "policyNumber", "claimAmount"]
}
```

**SchemaEditor Analysis:**
- **No Breaking Changes**: All additions are optional or extend existing enums
- **Suggested Version**: v1.9.0 (MINOR)

---

## User Interactions

### 1. Loading a Schema

**Trigger:** User navigates to schema editor with schemaId

**Flow:**
1. Show loading skeleton in editor
2. Fetch schema from registry API: `GET /schemas/{schemaId}/versions/{version}`
3. If `previousSchema` prop provided, also fetch that version
4. Initialize Monaco editor with schema content
5. Run initial validation
6. If diff mode enabled, set up diff editor
7. Transition to `idle` state

**Error Handling:**
- 404: Schema not found → Show error message with "Create new schema" option
- 403: Permission denied → Show error message
- Network error: Show retry button

### 2. Editing Schema Content

**Trigger:** User types in Monaco editor

**Flow:**
1. Capture keystroke in Monaco
2. Update editor content (Monaco handles this)
3. Mark as `isDirty = true`
4. Start debounce timer (300ms)
5. On timer complete:
   - Parse JSON
   - Run syntax validation
   - Run semantic validation
   - If `previousSchema` exists, run compatibility check
   - Update validation panel
   - Update compatibility panel
   - Calculate suggested version bump

**Optimizations:**
- Debounce validation to avoid lag
- Use Web Workers for heavy validation (large schemas)
- Cache validation results for unchanged portions

### 3. Formatting Schema

**Trigger:** User clicks "Format" button or presses Cmd+K

**Flow:**
1. Get current editor content
2. Parse JSON
3. If invalid JSON, show error toast
4. If valid:
   - Format with 2-space indentation
   - Sort keys alphabetically (optional)
   - Apply to editor
   - Mark as `isDirty = true`

### 4. Toggling Diff View

**Trigger:** User clicks "Diff" toggle button

**Flow:**
1. If toggling ON:
   - Hide single editor
   - Show diff editor (split pane)
   - Load `previousSchema` in left pane
   - Load `currentSchema` in right pane
   - Highlight differences
2. If toggling OFF:
   - Hide diff editor
   - Show single editor
   - Preserve cursor position

### 5. Publishing Schema

**Trigger:** User clicks "Publish" button

**Preconditions:**
- Schema is valid (no syntax/semantic errors)
- User has permission to publish

**Flow:**
1. Show confirmation dialog:
   ```
   ┌────────────────────────────────────────────┐
   │ Publish Schema?                            │
   ├────────────────────────────────────────────┤
   │ Schema: payment_event_schema               │
   │ Current Version: v2.1.0                    │
   │ New Version: v3.0.0 (MAJOR)                │
   │                                            │
   │ Breaking Changes:                          │
   │  • currency.enum reduced (removed GBP)     │
   │                                            │
   │ Impact: 234 records need migration         │
   │                                            │
   │ [Cancel] [Publish v3.0.0]                  │
   └────────────────────────────────────────────┘
   ```
2. If user confirms:
   - Set state to `publishing`
   - Call `onPublish(schema, versionBump, breakingChanges)`
   - Show progress indicator
   - On success:
     - Show success message
     - Transition to `success` state
     - Optionally redirect to schema details page
   - On error:
     - Show error message
     - Keep editor open with unsaved changes

### 6. Auto-Save

**Trigger:** Auto-save timer (default 30s)

**Flow:**
1. If `isDirty` and `isValid`:
   - Call `onAutoSave(schema)`
   - Save to localStorage or backend
   - Update "Last saved" timestamp
   - Keep `isDirty = true` (auto-save is not a manual save)

### 7. Inserting Template

**Trigger:** User clicks "Templates" dropdown and selects a template

**Flow:**
1. Show template picker dialog
2. User selects template (e.g., "Email field with validation")
3. Get cursor position in editor
4. Insert template JSON at cursor:
   ```json
   "email": {
     "type": "string",
     "format": "email",
     "description": "User email address"
   }
   ```
5. Auto-format inserted content
6. Mark as `isDirty = true`

### 8. Handling Validation Errors

**Trigger:** Validation detects errors

**Flow:**
1. Update `validationErrors` array
2. Display errors in ValidationPanel
3. Add error markers to Monaco editor (red squiggles)
4. Add gutter icons (🔴 for errors, ⚠️ for warnings)
5. Disable "Publish" button
6. On hover over error, show tooltip with details

**Interactive Error Fixes:**
Some errors offer "Quick Fix" buttons:
- Missing comma → "Add comma"
- Invalid enum → "Replace with valid value"
- Missing required field → "Add field to properties"

### 9. Collaborative Editing

**Trigger:** `enableCollaboration=true` and WebSocket connection established

**Flow:**
1. Connect to collaboration session via WebSocket
2. Broadcast cursor position on change
3. Receive remote cursor positions from other users
4. Display remote cursors in editor (colored carets)
5. Show active users in header
6. On remote edit:
   - Apply Operational Transform (OT) to resolve conflicts
   - Update editor content
   - Preserve local cursor position

**Conflict Resolution:**
- Use OT algorithm (similar to Google Docs)
- Last write wins for simple conflicts
- Show conflict banner if major divergence detected

---

## Accessibility

### WCAG 2.1 AA Compliance

#### 1. Keyboard Navigation

All editor functions are accessible via keyboard:

| Action | Keyboard Shortcut | Focus Behavior |
|--------|------------------|----------------|
| Navigate to editor | `Tab` | Focus enters Monaco editor |
| Navigate toolbar | `Shift+Tab` | Focus moves to toolbar |
| Open template picker | `Alt+T` | Opens dropdown, focus on first template |
| Publish schema | `Cmd+Enter` | Opens publish dialog |
| Cancel | `Esc` | Closes dialogs, returns to editor |
| Navigate errors | `F8` (next), `Shift+F8` (prev) | Jump to next/previous error |

#### 2. Screen Reader Support

**ARIA Labels:**
```html
<div role="application" aria-label="Schema Editor">
  <div role="toolbar" aria-label="Editor Actions">
    <button aria-label="Undo last change" aria-keyshortcuts="Ctrl+Z">↶</button>
    <button aria-label="Redo last change" aria-keyshortcuts="Ctrl+Y">↷</button>
    <button aria-label="Format JSON schema" aria-keyshortcuts="Ctrl+K">🎨 Format</button>
  </div>

  <div role="textbox" aria-label="Schema content editor" aria-multiline="true" aria-describedby="editor-help">
    <!-- Monaco editor -->
  </div>

  <div role="region" aria-label="Validation results" aria-live="polite">
    <ul role="list">
      <li role="listitem">
        <span aria-label="Error on line 11">🔴 Line 11: Enum value removed</span>
      </li>
    </ul>
  </div>
</div>
```

**Live Regions:**
- Validation panel: `aria-live="polite"` announces new errors
- Compatibility panel: `aria-live="polite"` announces breaking changes
- Status bar: `aria-live="polite"` announces save status

**Announcements:**
```
"Schema validated successfully. No errors found."
"Breaking change detected on line 11. Enum value removed."
"Schema published as version 3.0.0."
```

#### 3. Color Contrast

All colors meet WCAG AA standards (4.5:1 ratio):

| Element | Foreground | Background | Ratio |
|---------|-----------|------------|-------|
| Error text | #DC2626 | #FFFFFF | 5.8:1 |
| Warning text | #D97706 | #FFFFFF | 4.9:1 |
| Success text | #059669 | #FFFFFF | 4.7:1 |
| Editor text | #1F2937 | #FFFFFF | 15.3:1 |

**High Contrast Mode:**
When user enables high contrast (prefers-contrast: high):
- Increase border thickness (1px → 2px)
- Use pure black/white instead of grays
- Add underlines to error text (not just color)

#### 4. Focus Indicators

All interactive elements have visible focus indicators:
```css
button:focus-visible {
  outline: 2px solid #2563EB;
  outline-offset: 2px;
  border-radius: 4px;
}
```

#### 5. Error Identification

Errors are identified in multiple ways (not just color):
- Icon: 🔴 (visual indicator)
- Text: "Error" label
- Position: "Line 11" location
- Sound: Optional beep on validation failure (user preference)

#### 6. Text Resizing

Editor supports text resizing up to 200%:
- Monaco editor font size: configurable (default 14px)
- UI scales proportionally
- No horizontal scrolling required at 200% zoom

---

## Performance Optimizations

### 1. Debounced Validation

Avoid running expensive validation on every keystroke:

```typescript
const debouncedValidate = useMemo(
  () => debounce((schema: string) => {
    validateSchema(schema)
    checkCompatibility(schema)
  }, 300),
  []
)

// In Monaco onChange
editor.onDidChangeModelContent(() => {
  debouncedValidate(editor.getValue())
})
```

### 2. Web Workers for Validation

For large schemas (>1000 lines), run validation in Web Worker:

```typescript
// validation.worker.ts
self.addEventListener('message', (e) => {
  const { schema, previousSchema } = e.data

  const validationErrors = validateSchema(schema)
  const compatibilityReport = checkCompatibility(schema, previousSchema)

  self.postMessage({ validationErrors, compatibilityReport })
})

// In component
const validationWorker = useRef<Worker>()

useEffect(() => {
  validationWorker.current = new Worker('/validation.worker.js')

  validationWorker.current.onmessage = (e) => {
    setValidationErrors(e.data.validationErrors)
    setCompatibilityReport(e.data.compatibilityReport)
  }

  return () => validationWorker.current?.terminate()
}, [])
```

### 3. Virtualized Rendering

For very long schemas, Monaco editor automatically virtualizes rendering (only visible lines are rendered).

### 4. Memoized Diff Calculation

Cache diff calculation results:

```typescript
const diffResult = useMemo(() => {
  if (!previousSchema || !currentSchema) return null
  return calculateDiff(previousSchema, currentSchema)
}, [previousSchema, currentSchema])
```

### 5. Incremental Validation

Only re-validate changed portions of schema:

```typescript
function incrementalValidation(schema: object, changedPath: string[]) {
  // Only validate the subtree that changed
  const subtree = getAtPath(schema, changedPath)
  return validateSubtree(subtree, changedPath)
}
```

### 6. Lazy Loading of Monaco Editor

Code-split Monaco editor to reduce initial bundle size:

```typescript
const MonacoEditor = lazy(() => import('@monaco-editor/react'))

// In component
<Suspense fallback={<EditorSkeleton />}>
  <MonacoEditor {...props} />
</Suspense>
```

### 7. Optimistic UI Updates

For auto-save, update UI immediately without waiting for server response:

```typescript
const handleAutoSave = async (schema: string) => {
  setLastSaved(new Date())  // Optimistic update

  try {
    await api.saveSchema(schema)
  } catch (error) {
    setLastSaved(null)  // Rollback on error
    toast.error('Auto-save failed')
  }
}
```

---

## Integration Examples

### Example 1: React Integration

```typescript
import { SchemaEditor } from '@/components/il/SchemaEditor'
import { useState } from 'react'
import { useRouter } from 'next/navigation'

export default function SchemaEditPage({ params }: { params: { schemaId: string } }) {
  const router = useRouter()
  const [schema, setSchema] = useState<string>('')
  const [isValid, setIsValid] = useState(false)

  const handlePublish = async (
    schema: string,
    versionBump: VersionBump,
    breakingChanges: BreakingChange[]
  ) => {
    const response = await fetch(`/api/schemas/${params.schemaId}/publish`, {
      method: 'POST',
      body: JSON.stringify({
        schema: JSON.parse(schema),
        versionBump,
        breakingChanges
      })
    })

    if (response.ok) {
      const { version } = await response.json()
      toast.success(`Published as ${version}`)
      router.push(`/schemas/${params.schemaId}`)
    } else {
      const { error } = await response.json()
      toast.error(`Publish failed: ${error}`)
    }
  }

  return (
    <SchemaEditor
      schemaId={params.schemaId}
      currentVersion="2.1.0"
      initialSchema={schema}
      previousSchema={previousSchemaContent}
      previousVersion="2.0.0"
      onChange={(newSchema, valid) => {
        setSchema(newSchema)
        setIsValid(valid)
      }}
      onPublish={handlePublish}
      onCancel={() => router.back()}
      features={{
        autoCompletion: true,
        syntaxValidation: true,
        compatibilityCheck: true,
        diffView: true
      }}
      theme="vs-dark"
      height="600px"
    />
  )
}
```

### Example 2: With React Query

```typescript
import { useQuery, useMutation } from '@tanstack/react-query'

export function useSchemaEditor(schemaId: string, version: string) {
  // Fetch current schema
  const { data: currentSchema, isLoading } = useQuery({
    queryKey: ['schema', schemaId, version],
    queryFn: () => api.getSchema(schemaId, version)
  })

  // Fetch previous schema for diff
  const { data: previousSchema } = useQuery({
    queryKey: ['schema', schemaId, getPreviousVersion(version)],
    queryFn: () => api.getSchema(schemaId, getPreviousVersion(version)),
    enabled: !!version
  })

  // Publish mutation
  const publishMutation = useMutation({
    mutationFn: (data: PublishSchemaRequest) => api.publishSchema(schemaId, data),
    onSuccess: () => {
      queryClient.invalidateQueries(['schema', schemaId])
    }
  })

  return {
    currentSchema,
    previousSchema,
    isLoading,
    publish: publishMutation.mutate,
    isPublishing: publishMutation.isPending
  }
}

// In component
function SchemaEditorPage({ schemaId }: { schemaId: string }) {
  const { currentSchema, previousSchema, publish, isPublishing } = useSchemaEditor(schemaId, '2.1.0')

  return (
    <SchemaEditor
      schemaId={schemaId}
      currentVersion="2.1.0"
      initialSchema={currentSchema?.content}
      previousSchema={previousSchema?.content}
      previousVersion={previousSchema?.version}
      onPublish={(schema, versionBump, breakingChanges) => {
        publish({ schema, versionBump, breakingChanges })
      }}
      isLoading={isPublishing}
    />
  )
}
```

### Example 3: With Custom Validation Rules

```typescript
const customValidationRules: ValidationRule[] = [
  {
    id: 'require-descriptions',
    name: 'Require Field Descriptions',
    validate: (schema) => {
      const errors: ValidationError[] = []

      function traverse(obj: any, path: string = '$') {
        if (obj.properties) {
          Object.keys(obj.properties).forEach(key => {
            const field = obj.properties[key]
            if (!field.description) {
              errors.push({
                line: 0,  // Would need to calculate actual line
                column: 0,
                message: `Missing description for field '${key}'`,
                severity: 'error',
                code: 'MISSING_DESCRIPTION'
              })
            }
            traverse(field, `${path}.properties.${key}`)
          })
        }
      }

      traverse(schema)
      return errors
    }
  },
  {
    id: 'snake-case-fields',
    name: 'Enforce Snake Case',
    validate: (schema) => {
      const errors: ValidationError[] = []

      function traverse(obj: any, path: string = '$') {
        if (obj.properties) {
          Object.keys(obj.properties).forEach(key => {
            if (!/^[a-z][a-z0-9_]*$/.test(key)) {
              errors.push({
                line: 0,
                column: 0,
                message: `Field '${key}' should be in snake_case`,
                severity: 'warning',
                code: 'INVALID_FIELD_NAME'
              })
            }
            traverse(obj.properties[key], `${path}.properties.${key}`)
          })
        }
      }

      traverse(schema)
      return errors
    }
  }
]

<SchemaEditor
  schemaId="payment_event"
  currentVersion="2.1.0"
  initialSchema={schema}
  validationRules={{
    enforceRequired: true,
    enforceDescriptions: true,
    customRules: customValidationRules
  }}
/>
```

### Example 4: Collaborative Editing

```typescript
import { useCollaboration } from '@/hooks/useCollaboration'

function CollaborativeSchemaEditor({ schemaId }: { schemaId: string }) {
  const { session, connectedUsers, remoteCursors } = useCollaboration(schemaId)

  return (
    <SchemaEditor
      schemaId={schemaId}
      currentVersion="3.0.0"
      initialSchema={schema}
      enableCollaboration={true}
      collaborationSessionId={session.id}
      currentUser={{
        id: 'user_123',
        name: 'Alice Chen',
        avatar: '/avatars/alice.jpg',
        color: '#3B82F6'
      }}
      onChange={(schema) => {
        // Broadcast changes to other users
        session.broadcast({ type: 'schema_change', schema })
      }}
    />
  )
}
```

---

## Related Components

### 1. MigrationWizard

**Relationship:** SchemaEditor → MigrationWizard

When user publishes a schema with breaking changes, they can launch the MigrationWizard to:
- Review migration plan
- Configure migration options
- Execute data migration
- Verify results

**Integration:**
```typescript
<SchemaEditor
  onPublish={(schema, versionBump, breakingChanges) => {
    if (breakingChanges.length > 0) {
      // Open migration wizard
      setShowMigrationWizard(true)
    } else {
      // Direct publish
      publishSchema(schema)
    }
  }}
/>

{showMigrationWizard && (
  <MigrationWizard
    schemaId={schemaId}
    fromVersion={currentVersion}
    toVersion={suggestedNextVersion}
    breakingChanges={breakingChanges}
    onComplete={() => {
      publishSchema(schema)
      setShowMigrationWizard(false)
    }}
  />
)}
```

### 2. CompatibilityViewer

**Relationship:** SchemaEditor uses CompatibilityViewer internally

The CompatibilityPanel inside SchemaEditor can be extracted to a standalone CompatibilityViewer component for use in other contexts (e.g., schema history page).

**Integration:**
```typescript
<SchemaEditor
  schemaId={schemaId}
  currentVersion="2.1.0"
  initialSchema={schema}
  previousSchema={previousSchema}
  onCompatibilityCheck={(report) => {
    console.log('Compatibility report:', report)
  }}
  // Internally uses CompatibilityViewer component
/>
```

### 3. SchemaHistory

**Relationship:** SchemaEditor ← SchemaHistory

Users can navigate to SchemaEditor from SchemaHistory component:

```typescript
<SchemaHistory schemaId={schemaId}>
  {versions.map(version => (
    <VersionCard
      key={version.id}
      version={version}
      onEdit={() => {
        // Open SchemaEditor with this version
        router.push(`/schemas/${schemaId}/edit?version=${version.number}`)
      }}
    />
  ))}
</SchemaHistory>
```

### 4. SchemaConsumersList

**Relationship:** SchemaEditor → SchemaConsumersList

Before publishing breaking changes, show list of downstream consumers:

```typescript
<Dialog open={showPublishConfirmation}>
  <DialogContent>
    <h2>Publish Schema v3.0.0?</h2>

    <CompatibilityViewer
      oldSchema={previousSchema}
      newSchema={currentSchema}
      compatibilityReport={report}
    />

    <h3>Affected Consumers</h3>
    <SchemaConsumersList
      schemaId={schemaId}
      consumers={consumers}
      showImpact={true}
    />

    <DialogFooter>
      <Button onClick={confirmPublish}>Publish</Button>
    </DialogFooter>
  </DialogContent>
</Dialog>
```

### 5. SchemaValidator (Utility)

**Relationship:** SchemaEditor uses SchemaValidator service

SchemaValidator is a utility service (not a UI component) that handles validation logic:

```typescript
import { SchemaValidator } from '@/services/schema-validator'

const validator = new SchemaValidator({
  draft: '2020-12',
  strictMode: true,
  customRules: [...]
})

<SchemaEditor
  schemaId={schemaId}
  initialSchema={schema}
  onValidationError={(errors) => {
    // Errors from SchemaValidator service
  }}
/>
```

---

## References

### JSON Schema Specification
- [JSON Schema Draft 7](https://json-schema.org/draft-07/json-schema-release-notes.html)
- [JSON Schema 2020-12](https://json-schema.org/draft/2020-12/release-notes)
- [Understanding JSON Schema](https://json-schema.org/understanding-json-schema/)

### Monaco Editor
- [Monaco Editor API](https://microsoft.github.io/monaco-editor/api/index.html)
- [Monaco React](https://github.com/suren-atoyan/monaco-react)
- [Custom Language Support](https://microsoft.github.io/monaco-editor/monarch.html)

### Schema Evolution & Compatibility
- [Schema Evolution in Event Streaming](https://www.confluent.io/blog/schema-evolution-in-avro-protobuf-json-schema/)
- [Breaking Changes in APIs](https://docs.microsoft.com/en-us/azure/architecture/best-practices/api-design#versioning-a-restful-web-api)
- [Semantic Versioning](https://semver.org/)

### Diff Algorithms
- [Myers Diff Algorithm](https://blog.jcoglan.com/2017/02/12/the-myers-diff-algorithm-part-1/)
- [Diff Match Patch](https://github.com/google/diff-match-patch)

### Collaborative Editing
- [Operational Transformation](https://en.wikipedia.org/wiki/Operational_transformation)
- [Conflict-free Replicated Data Types (CRDTs)](https://crdt.tech/)
- [Yjs - Shared Editing Framework](https://github.com/yjs/yjs)

### Accessibility
- [WCAG 2.1 Guidelines](https://www.w3.org/WAI/WCAG21/quickref/)
- [ARIA Authoring Practices](https://www.w3.org/WAI/ARIA/apg/)
- [Monaco Editor Accessibility](https://github.com/microsoft/monaco-editor/wiki/Monaco-Editor-Accessibility-Guide)

### Related Projects
- [Confluent Schema Registry](https://docs.confluent.io/platform/current/schema-registry/index.html)
- [AWS Glue Schema Registry](https://docs.aws.amazon.com/glue/latest/dg/schema-registry.html)
- [Buf Schema Registry](https://buf.build/docs/bsr/introduction)

---

**Document Version:** 1.0
**Word Count:** ~9,800 words (~980 lines)
**Last Updated:** 2025-10-25
**Maintained By:** Platform Team
