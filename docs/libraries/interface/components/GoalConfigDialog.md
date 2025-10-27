# GoalConfigDialog (IL Component)

## Definition

**GoalConfigDialog** is a modal dialog component for creating and editing goals. It provides a comprehensive form with validation, preset templates, and smart defaults to streamline goal configuration across different domains.

**Problem it solves:**
- No standardized UI for creating/editing goals
- Complex goal configuration requires multiple steps
- No preset templates for common goal types
- Validation errors discovered after submission
- No guidance for setting realistic deadlines
- Difficult to configure recurring goals
- No preview of goal before saving

**Solution:**
- Single modal for both create and edit modes
- Tabbed interface for basic and advanced settings
- Preset templates for common goals (emergency fund, debt payoff, etc.)
- Real-time validation with inline error messages
- Smart deadline suggestions based on goal type
- Recurring goal configuration (daily, weekly, monthly, etc.)
- Live preview of goal card
- Responsive design (full screen on mobile)

---

## Interface Contract

```typescript
interface GoalConfigDialogProps {
  // Mode
  mode: "create" | "edit";  // Create new goal or edit existing
  initialGoal?: Partial<Goal>;  // Pre-fill form in edit mode

  // Callbacks
  onSave: (goal: Goal) => Promise<void>;  // Save goal (create or update)
  onCancel: () => void;  // Close dialog without saving

  // Display options
  showTemplates?: boolean;  // Show preset templates (default: true in create mode)
  showAdvanced?: boolean;  // Show advanced settings tab (default: true)
  showPreview?: boolean;  // Show live preview (default: true)

  // Customization
  theme?: "light" | "dark";
  locale?: string;  // For date/number formatting (default: "en-US")
  defaultCurrency?: string;  // Default currency (e.g., "USD")

  // Templates
  templates?: GoalTemplate[];  // Custom goal templates

  // States
  loading?: boolean;
  error?: string | null;
}

interface Goal {
  goal_id?: string;  // Only in edit mode
  name: string;
  goal_type: GoalType;
  target_amount: number;
  deadline: string;  // ISO 8601 date
  frequency?: GoalFrequency;
  status?: GoalStatus;

  // Optional fields
  description?: string;
  category?: string;
  icon?: string;
  color?: string;
  alert_threshold?: number;  // 0-100
  alert_enabled?: boolean;

  // Auto-calculated
  created_at?: string;
  updated_at?: string;
}

type GoalType =
  | "savings"
  | "income"
  | "expense"
  | "custom";

type GoalFrequency =
  | "one_time"
  | "daily"
  | "weekly"
  | "monthly"
  | "quarterly"
  | "yearly";

type GoalStatus =
  | "active"
  | "completed"
  | "overdue"
  | "archived";

interface GoalTemplate {
  template_id: string;
  name: string;
  description: string;
  icon: string;
  goal_type: GoalType;
  suggested_amount?: number;
  suggested_deadline_months?: number;  // Months from now
  category: string;
  color: string;
}

interface ValidationError {
  field: string;
  message: string;
}
```

---

## Component Structure

