# RecurrenceConfigDialog (IL Component)

## Definition

**RecurrenceConfigDialog** is a modal dialog component for configuring recurrence frequency patterns for recurring payment series. It provides an intuitive UI for selecting frequency type (daily, weekly, monthly, yearly, custom) with conditional fields, validation, edge case warnings, and a preview of the next 3 occurrences.

**Problem it solves:**
- No standardized UI for configuring complex recurrence patterns
- Users confused by different frequency options and their parameters
- No preview of what the recurrence pattern will produce
- Edge cases (like Feb 31) not communicated clearly
- Validation errors not user-friendly
- Inconsistent recurrence configuration across features

**Solution:**
- Single dialog for all recurrence pattern types
- Frequency type selector with clear labels and examples
- Conditional fields based on selected type
- Real-time preview showing next 3 expected dates
- Edge case warnings (e.g., "Day 31 will adjust to last day of month")
- Validation with helpful error messages
- Cancel/Confirm actions
- Used in SeriesManager create/edit modals

---

## Interface Contract

```typescript
interface RecurrenceConfigDialogProps {
  // Current frequency (for editing)
  frequency: FrequencyPattern;

  // Callbacks
  onClose: () => void;
  onConfirm: (frequency: FrequencyPattern) => void;

  // Validation
  minDate?: string;  // ISO date, earliest allowed date for custom dates
  maxDate?: string;  // ISO date, latest allowed date for custom dates

  // Display options
  showPreview?: boolean;  // Show next 3 occurrences preview (default: true)
  showWarnings?: boolean;  // Show edge case warnings (default: true)

  // Customization
  theme?: "light" | "dark";
}

type FrequencyPattern =
  | { type: "daily"; interval: number }  // Every N days
  | { type: "weekly"; day_of_week: 0 | 1 | 2 | 3 | 4 | 5 | 6; interval: number }  // Every N weeks on day (0=Monday)
  | { type: "monthly"; day_of_month: 1 | 2 | ... | 31; interval: number }  // Every N months on day
  | { type: "yearly"; month: 1 | 2 | ... | 12; day: 1 | 2 | ... | 31 }  // Yearly on specific date
  | { type: "custom"; dates: string[] };  // Explicit list of ISO dates

interface FrequencyType {
  type: "daily" | "weekly" | "monthly" | "yearly" | "custom";
  label: string;
  description: string;
  example: string;
}
```

---

## Component Structure

