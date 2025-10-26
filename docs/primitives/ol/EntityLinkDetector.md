# OL Primitive: EntityLinkDetector

**Type**: Entity Relationship Detection
**Domain**: Universal (domain-agnostic)
**Version**: 1.0
**Status**: Specification
**Introduced in**: Vertical 3.5 (Relationships)
**Renamed from**: TransferDetector (finance-specific name)

---

## Purpose

Universal interface for auto-detecting relationships between entities with confidence scoring. Identifies candidate entity pairs that likely represent the same logical relationship (transfers, citations, payments, etc.) using configurable matching criteria.

---

## Interface Contract

```python
from abc import ABC, abstractmethod
from typing import List, Generic, TypeVar
from dataclasses import dataclass

TEntity = TypeVar('TEntity')

@dataclass
class LinkSuggestion:
    """
    Suggested link between two entities with confidence score.
    """
    candidate_id: str
    confidence: float  # 0.0-1.0
    link_type: str  # Domain-specific (transfer, citation, payment, etc.)
    metadata: dict  # Additional matching details

class EntityLinkDetector(ABC, Generic[TEntity]):
    @abstractmethod
    def detect(
        self,
        entity: TEntity,
        candidate_pool: List[TEntity]
    ) -> List[LinkSuggestion]:
        """
        Find candidate entities that likely link to given entity.

        Args:
            entity: The entity to find links for
            candidate_pool: Pool of potential matching entities

        Returns:
            List of suggestions sorted by confidence (highest first)
            Only returns suggestions above confidence threshold (typically 0.70)
        """
        pass

    @abstractmethod
    def calculate_confidence(
        self,
        entity_a: TEntity,
        entity_b: TEntity
    ) -> float:
        """
        Calculate confidence score for potential link between two entities.

        Returns:
            Float between 0.0 (no match) and 1.0 (perfect match)
        """
        pass
```

---

## Multi-Domain Applicability

### Finance: TransferDetector

**Entity Pairs:** Transaction ↔ Transaction
**Link Types:** Transfer, FX conversion, Reimbursement
**Matching Criteria:** Amount (±$1), Date (±3 days), Opposite signs, Different accounts

```python
class TransferDetector(EntityLinkDetector[CanonicalTransaction]):
    def detect(
        self,
        transaction: CanonicalTransaction,
        candidate_pool: List[CanonicalTransaction]
    ) -> List[LinkSuggestion]:
        """
        Detect transfer pairs (e.g., BofA -$2000 → Wise +$2000).
        """
        suggestions = []

        for candidate in candidate_pool:
            # Filter: Opposite amount, ±3 days, different account
            if not self._is_potential_transfer(transaction, candidate):
                continue

            confidence = self.calculate_confidence(transaction, candidate)
            if confidence >= 0.70:
                suggestions.append(LinkSuggestion(
                    candidate_id=candidate.canonical_id,
                    confidence=confidence,
                    link_type="transfer",
                    metadata={
                        "amount_diff": abs(transaction.amount + candidate.amount),
                        "date_diff_days": abs((transaction.date - candidate.date).days)
                    }
                ))

        return sorted(suggestions, key=lambda s: s.confidence, reverse=True)

    def calculate_confidence(
        self,
        t1: CanonicalTransaction,
        t2: CanonicalTransaction
    ) -> float:
        """
        Weighted scoring:
        - Amount match (40%): Within $1
        - Date proximity (30%): Within 7 days
        - Opposite signs (20%): One negative, one positive
        - Different accounts (10%): Not same account
        """
        # Amount score (40%)
        amount_diff = abs(t1.amount + t2.amount)
        amount_score = 1.0 if amount_diff < 1.0 else 0.0

        # Date score (30%)
        days_diff = abs((t1.date - t2.date).days)
        date_score = max(0, 1 - days_diff / 7)

        # Sign score (20%)
        sign_score = 1.0 if (t1.amount * t2.amount) < 0 else 0.0

        # Account score (10%)
        account_score = 1.0 if t1.account != t2.account else 0.0

        return (amount_score * 0.4 + date_score * 0.3 +
                sign_score * 0.2 + account_score * 0.1)

    def _is_potential_transfer(self, t1, t2) -> bool:
        # Quick filter before expensive confidence calculation
        if abs(t1.amount + t2.amount) > 1.0:  # Not opposite amounts
            return False
        if abs((t1.date - t2.date).days) > 7:  # Too far apart
            return False
        if t1.account == t2.account:  # Same account (not a transfer)
            return False
        return True
```

**Use Case:** Auto-detect BofA → Wise → Scotia transfer chains (3-4 per week)