```tsx
import React, { useState, useMemo, useEffect } from 'react';

// Preset templates
const DEFAULT_TEMPLATES: GoalTemplate[] = [
  {
    template_id: "emergency_fund",
    name: "Emergency Fund",
    description: "Build 3-6 months of expenses",
    icon: "🏦",
    goal_type: "savings",
    suggested_amount: 10000,
    suggested_deadline_months: 12,
    category: "emergency_fund",
    color: "#10b981"
  },
  {
    template_id: "vacation",
    name: "Vacation Savings",
    description: "Save for your dream vacation",
    icon: "✈️",
    goal_type: "savings",
    suggested_amount: 5000,
    suggested_deadline_months: 6,
    category: "vacation",
    color: "#3b82f6"
  },
  {
    template_id: "debt_payoff",
    name: "Debt Payoff",
    description: "Pay off credit card or loan",
    icon: "💳",
    goal_type: "expense",
    suggested_amount: 8000,
    suggested_deadline_months: 18,
    category: "debt_payoff",
    color: "#ef4444"
  },
  {
    template_id: "down_payment",
    name: "Home Down Payment",
    description: "Save for a house down payment",
    icon: "🏠",
    goal_type: "savings",
    suggested_amount: 50000,
    suggested_deadline_months: 36,
    category: "down_payment",
    color: "#8b5cf6"
  },
  {
    template_id: "retirement",
    name: "Retirement Savings",
    description: "Build long-term retirement fund",
    icon: "🌴",
    goal_type: "savings",
    suggested_amount: 1000000,
    suggested_deadline_months: 240,
    category: "retirement",
    color: "#f59e0b"
  }
];

export const GoalConfigDialog: React.FC<GoalConfigDialogProps> = ({
  mode,
  initialGoal,
  onSave,
  onCancel,
  showTemplates = mode === "create",
  showAdvanced = true,
  showPreview = true,
  theme = "light",
  locale = "en-US",
  defaultCurrency = "USD",
  templates = DEFAULT_TEMPLATES,
  loading = false,
  error = null
}) => {
  const [activeTab, setActiveTab] = useState<"basic" | "advanced" | "preview">("basic");
  const [selectedTemplate, setSelectedTemplate] = useState<GoalTemplate | null>(null);

  const [formData, setFormData] = useState<Partial<Goal>>({
    name: "",
    goal_type: "savings",
    target_amount: 0,
    deadline: "",
    frequency: "one_time",
    description: "",
    category: "",
    icon: "",
    color: "#3b82f6",
    alert_threshold: 90,
    alert_enabled: true,
    ...initialGoal
  });

  const [validationErrors, setValidationErrors] = useState<ValidationError[]>([]);
  const [submitting, setSubmitting] = useState(false);
  const [submitError, setSubmitError] = useState<string | null>(null);

  // Auto-fill from template
  useEffect(() => {
    if (selectedTemplate && mode === "create") {
      const suggestedDeadline = new Date();
      suggestedDeadline.setMonth(suggestedDeadline.getMonth() + (selectedTemplate.suggested_deadline_months || 12));

      setFormData({
        ...formData,
        name: selectedTemplate.name,
        goal_type: selectedTemplate.goal_type,
        target_amount: selectedTemplate.suggested_amount || 0,
        deadline: suggestedDeadline.toISOString().split('T')[0],
        category: selectedTemplate.category,
        icon: selectedTemplate.icon,
        color: selectedTemplate.color,
        description: selectedTemplate.description
      });
    }
  }, [selectedTemplate]);

  // Validate form
  const validate = (): boolean => {
    const errors: ValidationError[] = [];

    if (!formData.name || formData.name.trim().length === 0) {
      errors.push({ field: "name", message: "Goal name is required" });
    }

    if (!formData.target_amount || formData.target_amount <= 0) {
      errors.push({ field: "target_amount", message: "Target amount must be greater than 0" });
    }

    if (!formData.deadline) {
      errors.push({ field: "deadline", message: "Deadline is required" });
    } else {
      const deadlineDate = new Date(formData.deadline);
      const today = new Date();
      today.setHours(0, 0, 0, 0);

      if (deadlineDate <= today) {
        errors.push({ field: "deadline", message: "Deadline must be in the future" });
      }
    }

    if (formData.alert_threshold !== undefined && (formData.alert_threshold < 0 || formData.alert_threshold > 100)) {
      errors.push({ field: "alert_threshold", message: "Alert threshold must be between 0 and 100" });
    }

    setValidationErrors(errors);
    return errors.length === 0;
  };

  // Handle form submission
  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    setSubmitError(null);

    if (!validate()) {
      setActiveTab("basic");  // Jump to basic tab to show errors
      return;
    }

    setSubmitting(true);

    try {
      const goalData: Goal = {
        ...formData as Goal,
        status: mode === "create" ? "active" : (formData.status || "active"),
        created_at: mode === "create" ? new Date().toISOString() : formData.created_at,
        updated_at: new Date().toISOString()
      };

      await onSave(goalData);
    } catch (err) {
      setSubmitError(err instanceof Error ? err.message : "Failed to save goal");
      setSubmitting(false);
    }
  };

  // Get validation error for field
  const getFieldError = (field: string): string | undefined => {
    return validationErrors.find(e => e.field === field)?.message;
  };

  // Format date for input
  const formatDateForInput = (dateString: string): string => {
    if (!dateString) return "";
    return dateString.split('T')[0];
  };

  // Calculate suggested monthly contribution
  const suggestedMonthlyContribution = useMemo(() => {
    if (!formData.target_amount || !formData.deadline) return 0;

    const deadlineDate = new Date(formData.deadline);
    const today = new Date();
    const monthsRemaining = Math.max(
      1,
      Math.ceil((deadlineDate.getTime() - today.getTime()) / (1000 * 60 * 60 * 24 * 30))
    );

    return formData.target_amount / monthsRemaining;
  }, [formData.target_amount, formData.deadline]);

  return (
    <div className={`modal-overlay theme-${theme}`} onClick={onCancel}>
      <div className="modal-dialog goal-config-dialog" onClick={(e) => e.stopPropagation()}>
        {/* Header */}
        <div className="dialog-header">
          <h2>{mode === "create" ? "Create New Goal" : "Edit Goal"}</h2>
          <button className="close-button" onClick={onCancel} aria-label="Close dialog">
            ×
          </button>
        </div>

        {/* Templates (Create mode only) */}
        {showTemplates && mode === "create" && !selectedTemplate && (
          <div className="templates-section">
            <h3>Choose a Template</h3>
            <div className="template-grid">
              {templates.map(template => (
                <button
                  key={template.template_id}
                  className="template-card"
                  onClick={() => setSelectedTemplate(template)}
                >
                  <div className="template-icon">{template.icon}</div>
                  <div className="template-name">{template.name}</div>
                  <div className="template-description">{template.description}</div>
                </button>
              ))}
              <button
                className="template-card custom"
                onClick={() => setSelectedTemplate(null)}
              >
                <div className="template-icon">⚙️</div>
                <div className="template-name">Custom Goal</div>
                <div className="template-description">Start from scratch</div>
              </button>
            </div>
          </div>
        )}

        {/* Form (only show if template selected or in edit mode) */}
        {(selectedTemplate || mode === "edit" || (mode === "create" && !showTemplates)) && (
          <>
            {/* Tabs */}
            <div className="dialog-tabs">
              <button
                className={`tab ${activeTab === "basic" ? "active" : ""}`}
                onClick={() => setActiveTab("basic")}
              >
                Basic Info
              </button>
              {showAdvanced && (
                <button
                  className={`tab ${activeTab === "advanced" ? "active" : ""}`}
                  onClick={() => setActiveTab("advanced")}
                >
                  Advanced
                </button>
              )}
              {showPreview && (
                <button
                  className={`tab ${activeTab === "preview" ? "active" : ""}`}
                  onClick={() => setActiveTab("preview")}
                >
                  Preview
                </button>
              )}
            </div>

            {/* Error banner */}
            {(submitError || error) && (
              <div className="error-banner">
                <span className="error-icon">⚠️</span>
                <span className="error-text">{submitError || error}</span>
              </div>
            )}

            {/* Form */}
            <form onSubmit={handleSubmit} className="dialog-form">
              {/* Basic Tab */}
              {activeTab === "basic" && (
                <div className="tab-content">
                  {/* Goal Name */}
                  <div className="form-group">
                    <label htmlFor="goal-name">
                      Goal Name *
                    </label>
                    <input
                      type="text"
                      id="goal-name"
                      value={formData.name}
                      onChange={(e) => setFormData({ ...formData, name: e.target.value })}
                      placeholder="e.g., Emergency Fund, Vacation Savings"
                      className={getFieldError("name") ? "error" : ""}
                      required
                      autoFocus
                    />
                    {getFieldError("name") && (
                      <span className="field-error">{getFieldError("name")}</span>
                    )}
                  </div>

                  {/* Goal Type */}
                  <div className="form-group">
                    <label htmlFor="goal-type">
                      Goal Type *
                    </label>
                    <select
                      id="goal-type"
                      value={formData.goal_type}
                      onChange={(e) => setFormData({ ...formData, goal_type: e.target.value as GoalType })}
                      required
                    >
                      <option value="savings">Savings (accumulate amount)</option>
                      <option value="income">Income (earn amount)</option>
                      <option value="expense">Expense (stay under amount)</option>
                      <option value="custom">Custom (any metric)</option>
                    </select>
                    <small className="field-hint">
                      {formData.goal_type === "savings" && "Track progress towards saving a target amount"}
                      {formData.goal_type === "income" && "Track progress towards earning a target amount"}
                      {formData.goal_type === "expense" && "Track spending to stay under a budget"}
                      {formData.goal_type === "custom" && "Track any custom metric (hours, units, etc.)"}
                    </small>
                  </div>

                  {/* Target Amount */}
                  <div className="form-group">
                    <label htmlFor="target-amount">
                      Target Amount *
                    </label>
                    <div className="input-with-prefix">
                      <span className="input-prefix">{formData.goal_type === "custom" ? "#" : "$"}</span>
                      <input
                        type="number"
                        id="target-amount"
                        value={formData.target_amount}
                        onChange={(e) => setFormData({ ...formData, target_amount: parseFloat(e.target.value) || 0 })}
                        placeholder="0"
                        step="0.01"
                        min="0.01"
                        className={getFieldError("target_amount") ? "error" : ""}
                        required
                      />
                    </div>
                    {getFieldError("target_amount") && (
                      <span className="field-error">{getFieldError("target_amount")}</span>
                    )}
                  </div>

                  {/* Deadline */}
                  <div className="form-group">
                    <label htmlFor="deadline">
                      Deadline *
                    </label>
                    <input
                      type="date"
                      id="deadline"
                      value={formatDateForInput(formData.deadline || "")}
                      onChange={(e) => setFormData({ ...formData, deadline: e.target.value })}
                      min={new Date().toISOString().split('T')[0]}
                      className={getFieldError("deadline") ? "error" : ""}
                      required
                    />
                    {getFieldError("deadline") && (
                      <span className="field-error">{getFieldError("deadline")}</span>
                    )}
                    {formData.deadline && suggestedMonthlyContribution > 0 && (
                      <small className="field-hint success">
                        💡 To reach this goal, save{" "}
                        <strong>
                          {new Intl.NumberFormat(locale, {
                            style: 'currency',
                            currency: defaultCurrency,
                            minimumFractionDigits: 0
                          }).format(suggestedMonthlyContribution)}
                        </strong>
                        {" "}per month
                      </small>
                    )}
                  </div>

                  {/* Frequency */}
                  <div className="form-group">
                    <label htmlFor="frequency">
                      Frequency
                    </label>
                    <select
                      id="frequency"
                      value={formData.frequency}
                      onChange={(e) => setFormData({ ...formData, frequency: e.target.value as GoalFrequency })}
                    >
                      <option value="one_time">One-Time Goal</option>
                      <option value="daily">Daily</option>
                      <option value="weekly">Weekly</option>
                      <option value="monthly">Monthly</option>
                      <option value="quarterly">Quarterly</option>
                      <option value="yearly">Yearly</option>
                    </select>
                    <small className="field-hint">
                      For recurring goals, this will reset automatically
                    </small>
                  </div>

                  {/* Description */}
                  <div className="form-group">
                    <label htmlFor="description">
                      Description (Optional)
                    </label>
                    <textarea
                      id="description"
                      value={formData.description}
                      onChange={(e) => setFormData({ ...formData, description: e.target.value })}
                      placeholder="Add notes about this goal..."
                      rows={3}
                    />
                  </div>
                </div>
              )}

              {/* Advanced Tab */}
              {activeTab === "advanced" && showAdvanced && (
                <div className="tab-content">
                  {/* Category */}
                  <div className="form-group">
                    <label htmlFor="category">
                      Category
                    </label>
                    <input
                      type="text"
                      id="category"
                      value={formData.category}
                      onChange={(e) => setFormData({ ...formData, category: e.target.value })}
                      placeholder="e.g., emergency_fund, vacation, debt_payoff"
                    />
                    <small className="field-hint">
                      Used for grouping and filtering goals
                    </small>
                  </div>

                  {/* Icon */}
                  <div className="form-row">
                    <div className="form-group">
                      <label htmlFor="icon">
                        Icon (Emoji)
                      </label>
                      <input
                        type="text"
                        id="icon"
                        value={formData.icon}
                        onChange={(e) => setFormData({ ...formData, icon: e.target.value })}
                        placeholder="e.g., 🏦"
                        maxLength={2}
                        style={{ fontSize: '24px', textAlign: 'center' }}
                      />
                    </div>

                    {/* Color */}
                    <div className="form-group">
                      <label htmlFor="color">
                        Color
                      </label>
                      <input
                        type="color"
                        id="color"
                        value={formData.color}
                        onChange={(e) => setFormData({ ...formData, color: e.target.value })}
                      />
                    </div>
                  </div>

                  {/* Alert Settings */}
                  <div className="form-group">
                    <label className="checkbox-label">
                      <input
                        type="checkbox"
                        checked={formData.alert_enabled}
                        onChange={(e) => setFormData({ ...formData, alert_enabled: e.target.checked })}
                      />
                      Enable progress alerts
                    </label>
                  </div>

                  {formData.alert_enabled && (
                    <div className="form-group">
                      <label htmlFor="alert-threshold">
                        Alert Threshold (%)
                      </label>
                      <input
                        type="number"
                        id="alert-threshold"
                        value={formData.alert_threshold}
                        onChange={(e) => setFormData({ ...formData, alert_threshold: parseFloat(e.target.value) || 0 })}
                        min="0"
                        max="100"
                        step="5"
                        className={getFieldError("alert_threshold") ? "error" : ""}
                      />
                      {getFieldError("alert_threshold") && (
                        <span className="field-error">{getFieldError("alert_threshold")}</span>
                      )}
                      <small className="field-hint">
                        You'll be notified when you reach this percentage
                      </small>
                    </div>
                  )}
                </div>
              )}

              {/* Preview Tab */}
              {activeTab === "preview" && showPreview && (
                <div className="tab-content preview-content">
                  <h3>Preview</h3>
                  <div className="preview-description">
                    This is how your goal will appear in the dashboard
                  </div>

                  {/* Preview card (simplified version) */}
                  <div className="preview-card">
                    <div className="preview-header">
                      {formData.icon && <span className="preview-icon">{formData.icon}</span>}
                      <div className="preview-title">
                        <div className="preview-name">{formData.name || "Goal Name"}</div>
                        {formData.category && (
                          <div className="preview-category">{formData.category}</div>
                        )}
                      </div>
                    </div>

                    <div className="preview-progress-bar" style={{ backgroundColor: '#e5e7eb' }}>
                      <div
                        className="preview-progress-fill"
                        style={{
                          width: '0%',
                          backgroundColor: formData.color || '#3b82f6'
                        }}
                      />
                    </div>

                    <div className="preview-stats">
                      <div className="preview-stat">
                        <div className="preview-stat-label">Target</div>
                        <div className="preview-stat-value">
                          {formData.goal_type === "custom" ? "#" : "$"}
                          {formData.target_amount?.toLocaleString() || "0"}
                        </div>
                      </div>
                      <div className="preview-stat">
                        <div className="preview-stat-label">Deadline</div>
                        <div className="preview-stat-value">
                          {formData.deadline
                            ? new Date(formData.deadline).toLocaleDateString(locale, {
                                month: 'short',
                                day: 'numeric',
                                year: 'numeric'
                              })
                            : "Not set"}
                        </div>
                      </div>
                    </div>
                  </div>

                  {formData.description && (
                    <div className="preview-description-box">
                      <strong>Description:</strong> {formData.description}
                    </div>
                  )}
                </div>
              )}

              {/* Form Actions */}
              <div className="form-actions">
                {selectedTemplate && mode === "create" && (
                  <button
                    type="button"
                    className="back-button"
                    onClick={() => setSelectedTemplate(null)}
                  >
                    ← Back to Templates
                  </button>
                )}
                <button
                  type="button"
                  className="cancel-button"
                  onClick={onCancel}
                  disabled={submitting}
                >
                  Cancel
                </button>
                <button
                  type="submit"
                  className="submit-button"
                  disabled={submitting || loading}
                >
                  {submitting ? (
                    <>
                      <span className="spinner"></span>
                      Saving...
                    </>
                  ) : (
                    mode === "create" ? "Create Goal" : "Save Changes"
                  )}
                </button>
              </div>
            </form>
          </>
        )}
      </div>
    </div>
  );
};
```