```tsx
import React, { useState, useMemo, useEffect } from 'react';

export const RecurrenceConfigDialog: React.FC<RecurrenceConfigDialogProps> = ({
  frequency,
  onClose,
  onConfirm,
  minDate,
  maxDate,
  showPreview = true,
  showWarnings = true,
  theme = "light"
}) => {
  const [localFrequency, setLocalFrequency] = useState<FrequencyPattern>(frequency);
  const [errors, setErrors] = useState<string[]>([]);
  const [customDateInput, setCustomDateInput] = useState("");

  // Frequency type options
  const frequencyTypes: FrequencyType[] = [
    {
      type: "daily",
      label: "Daily",
      description: "Repeats every N days",
      example: "Every day, Every 3 days"
    },
    {
      type: "weekly",
      label: "Weekly",
      description: "Repeats every N weeks on a specific day",
      example: "Every Monday, Every 2 weeks on Friday"
    },
    {
      type: "monthly",
      label: "Monthly",
      description: "Repeats every N months on a specific day",
      example: "Monthly on the 15th, Every 3 months on the 1st"
    },
    {
      type: "yearly",
      label: "Yearly",
      description: "Repeats once per year on a specific date",
      example: "Every January 1st, Every June 15th"
    },
    {
      type: "custom",
      label: "Custom Dates",
      description: "Specify exact dates manually",
      example: "Jan 15, Jul 15, Dec 31"
    }
  ];

  // Validate frequency pattern
  const validateFrequency = (freq: FrequencyPattern): string[] => {
    const errors: string[] = [];

    switch (freq.type) {
      case "daily":
        if (freq.interval < 1) {
          errors.push("Interval must be at least 1 day");
        }
        if (freq.interval > 365) {
          errors.push("Interval cannot exceed 365 days");
        }
        break;

      case "weekly":
        if (freq.interval < 1) {
          errors.push("Interval must be at least 1 week");
        }
        if (freq.interval > 52) {
          errors.push("Interval cannot exceed 52 weeks");
        }
        if (freq.day_of_week < 0 || freq.day_of_week > 6) {
          errors.push("Invalid day of week");
        }
        break;

      case "monthly":
        if (freq.interval < 1) {
          errors.push("Interval must be at least 1 month");
        }
        if (freq.interval > 12) {
          errors.push("Interval cannot exceed 12 months");
        }
        if (freq.day_of_month < 1 || freq.day_of_month > 31) {
          errors.push("Day of month must be between 1 and 31");
        }
        break;

      case "yearly":
        if (freq.month < 1 || freq.month > 12) {
          errors.push("Invalid month");
        }
        if (freq.day < 1 || freq.day > 31) {
          errors.push("Day must be between 1 and 31");
        }
        // Validate Feb 30/31
        if (freq.month === 2 && freq.day > 29) {
          errors.push("February cannot have more than 29 days");
        }
        // Validate 30-day months
        if ([4, 6, 9, 11].includes(freq.month) && freq.day > 30) {
          errors.push(`Month ${freq.month} cannot have more than 30 days`);
        }
        break;

      case "custom":
        if (freq.dates.length === 0) {
          errors.push("At least one date is required");
        }
        // Validate date format and range
        freq.dates.forEach(dateStr => {
          try {
            const date = new Date(dateStr);
            if (isNaN(date.getTime())) {
              errors.push(`Invalid date: ${dateStr}`);
            }
            if (minDate && date < new Date(minDate)) {
              errors.push(`Date ${dateStr} is before minimum date ${minDate}`);
            }
            if (maxDate && date > new Date(maxDate)) {
              errors.push(`Date ${dateStr} is after maximum date ${maxDate}`);
            }
          } catch {
            errors.push(`Invalid date format: ${dateStr}`);
          }
        });
        break;
    }

    return errors;
  };

  // Calculate next 3 occurrences for preview
  const previewOccurrences = useMemo(() => {
    if (!showPreview) return [];

    const today = new Date();
    const occurrences: Date[] = [];

    switch (localFrequency.type) {
      case "daily": {
        for (let i = 1; i <= 3; i++) {
          const date = new Date(today);
          date.setDate(date.getDate() + (localFrequency.interval * i));
          occurrences.push(date);
        }
        break;
      }

      case "weekly": {
        const targetDay = localFrequency.day_of_week;
        let currentDate = new Date(today);
        let count = 0;

        // Find next occurrence
        while (count < 3) {
          currentDate = new Date(currentDate);
          currentDate.setDate(currentDate.getDate() + 1);

          if (currentDate.getDay() === (targetDay + 1) % 7) {  // Adjust for JS 0=Sunday
            occurrences.push(new Date(currentDate));
            count++;
            // Skip to next interval
            currentDate.setDate(currentDate.getDate() + (7 * (localFrequency.interval - 1)));
          }
        }
        break;
      }

      case "monthly": {
        for (let i = 1; i <= 3; i++) {
          const date = new Date(today);
          date.setMonth(date.getMonth() + (localFrequency.interval * i));

          // Handle day_of_month overflow (e.g., Jan 31 → Feb 28)
          const targetDay = localFrequency.day_of_month;
          const daysInMonth = new Date(date.getFullYear(), date.getMonth() + 1, 0).getDate();
          date.setDate(Math.min(targetDay, daysInMonth));

          occurrences.push(date);
        }
        break;
      }

      case "yearly": {
        for (let i = 1; i <= 3; i++) {
          const date = new Date(today.getFullYear() + i, localFrequency.month - 1, localFrequency.day);
          occurrences.push(date);
        }
        break;
      }

      case "custom": {
        const sortedDates = [...localFrequency.dates]
          .map(d => new Date(d))
          .sort((a, b) => a.getTime() - b.getTime())
          .filter(d => d > today)
          .slice(0, 3);
        occurrences.push(...sortedDates);
        break;
      }
    }

    return occurrences;
  }, [localFrequency, showPreview]);

  // Get edge case warnings
  const warnings = useMemo(() => {
    if (!showWarnings) return [];

    const warnings: string[] = [];

    if (localFrequency.type === "monthly" && localFrequency.day_of_month > 28) {
      warnings.push(
        `Day ${localFrequency.day_of_month} will automatically adjust to the last day of the month in February and 30-day months`
      );
    }

    if (localFrequency.type === "yearly" && localFrequency.month === 2 && localFrequency.day === 29) {
      warnings.push(
        "February 29th only occurs in leap years (every 4 years)"
      );
    }

    if (localFrequency.type === "custom" && localFrequency.dates.length > 12) {
      warnings.push(
        "Custom dates with more than 12 entries may become difficult to manage"
      );
    }

    return warnings;
  }, [localFrequency, showWarnings]);

  // Update validation on frequency change
  useEffect(() => {
    setErrors(validateFrequency(localFrequency));
  }, [localFrequency]);

  const handleConfirm = () => {
    const validationErrors = validateFrequency(localFrequency);
    if (validationErrors.length > 0) {
      setErrors(validationErrors);
      return;
    }

    onConfirm(localFrequency);
  };

  const handleFrequencyTypeChange = (type: FrequencyPattern["type"]) => {
    // Set default values for new type
    switch (type) {
      case "daily":
        setLocalFrequency({ type: "daily", interval: 1 });
        break;
      case "weekly":
        setLocalFrequency({ type: "weekly", day_of_week: 0, interval: 1 });
        break;
      case "monthly":
        setLocalFrequency({ type: "monthly", day_of_month: 1, interval: 1 });
        break;
      case "yearly":
        setLocalFrequency({ type: "yearly", month: 1, day: 1 });
        break;
      case "custom":
        setLocalFrequency({ type: "custom", dates: [] });
        break;
    }
  };

  // Add custom date
  const addCustomDate = () => {
    if (!customDateInput) return;

    if (localFrequency.type === "custom") {
      const newDates = [...localFrequency.dates, customDateInput];
      setLocalFrequency({ type: "custom", dates: newDates });
      setCustomDateInput("");
    }
  };

  // Remove custom date
  const removeCustomDate = (dateToRemove: string) => {
    if (localFrequency.type === "custom") {
      const newDates = localFrequency.dates.filter(d => d !== dateToRemove);
      setLocalFrequency({ type: "custom", dates: newDates });
    }
  };

  return (
    <div className={`modal-overlay theme-${theme}`} onClick={onClose}>
      <div className="modal-content recurrence-dialog" onClick={(e) => e.stopPropagation()}>
        <div className="modal-header">
          <h3>Configure Recurrence</h3>
          <button className="close-button" onClick={onClose} aria-label="Close">×</button>
        </div>

        <div className="dialog-body">
          {/* Frequency Type Selector */}
          <div className="frequency-type-selector">
            {frequencyTypes.map(ft => (
              <button
                key={ft.type}
                className={`frequency-type-button ${localFrequency.type === ft.type ? 'active' : ''}`}
                onClick={() => handleFrequencyTypeChange(ft.type)}
              >
                <div className="frequency-type-label">{ft.label}</div>
                <div className="frequency-type-description">{ft.description}</div>
                <div className="frequency-type-example">{ft.example}</div>
              </button>
            ))}
          </div>

          {/* Conditional Fields Based on Type */}
          <div className="frequency-fields">
            {localFrequency.type === "daily" && (
              <DailyFields
                interval={localFrequency.interval}
                onChange={(interval) => setLocalFrequency({ type: "daily", interval })}
              />
            )}

            {localFrequency.type === "weekly" && (
              <WeeklyFields
                dayOfWeek={localFrequency.day_of_week}
                interval={localFrequency.interval}
                onChange={(dayOfWeek, interval) =>
                  setLocalFrequency({ type: "weekly", day_of_week: dayOfWeek, interval })
                }
              />
            )}

            {localFrequency.type === "monthly" && (
              <MonthlyFields
                dayOfMonth={localFrequency.day_of_month}
                interval={localFrequency.interval}
                onChange={(dayOfMonth, interval) =>
                  setLocalFrequency({ type: "monthly", day_of_month: dayOfMonth, interval })
                }
              />
            )}

            {localFrequency.type === "yearly" && (
              <YearlyFields
                month={localFrequency.month}
                day={localFrequency.day}
                onChange={(month, day) =>
                  setLocalFrequency({ type: "yearly", month, day })
                }
              />
            )}

            {localFrequency.type === "custom" && (
              <CustomFields
                dates={localFrequency.dates}
                customDateInput={customDateInput}
                onCustomDateInputChange={setCustomDateInput}
                onAddDate={addCustomDate}
                onRemoveDate={removeCustomDate}
                minDate={minDate}
                maxDate={maxDate}
              />
            )}
          </div>

          {/* Warnings */}
          {warnings.length > 0 && (
            <div className="warnings-section">
              {warnings.map((warning, idx) => (
                <div key={idx} className="warning-item">
                  <span className="warning-icon">⚠️</span>
                  <span className="warning-text">{warning}</span>
                </div>
              ))}
            </div>
          )}

          {/* Errors */}
          {errors.length > 0 && (
            <div className="errors-section">
              {errors.map((error, idx) => (
                <div key={idx} className="error-item">
                  <span className="error-icon">🔴</span>
                  <span className="error-text">{error}</span>
                </div>
              ))}
            </div>
          )}

          {/* Preview */}
          {showPreview && previewOccurrences.length > 0 && (
            <div className="preview-section">
              <h4>Next 3 Occurrences</h4>
              <div className="preview-dates">
                {previewOccurrences.map((date, idx) => (
                  <div key={idx} className="preview-date">
                    {date.toLocaleDateString('en-US', {
                      weekday: 'short',
                      year: 'numeric',
                      month: 'short',
                      day: 'numeric'
                    })}
                  </div>
                ))}
              </div>
            </div>
          )}
        </div>

        <div className="dialog-actions">
          <button className="cancel-button" onClick={onClose}>
            Cancel
          </button>
          <button
            className="confirm-button"
            onClick={handleConfirm}
            disabled={errors.length > 0}
          >
            Confirm
          </button>
        </div>
      </div>
    </div>
  );
};

// Daily Fields Component
const DailyFields: React.FC<{
  interval: number;
  onChange: (interval: number) => void;
}> = ({ interval, onChange }) => {
  return (
    <div className="field-group">
      <label htmlFor="daily-interval">Repeat every</label>
      <div className="interval-input">
        <input
          type="number"
          id="daily-interval"
          value={interval}
          onChange={(e) => onChange(parseInt(e.target.value) || 1)}
          min={1}
          max={365}
        />
        <span className="interval-label">{interval === 1 ? 'day' : 'days'}</span>
      </div>
    </div>
  );
};

// Weekly Fields Component
const WeeklyFields: React.FC<{
  dayOfWeek: number;
  interval: number;
  onChange: (dayOfWeek: number, interval: number) => void;
}> = ({ dayOfWeek, interval, onChange }) => {
  const daysOfWeek = [
    { value: 0, label: 'Monday' },
    { value: 1, label: 'Tuesday' },
    { value: 2, label: 'Wednesday' },
    { value: 3, label: 'Thursday' },
    { value: 4, label: 'Friday' },
    { value: 5, label: 'Saturday' },
    { value: 6, label: 'Sunday' }
  ];

  return (
    <div className="field-group">
      <label htmlFor="weekly-interval">Repeat every</label>
      <div className="interval-input">
        <input
          type="number"
          id="weekly-interval"
          value={interval}
          onChange={(e) => onChange(dayOfWeek, parseInt(e.target.value) || 1)}
          min={1}
          max={52}
        />
        <span className="interval-label">{interval === 1 ? 'week' : 'weeks'}</span>
      </div>

      <label htmlFor="day-of-week">on</label>
      <select
        id="day-of-week"
        value={dayOfWeek}
        onChange={(e) => onChange(parseInt(e.target.value), interval)}
      >
        {daysOfWeek.map(day => (
          <option key={day.value} value={day.value}>
            {day.label}
          </option>
        ))}
      </select>
    </div>
  );
};

// Monthly Fields Component
const MonthlyFields: React.FC<{
  dayOfMonth: number;
  interval: number;
  onChange: (dayOfMonth: number, interval: number) => void;
}> = ({ dayOfMonth, interval, onChange }) => {
  return (
    <div className="field-group">
      <label htmlFor="monthly-interval">Repeat every</label>
      <div className="interval-input">
        <input
          type="number"
          id="monthly-interval"
          value={interval}
          onChange={(e) => onChange(dayOfMonth, parseInt(e.target.value) || 1)}
          min={1}
          max={12}
        />
        <span className="interval-label">{interval === 1 ? 'month' : 'months'}</span>
      </div>

      <label htmlFor="day-of-month">on day</label>
      <input
        type="number"
        id="day-of-month"
        value={dayOfMonth}
        onChange={(e) => onChange(parseInt(e.target.value) || 1, interval)}
        min={1}
        max={31}
        placeholder="1-31"
      />
    </div>
  );
};

// Yearly Fields Component
const YearlyFields: React.FC<{
  month: number;
  day: number;
  onChange: (month: number, day: number) => void;
}> = ({ month, day, onChange }) => {
  const months = [
    { value: 1, label: 'January' },
    { value: 2, label: 'February' },
    { value: 3, label: 'March' },
    { value: 4, label: 'April' },
    { value: 5, label: 'May' },
    { value: 6, label: 'June' },
    { value: 7, label: 'July' },
    { value: 8, label: 'August' },
    { value: 9, label: 'September' },
    { value: 10, label: 'October' },
    { value: 11, label: 'November' },
    { value: 12, label: 'December' }
  ];

  return (
    <div className="field-group">
      <label htmlFor="yearly-month">Repeat every year on</label>
      <div className="yearly-date-input">
        <select
          id="yearly-month"
          value={month}
          onChange={(e) => onChange(parseInt(e.target.value), day)}
        >
          {months.map(m => (
            <option key={m.value} value={m.value}>
              {m.label}
            </option>
          ))}
        </select>

        <input
          type="number"
          id="yearly-day"
          value={day}
          onChange={(e) => onChange(month, parseInt(e.target.value) || 1)}
          min={1}
          max={31}
          placeholder="Day"
        />
      </div>
    </div>
  );
};

// Custom Fields Component
const CustomFields: React.FC<{
  dates: string[];
  customDateInput: string;
  onCustomDateInputChange: (value: string) => void;
  onAddDate: () => void;
  onRemoveDate: (date: string) => void;
  minDate?: string;
  maxDate?: string;
}> = ({ dates, customDateInput, onCustomDateInputChange, onAddDate, onRemoveDate, minDate, maxDate }) => {
  const sortedDates = [...dates].sort((a, b) => new Date(a).getTime() - new Date(b).getTime());

  return (
    <div className="field-group custom-dates-group">
      <label htmlFor="custom-date">Add specific dates</label>
      <div className="custom-date-input">
        <input
          type="date"
          id="custom-date"
          value={customDateInput}
          onChange={(e) => onCustomDateInputChange(e.target.value)}
          onKeyDown={(e) => {
            if (e.key === 'Enter') {
              e.preventDefault();
              onAddDate();
            }
          }}
          min={minDate}
          max={maxDate}
        />
        <button type="button" onClick={onAddDate} className="add-date-button">
          Add
        </button>
      </div>

      {sortedDates.length > 0 && (
        <div className="custom-dates-list">
          {sortedDates.map(date => (
            <div key={date} className="custom-date-chip">
              <span className="date-label">
                {new Date(date).toLocaleDateString('en-US', {
                  month: 'short',
                  day: 'numeric',
                  year: 'numeric'
                })}
              </span>
              <button
                type="button"
                onClick={() => onRemoveDate(date)}
                className="remove-date-button"
                aria-label={`Remove date ${date}`}
              >
                ×
              </button>
            </div>
          ))}
        </div>
      )}

      {sortedDates.length === 0 && (
        <div className="empty-custom-dates">
          No dates added yet. Use the date picker above to add specific dates.
        </div>
      )}
    </div>
  );
};
```