---

### Research (RSRCH): FactLinkDetector

**Entity Pairs:** Fact ↔ Fact (claims about founders, companies, investments)
**Link Types:** Confirms, Contradicts, Updates, Duplicates
**Matching Criteria:** Subject entity match, Fact type match, Source credibility, Temporal proximity

**RSRCH Context:** Research system for founders/companies/entities collecting facts from web, interviews, tweets, podcasts, transcripts.

**Example Facts to Link:**
- TechCrunch: "Sam Altman invested in Helion Energy in 2021"
- Y Combinator Podcast (transcript): "Sam backed Helion, a fusion energy company, back in '21"
- Twitter @sama: "Excited to support @HelionEnergy's mission"

```python
@dataclass
class Fact:
    """Truth Construction primitive: claim extracted from source."""
    fact_id: str
    subject_entity: str  # "Sam Altman", "OpenAI", "Y Combinator"
    fact_type: str  # "investment", "founding", "acquisition", "statement"
    claim: str  # Natural language claim
    source_url: str  # Where this came from
    source_type: str  # "web_article", "interview_transcript", "tweet", "podcast"
    extracted_date: date
    confidence: float  # 0.0-1.0 (source credibility)
    metadata: dict  # Amount, date mentioned, entities involved

class FactLinkDetector(EntityLinkDetector[Fact]):
    def detect(
        self,
        fact: Fact,
        candidate_pool: List[Fact]
    ) -> List[LinkSuggestion]:
        """
        Detect facts that confirm, contradict, or duplicate given fact.

        Example:
        - Input: Fact "Sam Altman invested in OpenAI" (TechCrunch)
        - Finds: Fact "Sam backed OpenAI" (interview transcript) → confidence 0.95 (confirms)
        - Finds: Fact "Sam Altman is CEO of OpenAI" (Wikipedia) → confidence 0.60 (related, not duplicate)
        """
        suggestions = []

        for candidate in candidate_pool:
            if candidate.fact_id == fact.fact_id:
                continue  # Skip self

            # Quick filter: Must be about same subject entity
            if not self._same_subject(fact, candidate):
                continue

            confidence = self.calculate_confidence(fact, candidate)
            if confidence >= 0.70:
                link_type = self._determine_link_type(fact, candidate)

                suggestions.append(LinkSuggestion(
                    candidate_id=candidate.fact_id,
                    confidence=confidence,
                    link_type=link_type,
                    metadata={
                        "claim_similarity": self._claim_similarity(fact, candidate),
                        "source_credibility_delta": abs(fact.confidence - candidate.confidence),
                        "temporal_gap_days": abs((fact.extracted_date - candidate.extracted_date).days)
                    }
                ))

        return sorted(suggestions, key=lambda s: s.confidence, reverse=True)

    def calculate_confidence(self, f1: Fact, f2: Fact) -> float:
        """
        Weighted scoring:
        - Subject entity match (30%): Same person/company
        - Claim similarity (40%): NLP semantic similarity
        - Fact type match (20%): Both "investment", both "founding", etc.
        - Source credibility (10%): Higher if both high-quality sources
        """
        # Subject entity score (30%)
        subject_score = 1.0 if self._normalize_entity(f1.subject_entity) == self._normalize_entity(f2.subject_entity) else 0.0

        # Claim similarity (40%) - semantic NLP
        claim_score = self._claim_similarity(f1, f2)

        # Fact type score (20%)
        type_score = 1.0 if f1.fact_type == f2.fact_type else 0.3

        # Source credibility (10%)
        avg_credibility = (f1.confidence + f2.confidence) / 2
        credibility_score = avg_credibility

        return (subject_score * 0.3 + claim_score * 0.4 +
                type_score * 0.2 + credibility_score * 0.1)

    def _claim_similarity(self, f1: Fact, f2: Fact) -> float:
        """
        Semantic similarity using NLP (embeddings, fuzzy match, etc.)

        Example:
        - "Sam invested in Helion" vs "Sam backed Helion" → 0.95 (same meaning)
        - "Sam invested in Helion" vs "Sam is CEO of OpenAI" → 0.20 (different)
        """
        from Levenshtein import ratio
        # Simple fuzzy match (production would use sentence embeddings)
        return ratio(f1.claim.lower(), f2.claim.lower())

    def _normalize_entity(self, entity: str) -> str:
        """Normalize entity names: '@sama' → 'Sam Altman', 'Elon' → 'Elon Musk'"""
        # Would use entity resolution service
        entity_map = {
            "@sama": "Sam Altman",
            "sama": "Sam Altman",
            "sam": "Sam Altman",
            "@elonmusk": "Elon Musk",
            "elon": "Elon Musk"
        }
        return entity_map.get(entity.lower(), entity)

    def _determine_link_type(self, f1: Fact, f2: Fact) -> str:
        """
        Determine relationship type between facts.

        Returns: "confirms" | "contradicts" | "updates" | "duplicates"
        """
        similarity = self._claim_similarity(f1, f2)

        if similarity > 0.90:
            return "duplicates"  # Almost identical claims
        elif similarity > 0.70:
            return "confirms"  # Similar but different wording
        elif self._check_contradiction(f1, f2):
            return "contradicts"  # Conflicting claims
        else:
            return "updates"  # New information about same subject

    def _same_subject(self, f1: Fact, f2: Fact) -> bool:
        """Check if facts are about same entity."""
        return self._normalize_entity(f1.subject_entity) == self._normalize_entity(f2.subject_entity)

    def _check_contradiction(self, f1: Fact, f2: Fact) -> bool:
        """Detect contradicting facts (simplified)."""
        # Example: "Sam invested in X" vs "Sam did not invest in X"
        if "not" in f1.claim.lower() and "not" not in f2.claim.lower():
            return True
        return False
```