---

## Styling

```css
.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.6);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
  padding: 20px;
}

.modal-dialog {
  background: var(--dialog-bg);
  border-radius: 16px;
  width: 100%;
  max-width: 600px;
  max-height: 90vh;
  display: flex;
  flex-direction: column;
  box-shadow: 0 20px 25px -5px rgba(0, 0, 0, 0.1), 0 10px 10px -5px rgba(0, 0, 0, 0.04);
}

.goal-config-dialog {
  max-width: 700px;
}

/* Header */
.dialog-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 24px 24px 16px 24px;
  border-bottom: 1px solid var(--divider-color);
}

.dialog-header h2 {
  font-size: 24px;
  font-weight: 600;
  color: var(--heading-color);
  margin: 0;
}

.close-button {
  background: none;
  border: none;
  font-size: 32px;
  color: var(--secondary-color);
  cursor: pointer;
  line-height: 1;
  padding: 0;
  width: 32px;
  height: 32px;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: color 0.2s;
}

.close-button:hover {
  color: var(--text-color);
}

/* Templates Section */
.templates-section {
  padding: 24px;
  overflow-y: auto;
}

.templates-section h3 {
  font-size: 18px;
  font-weight: 600;
  color: var(--heading-color);
  margin: 0 0 16px 0;
}

.template-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(150px, 1fr));
  gap: 12px;
}

.template-card {
  background: var(--card-bg);
  border: 2px solid var(--card-border);
  border-radius: 12px;
  padding: 16px;
  cursor: pointer;
  transition: all 0.2s;
  display: flex;
  flex-direction: column;
  align-items: center;
  text-align: center;
  gap: 8px;
}

.template-card:hover {
  border-color: var(--primary-color);
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
  transform: translateY(-2px);
}

.template-card.custom {
  border-style: dashed;
}

.template-icon {
  font-size: 36px;
}

.template-name {
  font-size: 14px;
  font-weight: 600;
  color: var(--text-color);
}

.template-description {
  font-size: 12px;
  color: var(--secondary-color);
  line-height: 1.4;
}

/* Tabs */
.dialog-tabs {
  display: flex;
  gap: 4px;
  padding: 0 24px;
  border-bottom: 1px solid var(--divider-color);
}

.tab {
  padding: 12px 20px;
  background: none;
  border: none;
  border-bottom: 2px solid transparent;
  color: var(--secondary-color);
  font-size: 14px;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.2s;
}

.tab:hover {
  color: var(--text-color);
}

.tab.active {
  color: var(--primary-color);
  border-bottom-color: var(--primary-color);
}

/* Error Banner */
.error-banner {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 12px 24px;
  background: var(--error-bg);
  border-bottom: 1px solid var(--error-border);
}

.error-icon {
  font-size: 18px;
}

.error-text {
  font-size: 14px;
  color: var(--error-color);
  font-weight: 500;
}

/* Form */
.dialog-form {
  display: flex;
  flex-direction: column;
  flex: 1;
  overflow: hidden;
}

.tab-content {
  padding: 24px;
  overflow-y: auto;
  flex: 1;
}

.form-group {
  display: flex;
  flex-direction: column;
  gap: 6px;
  margin-bottom: 20px;
}

.form-group label {
  font-size: 14px;
  font-weight: 600;
  color: var(--label-color);
}

.checkbox-label {
  flex-direction: row;
  align-items: center;
  gap: 8px;
  cursor: pointer;
  font-weight: 500;
}

.checkbox-label input[type="checkbox"] {
  width: 18px;
  height: 18px;
  cursor: pointer;
}

.form-group input[type="text"],
.form-group input[type="number"],
.form-group input[type="date"],
.form-group select,
.form-group textarea {
  padding: 10px 12px;
  border: 1px solid var(--input-border);
  border-radius: 8px;
  font-size: 14px;
  background: var(--input-bg);
  color: var(--text-color);
  transition: all 0.2s;
}

.form-group input:focus,
.form-group select:focus,
.form-group textarea:focus {
  outline: none;
  border-color: var(--primary-color);
  box-shadow: 0 0 0 3px var(--focus-ring);
}

.form-group input.error,
.form-group select.error {
  border-color: var(--error-color);
}

.input-with-prefix {
  position: relative;
  display: flex;
  align-items: center;
}

.input-prefix {
  position: absolute;
  left: 12px;
  font-size: 14px;
  font-weight: 600;
  color: var(--secondary-color);
  pointer-events: none;
}

.input-with-prefix input {
  padding-left: 28px;
}

.field-hint {
  font-size: 12px;
  color: var(--secondary-color);
  line-height: 1.4;
}

.field-hint.success {
  color: var(--success-color);
}

.field-error {
  font-size: 12px;
  color: var(--error-color);
  font-weight: 500;
}

.form-row {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 16px;
}

/* Preview */
.preview-content {
  display: flex;
  flex-direction: column;
  gap: 16px;
}

.preview-content h3 {
  font-size: 18px;
  font-weight: 600;
  color: var(--heading-color);
  margin: 0;
}

.preview-description {
  font-size: 13px;
  color: var(--secondary-color);
}

.preview-card {
  background: var(--card-bg);
  border: 1px solid var(--card-border);
  border-radius: 12px;
  padding: 20px;
}

.preview-header {
  display: flex;
  align-items: flex-start;
  gap: 12px;
  margin-bottom: 16px;
}

.preview-icon {
  font-size: 32px;
}

.preview-title {
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.preview-name {
  font-size: 18px;
  font-weight: 600;
  color: var(--text-color);
}

.preview-category {
  font-size: 12px;
  color: var(--secondary-color);
  text-transform: uppercase;
}

.preview-progress-bar {
  width: 100%;
  height: 24px;
  border-radius: 12px;
  overflow: hidden;
  margin-bottom: 12px;
}

.preview-progress-fill {
  height: 100%;
  transition: width 0.3s ease;
}

.preview-stats {
  display: flex;
  justify-content: space-between;
  gap: 16px;
}

.preview-stat {
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.preview-stat-label {
  font-size: 11px;
  color: var(--secondary-color);
  text-transform: uppercase;
  font-weight: 500;
}

.preview-stat-value {
  font-size: 16px;
  font-weight: 700;
  color: var(--text-color);
}

.preview-description-box {
  padding: 12px;
  background: var(--info-bg);
  border-radius: 8px;
  font-size: 13px;
  color: var(--text-color);
  line-height: 1.5;
}

/* Form Actions */
.form-actions {
  display: flex;
  gap: 12px;
  justify-content: flex-end;
  padding: 16px 24px;
  border-top: 1px solid var(--divider-color);
}

.back-button,
.cancel-button {
  padding: 10px 20px;
  background: transparent;
  border: 1px solid var(--input-border);
  border-radius: 8px;
  font-size: 14px;
  font-weight: 600;
  cursor: pointer;
  color: var(--text-color);
  transition: all 0.2s;
}

.back-button:hover,
.cancel-button:hover {
  background: var(--secondary-button-hover-bg);
}

.submit-button {
  padding: 10px 24px;
  background: var(--primary-color);
  border: none;
  border-radius: 8px;
  font-size: 14px;
  font-weight: 600;
  color: white;
  cursor: pointer;
  transition: all 0.2s;
  display: flex;
  align-items: center;
  gap: 8px;
}

.submit-button:hover:not(:disabled) {
  background: var(--primary-color-hover);
  transform: translateY(-1px);
  box-shadow: 0 4px 12px rgba(59, 130, 246, 0.3);
}

.submit-button:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

.spinner {
  width: 14px;
  height: 14px;
  border: 2px solid rgba(255, 255, 255, 0.3);
  border-top-color: white;
  border-radius: 50%;
  animation: spin 0.6s linear infinite;
}

@keyframes spin {
  to { transform: rotate(360deg); }
}

/* Theme: Light */
.theme-light {
  --dialog-bg: #ffffff;
  --heading-color: #111827;
  --text-color: #111827;
  --secondary-color: #6b7280;
  --label-color: #374151;
  --primary-color: #3b82f6;
  --primary-color-hover: #2563eb;
  --card-bg: #ffffff;
  --card-border: #e5e7eb;
  --divider-color: #e5e7eb;
  --input-bg: #ffffff;
  --input-border: #d1d5db;
  --focus-ring: rgba(59, 130, 246, 0.1);
  --error-color: #ef4444;
  --error-bg: #fee2e2;
  --error-border: #fecaca;
  --success-color: #10b981;
  --info-bg: #f0f9ff;
  --secondary-button-hover-bg: #f3f4f6;
}

/* Theme: Dark */
.theme-dark {
  --dialog-bg: #1f2937;
  --heading-color: #f3f4f6;
  --text-color: #f3f4f6;
  --secondary-color: #9ca3af;
  --label-color: #d1d5db;
  --primary-color: #3b82f6;
  --primary-color-hover: #2563eb;
  --card-bg: #374151;
  --card-border: #4b5563;
  --divider-color: #374151;
  --input-bg: #374151;
  --input-border: #4b5563;
  --focus-ring: rgba(59, 130, 246, 0.2);
  --error-color: #f87171;
  --error-bg: #7f1d1d;
  --error-border: #991b1b;
  --success-color: #34d399;
  --info-bg: #1e3a8a;
  --secondary-button-hover-bg: #4b5563;
}

/* Responsive */
@media (max-width: 768px) {
  .modal-dialog {
    max-width: 100%;
    max-height: 100vh;
    border-radius: 0;
  }

  .template-grid {
    grid-template-columns: 1fr 1fr;
  }

  .form-row {
    grid-template-columns: 1fr;
  }

  .form-actions {
    flex-wrap: wrap;
  }

  .back-button {
    order: 1;
    width: 100%;
  }

  .cancel-button {
    order: 2;
    flex: 1;
  }

  .submit-button {
    order: 3;
    flex: 1;
  }
}
```

