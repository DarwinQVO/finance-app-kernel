# FieldOverrideIndicator

**Version:** 1.0.0
**Last Updated:** 2025-10-24
**Owner:** Information Layer (IL)
**Domain:** Corrections Flow (Vertical 4.3)
**Stability:** Stable

---

## Overview

### Purpose

The **FieldOverrideIndicator** is a small, unobtrusive visual component that signals when a field value has been manually corrected or overridden from its original state. It provides immediate visual feedback about data integrity interventions while offering detailed correction metadata through interactive tooltips.

This component is essential for maintaining transparency in data correction workflows, enabling users to quickly identify which fields have been manually adjusted and by whom, supporting audit trails and data governance requirements.

### Primary Use Cases

1. **Correction Transparency** - Visual indication of manual data interventions
2. **Audit Trail Access** - Quick access to correction history and metadata
3. **Data Quality Signaling** - Distinguish between original and corrected data
4. **User Attribution** - Show who made corrections and when
5. **Confidence Indicators** - Signal human verification of critical fields
6. **Compliance Documentation** - Support regulatory audit requirements

### Component Classification

- **Type:** Indicator/Badge Component
- **Layer:** Information Layer (IL)
- **Complexity:** Low
- **Interactivity:** Medium (tooltip, optional click)
- **Reusability:** Very High (used across all domains)

---

## Visual Design

### Variants

#### 1. Badge Variant
```
┌─────────────────────┐
│ Field Label         │
│ ┌─────────┐         │
│ │ Edited  │ Value   │
│ └─────────┘         │
└─────────────────────┘
```

#### 2. Icon Variant
```
┌─────────────────────┐
│ Field Label     ✏️  │
│ Value               │
└─────────────────────┘
```

#### 3. Dot Variant
```
┌─────────────────────┐
│ Field Label      •  │
│ Value               │
└─────────────────────┘
```

### Size Variations

```
Small (sm):
[Edited] or ✏️ or •
12px height, 8px padding

Medium (md):
[Edited] or ✏️ or •
16px height, 10px padding

Large (lg):
[Edited] or ✏️ or •
20px height, 12px padding
```

### Tooltip Design

```
┌─────────────────────────────────────┐
│ Manually Corrected                  │
│                                     │
│ By: Sarah Chen                      │
│ Date: Oct 24, 2025 at 2:45 PM      │
│                                     │
│ Original Value: "Starbucks Corp"    │
│ Current Value: "Starbucks"          │
│                                     │
│ Click to view full audit trail →   │
└─────────────────────────────────────┘
```

### Color Scheme

```typescript
const colorScheme = {
  default: {
    background: 'hsl(var(--orange-100))',
    foreground: 'hsl(var(--orange-900))',
    border: 'hsl(var(--orange-300))'
  },
  recent: {
    background: 'hsl(var(--blue-100))',
    foreground: 'hsl(var(--blue-900))',
    border: 'hsl(var(--blue-300))'
  },
  dark: {
    background: 'hsl(var(--orange-900))',
    foreground: 'hsl(var(--orange-100))',
    border: 'hsl(var(--orange-700))'
  }
}
```

### Animation States

```typescript
const animations = {
  entrance: {
    initial: { opacity: 0, scale: 0.8 },
    animate: { opacity: 1, scale: 1 },
    transition: { duration: 0.2, ease: 'easeOut' }
  },
  hover: {
    scale: 1.05,
    transition: { duration: 0.15 }
  },
  tooltipEnter: {
    initial: { opacity: 0, y: -8 },
    animate: { opacity: 1, y: 0 },
    transition: { duration: 0.15 }
  }
}
```

---

## Component Interface

### TypeScript Definition

```typescript
import { ReactNode } from 'react'
import { LucideIcon } from 'lucide-react'

/**
 * FieldOverrideIndicator - Visual indicator for manually corrected fields
 *
 * Displays a badge, icon, or dot to show field has been overridden,
 * with optional tooltip showing correction metadata.
 */
export interface FieldOverrideIndicatorProps {
  // --- Data Props ---

  /**
   * Whether the field has been overridden
   * Controls visibility of the entire component
   */
  isOverridden: boolean

  /**
   * User who made the correction
   * @example "Sarah Chen" | "sarah.chen@company.com"
   */
  overriddenBy?: string

  /**
   * ISO timestamp of when correction was made
   * @example "2025-10-24T14:45:00Z"
   */
  overriddenAt?: string

  /**
   * Original value before correction
   * Can be any type (string, number, object)
   */
  originalValue?: any

  /**
   * Current/corrected value
   * Used for comparison in tooltip
   */
  currentValue?: any

  /**
   * Reason for correction (optional)
   * @example "OCR misread" | "User reported error"
   */
  correctionReason?: string

  // --- Display Props ---

  /**
   * Visual variant of the indicator
   * @default 'badge'
   */
  variant?: 'badge' | 'icon' | 'dot'

  /**
   * Size of the indicator
   * @default 'md'
   */
  size?: 'sm' | 'md' | 'lg'

  /**
   * Custom color (overrides default orange)
   * @example 'blue' | 'purple' | '#ff6600'
   */
  color?: string

  /**
   * Position relative to field
   * @default 'inline-end'
   */
  position?: 'inline-start' | 'inline-end' | 'above' | 'below' | 'absolute'

  /**
   * Label text for badge variant
   * @default 'Edited'
   */
  label?: string

  /**
   * Custom icon for icon variant
   * @default Edit2 from lucide-react
   */
  icon?: LucideIcon

  /**
   * Show "recent" styling for corrections in last 24h
   * @default true
   */
  highlightRecent?: boolean

  // --- Tooltip Props ---

  /**
   * Show tooltip on hover
   * @default true
   */
  showTooltip?: boolean

  /**
   * Custom tooltip content (replaces default)
   */
  tooltipContent?: ReactNode

  /**
   * Tooltip placement
   * @default 'top'
   */
  tooltipPlacement?: 'top' | 'bottom' | 'left' | 'right'

  /**
   * Delay before showing tooltip (ms)
   * @default 200
   */
  tooltipDelay?: number

  /**
   * Show before/after comparison in tooltip
   * @default true if originalValue provided
   */
  showComparison?: boolean

  // --- Interaction Props ---

  /**
   * Click handler - typically opens audit trail
   */
  onClick?: () => void

  /**
   * Whether indicator is clickable
   * @default true if onClick provided
   */
  interactive?: boolean

  /**
   * Aria label for accessibility
   * @default "Field manually corrected"
   */
  ariaLabel?: string

  // --- Styling Props ---

  /**
   * Additional CSS classes
   */
  className?: string

  /**
   * Inline styles
   */
  style?: React.CSSProperties

  /**
   * Test ID for automated testing
   */
  testId?: string
}

/**
 * Metadata about a field correction
 */
export interface CorrectionMetadata {
  fieldName: string
  originalValue: any
  correctedValue: any
  correctedBy: string
  correctedAt: string
  reason?: string
  source?: 'manual' | 'ai-assisted' | 'batch-import'
  confidence?: number
  validatedBy?: string[]
}

/**
 * Props for the internal tooltip content component
 */
export interface TooltipContentProps {
  metadata: CorrectionMetadata
  showComparison: boolean
  hasClickAction: boolean
}
```

