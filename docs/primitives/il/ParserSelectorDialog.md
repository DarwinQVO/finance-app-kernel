# Component: ParserSelectorDialog (IL)

> **Layer:** Interaction Layer (IL)
> **Type:** Modal Dialog
> **Vertical:** 3.7 Parser Registry
> **Last Updated:** 2025-10-24

---

## Overview

**ParserSelectorDialog** is a modal dialog for manually selecting a parser when auto-detection fails or user wants to override. Shows ranked parser options with capabilities, confidence scores, and version information.

**Key Features:**
- Display parser list with confidence badges
- Show parser capabilities inline (tooltip/expandable)
- Version selector per parser
- Highlight recommended parser (highest confidence)
- Search/filter parsers by name or capability

**Props:**
```typescript
interface ParserSelectorDialogProps {
  isOpen: boolean
  fileMetadata: FileMetadata
  availableParsers: RankedParser[]
  selectedParserId?: string
  onSelect: (parserId: string, version: string) => void
  onCancel: () => void
}
```

**States:**
- `loading` - Fetching parser list
- `loaded` - Parsers displayed
- `selecting` - User clicked parser
- `error` - Failed to load parsers

---

## Wireframe

```
┌──────────────────────────────────────────────────┐
│ Select Parser                                  [X]│
├──────────────────────────────────────────────────┤
│ File: BofA_Statement_Nov2024.pdf                 │
│                                                   │
│ ┌───────────────────────────────────────────┐    │
│ │ [🔍] Search parsers...                    │    │
│ └───────────────────────────────────────────┘    │
│                                                   │
│ ┌─────────────────────────────────────────┐      │
│ │ 💡 Recommended                           │      │
│ │ Bank of America PDF Parser v2.1.0        │      │
│ │ 98% confidence                           │      │
│ │ ✅ date ✅ amount ✅ merchant ✅ balance  │      │
│ │                              [Select] ✓  │      │
│ └─────────────────────────────────────────┘      │
│                                                   │
│ ┌─────────────────────────────────────────┐      │
│ │ Chase PDF Parser v1.5.3                  │      │
│ │ 55% confidence                           │      │
│ │ ✅ date ✅ amount ✅ merchant ❌ balance  │      │
│ │                              [Select]    │      │
│ └─────────────────────────────────────────┘      │
│                                                   │
│ ┌─────────────────────────────────────────┐      │
│ │ Generic PDF Parser v1.0.0                │      │
│ │ 40% confidence                           │      │
│ │ ✅ date ✅ amount ⚠️ merchant ❌ balance  │      │
│ │                              [Select]    │      │
│ └─────────────────────────────────────────┘      │
│                                                   │
│                          [Cancel] [Continue]      │
└──────────────────────────────────────────────────┘
```

---

## Behavior

### On Open
```typescript
useEffect(() => {
  if (isOpen) {
    setState('loading')
    fetchParsers(fileMetadata)
      .then(parsers => {
        setState('loaded')
        // Auto-highlight recommended parser (highest confidence)
        if (parsers.length > 0) {
          setRecommended(parsers[0].parser_id)
        }
      })
      .catch(error => setState('error'))
  }
}, [isOpen, fileMetadata])
```

### On Select Parser
```typescript
const handleSelect = (parserId: string, version: string) => {
  setState('selecting')
  onSelect(parserId, version)
  // Dialog closed by parent component
}
```

### Search/Filter
```typescript
const filteredParsers = availableParsers.filter(parser =>
  parser.name.toLowerCase().includes(searchQuery.toLowerCase()) ||
  parser.capabilities.some(cap => cap.includes(searchQuery.toLowerCase()))
)
```

---

## Multi-Domain Examples

### Finance: BofA Statement
```
File: BofA_Statement_Nov2024.pdf
Recommended: parser_bofa_pdf (98% confidence)
Alternatives: parser_generic_pdf (40%)
```

### Healthcare: HL7 Message
```
File: ADT_A01_20241101.hl7
Recommended: parser_hl7_v2_5_adt (95% confidence)
Alternatives: parser_hl7_v2_3_adt (60%), parser_generic_hl7 (50%)
```

---

## Summary

ParserSelectorDialog provides **manual parser selection** with ranked options, capability display, and confidence scoring for informed user decisions.
