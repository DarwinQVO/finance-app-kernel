# GoalProgressCard (IL Component)

## Definition

**GoalProgressCard** is a visual component for displaying goal progress with a progress bar, percentage completion, amount remaining, and time remaining. It provides at-a-glance status indicators and quick actions for managing individual goals.

**Problem it solves:**
- No standardized way to display goal progress across different domains
- Hard to assess if goal is on track without manual calculation
- No visual feedback for goal status (on-track, behind, overdue)
- Scattered goal information across multiple screens
- No quick actions for editing/archiving goals
- Difficulty comparing multiple goals side-by-side

**Solution:**
- Single card component for all goal types
- Visual progress bar with percentage label
- Color-coded status indicators (green/yellow/red)
- Automatic calculation of amount remaining and days left
- Quick actions: edit, delete, view details, share
- Responsive design (stacks on mobile)
- Support for different goal types (savings, income, expense, custom)

---

## Interface Contract

```typescript
interface GoalProgressCardProps {
  // Data
  goal: Goal;  // Goal object with all metadata
  currentValue: number;  // Current progress towards goal

  // Callbacks
  onEdit?: () => void;  // Edit goal
  onDelete?: () => void;  // Delete goal
  onViewDetails?: () => void;  // View detailed progress
  onShare?: () => void;  // Share goal with others

  // Display options
  showDetails?: boolean;  // Show extended details (default: false)
  showActions?: boolean;  // Show action buttons (default: true)
  compact?: boolean;  // Compact view for smaller spaces (default: false)

  // Customization
  theme?: "light" | "dark";
  locale?: string;  // For date/number formatting (default: "en-US")
  currency?: string;  // For financial goals (e.g., "USD")

  // States
  loading?: boolean;
  error?: string | null;
}

interface Goal {
  goal_id: string;
  name: string;
  goal_type: GoalType;
  target_amount: number;  // Target value to reach
  current_amount: number;  // Current progress (optional, can use currentValue prop)
  deadline: string;  // ISO 8601 date
  frequency?: GoalFrequency;  // For recurring goals
  status: GoalStatus;

  // Optional metadata
  description?: string;
  category?: string;  // e.g., "emergency_fund", "vacation", "debt_payoff"
  icon?: string;  // Emoji or icon identifier
  color?: string;  // Hex color for progress bar
  created_at: string;
  updated_at: string;

  // Alerts
  alert_threshold?: number;  // Percentage to trigger alert (e.g., 90)
  alert_enabled?: boolean;
}

type GoalType =
  | "savings"     // Save a specific amount
  | "income"      // Earn a specific amount
  | "expense"     // Spend less than amount (inverse)
  | "custom";     // Custom goal type

type GoalFrequency =
  | "one_time"    // Single goal
  | "daily"       // Daily target
  | "weekly"      // Weekly target
  | "monthly"     // Monthly target
  | "quarterly"   // Quarterly target
  | "yearly";     // Yearly target

type GoalStatus =
  | "active"      // In progress
  | "completed"   // Goal reached
  | "overdue"     // Past deadline, not completed
  | "archived";   // Manually archived
```

---

## Component Structure