### Default Props

```typescript
const defaultProps: Partial<FieldOverrideIndicatorProps> = {
  variant: 'badge',
  size: 'md',
  position: 'inline-end',
  label: 'Edited',
  showTooltip: true,
  tooltipPlacement: 'top',
  tooltipDelay: 200,
  highlightRecent: true,
  ariaLabel: 'Field manually corrected',
  testId: 'field-override-indicator'
}
```

---

## Implementation

### Core Component

```typescript
'use client'

import React, { useMemo } from 'react'
import { motion, AnimatePresence } from 'framer-motion'
import { Edit2, PenLine, Check } from 'lucide-react'
import { Badge } from '@/components/ui/badge'
import { Tooltip, TooltipContent, TooltipProvider, TooltipTrigger } from '@/components/ui/tooltip'
import { cn } from '@/lib/utils'
import { formatDistanceToNow, parseISO, format } from 'date-fns'
import type { FieldOverrideIndicatorProps, CorrectionMetadata } from './types'

export function FieldOverrideIndicator({
  isOverridden,
  overriddenBy,
  overriddenAt,
  originalValue,
  currentValue,
  correctionReason,
  variant = 'badge',
  size = 'md',
  color,
  position = 'inline-end',
  label = 'Edited',
  icon: CustomIcon,
  highlightRecent = true,
  showTooltip = true,
  tooltipContent,
  tooltipPlacement = 'top',
  tooltipDelay = 200,
  showComparison = true,
  onClick,
  interactive = !!onClick,
  ariaLabel = 'Field manually corrected',
  className,
  style,
  testId = 'field-override-indicator',
}: FieldOverrideIndicatorProps) {

  // Don't render if not overridden
  if (!isOverridden) return null

  // Determine if correction is recent (last 24 hours)
  const isRecent = useMemo(() => {
    if (!highlightRecent || !overriddenAt) return false
    const correctionDate = parseISO(overriddenAt)
    const hoursSince = (Date.now() - correctionDate.getTime()) / (1000 * 60 * 60)
    return hoursSince < 24
  }, [overriddenAt, highlightRecent])

  // Build correction metadata
  const metadata: CorrectionMetadata = useMemo(() => ({
    fieldName: '', // Provided by parent context
    originalValue,
    correctedValue: currentValue,
    correctedBy: overriddenBy || 'Unknown',
    correctedAt: overriddenAt || new Date().toISOString(),
    reason: correctionReason,
    source: 'manual'
  }), [originalValue, currentValue, overriddenBy, overriddenAt, correctionReason])

  // Size mappings
  const sizeClasses = {
    sm: 'h-4 text-xs px-1.5 gap-0.5',
    md: 'h-5 text-xs px-2 gap-1',
    lg: 'h-6 text-sm px-2.5 gap-1'
  }

  const iconSizes = {
    sm: 10,
    md: 12,
    lg: 14
  }

  const dotSizes = {
    sm: 'w-1.5 h-1.5',
    md: 'w-2 h-2',
    lg: 'w-2.5 h-2.5'
  }

  // Color scheme
  const colorScheme = useMemo(() => {
    if (color) return { bg: color, fg: 'white', border: color }

    const theme = isRecent ? 'blue' : 'orange'

    return {
      bg: `var(--${theme}-100)`,
      fg: `var(--${theme}-900)`,
      border: `var(--${theme}-300)`,
      bgDark: `var(--${theme}-900)`,
      fgDark: `var(--${theme}-100)`,
      borderDark: `var(--${theme}-700)`
    }
  }, [color, isRecent])

  // Icon selection
  const Icon = CustomIcon || Edit2

  // Render indicator based on variant
  const renderIndicator = () => {
    switch (variant) {
      case 'badge':
        return (
          <Badge
            variant="outline"
            className={cn(
              'inline-flex items-center font-medium cursor-default',
              sizeClasses[size],
              interactive && 'cursor-pointer hover:scale-105 transition-transform',
              className
            )}
            style={{
              backgroundColor: colorScheme.bg,
              color: colorScheme.fg,
              borderColor: colorScheme.border,
              ...style
            }}
            onClick={onClick}
          >
            <Icon size={iconSizes[size]} className="shrink-0" />
            <span>{label}</span>
          </Badge>
        )

      case 'icon':
        return (
          <div
            className={cn(
              'inline-flex items-center justify-center rounded-full',
              interactive && 'cursor-pointer hover:scale-110 transition-transform',
              className
            )}
            style={{
              color: colorScheme.fg,
              ...style
            }}
            onClick={onClick}
            role={interactive ? 'button' : undefined}
            tabIndex={interactive ? 0 : undefined}
          >
            <Icon size={iconSizes[size]} />
          </div>
        )

      case 'dot':
        return (
          <div
            className={cn(
              'rounded-full shrink-0',
              dotSizes[size],
              interactive && 'cursor-pointer hover:scale-125 transition-transform',
              className
            )}
            style={{
              backgroundColor: colorScheme.fg,
              ...style
            }}
            onClick={onClick}
            role={interactive ? 'button' : undefined}
            tabIndex={interactive ? 0 : undefined}
          />
        )
    }
  }

  const indicator = (
    <motion.div
      initial={{ opacity: 0, scale: 0.8 }}
      animate={{ opacity: 1, scale: 1 }}
      transition={{ duration: 0.2, ease: 'easeOut' }}
      data-testid={testId}
      aria-label={ariaLabel}
      role="status"
    >
      {renderIndicator()}
    </motion.div>
  )

  // Return without tooltip if disabled
  if (!showTooltip) {
    return indicator
  }

  // Return with tooltip
  return (
    <TooltipProvider delayDuration={tooltipDelay}>
      <Tooltip>
        <TooltipTrigger asChild>
          {indicator}
        </TooltipTrigger>
        <TooltipContent side={tooltipPlacement} className="max-w-xs">
          {tooltipContent || (
            <CorrectionTooltip
              metadata={metadata}
              showComparison={showComparison && !!originalValue}
              hasClickAction={interactive}
            />
          )}
        </TooltipContent>
      </Tooltip>
    </TooltipProvider>
  )
}

/**
 * Default tooltip content showing correction metadata
 */
function CorrectionTooltip({
  metadata,
  showComparison,
  hasClickAction
}: {
  metadata: CorrectionMetadata
  showComparison: boolean
  hasClickAction: boolean
}) {
  const correctionDate = parseISO(metadata.correctedAt)

  return (
    <div className="space-y-2 text-sm">
      <div className="font-semibold text-foreground">
        Manually Corrected
      </div>

      <div className="space-y-1 text-muted-foreground">
        <div className="flex justify-between gap-4">
          <span>By:</span>
          <span className="font-medium text-foreground">
            {metadata.correctedBy}
          </span>
        </div>

        <div className="flex justify-between gap-4">
          <span>Date:</span>
          <span className="font-medium text-foreground">
            {format(correctionDate, 'MMM d, yyyy')}
          </span>
        </div>

        <div className="flex justify-between gap-4">
          <span>Time:</span>
          <span className="font-medium text-foreground">
            {format(correctionDate, 'h:mm a')}
          </span>
        </div>
      </div>

      {metadata.reason && (
        <div className="pt-2 border-t border-border">
          <div className="text-xs text-muted-foreground">Reason:</div>
          <div className="text-foreground">{metadata.reason}</div>
        </div>
      )}

      {showComparison && (
        <div className="pt-2 border-t border-border space-y-1">
          <div className="grid grid-cols-[80px_1fr] gap-2 text-xs">
            <span className="text-muted-foreground">Original:</span>
            <span className="font-mono text-destructive line-through">
              {formatValue(metadata.originalValue)}
            </span>
          </div>
          <div className="grid grid-cols-[80px_1fr] gap-2 text-xs">
            <span className="text-muted-foreground">Corrected:</span>
            <span className="font-mono text-green-600 dark:text-green-400">
              {formatValue(metadata.correctedValue)}
            </span>
          </div>
        </div>
      )}

      {hasClickAction && (
        <div className="pt-2 border-t border-border text-xs text-primary">
          Click to view full audit trail →
        </div>
      )}
    </div>
  )
}

/**
 * Format value for display in tooltip
 */
function formatValue(value: any): string {
  if (value === null || value === undefined) return '(empty)'
  if (typeof value === 'object') return JSON.stringify(value)
  return String(value)
}

// Display name for debugging
FieldOverrideIndicator.displayName = 'FieldOverrideIndicator'
```