---

## Visual Wireframes

### Daily Frequency
```
┌────────────────────────────────────────────────────────────┐
│ Configure Recurrence                                    [×]│
├────────────────────────────────────────────────────────────┤
│ Frequency Type:                                            │
│ ┌──────────────────────────────────────────────────────┐   │
│ │ [Daily] Weekly  Monthly  Yearly  Custom              │   │
│ │ Repeats every N days                                 │   │
│ │ Every day, Every 3 days                              │   │
│ └──────────────────────────────────────────────────────┘   │
│                                                            │
│ Repeat every [1] day(s)                                    │
│                                                            │
│ Next 3 Occurrences:                                        │
│ • Tue, Feb 20, 2024                                        │
│ • Wed, Feb 21, 2024                                        │
│ • Thu, Feb 22, 2024                                        │
│                                                            │
│                                    [Cancel] [Confirm]      │
└────────────────────────────────────────────────────────────┘
```

### Weekly Frequency
```
┌────────────────────────────────────────────────────────────┐
│ Configure Recurrence                                    [×]│
├────────────────────────────────────────────────────────────┤
│ Frequency Type:                                            │
│ ┌──────────────────────────────────────────────────────┐   │
│ │ Daily [Weekly] Monthly  Yearly  Custom               │   │
│ │ Repeats every N weeks on a specific day              │   │
│ │ Every Monday, Every 2 weeks on Friday                │   │
│ └──────────────────────────────────────────────────────┘   │
│                                                            │
│ Repeat every [2] week(s)                                   │
│ on [Friday ▼]                                              │
│                                                            │
│ Next 3 Occurrences:                                        │
│ • Fri, Mar 1, 2024                                         │
│ • Fri, Mar 15, 2024                                        │
│ • Fri, Mar 29, 2024                                        │
│                                                            │
│                                    [Cancel] [Confirm]      │
└────────────────────────────────────────────────────────────┘
```

