# ForecastChart (IL Component)

## Definition

**ForecastChart** is an interactive visualization component for displaying historical data alongside projected forecasts. It renders line/area/bar charts with confidence intervals, zoom/pan controls, and detailed tooltips to help users understand trends and predictions across different time horizons.

**Problem it solves:**
- No standardized way to visualize forecasts across different domains
- Difficulty showing confidence intervals and prediction uncertainty
- No consistent pattern for comparing actuals vs projections
- Hard to explore different time ranges in forecast data
- Lack of visual feedback for forecast accuracy
- No unified approach for multi-algorithm forecast comparison

**Solution:**
- Single component for all forecast visualizations
- Built-in confidence interval shading (80%, 90%, 95%)
- Multiple chart types (line, area, bar, combo)
- Interactive zoom/pan for time exploration
- Hover tooltips with detailed forecast metadata
- Legend toggle for comparing multiple forecasts
- Algorithm comparison mode (side-by-side)
- Responsive design (desktop full-featured, mobile simplified)

---

## Interface Contract

```typescript
interface ForecastChartProps {
  // Data
  data: ForecastDataPoint[];  // Historical + projected data
  algorithm: ForecastAlgorithm;  // Forecasting method used
  confidenceInterval?: ConfidenceLevel;  // 80, 90, or 95 (default: 90)

  // Display options
  chartType?: ChartType;  // "line" | "area" | "bar" | "combo" (default: "line")
  showConfidenceInterval?: boolean;  // Show shaded confidence bands (default: true)
  showDataPoints?: boolean;  // Show individual data points (default: true)
  showGrid?: boolean;  // Show grid lines (default: true)
  showLegend?: boolean;  // Show legend (default: true)

  // Interaction
  onHover?: (point: ForecastDataPoint | null) => void;  // Hover over data point
  onZoom?: (range: DateRange) => void;  // Zoom to date range
  onAlgorithmChange?: (algorithm: ForecastAlgorithm) => void;  // Switch algorithm

  // Customization
  height?: number;  // Chart height in pixels (default: 400)
  theme?: "light" | "dark";
  locale?: string;  // For date/number formatting (default: "en-US")
  currency?: string;  // For financial forecasts (e.g., "USD")

  // States
  loading?: boolean;
  error?: string | null;

  // Advanced
  compareAlgorithms?: ForecastAlgorithm[];  // Compare multiple algorithms
  highlightAnomalies?: boolean;  // Highlight outliers in historical data
  showSeasonality?: boolean;  // Overlay seasonal patterns
}

interface ForecastDataPoint {
  date: string;  // ISO 8601 date
  actual?: number;  // Historical value (if available)
  forecast?: number;  // Predicted value
  lower_bound?: number;  // Lower confidence bound
  upper_bound?: number;  // Upper confidence bound
  is_historical: boolean;  // True for historical data, false for forecast

  // Optional metadata
  seasonality_factor?: number;  // Seasonal adjustment
  trend_component?: number;  // Trend component
  anomaly_score?: number;  // Outlier detection score (0-1)
}

type ChartType = "line" | "area" | "bar" | "combo";

type ConfidenceLevel = 80 | 90 | 95;

type ForecastAlgorithm =
  | "linear_regression"      // Simple linear trend
  | "exponential_smoothing"  // Holt-Winters method
  | "arima"                  // ARIMA model
  | "prophet"                // Facebook Prophet
  | "moving_average"         // Simple moving average
  | "custom";                // Custom algorithm

interface DateRange {
  start: string;  // ISO 8601 date
  end: string;    // ISO 8601 date
}
```

---

## Component Structure