### Helper Hooks

```typescript
/**
 * Hook to track field override state
 */
export function useFieldOverride(
  fieldId: string,
  currentValue: any
) {
  const [overrideState, setOverrideState] = useState<{
    isOverridden: boolean
    metadata?: CorrectionMetadata
  }>({
    isOverridden: false
  })

  useEffect(() => {
    // Check if field has override metadata in storage/API
    const checkOverride = async () => {
      const metadata = await fetchOverrideMetadata(fieldId)
      if (metadata) {
        setOverrideState({
          isOverridden: true,
          metadata
        })
      }
    }

    checkOverride()
  }, [fieldId])

  const markAsOverridden = useCallback(
    (originalValue: any, reason?: string) => {
      const metadata: CorrectionMetadata = {
        fieldName: fieldId,
        originalValue,
        correctedValue: currentValue,
        correctedBy: getCurrentUser(),
        correctedAt: new Date().toISOString(),
        reason,
        source: 'manual'
      }

      setOverrideState({
        isOverridden: true,
        metadata
      })

      // Persist to storage/API
      saveOverrideMetadata(fieldId, metadata)
    },
    [fieldId, currentValue]
  )

  return {
    ...overrideState,
    markAsOverridden
  }
}

/**
 * Hook for batch override operations
 */
export function useBatchFieldOverrides(fieldIds: string[]) {
  const [overrides, setOverrides] = useState<Map<string, CorrectionMetadata>>(
    new Map()
  )

  useEffect(() => {
    const loadOverrides = async () => {
      const metadata = await fetchBatchOverrideMetadata(fieldIds)
      setOverrides(new Map(metadata))
    }

    loadOverrides()
  }, [fieldIds])

  return {
    overrides,
    getOverride: (fieldId: string) => overrides.get(fieldId),
    hasOverride: (fieldId: string) => overrides.has(fieldId)
  }
}
```

### Utility Functions

```typescript
/**
 * Calculate if correction is recent (within threshold)
 */
export function isRecentCorrection(
  correctedAt: string,
  thresholdHours: number = 24
): boolean {
  const correctionDate = parseISO(correctedAt)
  const hoursSince = (Date.now() - correctionDate.getTime()) / (1000 * 60 * 60)
  return hoursSince < thresholdHours
}

/**
 * Format correction metadata for display
 */
export function formatCorrectionSummary(metadata: CorrectionMetadata): string {
  const date = parseISO(metadata.correctedAt)
  const timeAgo = formatDistanceToNow(date, { addSuffix: true })

  return `Corrected by ${metadata.correctedBy} ${timeAgo}`
}

/**
 * Determine appropriate variant based on context
 */
export function getRecommendedVariant(context: {
  spaceAvailable: 'tight' | 'normal' | 'spacious'
  importanceLevel: 'low' | 'medium' | 'high'
}): 'badge' | 'icon' | 'dot' {
  if (context.spaceAvailable === 'tight') return 'dot'
  if (context.importanceLevel === 'high') return 'badge'
  return 'icon'
}

/**
 * Get color based on correction age
 */
export function getCorrectionColor(
  correctedAt: string,
  scheme: 'age-based' | 'status-based' = 'age-based'
): string {
  if (scheme === 'age-based') {
    const hours = (Date.now() - parseISO(correctedAt).getTime()) / (1000 * 60 * 60)
    if (hours < 1) return 'blue'
    if (hours < 24) return 'orange'
    return 'gray'
  }

  return 'orange'
}
```

---

## Multi-Domain Usage Examples

### Example 1: Finance - Transaction Merchant Correction

```typescript
'use client'

import { FieldOverrideIndicator } from '@/components/il/field-override-indicator'
import { useState } from 'react'

interface Transaction {
  id: string
  merchant: string
  merchantOverride?: {
    original: string
    by: string
    at: string
    reason: string
  }
}

function TransactionMerchantField({ transaction }: { transaction: Transaction }) {
  const [showAuditTrail, setShowAuditTrail] = useState(false)

  return (
    <div className="flex items-center gap-2">
      <span className="font-medium">{transaction.merchant}</span>

      <FieldOverrideIndicator
        isOverridden={!!transaction.merchantOverride}
        overriddenBy={transaction.merchantOverride?.by}
        overriddenAt={transaction.merchantOverride?.at}
        originalValue={transaction.merchantOverride?.original}
        currentValue={transaction.merchant}
        correctionReason={transaction.merchantOverride?.reason}
        variant="badge"
        size="sm"
        onClick={() => setShowAuditTrail(true)}
      />

      {showAuditTrail && (
        <AuditTrailDialog
          transactionId={transaction.id}
          onClose={() => setShowAuditTrail(false)}
        />
      )}
    </div>
  )
}

// Example usage
export function TransactionsList() {
  const transactions: Transaction[] = [
    {
      id: 'txn_001',
      merchant: 'Starbucks',
      merchantOverride: {
        original: 'STARBUCKS #2847',
        by: 'Sarah Chen',
        at: '2025-10-24T14:30:00Z',
        reason: 'Normalized merchant name for categorization'
      }
    },
    {
      id: 'txn_002',
      merchant: 'Amazon',
      // No override
    }
  ]

  return (
    <div className="space-y-2">
      {transactions.map(txn => (
        <TransactionMerchantField key={txn.id} transaction={txn} />
      ))}
    </div>
  )
}
```

**Output:**
```
Starbucks [Edited ✏️]
Amazon
```

---

### Example 2: Healthcare - Diagnosis Code Correction