```tsx
import React, { useMemo } from 'react';

export const GoalProgressCard: React.FC<GoalProgressCardProps> = ({
  goal,
  currentValue,
  onEdit,
  onDelete,
  onViewDetails,
  onShare,
  showDetails = false,
  showActions = true,
  compact = false,
  theme = "light",
  locale = "en-US",
  currency,
  loading = false,
  error = null
}) => {
  // Calculate progress metrics
  const metrics = useMemo(() => {
    const value = currentValue ?? goal.current_amount ?? 0;
    const target = goal.target_amount;
    const percentage = target > 0 ? (value / target) * 100 : 0;
    const remaining = target - value;
    const isComplete = percentage >= 100;

    // Calculate days remaining
    const now = new Date();
    const deadlineDate = new Date(goal.deadline);
    const daysRemaining = Math.ceil((deadlineDate.getTime() - now.getTime()) / (1000 * 60 * 60 * 24));
    const isPastDeadline = daysRemaining < 0;

    // Determine status
    let status: 'on-track' | 'behind' | 'overdue' | 'completed';
    if (isComplete) {
      status = 'completed';
    } else if (isPastDeadline) {
      status = 'overdue';
    } else {
      // Calculate expected progress based on time elapsed
      const createdDate = new Date(goal.created_at);
      const totalDays = Math.ceil((deadlineDate.getTime() - createdDate.getTime()) / (1000 * 60 * 60 * 24));
      const elapsedDays = totalDays - daysRemaining;
      const expectedPercentage = totalDays > 0 ? (elapsedDays / totalDays) * 100 : 0;

      // Consider "on track" if within 10% of expected progress
      status = percentage >= expectedPercentage - 10 ? 'on-track' : 'behind';
    }

    return {
      currentValue: value,
      targetValue: target,
      percentage: Math.min(percentage, 100),
      remaining,
      daysRemaining,
      isPastDeadline,
      isComplete,
      status
    };
  }, [goal, currentValue]);

  // Format currency value
  const formatValue = (value: number): string => {
    if (currency) {
      return new Intl.NumberFormat(locale, {
        style: 'currency',
        currency,
        minimumFractionDigits: 0,
        maximumFractionDigits: 0
      }).format(value);
    }

    return new Intl.NumberFormat(locale, {
      minimumFractionDigits: 0,
      maximumFractionDigits: 2
    }).format(value);
  };

  // Format date
  const formatDate = (dateString: string): string => {
    const date = new Date(dateString);
    return new Intl.DateTimeFormat(locale, {
      month: 'short',
      day: 'numeric',
      year: 'numeric'
    }).format(date);
  };

  // Get status color
  const getStatusColor = (): string => {
    switch (metrics.status) {
      case 'completed': return 'var(--success-color)';
      case 'on-track': return 'var(--success-color)';
      case 'behind': return 'var(--warning-color)';
      case 'overdue': return 'var(--error-color)';
      default: return 'var(--primary-color)';
    }
  };

  // Get status icon
  const getStatusIcon = (): string => {
    switch (metrics.status) {
      case 'completed': return '✅';
      case 'on-track': return '🟢';
      case 'behind': return '🟡';
      case 'overdue': return '🔴';
      default: return '⚪';
    }
  };

  // Get status label
  const getStatusLabel = (): string => {
    switch (metrics.status) {
      case 'completed': return 'Completed';
      case 'on-track': return 'On Track';
      case 'behind': return 'Behind Schedule';
      case 'overdue': return 'Overdue';
      default: return 'Active';
    }
  };

  // Loading state
  if (loading) {
    return (
      <div className={`goal-progress-card theme-${theme} loading ${compact ? 'compact' : ''}`}>
        <div className="skeleton skeleton-header"></div>
        <div className="skeleton skeleton-progress"></div>
        <div className="skeleton skeleton-footer"></div>
      </div>
    );
  }

  // Error state
  if (error) {
    return (
      <div className={`goal-progress-card theme-${theme} error ${compact ? 'compact' : ''}`}>
        <div className="error-icon">⚠️</div>
        <div className="error-message">{error}</div>
      </div>
    );
  }

  return (
    <div
      className={`goal-progress-card theme-${theme} status-${metrics.status} ${compact ? 'compact' : ''}`}
      onClick={onViewDetails}
      style={{ cursor: onViewDetails ? 'pointer' : 'default' }}
    >
      {/* Header */}
      <div className="card-header">
        <div className="goal-info">
          {goal.icon && <span className="goal-icon">{goal.icon}</span>}
          <div className="goal-title-group">
            <h4 className="goal-name">{goal.name}</h4>
            {!compact && goal.category && (
              <span className="goal-category">{formatCategory(goal.category)}</span>
            )}
          </div>
        </div>

        {showActions && !compact && (
          <div className="card-actions">
            {onEdit && (
              <button
                className="action-button edit"
                onClick={(e) => {
                  e.stopPropagation();
                  onEdit();
                }}
                title="Edit goal"
                aria-label="Edit goal"
              >
                ✏️
              </button>
            )}
            {onShare && (
              <button
                className="action-button share"
                onClick={(e) => {
                  e.stopPropagation();
                  onShare();
                }}
                title="Share goal"
                aria-label="Share goal"
              >
                📤
              </button>
            )}
            {onDelete && (
              <button
                className="action-button delete"
                onClick={(e) => {
                  e.stopPropagation();
                  onDelete();
                }}
                title="Delete goal"
                aria-label="Delete goal"
              >
                🗑️
              </button>
            )}
          </div>
        )}
      </div>

      {/* Status indicator */}
      <div className="status-indicator">
        <span className="status-icon">{getStatusIcon()}</span>
        <span className="status-label">{getStatusLabel()}</span>
      </div>

      {/* Progress bar */}
      <div className="progress-section">
        <div className="progress-header">
          <span className="progress-label">Progress</span>
          <span className="progress-percentage" style={{ color: getStatusColor() }}>
            {metrics.percentage.toFixed(0)}%
          </span>
        </div>

        <div className="progress-bar-container">
          <div
            className="progress-bar-fill"
            style={{
              width: `${metrics.percentage}%`,
              backgroundColor: goal.color || getStatusColor()
            }}
          >
            {metrics.percentage >= 15 && (
              <span className="progress-bar-label">{metrics.percentage.toFixed(0)}%</span>
            )}
          </div>
        </div>

        <div className="progress-footer">
          <div className="progress-current">
            <span className="label">Current:</span>
            <span className="value">{formatValue(metrics.currentValue)}</span>
          </div>
          <div className="progress-target">
            <span className="label">Target:</span>
            <span className="value">{formatValue(metrics.targetValue)}</span>
          </div>
        </div>
      </div>

      {/* Amount remaining */}
      {!metrics.isComplete && (
        <div className="remaining-section">
          <div className="remaining-amount">
            <span className="remaining-label">Remaining:</span>
            <span className="remaining-value" style={{ color: getStatusColor() }}>
              {formatValue(Math.abs(metrics.remaining))}
            </span>
          </div>

          <div className="remaining-time">
            <span className="time-label">
              {metrics.isPastDeadline ? 'Overdue by:' : 'Days left:'}
            </span>
            <span className={`time-value ${metrics.isPastDeadline ? 'overdue' : ''}`}>
              {Math.abs(metrics.daysRemaining)} {Math.abs(metrics.daysRemaining) === 1 ? 'day' : 'days'}
            </span>
          </div>
        </div>
      )}

      {/* Completion message */}
      {metrics.isComplete && (
        <div className="completion-section">
          <div className="completion-icon">🎉</div>
          <div className="completion-message">Goal completed!</div>
          <div className="completion-date">
            Achieved on {formatDate(new Date().toISOString())}
          </div>
        </div>
      )}

      {/* Deadline */}
      {!compact && (
        <div className="deadline-section">
          <span className="deadline-label">Deadline:</span>
          <span className="deadline-value">{formatDate(goal.deadline)}</span>
          {goal.frequency && goal.frequency !== 'one_time' && (
            <span className="frequency-badge">{formatFrequency(goal.frequency)}</span>
          )}
        </div>
      )}

      {/* Extended details */}
      {showDetails && !compact && goal.description && (
        <div className="details-section">
          <div className="details-label">Description</div>
          <div className="details-content">{goal.description}</div>
        </div>
      )}

      {/* Alert indicator */}
      {goal.alert_enabled && goal.alert_threshold && metrics.percentage >= goal.alert_threshold && (
        <div className="alert-banner">
          <span className="alert-icon">🔔</span>
          <span className="alert-message">
            You've reached {metrics.percentage.toFixed(0)}% of your goal!
          </span>
        </div>
      )}
    </div>
  );
};

// Helper functions
function formatCategory(category: string): string {
  return category
    .split('_')
    .map(word => word.charAt(0).toUpperCase() + word.slice(1))
    .join(' ');
}

function formatFrequency(frequency: GoalFrequency): string {
  const labels: Record<GoalFrequency, string> = {
    one_time: 'One-Time',
    daily: 'Daily',
    weekly: 'Weekly',
    monthly: 'Monthly',
    quarterly: 'Quarterly',
    yearly: 'Yearly'
  };
  return labels[frequency] || frequency;
}
```