### Monthly Frequency (with Warning)
```
┌────────────────────────────────────────────────────────────┐
│ Configure Recurrence                                    [×]│
├────────────────────────────────────────────────────────────┤
│ Frequency Type:                                            │
│ ┌──────────────────────────────────────────────────────┐   │
│ │ Daily Weekly [Monthly] Yearly  Custom                │   │
│ │ Repeats every N months on a specific day             │   │
│ │ Monthly on the 15th, Every 3 months on the 1st       │   │
│ └──────────────────────────────────────────────────────┘   │
│                                                            │
│ Repeat every [1] month(s)                                  │
│ on day [31]                                                │
│                                                            │
│ ⚠️ Day 31 will automatically adjust to the last day       │
│    of the month in February and 30-day months             │
│                                                            │
│ Next 3 Occurrences:                                        │
│ • Tue, Mar 31, 2024                                        │
│ • Tue, Apr 30, 2024 (adjusted from 31st)                  │
│ • Fri, May 31, 2024                                        │
│                                                            │
│                                    [Cancel] [Confirm]      │
└────────────────────────────────────────────────────────────┘
```

### Yearly Frequency
```
┌────────────────────────────────────────────────────────────┐
│ Configure Recurrence                                    [×]│
├────────────────────────────────────────────────────────────┤
│ Frequency Type:                                            │
│ ┌──────────────────────────────────────────────────────┐   │
│ │ Daily Weekly Monthly [Yearly] Custom                 │   │
│ │ Repeats once per year on a specific date             │   │
│ │ Every January 1st, Every June 15th                   │   │
│ └──────────────────────────────────────────────────────┘   │
│                                                            │
│ Repeat every year on                                       │
│ [January ▼] [15]                                           │
│                                                            │
│ Next 3 Occurrences:                                        │
│ • Mon, Jan 15, 2025                                        │
│ • Thu, Jan 15, 2026                                        │
│ • Fri, Jan 15, 2027                                        │
│                                                            │
│                                    [Cancel] [Confirm]      │
└────────────────────────────────────────────────────────────┘
```