```typescript
'use client'

import { FieldOverrideIndicator } from '@/components/il/field-override-indicator'

interface DiagnosisCode {
  code: string
  description: string
  override?: {
    originalCode: string
    originalDescription: string
    correctedBy: string
    correctedAt: string
    validatedBy: string[]
  }
}

function DiagnosisCodeField({ diagnosis }: { diagnosis: DiagnosisCode }) {
  return (
    <div className="flex items-start gap-3 p-3 border rounded-lg">
      <div className="flex-1">
        <div className="flex items-center gap-2">
          <code className="font-mono font-semibold">{diagnosis.code}</code>

          <FieldOverrideIndicator
            isOverridden={!!diagnosis.override}
            overriddenBy={diagnosis.override?.correctedBy}
            overriddenAt={diagnosis.override?.correctedAt}
            originalValue={diagnosis.override?.originalCode}
            currentValue={diagnosis.code}
            variant="icon"
            size="sm"
            color="purple"
            tooltipContent={
              diagnosis.override && (
                <div className="space-y-2 text-sm">
                  <div className="font-semibold">Diagnosis Code Corrected</div>
                  <div className="space-y-1">
                    <div>Original: <code className="text-destructive">{diagnosis.override.originalCode}</code></div>
                    <div>Corrected: <code className="text-green-600">{diagnosis.code}</code></div>
                    <div>By: {diagnosis.override.correctedBy}</div>
                    {diagnosis.override.validatedBy.length > 0 && (
                      <div className="pt-2 border-t">
                        <div className="text-xs text-muted-foreground">Validated by:</div>
                        {diagnosis.override.validatedBy.map(validator => (
                          <div key={validator} className="text-xs">✓ {validator}</div>
                        ))}
                      </div>
                    )}
                  </div>
                </div>
              )
            }
          />
        </div>
        <p className="text-sm text-muted-foreground mt-1">
          {diagnosis.description}
        </p>
      </div>
    </div>
  )
}

// Example usage
export function PatientDiagnoses() {
  const diagnoses: DiagnosisCode[] = [
    {
      code: 'E11.9',
      description: 'Type 2 diabetes mellitus without complications',
      override: {
        originalCode: 'E11.65',
        originalDescription: 'Type 2 diabetes mellitus with hyperglycemia',
        correctedBy: 'Dr. Michael Torres',
        correctedAt: '2025-10-23T09:15:00Z',
        validatedBy: ['Dr. Lisa Park', 'Nurse Sarah Johnson']
      }
    }
  ]

  return (
    <div className="space-y-3">
      {diagnoses.map((d, i) => (
        <DiagnosisCodeField key={i} diagnosis={d} />
      ))}
    </div>
  )
}
```

---

### Example 3: Legal - Case Number Correction

```typescript
'use client'

import { FieldOverrideIndicator } from '@/components/il/field-override-indicator'
import { AlertTriangle } from 'lucide-react'

interface LegalCase {
  id: string
  caseNumber: string
  title: string
  caseNumberOverride?: {
    original: string
    correctedBy: string
    correctedAt: string
    reason: string
    severity: 'low' | 'medium' | 'high'
  }
}

function CaseNumberDisplay({ legalCase }: { legalCase: LegalCase }) {
  const isHighSeverity = legalCase.caseNumberOverride?.severity === 'high'

  return (
    <div className="space-y-1">
      <div className="flex items-center gap-2">
        <h3 className="font-semibold text-lg">{legalCase.caseNumber}</h3>

        {isHighSeverity && (
          <AlertTriangle className="w-4 h-4 text-amber-500" />
        )}

        <FieldOverrideIndicator
          isOverridden={!!legalCase.caseNumberOverride}
          overriddenBy={legalCase.caseNumberOverride?.correctedBy}
          overriddenAt={legalCase.caseNumberOverride?.correctedAt}
          originalValue={legalCase.caseNumberOverride?.original}
          currentValue={legalCase.caseNumber}
          correctionReason={legalCase.caseNumberOverride?.reason}
          variant="badge"
          size="md"
          label={isHighSeverity ? 'Critical Edit' : 'Edited'}
          color={isHighSeverity ? '#ef4444' : undefined}
        />
      </div>

      <p className="text-sm text-muted-foreground">{legalCase.title}</p>
    </div>
  )
}

// Example usage
export function CasesList() {
  const cases: LegalCase[] = [
    {
      id: 'case_001',
      caseNumber: '2025-CV-12345',
      title: 'Smith v. Johnson Industries',
      caseNumberOverride: {
        original: '2024-CV-12345',
        correctedBy: 'Jennifer Martinez, Esq.',
        correctedAt: '2025-10-24T11:00:00Z',
        reason: 'Incorrect filing year discovered during review',
        severity: 'high'
      }
    }
  ]

  return (
    <div className="space-y-4">
      {cases.map(c => (
        <CaseNumberDisplay key={c.id} legalCase={c} />
      ))}
    </div>
  )
}
```

---

### Example 4: Research - Author Name Correction

```typescript
'use client'

import { FieldOverrideIndicator } from '@/components/il/field-override-indicator'

interface ResearchPaper {
  id: string
  title: string
  authors: Author[]
}

interface Author {
  name: string
  affiliation: string
  override?: {
    originalName: string
    correctedBy: string
    correctedAt: string
    source: 'orcid' | 'manual' | 'publisher'
  }
}

function AuthorList({ authors }: { authors: Author[] }) {
  return (
    <div className="space-y-1">
      {authors.map((author, idx) => (
        <div key={idx} className="flex items-center gap-2">
          <span className="font-medium">{author.name}</span>

          <FieldOverrideIndicator
            isOverridden={!!author.override}
            overriddenBy={author.override?.correctedBy}
            overriddenAt={author.override?.correctedAt}
            originalValue={author.override?.originalName}
            currentValue={author.name}
            variant="dot"
            size="sm"
            tooltipContent={
              author.override && (
                <div className="space-y-2 text-sm">
                  <div className="font-semibold">Author Name Corrected</div>
                  <div>Original: {author.override.originalName}</div>
                  <div>Corrected: {author.name}</div>
                  <div className="pt-2 border-t text-xs text-muted-foreground">
                    Source: {author.override.source.toUpperCase()}
                  </div>
                </div>
              )
            }
          />

          <span className="text-sm text-muted-foreground">
            {author.affiliation}
          </span>
        </div>
      ))}
    </div>
  )
}

// Example usage
export function PaperMetadata() {
  const paper: ResearchPaper = {
    id: 'paper_001',
    title: 'Machine Learning Applications in Healthcare',
    authors: [
      {
        name: 'Dr. Sarah J. Chen',
        affiliation: 'Stanford University',
        override: {
          originalName: 'S. Chen',
          correctedBy: 'Research Librarian',
          correctedAt: '2025-10-20T16:00:00Z',
          source: 'orcid'
        }
      },
      {
        name: 'Dr. Michael Torres',
        affiliation: 'MIT'
      }
    ]
  }

  return (
    <div className="space-y-3">
      <h2 className="text-xl font-bold">{paper.title}</h2>
      <AuthorList authors={paper.authors} />
    </div>
  )
}
```

---

### Example 5: E-commerce - Product Category Correction