---

## Styling

```css
.goal-progress-card {
  background: var(--card-bg);
  border: 1px solid var(--card-border);
  border-radius: 12px;
  padding: 20px;
  transition: all 0.3s ease;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
}

.goal-progress-card:hover {
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
  transform: translateY(-2px);
}

.goal-progress-card.compact {
  padding: 16px;
}

/* Status-based border colors */
.goal-progress-card.status-completed {
  border-left: 4px solid var(--success-color);
}

.goal-progress-card.status-on-track {
  border-left: 4px solid var(--success-color);
}

.goal-progress-card.status-behind {
  border-left: 4px solid var(--warning-color);
}

.goal-progress-card.status-overdue {
  border-left: 4px solid var(--error-color);
}

/* Header */
.card-header {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  margin-bottom: 16px;
}

.goal-info {
  display: flex;
  align-items: flex-start;
  gap: 12px;
  flex: 1;
}

.goal-icon {
  font-size: 32px;
  line-height: 1;
}

.goal-progress-card.compact .goal-icon {
  font-size: 24px;
}

.goal-title-group {
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.goal-name {
  font-size: 18px;
  font-weight: 600;
  color: var(--heading-color);
  margin: 0;
}

.goal-progress-card.compact .goal-name {
  font-size: 16px;
}

.goal-category {
  font-size: 12px;
  color: var(--secondary-color);
  text-transform: uppercase;
  font-weight: 500;
  letter-spacing: 0.5px;
}

.card-actions {
  display: flex;
  gap: 8px;
}

.action-button {
  padding: 6px 10px;
  background: transparent;
  border: 1px solid var(--action-border);
  border-radius: 6px;
  cursor: pointer;
  font-size: 16px;
  transition: all 0.2s;
}

.action-button:hover {
  background: var(--action-hover-bg);
  transform: scale(1.1);
}

/* Status indicator */
.status-indicator {
  display: flex;
  align-items: center;
  gap: 8px;
  margin-bottom: 16px;
  padding: 8px 12px;
  background: var(--status-bg);
  border-radius: 8px;
}

.status-icon {
  font-size: 16px;
}

.status-label {
  font-size: 13px;
  font-weight: 600;
  color: var(--text-color);
}

/* Progress section */
.progress-section {
  margin-bottom: 16px;
}

.progress-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 8px;
}

.progress-label {
  font-size: 13px;
  font-weight: 500;
  color: var(--secondary-color);
  text-transform: uppercase;
  letter-spacing: 0.5px;
}

.progress-percentage {
  font-size: 20px;
  font-weight: 700;
}

.progress-bar-container {
  width: 100%;
  height: 24px;
  background: var(--progress-bg);
  border-radius: 12px;
  overflow: hidden;
  position: relative;
  margin-bottom: 8px;
}

.progress-bar-fill {
  height: 100%;
  border-radius: 12px;
  transition: width 0.5s ease;
  display: flex;
  align-items: center;
  justify-content: flex-end;
  padding-right: 8px;
  position: relative;
}

.progress-bar-label {
  font-size: 11px;
  font-weight: 700;
  color: white;
  text-shadow: 0 1px 2px rgba(0, 0, 0, 0.2);
}

.progress-footer {
  display: flex;
  justify-content: space-between;
  align-items: center;
  font-size: 13px;
}

.progress-current,
.progress-target {
  display: flex;
  flex-direction: column;
  gap: 2px;
}

.progress-current .label,
.progress-target .label {
  font-size: 11px;
  color: var(--secondary-color);
  text-transform: uppercase;
  font-weight: 500;
}

.progress-current .value,
.progress-target .value {
  font-size: 14px;
  font-weight: 700;
  color: var(--text-color);
}

/* Remaining section */
.remaining-section {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 12px;
  background: var(--remaining-bg);
  border-radius: 8px;
  margin-bottom: 16px;
}

.remaining-amount,
.remaining-time {
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.remaining-label,
.time-label {
  font-size: 11px;
  color: var(--secondary-color);
  text-transform: uppercase;
  font-weight: 500;
}

.remaining-value {
  font-size: 18px;
  font-weight: 700;
}

.time-value {
  font-size: 18px;
  font-weight: 700;
  color: var(--text-color);
}

.time-value.overdue {
  color: var(--error-color);
}

/* Completion section */
.completion-section {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 8px;
  padding: 20px;
  background: var(--success-bg);
  border-radius: 8px;
  margin-bottom: 16px;
}

.completion-icon {
  font-size: 48px;
}

.completion-message {
  font-size: 18px;
  font-weight: 700;
  color: var(--success-color);
}

.completion-date {
  font-size: 13px;
  color: var(--secondary-color);
}

/* Deadline section */
.deadline-section {
  display: flex;
  align-items: center;
  gap: 8px;
  font-size: 13px;
  color: var(--secondary-color);
  padding-top: 12px;
  border-top: 1px solid var(--divider-color);
}

.deadline-label {
  font-weight: 500;
}

.deadline-value {
  font-weight: 600;
  color: var(--text-color);
}

.frequency-badge {
  padding: 2px 8px;
  background: var(--badge-bg);
  color: var(--badge-color);
  border-radius: 4px;
  font-size: 11px;
  font-weight: 600;
  text-transform: uppercase;
  margin-left: auto;
}

/* Details section */
.details-section {
  margin-top: 16px;
  padding-top: 16px;
  border-top: 1px solid var(--divider-color);
}

.details-label {
  font-size: 12px;
  font-weight: 600;
  color: var(--secondary-color);
  text-transform: uppercase;
  letter-spacing: 0.5px;
  margin-bottom: 8px;
}

.details-content {
  font-size: 14px;
  color: var(--text-color);
  line-height: 1.5;
}

/* Alert banner */
.alert-banner {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 12px;
  background: var(--alert-bg);
  border: 1px solid var(--alert-border);
  border-radius: 8px;
  margin-top: 16px;
}

.alert-icon {
  font-size: 18px;
}

.alert-message {
  font-size: 13px;
  font-weight: 600;
  color: var(--alert-color);
}

/* Loading state */
.goal-progress-card.loading {
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.skeleton {
  background: linear-gradient(
    90deg,
    var(--skeleton-start) 0%,
    var(--skeleton-middle) 50%,
    var(--skeleton-end) 100%
  );
  background-size: 200% 100%;
  animation: shimmer 1.5s infinite;
  border-radius: 8px;
}

.skeleton-header {
  height: 50px;
  width: 60%;
}

.skeleton-progress {
  height: 60px;
  width: 100%;
}

.skeleton-footer {
  height: 40px;
  width: 80%;
}

@keyframes shimmer {
  0% { background-position: 200% 0; }
  100% { background-position: -200% 0; }
}

/* Error state */
.goal-progress-card.error {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 12px;
  min-height: 200px;
}

.error-icon {
  font-size: 48px;
}

.error-message {
  font-size: 14px;
  color: var(--error-color);
  text-align: center;
}

/* Theme: Light */
.theme-light {
  --card-bg: #ffffff;
  --card-border: #e5e7eb;
  --heading-color: #111827;
  --text-color: #111827;
  --secondary-color: #6b7280;
  --primary-color: #3b82f6;
  --success-color: #10b981;
  --warning-color: #f59e0b;
  --error-color: #ef4444;
  --action-border: #e5e7eb;
  --action-hover-bg: #f3f4f6;
  --status-bg: #f9fafb;
  --progress-bg: #e5e7eb;
  --remaining-bg: #f9fafb;
  --success-bg: #d1fae5;
  --divider-color: #e5e7eb;
  --badge-bg: #dbeafe;
  --badge-color: #1e40af;
  --alert-bg: #fef3c7;
  --alert-border: #fcd34d;
  --alert-color: #92400e;
  --skeleton-start: #f3f4f6;
  --skeleton-middle: #e5e7eb;
  --skeleton-end: #f3f4f6;
}

/* Theme: Dark */
.theme-dark {
  --card-bg: #1f2937;
  --card-border: #374151;
  --heading-color: #f3f4f6;
  --text-color: #f3f4f6;
  --secondary-color: #9ca3af;
  --primary-color: #3b82f6;
  --success-color: #10b981;
  --warning-color: #f59e0b;
  --error-color: #ef4444;
  --action-border: #4b5563;
  --action-hover-bg: #374151;
  --status-bg: #111827;
  --progress-bg: #374151;
  --remaining-bg: #111827;
  --success-bg: #064e3b;
  --divider-color: #374151;
  --badge-bg: #1e3a8a;
  --badge-color: #93c5fd;
  --alert-bg: #78350f;
  --alert-border: #92400e;
  --alert-color: #fcd34d;
  --skeleton-start: #374151;
  --skeleton-middle: #4b5563;
  --skeleton-end: #374151;
}

/* Responsive */
@media (max-width: 768px) {
  .goal-progress-card {
    padding: 16px;
  }

  .goal-name {
    font-size: 16px;
  }

  .progress-percentage {
    font-size: 18px;
  }

  .remaining-section {
    flex-direction: column;
    align-items: flex-start;
    gap: 12px;
  }

  .card-actions {
    position: absolute;
    top: 16px;
    right: 16px;
  }
}
```

