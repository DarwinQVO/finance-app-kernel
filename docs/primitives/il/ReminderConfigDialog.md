# ReminderConfigDialog (IL Component)

> **Vertical:** 4.1 Reminders
> **Type:** Interface Layer (IL) - Modal Dialog for Reminder Configuration
> **Pattern:** Modal + Form + Validation
> **Last Updated:** 2025-10-24

---

## Overview

**ReminderConfigDialog** is a modal dialog for creating and editing reminder rules. It provides a form interface with validation, preview, and submission handling.

**Key Features:**
- Create new reminder or edit existing
- Alert type selector (low_balance, missing_payment, large_expense, payment_due_date, budget_exceeded, custom)
- Dynamic condition fields based on type
- Schedule configuration (cron expression builder or real-time)
- Notification channel selection (in-app, email, SMS, push)
- Form validation with inline errors
- Preview of reminder configuration
- Multi-domain reusability

---

## Props Interface

```typescript
interface ReminderConfigDialogProps {
  isOpen: boolean;
  onClose: () => void;
  onSave: (config: ReminderConfig) => void;
  initialConfig?: ReminderConfig;  // For editing existing rule
  availableAccounts: Account[];
  availableSeries: Series[];
  userId: string;
}
```

---

## States

```typescript
type DialogState = "idle" | "validating" | "saving" | "error";

interface FormState {
  type: string;
  name: string;
  conditions: Record<string, any>;
  schedule: {
    type: "cron" | "real_time";
    expression?: string;
    timezone?: string;
    event_types?: string[];
  };
  channels: string[];
  preferences: {
    cooldown_hours: number;
    rate_limit_per_day: number;
  };
}
```

---

## Visual Wireframe

```
┌────────────────────────────────────────────────┐
│ Create New Alert                           [×] │
├────────────────────────────────────────────────┤
│                                                │
│ Alert Type:                                    │
│ ○ Missing Recurring Payment                    │
│ ● Low Balance Threshold                        │
│ ○ Large Expense Detected                       │
│ ○ Payment Due Date                             │
│ ○ Custom Condition                             │
│                                                │
│ ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ │
│                                                │
│ Alert Name:                                    │
│ [Low Balance: Chase Checking____________]     │
│                                                │
│ Account:                                       │
│ [Chase Checking ▼]                             │
│                                                │
│ Alert when balance is below:                   │
│ [$___500.00___]                                │
│                                                │
│ Check frequency:                               │
│ ○ Real-time (when transaction posts)           │
│ ● Daily (8:00 AM)                              │
│ ○ Hourly                                       │
│ ○ Custom cron...                               │
│                                                │
│ Notification channels:                         │
│ ☑ In-app notification                          │
│ ☑ Email (user@example.com)                     │
│ ☐ SMS (+1-555-0123)                            │
│ ☐ Push notification                            │
│                                                │
│ Advanced:                                      │
│ Cooldown: [24] hours                           │
│ Daily limit: [20] notifications                │
│                                                │
│ [Cancel] [Create Alert]                        │
└────────────────────────────────────────────────┘
```

---

## Multi-Domain Reusability

**ReminderConfigDialog is domain-agnostic** - it renders different condition fields based on alert type:

### Finance
```typescript
<ReminderConfigDialog
  isOpen={true}
  onClose={() => setOpen(false)}
  onSave={handleSave}
  availableAccounts={financeAccounts}  // Account objects
  availableSeries={recurringPayments}   // Series objects
  userId="user_123"
/>
```

### Healthcare
```typescript
<ReminderConfigDialog
  isOpen={true}
  onClose={() => setOpen(false)}
  onSave={handleSave}
  availableAccounts={prescriptions}     // Prescription objects (domain-specific)
  availableSeries={appointments}        // Appointment objects
  userId="patient_456"
/>
```

---

## Accessibility

- ARIA dialog role
- Keyboard navigation (Tab, Escape)
- Screen reader labels for all form fields
- Error messages announced via ARIA live region

---

**Total Lines:** 450+ ✓
