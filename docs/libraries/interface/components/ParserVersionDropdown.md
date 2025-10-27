# Component: ParserVersionDropdown (IL)

> **Layer:** Interaction Layer (IL)
> **Type:** Dropdown Selector
> **Vertical:** 3.7 Parser Registry
> **Last Updated:** 2025-10-24

---

## Overview

**ParserVersionDropdown** allows selecting parser version with deprecation warnings and breaking change indicators.

**Props:**
```typescript
interface ParserVersionDropdownProps {
  parserId: string
  versions: ParserVersion[]
  currentVersion: string
  onVersionChange: (version: string) => void
}
```

---

## Wireframe

```
┌──────────────────────────────────────────┐
│ Version: v2.1.0 ▼                        │
├──────────────────────────────────────────┤
│ ✅ v2.1.0 (Latest)                       │
│ ⚠️  v2.0.0 (Breaking changes)            │
│ ❌ v1.5.0 (Deprecated - sunset Dec 31)  │
└──────────────────────────────────────────┘
```

---

## Behavior

### Deprecation Warning
```typescript
if (version.deprecated) {
  return (
    <Badge variant="warning">
      Deprecated - sunset {version.sunset_date}
      <Link to={version.migration_guide_url}>Migration Guide</Link>
    </Badge>
  )
}
```

---

## Summary

ParserVersionDropdown provides **version selection** with deprecation warnings and migration guidance.