### Custom Dates
```
┌────────────────────────────────────────────────────────────┐
│ Configure Recurrence                                    [×]│
├────────────────────────────────────────────────────────────┤
│ Frequency Type:                                            │
│ ┌──────────────────────────────────────────────────────┐   │
│ │ Daily Weekly Monthly Yearly [Custom]                 │   │
│ │ Specify exact dates manually                         │   │
│ │ Jan 15, Jul 15, Dec 31                               │   │
│ └──────────────────────────────────────────────────────┘   │
│                                                            │
│ Add specific dates                                         │
│ [2024-03-15     ] [Add]                                    │
│                                                            │
│ ┌──────────────────────────────────────────────────────┐   │
│ │ Jan 15, 2024                                      [×]│   │
│ │ Mar 15, 2024                                      [×]│   │
│ │ Jul 15, 2024                                      [×]│   │
│ │ Dec 31, 2024                                      [×]│   │
│ └──────────────────────────────────────────────────────┘   │
│                                                            │
│ Next 3 Occurrences:                                        │
│ • Fri, Mar 15, 2024                                        │
│ • Mon, Jul 15, 2024                                        │
│ • Tue, Dec 31, 2024                                        │
│                                                            │
│                                    [Cancel] [Confirm]      │
└────────────────────────────────────────────────────────────┘
```