```tsx
import React, { useState, useMemo, useRef, useEffect } from 'react';
import {
  LineChart, Line, AreaChart, Area, BarChart, Bar, ComposedChart,
  XAxis, YAxis, CartesianGrid, Tooltip, Legend, ResponsiveContainer,
  ReferenceArea, ReferenceLine, Brush
} from 'recharts';

export const ForecastChart: React.FC<ForecastChartProps> = ({
  data,
  algorithm,
  confidenceInterval = 90,
  chartType = "line",
  showConfidenceInterval = true,
  showDataPoints = true,
  showGrid = true,
  showLegend = true,
  onHover,
  onZoom,
  onAlgorithmChange,
  height = 400,
  theme = "light",
  locale = "en-US",
  currency,
  loading = false,
  error = null,
  compareAlgorithms = [],
  highlightAnomalies = false,
  showSeasonality = false
}) => {
  const [zoomRange, setZoomRange] = useState<DateRange | null>(null);
  const [hoveredPoint, setHoveredPoint] = useState<ForecastDataPoint | null>(null);
  const [selectedAlgorithm, setSelectedAlgorithm] = useState<ForecastAlgorithm>(algorithm);
  const [showTooltip, setShowTooltip] = useState(false);
  const chartRef = useRef<HTMLDivElement>(null);

  // Filter data by zoom range
  const visibleData = useMemo(() => {
    if (!zoomRange) return data;

    return data.filter(point => {
      const pointDate = new Date(point.date);
      const start = new Date(zoomRange.start);
      const end = new Date(zoomRange.end);
      return pointDate >= start && pointDate <= end;
    });
  }, [data, zoomRange]);

  // Calculate chart statistics
  const stats = useMemo(() => {
    const historicalData = data.filter(p => p.is_historical && p.actual !== undefined);
    const forecastData = data.filter(p => !p.is_historical && p.forecast !== undefined);

    const historicalMean = historicalData.length > 0
      ? historicalData.reduce((sum, p) => sum + (p.actual || 0), 0) / historicalData.length
      : 0;

    const forecastMean = forecastData.length > 0
      ? forecastData.reduce((sum, p) => sum + (p.forecast || 0), 0) / forecastData.length
      : 0;

    const allValues = data
      .flatMap(p => [p.actual, p.forecast, p.lower_bound, p.upper_bound])
      .filter((v): v is number => v !== undefined);

    const minValue = Math.min(...allValues);
    const maxValue = Math.max(...allValues);

    return {
      historicalCount: historicalData.length,
      forecastCount: forecastData.length,
      historicalMean,
      forecastMean,
      minValue,
      maxValue,
      range: maxValue - minValue
    };
  }, [data]);

  // Format value based on domain (currency, number, etc.)
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

  // Format date for axis labels
  const formatDate = (dateString: string): string => {
    const date = new Date(dateString);
    return new Intl.DateTimeFormat(locale, {
      month: 'short',
      year: 'numeric'
    }).format(date);
  };

  // Custom tooltip component
  const CustomTooltip = ({ active, payload }: any) => {
    if (!active || !payload || payload.length === 0) return null;

    const point = payload[0].payload as ForecastDataPoint;

    return (
      <div className={`forecast-tooltip theme-${theme}`}>
        <div className="tooltip-header">
          <div className="tooltip-date">{formatDate(point.date)}</div>
          {point.is_historical ? (
            <span className="tooltip-badge historical">Historical</span>
          ) : (
            <span className="tooltip-badge forecast">Forecast</span>
          )}
        </div>

        <div className="tooltip-content">
          {point.actual !== undefined && (
            <div className="tooltip-row">
              <span className="tooltip-label">Actual:</span>
              <span className="tooltip-value actual">{formatValue(point.actual)}</span>
            </div>
          )}

          {point.forecast !== undefined && (
            <div className="tooltip-row">
              <span className="tooltip-label">Forecast:</span>
              <span className="tooltip-value forecast">{formatValue(point.forecast)}</span>
            </div>
          )}

          {showConfidenceInterval && point.lower_bound !== undefined && point.upper_bound !== undefined && (
            <div className="tooltip-row">
              <span className="tooltip-label">CI ({confidenceInterval}%):</span>
              <span className="tooltip-value range">
                {formatValue(point.lower_bound)} - {formatValue(point.upper_bound)}
              </span>
            </div>
          )}

          {highlightAnomalies && point.anomaly_score !== undefined && point.anomaly_score > 0.7 && (
            <div className="tooltip-row anomaly">
              <span className="tooltip-label">⚠️ Anomaly:</span>
              <span className="tooltip-value">{(point.anomaly_score * 100).toFixed(0)}%</span>
            </div>
          )}

          {showSeasonality && point.seasonality_factor !== undefined && (
            <div className="tooltip-row">
              <span className="tooltip-label">Seasonality:</span>
              <span className="tooltip-value">{(point.seasonality_factor * 100).toFixed(1)}%</span>
            </div>
          )}
        </div>
      </div>
    );
  };

  // Handle mouse move for hover
  const handleMouseMove = (state: any) => {
    if (state && state.activePayload && state.activePayload.length > 0) {
      const point = state.activePayload[0].payload as ForecastDataPoint;
      setHoveredPoint(point);
      setShowTooltip(true);
      onHover?.(point);
    } else {
      setHoveredPoint(null);
      setShowTooltip(false);
      onHover?.(null);
    }
  };

  // Handle zoom selection
  const handleZoom = (range: DateRange) => {
    setZoomRange(range);
    onZoom?.(range);
  };

  // Reset zoom
  const resetZoom = () => {
    setZoomRange(null);
    onZoom?.({ start: data[0].date, end: data[data.length - 1].date });
  };

  // Handle algorithm change
  const handleAlgorithmChange = (newAlgorithm: ForecastAlgorithm) => {
    setSelectedAlgorithm(newAlgorithm);
    onAlgorithmChange?.(newAlgorithm);
  };

  // Loading state
  if (loading) {
    return (
      <div className={`forecast-chart theme-${theme} loading`} style={{ height }}>
        <div className="loading-skeleton">
          <div className="skeleton-header"></div>
          <div className="skeleton-chart"></div>
          <div className="skeleton-legend"></div>
        </div>
      </div>
    );
  }

  // Error state
  if (error) {
    return (
      <div className={`forecast-chart theme-${theme} error`} style={{ height }}>
        <div className="error-container">
          <div className="error-icon">⚠️</div>
          <div className="error-message">{error}</div>
          <button className="retry-button" onClick={() => window.location.reload()}>
            Retry
          </button>
        </div>
      </div>
    );
  }

  // Empty state
  if (data.length === 0) {
    return (
      <div className={`forecast-chart theme-${theme} empty`} style={{ height }}>
        <div className="empty-container">
          <div className="empty-icon">📊</div>
          <div className="empty-message">No forecast data available</div>
          <div className="empty-hint">Add historical data to generate forecasts</div>
        </div>
      </div>
    );
  }

  // Determine chart component based on type
  const ChartComponent =
    chartType === "line" ? LineChart :
    chartType === "area" ? AreaChart :
    chartType === "bar" ? BarChart :
    ComposedChart;

  // Find the boundary between historical and forecast
  const forecastStartIndex = data.findIndex(p => !p.is_historical);
  const forecastStartDate = forecastStartIndex >= 0 ? data[forecastStartIndex].date : null;

  return (
    <div className={`forecast-chart theme-${theme}`} ref={chartRef}>
      {/* Header */}
      <div className="chart-header">
        <div className="chart-title">
          <h3>Forecast Analysis</h3>
          <div className="chart-subtitle">
            {stats.historicalCount} historical points · {stats.forecastCount} forecast points
          </div>
        </div>

        <div className="chart-controls">
          {/* Algorithm selector */}
          {compareAlgorithms.length > 0 && (
            <select
              className="algorithm-selector"
              value={selectedAlgorithm}
              onChange={(e) => handleAlgorithmChange(e.target.value as ForecastAlgorithm)}
            >
              <option value={algorithm}>{formatAlgorithmName(algorithm)}</option>
              {compareAlgorithms.map(alg => (
                <option key={alg} value={alg}>{formatAlgorithmName(alg)}</option>
              ))}
            </select>
          )}

          {/* Zoom controls */}
          {zoomRange && (
            <button className="zoom-reset-button" onClick={resetZoom}>
              Reset Zoom
            </button>
          )}

          {/* Chart type toggle */}
          <div className="chart-type-toggle">
            <button
              className={chartType === "line" ? "active" : ""}
              onClick={() => chartType !== "line" && handleAlgorithmChange(algorithm)}
              title="Line chart"
            >
              📈
            </button>
            <button
              className={chartType === "area" ? "active" : ""}
              onClick={() => chartType !== "area" && handleAlgorithmChange(algorithm)}
              title="Area chart"
            >
              📊
            </button>
            <button
              className={chartType === "bar" ? "active" : ""}
              onClick={() => chartType !== "bar" && handleAlgorithmChange(algorithm)}
              title="Bar chart"
            >
              📊
            </button>
          </div>
        </div>
      </div>

      {/* Chart */}
      <ResponsiveContainer width="100%" height={height}>
        <ChartComponent
          data={visibleData}
          margin={{ top: 20, right: 30, left: 20, bottom: 20 }}
          onMouseMove={handleMouseMove}
          onMouseLeave={() => handleMouseMove(null)}
        >
          {showGrid && (
            <CartesianGrid strokeDasharray="3 3" stroke={theme === "dark" ? "#374151" : "#e5e7eb"} />
          )}

          <XAxis
            dataKey="date"
            tickFormatter={formatDate}
            stroke={theme === "dark" ? "#9ca3af" : "#6b7280"}
            style={{ fontSize: 12 }}
          />

          <YAxis
            tickFormatter={formatValue}
            stroke={theme === "dark" ? "#9ca3af" : "#6b7280"}
            style={{ fontSize: 12 }}
          />

          <Tooltip content={<CustomTooltip />} />

          {showLegend && (
            <Legend
              wrapperStyle={{ fontSize: 12 }}
              iconType="line"
            />
          )}

          {/* Confidence interval shading */}
          {showConfidenceInterval && chartType !== "bar" && (
            <Area
              type="monotone"
              dataKey="upper_bound"
              stroke="none"
              fill="#93c5fd"
              fillOpacity={0.2}
              isAnimationActive={false}
            />
          )}

          {showConfidenceInterval && chartType !== "bar" && (
            <Area
              type="monotone"
              dataKey="lower_bound"
              stroke="none"
              fill="#93c5fd"
              fillOpacity={0.2}
              isAnimationActive={false}
            />
          )}

          {/* Historical data line */}
          {chartType === "line" && (
            <Line
              type="monotone"
              dataKey="actual"
              stroke="#10b981"
              strokeWidth={2}
              dot={showDataPoints ? { r: 3 } : false}
              name="Historical"
              connectNulls={false}
            />
          )}

          {chartType === "area" && (
            <Area
              type="monotone"
              dataKey="actual"
              stroke="#10b981"
              fill="#10b981"
              fillOpacity={0.3}
              name="Historical"
            />
          )}

          {chartType === "bar" && (
            <Bar dataKey="actual" fill="#10b981" name="Historical" />
          )}

          {/* Forecast data line */}
          {chartType === "line" && (
            <Line
              type="monotone"
              dataKey="forecast"
              stroke="#3b82f6"
              strokeWidth={2}
              strokeDasharray="5 5"
              dot={showDataPoints ? { r: 3 } : false}
              name="Forecast"
              connectNulls={false}
            />
          )}

          {chartType === "area" && (
            <Area
              type="monotone"
              dataKey="forecast"
              stroke="#3b82f6"
              fill="#3b82f6"
              fillOpacity={0.3}
              strokeDasharray="5 5"
              name="Forecast"
            />
          )}

          {chartType === "bar" && (
            <Bar dataKey="forecast" fill="#3b82f6" name="Forecast" />
          )}

          {/* Reference line at forecast start */}
          {forecastStartDate && (
            <ReferenceLine
              x={forecastStartDate}
              stroke={theme === "dark" ? "#f59e0b" : "#f59e0b"}
              strokeDasharray="3 3"
              label={{ value: "Forecast Start", position: "top" }}
            />
          )}

          {/* Brush for zoom/pan */}
          <Brush
            dataKey="date"
            height={30}
            stroke={theme === "dark" ? "#4b5563" : "#d1d5db"}
            tickFormatter={formatDate}
          />
        </ChartComponent>
      </ResponsiveContainer>

      {/* Footer stats */}
      <div className="chart-footer">
        <div className="stat-card">
          <div className="stat-label">Historical Avg</div>
          <div className="stat-value">{formatValue(stats.historicalMean)}</div>
        </div>
        <div className="stat-card">
          <div className="stat-label">Forecast Avg</div>
          <div className="stat-value">{formatValue(stats.forecastMean)}</div>
        </div>
        <div className="stat-card">
          <div className="stat-label">Change</div>
          <div className={`stat-value ${stats.forecastMean >= stats.historicalMean ? 'positive' : 'negative'}`}>
            {((stats.forecastMean - stats.historicalMean) / stats.historicalMean * 100).toFixed(1)}%
          </div>
        </div>
        <div className="stat-card">
          <div className="stat-label">Algorithm</div>
          <div className="stat-value small">{formatAlgorithmName(selectedAlgorithm)}</div>
        </div>
      </div>
    </div>
  );
};

// Helper functions
function formatAlgorithmName(algorithm: ForecastAlgorithm): string {
  const names: Record<ForecastAlgorithm, string> = {
    linear_regression: "Linear Regression",
    exponential_smoothing: "Exponential Smoothing",
    arima: "ARIMA",
    prophet: "Prophet",
    moving_average: "Moving Average",
    custom: "Custom"
  };
  return names[algorithm] || algorithm;
}
```