**Use Case Examples:**

1. **Founder Research:**
   - Fact 1 (TechCrunch): "Sam Altman invested $375M in Helion Energy"
   - Fact 2 (Bloomberg): "Altman backed Helion with $375 million"
   - Link: `duplicates` (confidence 0.96)

2. **Company Validation:**
   - Fact 1 (Twitter): "OpenAI raised $10B from Microsoft"
   - Fact 2 (SEC Filing): "Microsoft invested $10 billion in OpenAI"
   - Link: `confirms` (confidence 0.98, high credibility)

3. **Contradiction Detection:**
   - Fact 1 (Blog post): "Company X has 50 employees"
   - Fact 2 (LinkedIn): "Company X has 200 employees"
   - Link: `contradicts` (confidence 0.75)

**Truth Construction:** Link facts from multiple sources → Build canonical truth with provenance

---

### Healthcare: ClaimPaymentDetector

**Entity Pairs:** Claim ↔ Payment (EOB)
**Link Types:** Payment, Partial-payment, Denial
**Matching Criteria:** Claim amount, Service date, Patient ID, Procedure code

```python
class ClaimPaymentDetector(EntityLinkDetector[InsuranceClaim]):
    def detect(
        self,
        claim: InsuranceClaim,
        candidate_pool: List[Payment]
    ) -> List[LinkSuggestion]:
        """
        Detect payments matching insurance claim.
        """
        suggestions = []

        for payment in candidate_pool:
            if payment.claim_id == claim.claim_id:
                # Explicit match - perfect confidence
                suggestions.append(LinkSuggestion(
                    candidate_id=payment.payment_id,
                    confidence=1.0,
                    link_type="payment",
                    metadata={"match_type": "explicit_claim_id"}
                ))
                continue

            confidence = self.calculate_confidence(claim, payment)
            if confidence >= 0.70:
                link_type = self._determine_link_type(claim, payment)
                suggestions.append(LinkSuggestion(
                    candidate_id=payment.payment_id,
                    confidence=confidence,
                    link_type=link_type,
                    metadata={
                        "amount_diff": abs(claim.amount - payment.amount),
                        "date_diff_days": abs((claim.service_date - payment.payment_date).days)
                    }
                ))

        return sorted(suggestions, key=lambda s: s.confidence, reverse=True)

    def calculate_confidence(self, claim: InsuranceClaim, payment: Payment) -> float:
        """
        Weighted scoring:
        - Amount match (40%): Within 10% tolerance
        - Date proximity (30%): Within 60 days
        - Patient match (20%): Same patient ID
        - Procedure match (10%): Same procedure code
        """
        # Amount score (40%)
        amount_diff_pct = abs(claim.amount - payment.amount) / claim.amount
        amount_score = 1.0 if amount_diff_pct <= 0.10 else 0.0

        # Date score (30%)
        days_diff = abs((claim.service_date - payment.payment_date).days)
        date_score = max(0, 1 - days_diff / 60)

        # Patient score (20%)
        patient_score = 1.0 if claim.patient_id == payment.patient_id else 0.0

        # Procedure score (10%)
        procedure_score = 1.0 if claim.procedure_code == payment.procedure_code else 0.0

        return (amount_score * 0.4 + date_score * 0.3 +
                patient_score * 0.2 + procedure_score * 0.1)

    def _determine_link_type(self, claim, payment) -> str:
        if payment.amount >= claim.amount * 0.95:
            return "payment"  # Full payment
        elif payment.amount >= claim.amount * 0.20:
            return "partial_payment"
        else:
            return "denial"  # Rejected or minimal payment
```