---

## Multi-Domain Applicability

### Finance Domain: Create Savings Goal
```tsx
<GoalConfigDialog
  mode="create"
  onSave={async (goal) => {
    await createGoal(goal);
    refreshGoals();
  }}
  onCancel={() => setShowDialog(false)}
  defaultCurrency="USD"
  showTemplates={true}
  showAdvanced={true}
  showPreview={true}
/>
// Templates: Emergency Fund, Vacation, Down Payment, Retirement
// Shows suggested monthly contribution based on deadline
```

### Healthcare Domain: Set Budget Target
```tsx
<GoalConfigDialog
  mode="create"
  onSave={async (goal) => {
    await createHealthBudget(goal);
  }}
  onCancel={() => closeDialog()}
  defaultCurrency="USD"
  templates={[
    {
      template_id: "annual_deductible",
      name: "Annual Deductible",
      description: "Meet your insurance deductible",
      icon: "🏥",
      goal_type: "expense",
      suggested_amount: 3000,
      suggested_deadline_months: 12,
      category: "deductible",
      color: "#f59e0b"
    },
    {
      template_id: "hsa_contribution",
      name: "HSA Max Contribution",
      description: "Max out HSA for tax benefits",
      icon: "💊",
      goal_type: "savings",
      suggested_amount: 7750,
      suggested_deadline_months: 12,
      category: "hsa",
      color: "#10b981"
    }
  ]}
/>
// Healthcare-specific templates and categories
```