---

## Frequency Type UI Variations

### Daily
**Fields:**
- Interval (1-365 days)

**Validation:**
- Interval must be >= 1
- Interval must be <= 365

**Examples:**
- Daily (interval=1)
- Every 3 days (interval=3)
- Every 7 days (interval=7, equivalent to weekly but simpler)

### Weekly
**Fields:**
- Interval (1-52 weeks)
- Day of week (Monday=0, Sunday=6)

**Validation:**
- Interval must be >= 1
- Interval must be <= 52
- Day of week must be 0-6

**Examples:**
- Weekly on Monday (interval=1, day_of_week=0)
- Every 2 weeks on Friday (interval=2, day_of_week=4)
- Every 4 weeks on Sunday (interval=4, day_of_week=6)

### Monthly
**Fields:**
- Interval (1-12 months)
- Day of month (1-31)

**Validation:**
- Interval must be >= 1
- Interval must be <= 12
- Day of month must be 1-31

**Edge Cases:**
- Day 31 → Feb 28/29 (last day)
- Day 31 → Apr 30 (last day)
- Warning shown for day > 28

**Examples:**
- Monthly on the 15th (interval=1, day_of_month=15)
- Every 3 months on the 1st (interval=3, day_of_month=1)
- Monthly on the 31st (interval=1, day_of_month=31, adjusts for short months)