```typescript
'use client'

import { FieldOverrideIndicator } from '@/components/il/field-override-indicator'
import { Tag } from 'lucide-react'

interface Product {
  id: string
  name: string
  category: string
  categoryOverride?: {
    originalCategory: string
    correctedBy: string
    correctedAt: string
    reason: string
    confidence: number
  }
}

function ProductCategoryBadge({ product }: { product: Product }) {
  return (
    <div className="flex items-center gap-2 p-2 bg-muted/50 rounded-lg">
      <Tag className="w-4 h-4 text-muted-foreground" />

      <span className="text-sm font-medium">{product.category}</span>

      <FieldOverrideIndicator
        isOverridden={!!product.categoryOverride}
        overriddenBy={product.categoryOverride?.correctedBy}
        overriddenAt={product.categoryOverride?.correctedAt}
        originalValue={product.categoryOverride?.originalCategory}
        currentValue={product.category}
        correctionReason={product.categoryOverride?.reason}
        variant="badge"
        size="sm"
        tooltipContent={
          product.categoryOverride && (
            <div className="space-y-2 text-sm">
              <div className="font-semibold">Category Reclassified</div>
              <div className="space-y-1">
                <div>From: <span className="line-through text-destructive">{product.categoryOverride.originalCategory}</span></div>
                <div>To: <span className="text-green-600">{product.category}</span></div>
                <div className="pt-2 border-t">
                  <div className="text-xs text-muted-foreground">Reason:</div>
                  <div>{product.categoryOverride.reason}</div>
                </div>
                {product.categoryOverride.confidence && (
                  <div className="pt-2 border-t">
                    <div className="text-xs text-muted-foreground">Confidence:</div>
                    <div className="flex items-center gap-2">
                      <div className="flex-1 bg-muted rounded-full h-2">
                        <div
                          className="bg-green-600 h-2 rounded-full"
                          style={{ width: `${product.categoryOverride.confidence * 100}%` }}
                        />
                      </div>
                      <span className="text-xs font-medium">
                        {Math.round(product.categoryOverride.confidence * 100)}%
                      </span>
                    </div>
                  </div>
                )}
              </div>
            </div>
          )
        }
      />
    </div>
  )
}

// Example usage
export function ProductCard() {
  const product: Product = {
    id: 'prod_001',
    name: 'Wireless Bluetooth Headphones',
    category: 'Audio > Headphones > Over-Ear',
    categoryOverride: {
      originalCategory: 'Electronics > Accessories',
      correctedBy: 'Catalog Manager',
      correctedAt: '2025-10-22T13:45:00Z',
      reason: 'Moved to specific audio category for better discoverability',
      confidence: 0.95
    }
  }

  return (
    <div className="border rounded-lg p-4 space-y-3">
      <h3 className="font-semibold">{product.name}</h3>
      <ProductCategoryBadge product={product} />
    </div>
  )
}
```

---

### Example 6: SaaS - MRR Value Correction

```typescript
'use client'

import { FieldOverrideIndicator } from '@/components/il/field-override-indicator'
import { DollarSign } from 'lucide-react'

interface Customer {
  id: string
  name: string
  mrr: number
  mrrOverride?: {
    originalMrr: number
    correctedBy: string
    correctedAt: string
    reason: string
    impactedMetrics: string[]
  }
}

function CustomerMRRField({ customer }: { customer: Customer }) {
  const hasMajorChange = customer.mrrOverride
    ? Math.abs(customer.mrr - customer.mrrOverride.originalMrr) > 1000
    : false

  return (
    <div className="flex items-center justify-between p-3 border rounded-lg">
      <div className="flex items-center gap-2">
        <DollarSign className="w-5 h-5 text-muted-foreground" />
        <div>
          <div className="text-sm text-muted-foreground">Monthly Recurring Revenue</div>
          <div className="flex items-center gap-2">
            <span className="text-2xl font-bold">
              ${customer.mrr.toLocaleString()}
            </span>

            <FieldOverrideIndicator
              isOverridden={!!customer.mrrOverride}
              overriddenBy={customer.mrrOverride?.correctedBy}
              overriddenAt={customer.mrrOverride?.correctedAt}
              originalValue={customer.mrrOverride?.originalMrr}
              currentValue={customer.mrr}
              correctionReason={customer.mrrOverride?.reason}
              variant="badge"
              size="md"
              label={hasMajorChange ? 'Major Edit' : 'Edited'}
              color={hasMajorChange ? '#f59e0b' : undefined}
              tooltipContent={
                customer.mrrOverride && (
                  <div className="space-y-2 text-sm">
                    <div className="font-semibold">MRR Manually Corrected</div>
                    <div className="space-y-1">
                      <div className="grid grid-cols-[80px_1fr] gap-2">
                        <span className="text-muted-foreground">Original:</span>
                        <span className="font-mono text-destructive line-through">
                          ${customer.mrrOverride.originalMrr.toLocaleString()}
                        </span>
                      </div>
                      <div className="grid grid-cols-[80px_1fr] gap-2">
                        <span className="text-muted-foreground">Corrected:</span>
                        <span className="font-mono text-green-600">
                          ${customer.mrr.toLocaleString()}
                        </span>
                      </div>
                      <div className="grid grid-cols-[80px_1fr] gap-2">
                        <span className="text-muted-foreground">Difference:</span>
                        <span className={`font-mono font-semibold ${
                          customer.mrr > customer.mrrOverride.originalMrr
                            ? 'text-green-600'
                            : 'text-destructive'
                        }`}>
                          {customer.mrr > customer.mrrOverride.originalMrr ? '+' : ''}
                          ${(customer.mrr - customer.mrrOverride.originalMrr).toLocaleString()}
                        </span>
                      </div>
                    </div>
                    <div className="pt-2 border-t">
                      <div className="text-xs text-muted-foreground">Reason:</div>
                      <div>{customer.mrrOverride.reason}</div>
                    </div>
                    {customer.mrrOverride.impactedMetrics.length > 0 && (
                      <div className="pt-2 border-t">
                        <div className="text-xs text-muted-foreground mb-1">Impacted Metrics:</div>
                        {customer.mrrOverride.impactedMetrics.map(metric => (
                          <div key={metric} className="text-xs">• {metric}</div>
                        ))}
                      </div>
                    )}
                  </div>
                )
              }
            />
          </div>
        </div>
      </div>
    </div>
  )
}

// Example usage
export function CustomerMetrics() {
  const customer: Customer = {
    id: 'cust_001',
    name: 'Acme Corporation',
    mrr: 15000,
    mrrOverride: {
      originalMrr: 12000,
      correctedBy: 'Finance Team',
      correctedAt: '2025-10-24T10:30:00Z',
      reason: 'Added missing enterprise add-ons that were invoiced separately',
      impactedMetrics: ['Total MRR', 'Expansion Revenue', 'Annual Contract Value']
    }
  }

  return (
    <div className="space-y-4">
      <h2 className="text-xl font-bold">{customer.name}</h2>
      <CustomerMRRField customer={customer} />
    </div>
  )
}
```

---

### Example 7: Multi-Field Form with Override Tracking