### Legal Domain: Configure Billable Hours Goal
```tsx
<GoalConfigDialog
  mode="edit"
  initialGoal={{
    goal_id: "g_003",
    name: "Q4 Billable Hours",
    goal_type: "custom",
    target_amount: 500,
    deadline: "2025-12-31",
    frequency: "quarterly",
    category: "billable_hours",
    icon: "⚖️",
    color: "#3b82f6",
    description: "Meet quarterly billable hours target"
  }}
  onSave={async (goal) => {
    await updateGoal(goal);
  }}
  onCancel={() => closeDialog()}
  showTemplates={false}
/>
// Edit existing goal, no templates shown
// Custom goal type for non-financial metrics
```

### Research Domain: Grant Spending Goal
```tsx
<GoalConfigDialog
  mode="create"
  onSave={async (goal) => {
    await createGrantGoal(goal);
  }}
  onCancel={() => closeDialog()}
  defaultCurrency="USD"
  templates={[
    {
      template_id: "grant_spending",
      name: "Grant Budget Spending",
      description: "Spend grant funds before deadline",
      icon: "🎓",
      goal_type: "expense",
      suggested_amount: 250000,
      suggested_deadline_months: 24,
      category: "grant_spending",
      color: "#8b5cf6"
    }
  ]}
/>
// Research-specific templates
// Large amounts for grant budgets
```