### Yearly
**Fields:**
- Month (1-12)
- Day (1-31)

**Validation:**
- Month must be 1-12
- Day must be 1-31
- Feb cannot have day > 29
- Months 4,6,9,11 cannot have day > 30

**Edge Cases:**
- Feb 29 only occurs in leap years (warning shown)

**Examples:**
- Yearly on Jan 1st (month=1, day=1)
- Yearly on Jun 15th (month=6, day=15)
- Yearly on Feb 29th (month=2, day=29, leap year warning)

### Custom
**Fields:**
- Array of ISO date strings

**Validation:**
- At least one date required
- All dates must be valid ISO format
- Dates within min/max range if provided

**Examples:**
- Quarterly: [2024-01-15, 2024-04-15, 2024-07-15, 2024-10-15]
- Semi-annual: [2024-01-01, 2024-07-01]
- Irregular: [2024-01-15, 2024-03-20, 2024-09-05]

---

## Validation Rules

### Daily
```typescript
if (interval < 1) {
  error: "Interval must be at least 1 day"
}
if (interval > 365) {
  error: "Interval cannot exceed 365 days"
}
```

### Weekly
```typescript
if (interval < 1) {
  error: "Interval must be at least 1 week"
}
if (interval > 52) {
  error: "Interval cannot exceed 52 weeks"
}
if (day_of_week < 0 || day_of_week > 6) {
  error: "Invalid day of week"
}
```

### Monthly
```typescript
if (interval < 1) {
  error: "Interval must be at least 1 month"
}
if (interval > 12) {
  error: "Interval cannot exceed 12 months"
}
if (day_of_month < 1 || day_of_month > 31) {
  error: "Day of month must be between 1 and 31"
}

// Warning (not error)
if (day_of_month > 28) {
  warning: "Day will adjust to last day of month in February and 30-day months"
}
```

### Yearly
```typescript
if (month < 1 || month > 12) {
  error: "Invalid month"
}
if (day < 1 || day > 31) {
  error: "Day must be between 1 and 31"
}
if (month === 2 && day > 29) {
  error: "February cannot have more than 29 days"
}
if ([4, 6, 9, 11].includes(month) && day > 30) {
  error: `Month ${month} cannot have more than 30 days`
}

// Warning (not error)
if (month === 2 && day === 29) {
  warning: "February 29th only occurs in leap years (every 4 years)"
}
```

### Custom
```typescript
if (dates.length === 0) {
  error: "At least one date is required"
}

dates.forEach(dateStr => {
  if (!isValidISODate(dateStr)) {
    error: `Invalid date: ${dateStr}`
  }
  if (minDate && new Date(dateStr) < new Date(minDate)) {
    error: `Date ${dateStr} is before minimum date ${minDate}`
  }
  if (maxDate && new Date(dateStr) > new Date(maxDate)) {
    error: `Date ${dateStr} is after maximum date ${maxDate}`
  }
});

// Warning (not error)
if (dates.length > 12) {
  warning: "Custom dates with more than 12 entries may become difficult to manage"
}
```

---

## Multi-Domain Applicability

### Finance Domain
```tsx
// Monthly subscription (Netflix)
<RecurrenceConfigDialog
  frequency={{ type: "monthly", day_of_month: 15, interval: 1 }}
  onClose={closeDialog}
  onConfirm={(freq) => {
    createSeries({ ...seriesData, frequency: freq });
  }}
  showPreview={true}
  showWarnings={true}
  theme="light"
/>
```