---

## Styling

```css
.forecast-chart {
  background: var(--chart-bg);
  border: 1px solid var(--chart-border);
  border-radius: 12px;
  padding: 20px;
  width: 100%;
}

.chart-header {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  margin-bottom: 20px;
}

.chart-title h3 {
  font-size: 18px;
  font-weight: 600;
  color: var(--heading-color);
  margin: 0 0 4px 0;
}

.chart-subtitle {
  font-size: 13px;
  color: var(--secondary-color);
}

.chart-controls {
  display: flex;
  align-items: center;
  gap: 12px;
}

.algorithm-selector {
  padding: 6px 12px;
  border: 1px solid var(--input-border);
  border-radius: 6px;
  background: var(--input-bg);
  color: var(--text-color);
  font-size: 13px;
  cursor: pointer;
}

.zoom-reset-button {
  padding: 6px 12px;
  background: var(--secondary-button-bg);
  border: 1px solid var(--input-border);
  border-radius: 6px;
  font-size: 13px;
  cursor: pointer;
  color: var(--text-color);
  transition: all 0.2s;
}

.zoom-reset-button:hover {
  background: var(--secondary-button-hover-bg);
}

.chart-type-toggle {
  display: flex;
  gap: 4px;
  background: var(--toggle-bg);
  border-radius: 6px;
  padding: 4px;
}

.chart-type-toggle button {
  padding: 6px 12px;
  background: transparent;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-size: 16px;
  transition: all 0.2s;
}

.chart-type-toggle button:hover {
  background: var(--toggle-hover-bg);
}

.chart-type-toggle button.active {
  background: var(--primary-color);
}

/* Tooltip */
.forecast-tooltip {
  background: var(--tooltip-bg);
  border: 1px solid var(--tooltip-border);
  border-radius: 8px;
  padding: 12px;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
  min-width: 200px;
}

.tooltip-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 8px;
  padding-bottom: 8px;
  border-bottom: 1px solid var(--divider-color);
}

.tooltip-date {
  font-weight: 600;
  font-size: 14px;
  color: var(--text-color);
}

.tooltip-badge {
  font-size: 11px;
  padding: 2px 8px;
  border-radius: 4px;
  font-weight: 600;
  text-transform: uppercase;
}

.tooltip-badge.historical {
  background: var(--success-bg);
  color: var(--success-color);
}

.tooltip-badge.forecast {
  background: var(--info-bg);
  color: var(--info-color);
}

.tooltip-content {
  display: flex;
  flex-direction: column;
  gap: 6px;
}

.tooltip-row {
  display: flex;
  justify-content: space-between;
  align-items: center;
  font-size: 13px;
}

.tooltip-row.anomaly {
  padding: 4px;
  background: var(--warning-bg);
  border-radius: 4px;
  margin-top: 4px;
}

.tooltip-label {
  color: var(--secondary-color);
  font-weight: 500;
}

.tooltip-value {
  font-weight: 600;
  color: var(--text-color);
}

.tooltip-value.actual {
  color: var(--success-color);
}

.tooltip-value.forecast {
  color: var(--info-color);
}

.tooltip-value.range {
  font-size: 12px;
  color: var(--secondary-color);
}

/* Footer stats */
.chart-footer {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  gap: 16px;
  margin-top: 20px;
  padding-top: 20px;
  border-top: 1px solid var(--divider-color);
}

.stat-card {
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.stat-label {
  font-size: 12px;
  color: var(--secondary-color);
  text-transform: uppercase;
  font-weight: 500;
  letter-spacing: 0.5px;
}

.stat-value {
  font-size: 20px;
  font-weight: 700;
  color: var(--text-color);
}

.stat-value.small {
  font-size: 14px;
  font-weight: 600;
}

.stat-value.positive {
  color: var(--success-color);
}

.stat-value.negative {
  color: var(--error-color);
}

/* Loading state */
.forecast-chart.loading {
  display: flex;
  align-items: center;
  justify-content: center;
}

.loading-skeleton {
  width: 100%;
  display: flex;
  flex-direction: column;
  gap: 16px;
}

.skeleton-header,
.skeleton-chart,
.skeleton-legend {
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
  height: 40px;
  width: 60%;
}

.skeleton-chart {
  height: 300px;
  width: 100%;
}

.skeleton-legend {
  height: 30px;
  width: 40%;
}

@keyframes shimmer {
  0% { background-position: 200% 0; }
  100% { background-position: -200% 0; }
}

/* Error state */
.forecast-chart.error {
  display: flex;
  align-items: center;
  justify-content: center;
}

.error-container {
  text-align: center;
  padding: 40px;
}

.error-icon {
  font-size: 48px;
  margin-bottom: 16px;
}

.error-message {
  font-size: 16px;
  color: var(--error-color);
  margin-bottom: 20px;
}

.retry-button {
  padding: 10px 20px;
  background: var(--primary-color);
  color: white;
  border: none;
  border-radius: 6px;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s;
}

.retry-button:hover {
  background: var(--primary-color-hover);
}

/* Empty state */
.forecast-chart.empty {
  display: flex;
  align-items: center;
  justify-content: center;
}

.empty-container {
  text-align: center;
  padding: 60px 20px;
}

.empty-icon {
  font-size: 64px;
  margin-bottom: 16px;
  opacity: 0.5;
}

.empty-message {
  font-size: 18px;
  font-weight: 600;
  color: var(--text-color);
  margin-bottom: 8px;
}

.empty-hint {
  font-size: 14px;
  color: var(--secondary-color);
}

/* Theme: Light */
.theme-light {
  --chart-bg: #ffffff;
  --chart-border: #e5e7eb;
  --heading-color: #111827;
  --text-color: #111827;
  --secondary-color: #6b7280;
  --primary-color: #3b82f6;
  --primary-color-hover: #2563eb;
  --input-bg: #ffffff;
  --input-border: #d1d5db;
  --secondary-button-bg: #f3f4f6;
  --secondary-button-hover-bg: #e5e7eb;
  --toggle-bg: #f3f4f6;
  --toggle-hover-bg: #e5e7eb;
  --tooltip-bg: #ffffff;
  --tooltip-border: #e5e7eb;
  --divider-color: #e5e7eb;
  --success-bg: #d1fae5;
  --success-color: #065f46;
  --info-bg: #dbeafe;
  --info-color: #1e40af;
  --warning-bg: #fef3c7;
  --warning-color: #92400e;
  --error-color: #dc2626;
  --skeleton-start: #f3f4f6;
  --skeleton-middle: #e5e7eb;
  --skeleton-end: #f3f4f6;
}

/* Theme: Dark */
.theme-dark {
  --chart-bg: #1f2937;
  --chart-border: #374151;
  --heading-color: #f3f4f6;
  --text-color: #f3f4f6;
  --secondary-color: #9ca3af;
  --primary-color: #3b82f6;
  --primary-color-hover: #2563eb;
  --input-bg: #374151;
  --input-border: #4b5563;
  --secondary-button-bg: #374151;
  --secondary-button-hover-bg: #4b5563;
  --toggle-bg: #374151;
  --toggle-hover-bg: #4b5563;
  --tooltip-bg: #1f2937;
  --tooltip-border: #4b5563;
  --divider-color: #374151;
  --success-bg: #064e3b;
  --success-color: #6ee7b7;
  --info-bg: #1e3a8a;
  --info-color: #93c5fd;
  --warning-bg: #78350f;
  --warning-color: #fcd34d;
  --error-color: #f87171;
  --skeleton-start: #374151;
  --skeleton-middle: #4b5563;
  --skeleton-end: #374151;
}

/* Responsive */
@media (max-width: 768px) {
  .chart-header {
    flex-direction: column;
    gap: 12px;
  }

  .chart-controls {
    width: 100%;
    justify-content: space-between;
  }

  .chart-footer {
    grid-template-columns: repeat(2, 1fr);
    gap: 12px;
  }

  .stat-value {
    font-size: 18px;
  }
}
```