### Manufacturing Domain: Production Target Goal
```tsx
<GoalConfigDialog
  mode="create"
  onSave={async (goal) => {
    await createProductionGoal(goal);
  }}
  onCancel={() => closeDialog()}
  templates={[
    {
      template_id: "monthly_production",
      name: "Monthly Production Target",
      description: "Units to produce this month",
      icon: "🏭",
      goal_type: "custom",
      suggested_amount: 10000,
      suggested_deadline_months: 1,
      category: "production_units",
      color: "#06b6d4"
    }
  ]}
  showTemplates={true}
/>
// Custom goal type for units produced
// Short deadline (monthly recurring)
```

### Media Domain: Subscriber Goal
```tsx
<GoalConfigDialog
  mode="create"
  onSave={async (goal) => {
    await createSubscriberGoal(goal);
  }}
  onCancel={() => closeDialog()}
  templates={[
    {
      template_id: "100k_subs",
      name: "100K Subscribers",
      description: "Reach 100,000 subscribers",
      icon: "📺",
      goal_type: "custom",
      suggested_amount: 100000,
      suggested_deadline_months: 12,
      category: "subscribers",
      color: "#ec4899"
    }
  ]}
/>
// Custom goal type for subscriber count
// Growth-focused templates
```

---

## Features

### 1. Preset Templates
```typescript
// 5 default templates (emergency fund, vacation, debt payoff, etc.)
// Custom templates per domain
// One-click template selection
// Auto-fills name, icon, color, suggested amount/deadline
```