```typescript
'use client'

import { FieldOverrideIndicator } from '@/components/il/field-override-indicator'
import { useFieldOverride } from '@/components/il/field-override-indicator/hooks'
import { useState } from 'react'
import { Input } from '@/components/ui/input'
import { Label } from '@/components/ui/label'
import { Button } from '@/components/ui/button'

interface FormData {
  companyName: string
  industry: string
  employeeCount: number
  annualRevenue: number
}

function EditableFieldWithOverride({
  label,
  value,
  fieldId,
  type = 'text',
  onChange
}: {
  label: string
  value: string | number
  fieldId: string
  type?: 'text' | 'number'
  onChange: (value: any) => void
}) {
  const { isOverridden, metadata, markAsOverridden } = useFieldOverride(fieldId, value)
  const [isEditing, setIsEditing] = useState(false)
  const [editValue, setEditValue] = useState(value)

  const handleSave = () => {
    if (editValue !== value) {
      markAsOverridden(value, 'Manual correction')
      onChange(editValue)
    }
    setIsEditing(false)
  }

  return (
    <div className="space-y-2">
      <div className="flex items-center gap-2">
        <Label>{label}</Label>

        {isOverridden && metadata && (
          <FieldOverrideIndicator
            isOverridden={true}
            overriddenBy={metadata.correctedBy}
            overriddenAt={metadata.correctedAt}
            originalValue={metadata.originalValue}
            currentValue={value}
            variant="icon"
            size="sm"
          />
        )}
      </div>

      {isEditing ? (
        <div className="flex gap-2">
          <Input
            type={type}
            value={editValue}
            onChange={(e) => setEditValue(
              type === 'number' ? Number(e.target.value) : e.target.value
            )}
            className="flex-1"
          />
          <Button size="sm" onClick={handleSave}>Save</Button>
          <Button size="sm" variant="outline" onClick={() => setIsEditing(false)}>
            Cancel
          </Button>
        </div>
      ) : (
        <div
          className="flex items-center gap-2 cursor-pointer group"
          onClick={() => setIsEditing(true)}
        >
          <span className="font-medium">{value}</span>
          <span className="text-xs text-muted-foreground opacity-0 group-hover:opacity-100">
            Click to edit
          </span>
        </div>
      )}
    </div>
  )
}

export function CompanyProfileForm() {
  const [formData, setFormData] = useState<FormData>({
    companyName: 'Acme Corporation',
    industry: 'Technology',
    employeeCount: 500,
    annualRevenue: 50000000
  })

  return (
    <div className="space-y-4 p-6 border rounded-lg">
      <h2 className="text-xl font-bold">Company Profile</h2>

      <EditableFieldWithOverride
        label="Company Name"
        value={formData.companyName}
        fieldId="companyName"
        onChange={(v) => setFormData({ ...formData, companyName: v })}
      />

      <EditableFieldWithOverride
        label="Industry"
        value={formData.industry}
        fieldId="industry"
        onChange={(v) => setFormData({ ...formData, industry: v })}
      />

      <EditableFieldWithOverride
        label="Employee Count"
        value={formData.employeeCount}
        fieldId="employeeCount"
        type="number"
        onChange={(v) => setFormData({ ...formData, employeeCount: v })}
      />

      <EditableFieldWithOverride
        label="Annual Revenue"
        value={formData.annualRevenue}
        fieldId="annualRevenue"
        type="number"
        onChange={(v) => setFormData({ ...formData, annualRevenue: v })}
      />
    </div>
  )
}
```

---

## Edge Cases & Error Handling

### Edge Case 1: No Metadata Available

```typescript
function SafeFieldOverrideIndicator(props: FieldOverrideIndicatorProps) {
  // Gracefully handle missing metadata
  if (props.isOverridden && !props.overriddenBy && !props.overriddenAt) {
    return (
      <FieldOverrideIndicator
        {...props}
        overriddenBy="Unknown User"
        overriddenAt={new Date().toISOString()}
        tooltipContent={
          <div className="text-sm text-muted-foreground">
            This field was manually corrected, but metadata is unavailable.
          </div>
        }
      />
    )
  }

  return <FieldOverrideIndicator {...props} />
}
```

### Edge Case 2: Very Long User Names

```typescript
function truncateUserName(name: string, maxLength: number = 30): string {
  if (name.length <= maxLength) return name

  // Try to truncate at word boundary
  const truncated = name.substring(0, maxLength)
  const lastSpace = truncated.lastIndexOf(' ')

  if (lastSpace > maxLength * 0.7) {
    return truncated.substring(0, lastSpace) + '...'
  }

  return truncated + '...'
}

// Usage in component
function CorrectionTooltipWithTruncation({ metadata }: { metadata: CorrectionMetadata }) {
  const displayName = truncateUserName(metadata.correctedBy)
  const fullName = metadata.correctedBy

  return (
    <div className="space-y-2">
      <div className="flex justify-between gap-4">
        <span>By:</span>
        <span title={fullName}>{displayName}</span>
      </div>
      {/* ... rest of tooltip */}
    </div>
  )
}
```

### Edge Case 3: Multiple Rapid Corrections

```typescript
interface CorrectionHistory {
  corrections: CorrectionMetadata[]
  currentValue: any
}

function FieldWithCorrectionHistory({ history }: { history: CorrectionHistory }) {
  const latestCorrection = history.corrections[0]
  const correctionCount = history.corrections.length

  return (
    <FieldOverrideIndicator
      isOverridden={true}
      overriddenBy={latestCorrection.correctedBy}
      overriddenAt={latestCorrection.correctedAt}
      originalValue={history.corrections[history.corrections.length - 1].originalValue}
      currentValue={history.currentValue}
      variant="badge"
      label={correctionCount > 1 ? `Edited ${correctionCount}×` : 'Edited'}
      tooltipContent={
        <div className="space-y-2 text-sm">
          <div className="font-semibold">
            Corrected {correctionCount} {correctionCount === 1 ? 'time' : 'times'}
          </div>
          <div className="max-h-48 overflow-y-auto space-y-2">
            {history.corrections.map((correction, idx) => (
              <div key={idx} className="pb-2 border-b last:border-b-0">
                <div className="text-xs text-muted-foreground">
                  {format(parseISO(correction.correctedAt), 'MMM d, h:mm a')}
                </div>
                <div>By: {correction.correctedBy}</div>
                <div className="text-xs">
                  {correction.originalValue} → {correction.correctedValue}
                </div>
              </div>
            ))}
          </div>
        </div>
      }
    />
  )
}
```

### Edge Case 4: Dark Mode Styling

```typescript
// Add dark mode variants to component styles
const darkModeClasses = cn(
  'dark:bg-orange-900/20 dark:text-orange-100 dark:border-orange-800',
  isRecent && 'dark:bg-blue-900/20 dark:text-blue-100 dark:border-blue-800'
)

// Enhanced component with dark mode support
function FieldOverrideIndicatorWithDarkMode(props: FieldOverrideIndicatorProps) {
  return (
    <FieldOverrideIndicator
      {...props}
      className={cn(props.className, darkModeClasses)}
    />
  )
}
```

### Edge Case 5: Mobile Touch Targets

