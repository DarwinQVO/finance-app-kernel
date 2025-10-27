# Component: ParserCapabilitiesCard (IL)

> **Layer:** Interaction Layer (IL)
> **Type:** Card / Tooltip
> **Vertical:** 3.7 Parser Registry
> **Last Updated:** 2025-10-24

---

## Overview

**ParserCapabilitiesCard** displays parser capabilities as a card or tooltip with confidence badges for each extractable field.

**Props:**
```typescript
interface ParserCapabilitiesCardProps {
  parser: ParserRegistration
  capabilities: ParserCapability[]
  showVersion?: boolean
  onSelectVersion?: (version: string) => void
  displayMode: 'card' | 'tooltip'
}
```

---

## Wireframe (Card Mode)

```
┌──────────────────────────────────────────────┐
│ Bank of America PDF Parser v2.1.0            │
├──────────────────────────────────────────────┤
│ ✅ Date (98% confidence)                     │
│ ✅ Amount (98% confidence)                   │
│ ✅ Merchant (95% confidence)                 │
│ ✅ Account (98% confidence)                  │
│ ✅ Balance (95% confidence)                  │
│ ⚠️  Memo (70% confidence)                    │
│ ❌ Category (not supported)                  │
│                                              │
│ Supported formats: PDF                       │
│ Last updated: 2024-11-01                     │
└──────────────────────────────────────────────┘
```

---

## Multi-Domain Examples

### Healthcare: HL7 Parser
```
HL7 v2.5 ADT Parser v3.0.0
✅ Patient ID (99% confidence)
✅ Visit Number (99% confidence)
✅ Admission Date (98% confidence)
⚠️  Diagnosis Code (90% confidence)
❌ Insurance Info (not supported)
```

---

## Summary

ParserCapabilitiesCard provides **capability visualization** with confidence badges for transparent parser selection.