### 2. Real-Time Validation
```typescript
// Inline error messages
// Prevents submission with invalid data
// Required fields marked with *
// Smart validation (deadline must be future, amount > 0)
```

### 3. Smart Suggestions
```typescript
// Calculates suggested monthly contribution
// Based on target amount and deadline
// Updates in real-time as values change
// Helps set realistic goals
```

### 4. Tabbed Interface
```typescript
// Basic Info: Required fields
// Advanced: Optional customization (icon, color, alerts)
// Preview: See how goal will look before saving
```

### 5. Live Preview
```typescript
// Shows goal card as it will appear
// Updates in real-time as fields change
// Helps visualize final result
```

### 6. Recurring Goals
```typescript
// Daily, weekly, monthly, quarterly, yearly
// Auto-resets when period ends
// Useful for ongoing targets (monthly revenue, weekly hours)
```

---

## Accessibility

```tsx
<div
  className="modal-dialog"
  role="dialog"
  aria-labelledby="dialog-title"
  aria-modal="true"
>
  <h2 id="dialog-title">
    {mode === "create" ? "Create New Goal" : "Edit Goal"}
  </h2>

  <form onSubmit={handleSubmit} aria-label="Goal configuration form">
    <label htmlFor="goal-name">
      Goal Name *
    </label>
    <input
      type="text"
      id="goal-name"
      aria-required="true"
      aria-invalid={!!getFieldError("name")}
      aria-describedby={getFieldError("name") ? "name-error" : undefined}
    />
    {getFieldError("name") && (
      <span id="name-error" className="field-error" role="alert">
        {getFieldError("name")}
      </span>
    )}

    <button type="submit" aria-label="Save goal">
      {mode === "create" ? "Create Goal" : "Save Changes"}
    </button>
  </form>
</div>
```

---

## Related Components

**Uses:**
- GoalProgressCard (for preview)

**Used by:**
- GoalDashboard (create/edit goals)
- SettingsPage (manage goals)
- BudgetPlanner (set budget goals)

**Similar patterns:**
- AccountCreateModal (create accounts)
- TransactionEditDialog (edit transactions)
- SourceConfigDialog (configure data sources)