```typescript
'use client'

import { useMediaQuery } from '@/hooks/use-media-query'

function ResponsiveFieldOverrideIndicator(props: FieldOverrideIndicatorProps) {
  const isMobile = useMediaQuery('(max-width: 768px)')

  return (
    <FieldOverrideIndicator
      {...props}
      size={isMobile ? 'lg' : props.size} // Larger on mobile
      variant={isMobile && props.variant === 'dot' ? 'icon' : props.variant}
      className={cn(
        props.className,
        isMobile && 'min-w-[44px] min-h-[44px]' // iOS minimum touch target
      )}
    />
  )
}
```

### Edge Case 6: Async Metadata Loading

```typescript
'use client'

import { useEffect, useState } from 'react'

function AsyncFieldOverrideIndicator({
  fieldId,
  isOverridden,
  ...props
}: FieldOverrideIndicatorProps & { fieldId: string }) {
  const [metadata, setMetadata] = useState<CorrectionMetadata | null>(null)
  const [loading, setLoading] = useState(false)

  useEffect(() => {
    if (isOverridden && !props.overriddenBy) {
      setLoading(true)
      fetchOverrideMetadata(fieldId)
        .then(setMetadata)
        .finally(() => setLoading(false))
    }
  }, [fieldId, isOverridden])

  if (loading) {
    return (
      <div className="inline-flex items-center gap-1 text-xs text-muted-foreground">
        <div className="w-3 h-3 border-2 border-current border-t-transparent rounded-full animate-spin" />
        Loading...
      </div>
    )
  }

  if (!metadata && isOverridden && !props.overriddenBy) {
    return null // Failed to load metadata
  }

  return (
    <FieldOverrideIndicator
      {...props}
      isOverridden={isOverridden}
      overriddenBy={metadata?.correctedBy || props.overriddenBy}
      overriddenAt={metadata?.correctedAt || props.overriddenAt}
      originalValue={metadata?.originalValue || props.originalValue}
    />
  )
}
```

---

## Accessibility

### ARIA Labels

```typescript
function AccessibleFieldOverrideIndicator(props: FieldOverrideIndicatorProps) {
  const ariaLabel = useMemo(() => {
    if (!props.isOverridden) return undefined

    const parts = ['Field manually corrected']

    if (props.overriddenBy) {
      parts.push(`by ${props.overriddenBy}`)
    }

    if (props.overriddenAt) {
      const timeAgo = formatDistanceToNow(parseISO(props.overriddenAt), { addSuffix: true })
      parts.push(timeAgo)
    }

    if (props.originalValue) {
      parts.push(`Original value: ${formatValue(props.originalValue)}`)
    }

    return parts.join(', ')
  }, [props])

  return (
    <FieldOverrideIndicator
      {...props}
      ariaLabel={ariaLabel}
    />
  )
}
```

### Keyboard Navigation

```typescript
function KeyboardNavigableIndicator(props: FieldOverrideIndicatorProps) {
  const handleKeyDown = (e: React.KeyboardEvent) => {
    if (e.key === 'Enter' || e.key === ' ') {
      e.preventDefault()
      props.onClick?.()
    }
  }

  return (
    <div
      onKeyDown={handleKeyDown}
      role={props.onClick ? 'button' : 'status'}
      tabIndex={props.onClick ? 0 : -1}
      aria-label={props.ariaLabel}
    >
      <FieldOverrideIndicator {...props} />
    </div>
  )
}
```

### Screen Reader Announcements

```typescript
'use client'

import { useEffect, useRef } from 'react'

function AnnouncedFieldOverrideIndicator(props: FieldOverrideIndicatorProps) {
  const [previousOverride, setPreviousOverride] = useState(props.isOverridden)
  const announceRef = useRef<HTMLDivElement>(null)

  useEffect(() => {
    // Announce when override status changes
    if (props.isOverridden && !previousOverride) {
      if (announceRef.current) {
        announceRef.current.textContent = `Field corrected by ${props.overriddenBy || 'unknown user'}`
      }
    }
    setPreviousOverride(props.isOverridden)
  }, [props.isOverridden, props.overriddenBy, previousOverride])

  return (
    <>
      {/* Live region for screen reader announcements */}
      <div
        ref={announceRef}
        className="sr-only"
        role="status"
        aria-live="polite"
        aria-atomic="true"
      />

      <FieldOverrideIndicator {...props} />
    </>
  )
}
```

---

## Testing

### Unit Tests

```typescript
import { render, screen, waitFor } from '@testing-library/react'
import userEvent from '@testing-library/user-event'
import { FieldOverrideIndicator } from './field-override-indicator'

describe('FieldOverrideIndicator', () => {
  const mockMetadata = {
    overriddenBy: 'Sarah Chen',
    overriddenAt: '2025-10-24T14:30:00Z',
    originalValue: 'Original Value',
    currentValue: 'Corrected Value'
  }

  it('does not render when isOverridden is false', () => {
    const { container } = render(
      <FieldOverrideIndicator isOverridden={false} />
    )
    expect(container.firstChild).toBeNull()
  })

  it('renders badge variant correctly', () => {
    render(
      <FieldOverrideIndicator
        isOverridden={true}
        variant="badge"
        label="Edited"
        {...mockMetadata}
      />
    )
    expect(screen.getByText('Edited')).toBeInTheDocument()
  })

  it('renders icon variant correctly', () => {
    render(
      <FieldOverrideIndicator
        isOverridden={true}
        variant="icon"
        {...mockMetadata}
      />
    )
    expect(screen.getByTestId('field-override-indicator')).toBeInTheDocument()
  })

  it('shows tooltip on hover', async () => {
    render(
      <FieldOverrideIndicator
        isOverridden={true}
        showTooltip={true}
        {...mockMetadata}
      />
    )

    const indicator = screen.getByTestId('field-override-indicator')
    await userEvent.hover(indicator)

    await waitFor(() => {
      expect(screen.getByText('Manually Corrected')).toBeInTheDocument()
      expect(screen.getByText('Sarah Chen')).toBeInTheDocument()
    })
  })

  it('calls onClick when clicked', async () => {
    const handleClick = jest.fn()
    render(
      <FieldOverrideIndicator
        isOverridden={true}
        onClick={handleClick}
        {...mockMetadata}
      />
    )

    const indicator = screen.getByTestId('field-override-indicator')
    await userEvent.click(indicator)

    expect(handleClick).toHaveBeenCalledTimes(1)
  })

  it('displays comparison in tooltip when originalValue provided', async () => {
    render(
      <FieldOverrideIndicator
        isOverridden={true}
        showTooltip={true}
        showComparison={true}
        {...mockMetadata}
      />
    )

    const indicator = screen.getByTestId('field-override-indicator')
    await userEvent.hover(indicator)

    await waitFor(() => {
      expect(screen.getByText(/Original Value/)).toBeInTheDocument()
      expect(screen.getByText(/Corrected Value/)).toBeInTheDocument()
    })
  })

  it('applies custom color', () => {
    const { container } = render(
      <FieldOverrideIndicator
        isOverridden={true}
        variant="badge"
        color="purple"
        {...mockMetadata}
      />
    )

    const badge = container.querySelector('[style*="purple"]')
    expect(badge).toBeInTheDocument()
  })

  it('highlights recent corrections', () => {
    const recentTime = new Date().toISOString()
    render(
      <FieldOverrideIndicator
        isOverridden={true}
        overriddenAt={recentTime}
        highlightRecent={true}
        {...mockMetadata}
      />
    )

    // Component should apply blue color for recent edits
    // Test implementation depends on how you expose this
  })

  it('handles missing metadata gracefully', () => {
    render(
      <FieldOverrideIndicator
        isOverridden={true}
        showTooltip={true}
      />
    )

    expect(screen.getByTestId('field-override-indicator')).toBeInTheDocument()
  })
})
```