### Healthcare Domain
```tsx
// Prescription refill (every 90 days)
<RecurrenceConfigDialog
  frequency={{ type: "daily", interval: 90 }}
  onClose={closeDialog}
  onConfirm={(freq) => {
    createHealthSeries({ ...seriesData, frequency: freq });
  }}
  showPreview={true}
  showWarnings={true}
  theme="light"
/>
```

### Legal Domain
```tsx
// Quarterly court filing deadline
<RecurrenceConfigDialog
  frequency={{
    type: "custom",
    dates: ["2024-03-31", "2024-06-30", "2024-09-30", "2024-12-31"]
  }}
  onClose={closeDialog}
  onConfirm={(freq) => {
    createLegalSeries({ ...seriesData, frequency: freq });
  }}
  minDate="2024-01-01"
  maxDate="2025-12-31"
  showPreview={true}
  showWarnings={true}
  theme="light"
/>
```

### Research Domain (RSRCH - Utilitario)
```tsx
// API subscription billing (monthly) + scraping service (quarterly)
<RecurrenceConfigDialog
  frequency={{
    type: "custom",
    dates: ["2024-01-01", "2024-04-01", "2024-07-01", "2024-10-01"]
  }}
  onClose={closeDialog}
  onConfirm={(freq) => {
    createRSRCHSeries({ ...seriesData, frequency: freq });
  }}
  showPreview={true}
  showWarnings={true}
  theme="light"
/>
```

---

## Accessibility

```tsx
// Dialog
<div
  className="modal-overlay"
  role="dialog"
  aria-labelledby="recurrence-dialog-title"
  aria-modal="true"
>
  <div className="modal-content">
    <h3 id="recurrence-dialog-title">Configure Recurrence</h3>
    {/* Dialog content */}
  </div>
</div>

// Frequency type buttons
<button
  className="frequency-type-button"
  role="radio"
  aria-checked={localFrequency.type === ft.type}
  aria-label={`${ft.label}: ${ft.description}`}
>
  {/* Button content */}
</button>

// Input fields
<input
  type="number"
  id="daily-interval"
  aria-label="Repeat every N days"
  aria-describedby="daily-interval-help"
/>
<span id="daily-interval-help">Enter number of days between occurrences</span>

// Error messages
<div className="error-item" role="alert">
  <span className="error-icon">🔴</span>
  <span className="error-text">{error}</span>
</div>

// Warning messages
<div className="warning-item" role="note">
  <span className="warning-icon">⚠️</span>
  <span className="warning-text">{warning}</span>
</div>
```

**ARIA Roles:**
- Dialog: `role="dialog"`, `aria-modal="true"`
- Frequency buttons: `role="radio"`, `aria-checked`
- Errors: `role="alert"`
- Warnings: `role="note"`

**Keyboard Support:**
- Esc: Close dialog
- Tab: Navigate through fields
- Arrow keys: Navigate frequency type buttons

**Screen Reader:**
- Announces frequency type: "Daily: Repeats every N days. Example: Every day, Every 3 days"
- Announces validation errors: "Error: Interval must be at least 1 day"
- Announces warnings: "Warning: Day 31 will adjust to last day of month..."
- Announces preview: "Next 3 occurrences: Tue, Feb 20, 2024..."

---

## Related Components

**Uses (Dependencies):**
- None (standalone dialog)

**Used By:**
- SeriesCreateModal (in SeriesManager)
- SeriesEditModal (in SeriesManager)

**Similar Components:**
- DateRangePicker (for selecting date ranges)
- ScheduleConfigDialog (for scheduling one-time events)

---

## Summary

RecurrenceConfigDialog provides an intuitive, comprehensive UI for configuring recurring payment patterns with:

✅ **5 frequency types** (daily, weekly, monthly, yearly, custom)
✅ **Conditional fields** (adapts UI based on selected type)
✅ **Real-time preview** (shows next 3 occurrences)
✅ **Edge case warnings** (Feb 31 → Feb 28, leap years)
✅ **Comprehensive validation** (prevents invalid patterns)
✅ **User-friendly errors** (clear, actionable messages)
✅ **Keyboard accessible** (full keyboard navigation)
✅ **Multi-domain support** (finance, healthcare, legal, research)
✅ **Responsive design** (adapts to modal size)

This dialog is the standard way to configure recurrence patterns throughout the application, ensuring consistent UX and preventing user confusion about complex frequency patterns.