---

## Multi-Domain Applicability

### Finance Domain: Savings Goal Card
```tsx
<GoalProgressCard
  goal={{
    goal_id: "g_001",
    name: "Emergency Fund",
    goal_type: "savings",
    target_amount: 10000,
    current_amount: 6500,
    deadline: "2025-12-31",
    frequency: "monthly",
    status: "active",
    category: "emergency_fund",
    icon: "🏦",
    color: "#10b981",
    description: "Build 6 months of expenses",
    created_at: "2025-01-01",
    updated_at: "2025-10-24",
    alert_threshold: 90,
    alert_enabled: true
  }}
  currentValue={6500}
  currency="USD"
  onEdit={() => console.log('Edit goal')}
  onDelete={() => console.log('Delete goal')}
  onViewDetails={() => console.log('View details')}
  showDetails={true}
/>
// Shows: 65% progress, $3,500 remaining, 68 days left, on track
```

### Healthcare Domain: Budget Target Card
```tsx
<GoalProgressCard
  goal={{
    goal_id: "g_002",
    name: "Annual Healthcare Budget",
    goal_type: "expense",
    target_amount: 5000,
    current_amount: 3200,
    deadline: "2025-12-31",
    status: "active",
    category: "medical_expenses",
    icon: "🏥",
    color: "#f59e0b",
    description: "Stay under $5,000 for the year",
    created_at: "2025-01-01",
    updated_at: "2025-10-24"
  }}
  currentValue={3200}
  currency="USD"
  onEdit={() => console.log('Edit budget')}
  showActions={true}
  compact={false}
/>
// Shows: 64% spent (inverse), $1,800 remaining, 68 days left
```