**Use Case:** Reconcile insurance claims with EOB payments (thousands per month)

---

### E-commerce: OrderShipmentDetector

**Entity Pairs:** Order ↔ Shipment
**Link Types:** Shipped, Partial-shipment, Backordered
**Matching Criteria:** Order number, Customer, Amount, Date

```python
class OrderShipmentDetector(EntityLinkDetector[Order]):
    def detect(
        self,
        order: Order,
        candidate_pool: List[Shipment]
    ) -> List[LinkSuggestion]:
        """
        Detect shipments matching order.
        """
        suggestions = []

        for shipment in candidate_pool:
            # Explicit match by tracking number
            if shipment.order_number == order.order_number:
                suggestions.append(LinkSuggestion(
                    candidate_id=shipment.shipment_id,
                    confidence=1.0,
                    link_type="shipped",
                    metadata={"match_type": "explicit_order_number"}
                ))
                continue

            confidence = self.calculate_confidence(order, shipment)
            if confidence >= 0.70:
                suggestions.append(LinkSuggestion(
                    candidate_id=shipment.shipment_id,
                    confidence=confidence,
                    link_type="shipped",
                    metadata={
                        "customer_match": order.customer_id == shipment.customer_id,
                        "date_diff_days": abs((order.order_date - shipment.ship_date).days)
                    }
                ))

        return sorted(suggestions, key=lambda s: s.confidence, reverse=True)

    def calculate_confidence(self, order: Order, shipment: Shipment) -> float:
        """
        Weighted scoring:
        - Customer match (40%)
        - Amount match (30%)
        - Date proximity (20%)
        - Address match (10%)
        """
        customer_score = 1.0 if order.customer_id == shipment.customer_id else 0.0

        amount_diff_pct = abs(order.total - shipment.value) / order.total
        amount_score = 1.0 if amount_diff_pct <= 0.05 else 0.0

        days_diff = abs((order.order_date - shipment.ship_date).days)
        date_score = max(0, 1 - days_diff / 14)

        address_score = 1.0 if order.shipping_address == shipment.destination else 0.0

        return (customer_score * 0.4 + amount_score * 0.3 +
                date_score * 0.2 + address_score * 0.1)
```

**Use Case:** Track order fulfillment, detect missing shipments

---

## Responsibilities

✅ **DOES:**
- Auto-detect candidate entity pairs for linking
- Calculate confidence scores using domain-specific criteria
- Filter candidates by minimum confidence threshold
- Return suggestions sorted by confidence
- Support configurable matching weights
- Handle one-to-one and one-to-many relationships

❌ **DOES NOT:**
- Store relationships (use RelationshipStore)
- Make final linking decisions (user/threshold decides)
- Modify entities (read-only detection)
- Handle cross-domain links (single domain per detector)

---

## Implementation Notes

**Performance:**
- Use blocking strategy to pre-filter candidates (see ADR-0017)
- Calculate confidence only for promising candidates
- Early exit on hard filters (date range, amount range)

**Confidence Thresholds:**
- **0.95+**: Auto-link (very high confidence)
- **0.70-0.95**: Suggest to user for review
- **<0.70**: Ignore (low confidence)

**Storage:**
- Detector is stateless (no persistence)
- Results stored in RelationshipStore after user accepts

---

## Related Primitives

- **RelationshipStore** - Persist accepted links
- **RelationshipMatcher** - Fuzzy matching algorithms
- **FuzzyMatcher** - String similarity (Levenshtein, Jaro-Winkler)
- **BlockingStrategy** - Pre-filtering for performance (ADR-0017)

---

## Testing Strategy

```python
def test_transfer_detector():
    # Given: Two opposite transactions
    bofa_tx = CanonicalTransaction(
        amount=-2000.00,
        date=date(2025, 1, 15),
        account="bofa_debit"
    )
    wise_tx = CanonicalTransaction(
        amount=2000.00,
        date=date(2025, 1, 16),  # 1 day later
        account="wise"
    )

    # When: Detect links
    detector = TransferDetector()
    suggestions = detector.detect(bofa_tx, [wise_tx])

    # Then: High confidence suggestion
    assert len(suggestions) == 1
    assert suggestions[0].confidence >= 0.90
    assert suggestions[0].link_type == "transfer"
```

---

**Last Updated:** 2025-10-25
**Maintained By:** Truth Construction Primitives Team
**Status:** Production-ready for multi-domain use