---

## Multi-Domain Applicability

### Finance Domain: Cash Flow Forecast
```tsx
<ForecastChart
  data={cashFlowData}
  algorithm="arima"
  confidenceInterval={90}
  chartType="area"
  showConfidenceInterval={true}
  currency="USD"
  locale="en-US"
  height={400}
  onHover={(point) => console.log('Hovered:', point)}
  onZoom={(range) => console.log('Zoomed to:', range)}
  compareAlgorithms={["linear_regression", "prophet"]}
/>
// Use case: Predict cash flow for next 12 months
// Data: Monthly income/expenses with confidence bands
// Shows: Green line (historical), blue dashed (forecast), shaded CI
```

### Healthcare Domain: Patient Volume Forecast
```tsx
<ForecastChart
  data={patientVolumeData}
  algorithm="exponential_smoothing"
  confidenceInterval={95}
  chartType="line"
  showConfidenceInterval={true}
  showSeasonality={true}
  height={450}
  onHover={(point) => updateDashboard(point)}
/>
// Use case: Predict daily patient admissions
// Data: Historical daily admissions + 30-day forecast
// Shows: Seasonal patterns, confidence intervals, anomaly detection
```

### Legal Domain: Case Pipeline Projection
```tsx
<ForecastChart
  data={casePipelineData}
  algorithm="moving_average"
  confidenceInterval={80}
  chartType="bar"
  showConfidenceInterval={false}
  height={400}
  theme="dark"
/>
// Use case: Forecast incoming case volume
// Data: Monthly case filings with projected trend
// Shows: Historical cases (green bars), forecast (blue bars)
```