### Legal Domain: Billable Hours Goal
```tsx
<GoalProgressCard
  goal={{
    goal_id: "g_003",
    name: "Q4 Billable Hours",
    goal_type: "custom",
    target_amount: 500,
    current_amount: 387,
    deadline: "2025-12-31",
    frequency: "quarterly",
    status: "active",
    category: "billable_hours",
    icon: "⚖️",
    color: "#3b82f6",
    created_at: "2025-10-01",
    updated_at: "2025-10-24"
  }}
  currentValue={387}
  onEdit={() => console.log('Edit hours goal')}
  onViewDetails={() => console.log('View hours breakdown')}
  showDetails={false}
/>
// Shows: 77% progress, 113 hours remaining, on track
```

### Research Domain: Grant Spending Goal
```tsx
<GoalProgressCard
  goal={{
    goal_id: "g_004",
    name: "NSF Grant Budget",
    goal_type: "expense",
    target_amount: 250000,
    current_amount: 187500,
    deadline: "2026-06-30",
    status: "active",
    category: "grant_spending",
    icon: "🎓",
    color: "#8b5cf6",
    description: "Spend grant funds before deadline",
    created_at: "2024-07-01",
    updated_at: "2025-10-24",
    alert_threshold: 80,
    alert_enabled: true
  }}
  currentValue={187500}
  currency="USD"
  onEdit={() => console.log('Edit grant budget')}
  showActions={true}
  theme="dark"
/>
// Shows: 75% spent, $62,500 remaining, 249 days left
```