### Integration Tests

```typescript
import { render, screen } from '@testing-library/react'
import userEvent from '@testing-library/user-event'
import { TransactionMerchantField } from './examples/finance'

describe('FieldOverrideIndicator Integration', () => {
  it('integrates with form field and opens audit trail', async () => {
    const transaction = {
      id: 'txn_001',
      merchant: 'Starbucks',
      merchantOverride: {
        original: 'STARBUCKS #2847',
        by: 'Sarah Chen',
        at: '2025-10-24T14:30:00Z',
        reason: 'Normalized merchant name'
      }
    }

    render(<TransactionMerchantField transaction={transaction} />)

    // Field should show corrected value
    expect(screen.getByText('Starbucks')).toBeInTheDocument()

    // Override indicator should be visible
    const indicator = screen.getByText(/Edited/)
    expect(indicator).toBeInTheDocument()

    // Click should open audit trail
    await userEvent.click(indicator)
    // Assert audit trail dialog opened
  })
})
```

---

## Performance Considerations

### Memoization

```typescript
import { memo } from 'react'

export const FieldOverrideIndicator = memo(
  function FieldOverrideIndicator(props: FieldOverrideIndicatorProps) {
    // Component implementation...
  },
  (prevProps, nextProps) => {
    // Custom comparison - only re-render if override state changes
    return (
      prevProps.isOverridden === nextProps.isOverridden &&
      prevProps.overriddenAt === nextProps.overriddenAt &&
      prevProps.overriddenBy === nextProps.overriddenBy &&
      prevProps.variant === nextProps.variant
    )
  }
)
```

### Lazy Loading Metadata

```typescript
function LazyMetadataIndicator({ fieldId, isOverridden }: {
  fieldId: string
  isOverridden: boolean
}) {
  const [metadata, setMetadata] = useState<CorrectionMetadata | null>(null)
  const [tooltipOpen, setTooltipOpen] = useState(false)

  useEffect(() => {
    // Only fetch metadata when tooltip is about to open
    if (tooltipOpen && !metadata) {
      fetchOverrideMetadata(fieldId).then(setMetadata)
    }
  }, [tooltipOpen, fieldId, metadata])

  return (
    <Tooltip onOpenChange={setTooltipOpen}>
      <TooltipTrigger>
        <FieldOverrideIndicator
          isOverridden={isOverridden}
          {...(metadata && {
            overriddenBy: metadata.correctedBy,
            overriddenAt: metadata.correctedAt,
            originalValue: metadata.originalValue
          })}
        />
      </TooltipTrigger>
      <TooltipContent>
        {metadata ? (
          <CorrectionTooltip metadata={metadata} />
        ) : (
          <div>Loading...</div>
        )}
      </TooltipContent>
    </Tooltip>
  )
}
```

---

## Design System Integration

### Shadcn/ui Components Used

```typescript
// Required shadcn/ui components
import { Badge } from '@/components/ui/badge'
import { Tooltip, TooltipContent, TooltipProvider, TooltipTrigger } from '@/components/ui/tooltip'
import { HoverCard, HoverCardContent, HoverCardTrigger } from '@/components/ui/hover-card'

// Optional for advanced features
import { Popover, PopoverContent, PopoverTrigger } from '@/components/ui/popover'
import { Sheet, SheetContent, SheetTrigger } from '@/components/ui/sheet'
```

### Theme Variables

```css
/* Add to globals.css */
:root {
  --override-indicator-default: 39 100% 50%; /* orange */
  --override-indicator-recent: 221 83% 53%; /* blue */
  --override-indicator-critical: 0 84% 60%; /* red */
}

.dark {
  --override-indicator-default: 32 95% 44%;
  --override-indicator-recent: 217 91% 60%;
  --override-indicator-critical: 0 72% 51%;
}
```

---

## Related Components

### Component Relationships

```typescript
// Often used together with:
import { AuditTrailViewer } from '@/components/il/audit-trail-viewer'
import { FieldComparisonView } from '@/components/il/field-comparison-view'
import { ValidationStatusBadge } from '@/components/il/validation-status-badge'
import { ConfidenceIndicator } from '@/components/il/confidence-indicator'

// Example combined usage
function ComprehensiveFieldDisplay({ field }) {
  return (
    <div className="flex items-center gap-2">
      <span>{field.value}</span>

      {field.isOverridden && (
        <FieldOverrideIndicator
          isOverridden={true}
          {...field.overrideMetadata}
        />
      )}

      {field.confidence && (
        <ConfidenceIndicator score={field.confidence} />
      )}

      {field.validationStatus && (
        <ValidationStatusBadge status={field.validationStatus} />
      )}
    </div>
  )
}
```

---

## Migration Guide

### From Custom Implementation

```typescript
// Before (custom implementation)
function OldOverrideIndicator({ isEdited, editor, editDate }) {
  if (!isEdited) return null

  return (
    <span className="edited-badge" title={`Edited by ${editor} on ${editDate}`}>
      ✏️ Edited
    </span>
  )
}

// After (using FieldOverrideIndicator)
function NewOverrideIndicator({ isEdited, editor, editDate, originalValue }) {
  return (
    <FieldOverrideIndicator
      isOverridden={isEdited}
      overriddenBy={editor}
      overriddenAt={editDate}
      originalValue={originalValue}
      variant="badge"
      size="sm"
    />
  )
}
```

---

## Future Enhancements

### Planned Features

1. **Batch Override Indicators** - Show multiple corrections in one tooltip
2. **AI-Assisted Corrections** - Different styling for AI vs manual edits
3. **Confidence Scoring** - Show correction confidence level
4. **Rollback Action** - Quick revert to original value
5. **Diff View** - Visual diff for complex value changes
6. **Correction Templates** - Common correction reasons as quick-select
7. **Multi-Language Support** - i18n for all text content

---

## Changelog

### Version 1.0.0 (2025-10-24)

- Initial release
- Three variants: badge, icon, dot
- Interactive tooltip with metadata
- Multi-domain examples
- Comprehensive edge case handling
- Full accessibility support
- Dark mode styling
- Mobile responsiveness

---

## License

MIT License - Part of the Information Layer (IL) Component Library

---

## Support

For questions, issues, or feature requests related to the FieldOverrideIndicator component:

- **Documentation:** [IL Component Library Docs](/)
- **Issues:** [GitHub Issues](https://github.com/your-org/il-components/issues)
- **Examples:** [Storybook](https://storybook.il-components.dev)

---

**End of FieldOverrideIndicator Specification**

*Last Updated: 2025-10-24*
*Version: 1.0.0*
*Component Stability: Stable*