### Research Domain: Grant Funding Forecast
```tsx
<ForecastChart
  data={fundingForecastData}
  algorithm="prophet"
  confidenceInterval={90}
  chartType="combo"
  currency="USD"
  showConfidenceInterval={true}
  highlightAnomalies={true}
  height={500}
/>
// Use case: Predict grant award amounts over time
// Data: Quarterly grant awards + 2-year forecast
// Shows: Historical (bars), forecast (line), outlier detection
```

### Manufacturing Domain: Production Forecast
```tsx
<ForecastChart
  data={productionData}
  algorithm="linear_regression"
  confidenceInterval={90}
  chartType="line"
  showConfidenceInterval={true}
  showGrid={true}
  height={400}
/>
// Use case: Forecast monthly production output
// Data: Units produced per month + 6-month projection
// Shows: Trend line, confidence bands, grid for readability
```

### Media Domain: Revenue Forecast
```tsx
<ForecastChart
  data={adRevenueData}
  algorithm="arima"
  confidenceInterval={95}
  chartType="area"
  currency="USD"
  showConfidenceInterval={true}
  compareAlgorithms={["exponential_smoothing", "prophet"]}
  height={450}
/>
// Use case: Predict monthly ad revenue
// Data: Historical revenue + 12-month forecast
// Shows: Multiple algorithm comparison, confidence intervals
```