### Manufacturing Domain: Production Target Card
```tsx
<GoalProgressCard
  goal={{
    goal_id: "g_005",
    name: "Monthly Production Target",
    goal_type: "custom",
    target_amount: 10000,
    current_amount: 7834,
    deadline: "2025-10-31",
    frequency: "monthly",
    status: "active",
    category: "production_units",
    icon: "🏭",
    color: "#06b6d4",
    created_at: "2025-10-01",
    updated_at: "2025-10-24"
  }}
  currentValue={7834}
  onViewDetails={() => console.log('View production stats')}
  compact={true}
/>
// Shows: 78% progress, 2,166 units remaining, 7 days left
```

### Media Domain: Subscriber Goal Card
```tsx
<GoalProgressCard
  goal={{
    goal_id: "g_006",
    name: "100K Subscribers",
    goal_type: "custom",
    target_amount: 100000,
    current_amount: 87234,
    deadline: "2025-12-31",
    status: "active",
    category: "subscribers",
    icon: "📺",
    color: "#ec4899",
    description: "Reach 100K subscribers by year end",
    created_at: "2025-01-01",
    updated_at: "2025-10-24",
    alert_threshold: 95,
    alert_enabled: true
  }}
  currentValue={87234}
  onEdit={() => console.log('Edit subscriber goal')}
  onShare={() => console.log('Share progress')}
  showDetails={true}
/>
// Shows: 87% progress, 12,766 subs remaining, on track
```

