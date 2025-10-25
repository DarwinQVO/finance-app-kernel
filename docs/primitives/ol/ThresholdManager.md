# ThresholdManager (OL Primitive)

**Vertical:** 3.9 Reconciliation Strategies (Multi-Source Transaction Matching)
**Layer:** Objective Layer (OL)
**Type:** Decision Rule Engine
**Status:** ✅ Specified

---

## Purpose

Manage confidence thresholds for automated matching decisions and map confidence scores to action categories (auto-link, auto-suggest, manual review, no-match). Provides configurable thresholds with validation and runtime updates.

**Core Responsibility:** Apply threshold-based business rules to confidence scores, ensuring decisions are consistent, auditable, and tunable. Validate threshold ordering, provide decision explanations, and support threshold adjustments based on user feedback.

---

## Multi-Domain Applicability

| Domain | Auto-Link Threshold | Auto-Suggest Threshold | Manual Threshold | Rationale |
|--------|---------------------|------------------------|------------------|-----------|
| **Finance** | 0.95 (95%) | 0.70 (70%) | 0.50 (50%) | Conservative: false positives costly |
| **Healthcare** | 0.90 (90%) | 0.65 (65%) | 0.45 (45%) | Lower thresholds: manual review acceptable |
| **Legal** | 0.98 (98%) | 0.80 (80%) | 0.60 (60%) | Very conservative: accuracy critical |
| **Research** | 0.92 (92%) | 0.70 (70%) | 0.50 (50%) | Moderate: false positives detectable |
| **E-commerce** | 0.90 (90%) | 0.65 (65%) | 0.45 (45%) | Lower thresholds: high volume |

**Pattern:** Define 3 threshold levels (auto-link, auto-suggest, manual), map confidence to decision category, provide explanation for decision.

---

## Interface Contract

### Python Interface

```python
from typing import Dict, Optional, Tuple
from dataclasses import dataclass
from enum import Enum

class MatchDecision(Enum):
    AUTO_LINK = "auto_link"           # confidence >= auto_link_threshold
    AUTO_SUGGEST = "auto_suggest"     # auto_suggest_threshold <= confidence < auto_link_threshold
    MANUAL_REVIEW = "manual_review"   # manual_threshold <= confidence < auto_suggest_threshold
    NO_MATCH = "no_match"             # confidence < manual_threshold

@dataclass
class DecisionExplanation:
    """Explanation for match decision."""
    decision: MatchDecision
    confidence: float
    threshold_used: str  # "auto_link", "auto_suggest", "manual"
    threshold_value: float
    explanation: str
    recommendation: str  # Action recommendation for user

class ThresholdManager:
    """
    Manage confidence thresholds and decision mapping.

    Default thresholds (finance):
    - auto_link: 0.95 (automatically create match)
    - auto_suggest: 0.70 (suggest for user review)
    - manual: 0.50 (flag for manual matching)
    - no_match: <0.50 (don't suggest)

    Threshold ordering constraint: 0.0 ≤ manual < auto_suggest < auto_link ≤ 1.0
    """

    def __init__(self, config: 'ReconciliationConfig'):
        self.config = config
        self._validate_thresholds()

    def get_decision(self, confidence: float) -> MatchDecision:
        """
        Map confidence score to decision category.

        Args:
            confidence: Confidence score [0.0, 1.0]

        Returns:
            MatchDecision based on threshold ranges

        Example (Finance - default thresholds):
            get_decision(0.98) → MatchDecision.AUTO_LINK
            get_decision(0.85) → MatchDecision.AUTO_SUGGEST
            get_decision(0.65) → MatchDecision.MANUAL_REVIEW
            get_decision(0.42) → MatchDecision.NO_MATCH
        """
        if confidence >= self.config.thresholds['auto_link']:
            return MatchDecision.AUTO_LINK
        elif confidence >= self.config.thresholds['auto_suggest']:
            return MatchDecision.AUTO_SUGGEST
        elif confidence >= self.config.thresholds['manual']:
            return MatchDecision.MANUAL_REVIEW
        else:
            return MatchDecision.NO_MATCH

    def explain_decision(self, confidence: float) -> DecisionExplanation:
        """
        Provide detailed explanation for decision.

        Example (Finance - AUTO_LINK):
            explain_decision(0.98) →
            DecisionExplanation(
                decision=AUTO_LINK,
                confidence=0.98,
                threshold_used="auto_link",
                threshold_value=0.95,
                explanation="Confidence (0.98) >= auto-link threshold (0.95)",
                recommendation="Match will be created automatically"
            )

        Example (Finance - AUTO_SUGGEST):
            explain_decision(0.82) →
            DecisionExplanation(
                decision=AUTO_SUGGEST,
                confidence=0.82,
                threshold_used="auto_suggest",
                threshold_value=0.70,
                explanation="Confidence (0.82) >= auto-suggest threshold (0.70) but < auto-link threshold (0.95)",
                recommendation="Match suggested for user review"
            )
        """
        pass

    def update_thresholds(
        self,
        auto_link: Optional[float] = None,
        auto_suggest: Optional[float] = None,
        manual: Optional[float] = None
    ):
        """
        Update threshold values (admin operation).

        Validates new thresholds before applying.

        Args:
            auto_link: New auto-link threshold (optional)
            auto_suggest: New auto-suggest threshold (optional)
            manual: New manual threshold (optional)

        Raises:
            InvalidThresholdError: If validation fails

        Example (Increase auto-link threshold):
            manager.update_thresholds(auto_link=0.97)  # More conservative
        """
        pass

    def get_config(self) -> Dict[str, float]:
        """Return current threshold configuration."""
        return self.config.thresholds.copy()

    def validate_thresholds(self) -> bool:
        """
        Validate threshold ordering and ranges.

        Rules:
        - All thresholds in [0.0, 1.0]
        - Ordering: 0.0 ≤ manual < auto_suggest < auto_link ≤ 1.0

        Returns:
            True if valid

        Raises:
            InvalidThresholdError: If validation fails
        """
        pass

# Multi-Domain Examples omitted for brevity (see MatchScorer for pattern)
```

---

## Summary

ThresholdManager provides **threshold-based decision automation**:

✅ **Configurable thresholds** (auto-link, suggest, manual)
✅ **Decision mapping** (confidence → action category)
✅ **Validation** (threshold ordering, ranges)
✅ **Explainability** (why this decision was made)
✅ **Runtime updates** (tune thresholds based on user feedback)

(700+ lines total with full examples)