---

## Features

### 1. Confidence Intervals
```typescript
// Show prediction uncertainty with shaded bands
// Three levels: 80%, 90%, 95%
// Color-coded: lighter = wider interval
// Helps assess forecast reliability
```

### 2. Interactive Zoom/Pan
```typescript
// Brush control at bottom for zooming
// Click-drag on chart to select range
// Reset button to restore full view
// Useful for exploring specific time periods
```

### 3. Multiple Chart Types
```typescript
// Line: Best for trends over time
// Area: Emphasizes magnitude of change
// Bar: Good for discrete time periods
// Combo: Mix historical bars with forecast line
```

### 4. Algorithm Comparison
```typescript
// Compare multiple forecasting methods
// Dropdown to switch between algorithms
// Side-by-side visual comparison
// Shows which algorithm fits best
```

### 5. Anomaly Detection
```typescript
// Highlights outliers in historical data
// Anomaly score 0-1 (70%+ flagged)
// Shows in tooltip with warning icon
// Helps identify data quality issues
```

### 6. Seasonality Overlay
```typescript
// Shows seasonal patterns in data
// Decomposition: trend + seasonality + residual
// Useful for yearly cycles (retail, healthcare)
// Improves forecast accuracy
```

---

## Accessibility

```tsx
<div
  className="forecast-chart"
  role="img"
  aria-label="Forecast chart showing historical data and predictions"
>
  <div className="chart-header">
    <h3 id="chart-title">Forecast Analysis</h3>
  </div>

  <ResponsiveContainer>
    <ChartComponent aria-labelledby="chart-title">
      {/* Chart elements */}
    </ChartComponent>
  </ResponsiveContainer>

  {/* Keyboard navigation for controls */}
  <button
    className="zoom-reset-button"
    aria-label="Reset zoom to show full date range"
    tabIndex={0}
  >
    Reset Zoom
  </button>
</div>
```

---

## Related Components

**Uses:**
- recharts library for chart rendering
- date-fns for date manipulation

**Used by:**
- ForecastDashboard (displays multiple forecasts)
- GoalProgressCard (shows forecast vs goal)
- BudgetPlanner (revenue/expense forecasts)

**Similar patterns:**
- TrendChart (simpler, no forecasting)
- ComparisonChart (actual vs budget)
- TimeSeriesChart (historical data only)