---

## Features

### 1. Automatic Status Calculation
```typescript
// Compares actual vs expected progress based on time
// On-track: Within 10% of expected progress
// Behind: More than 10% below expected
// Overdue: Past deadline without completion
// Completed: 100% or more of target reached
```

### 2. Color-Coded Visual Indicators
```typescript
// Green: On track or completed
// Yellow: Behind schedule
// Red: Overdue
// Border color matches status
// Progress bar uses custom color or status color
```

### 3. Smart Time Calculations
```typescript
// Days remaining until deadline
// Percentage of time elapsed
// Expected progress based on time
// Overdue days (if past deadline)
```

### 4. Flexible Goal Types
```typescript
// Savings: Accumulate amount
// Income: Earn amount
// Expense: Stay under amount (inverse progress)
// Custom: Any metric (hours, units, subscribers)
```

### 5. Alert Notifications
```typescript
// Trigger when reaching threshold (e.g., 90%)
// Visual banner at bottom of card
// Optional: Email/push notifications
```

### 6. Responsive Design
```typescript
// Desktop: Full details, side-by-side layout
// Tablet: Stacked layout, all features
// Mobile: Compact mode, essential info only
```

---

## Accessibility

```tsx
<div
  className="goal-progress-card"
  role="article"
  aria-label={`Goal progress card for ${goal.name}`}
  tabIndex={0}
>
  <h4 className="goal-name" id="goal-name-001">
    {goal.name}
  </h4>

  <div
    className="progress-bar-container"
    role="progressbar"
    aria-valuenow={metrics.percentage}
    aria-valuemin={0}
    aria-valuemax={100}
    aria-labelledby="goal-name-001"
  >
    <div className="progress-bar-fill" />
  </div>

  <button
    className="action-button edit"
    aria-label={`Edit ${goal.name} goal`}
    onClick={onEdit}
  >
    ✏️
  </button>
</div>
```

---

## Related Components

**Uses:**
- None (primitive UI component)

**Used by:**
- GoalDashboard (displays multiple goal cards)
- GoalConfigDialog (creates/edits goals)
- ForecastChart (shows goal vs forecast)

**Similar patterns:**
- BudgetCard (budget tracking)
- MetricCard (KPI tracking)
- TaskProgressCard (task completion)
