# OL Primitive: ParserRegistry

**Type**: Service Discovery / Registry
**Domain**: Universal (domain-agnostic)
**Version**: 1.0
**Status**: Specification
**Introduced in**: Vertical 1.2 (Extraction)

---

## Purpose

Central registry for discovering and selecting appropriate Parser based on `source_type`. Enables pluggable parser architecture - add new parsers without modifying core extraction logic.

---

## Interface Contract

```python
from typing import Optional, List

class ParserRegistry:
    def register(self, parser: Parser) -> None:
        """Register a parser. Raises error if parser_id already registered."""
        pass

    def get_parser(self, source_type: str) -> Parser:
        """
        Get parser for source_type.
        Raises: ParserNotFoundError if no parser supports source_type
        """
        pass

    def get_all_parsers(self) -> List[Parser]:
        """Return all registered parsers."""
        pass

    def supports(self, source_type: str) -> bool:
        """Check if any parser supports source_type."""
        pass
```

---

## Behavior

**Registration:**
```python
registry = ParserRegistry()
registry.register(BofAPDFParser())
registry.register(ChaseCSVParser())
```

**Discovery:**
```python
parser = registry.get_parser("bofa_pdf")  # Returns BofAPDFParser
parser = registry.get_parser("chase_csv")  # Returns ChaseCSVParser
parser = registry.get_parser("unknown")    # Raises ParserNotFoundError
```

---

## Multi-Domain Applicability

**Finance:** Register bank-specific parsers (BoFA, Chase, Wells Fargo)
**Healthcare:** Register lab-specific parsers (LabCorp, Quest, hospital systems)
**Legal:** Register contract parsers (DocuSign, HelloSign, manual PDF)
**Research (RSRCH - Utilitario):** Register web scraping parsers (TechCrunch parser, Twitter parser, Lex Fridman podcast transcript parser)
**Manufacturing:** Register sensor parsers (CSV, JSON, proprietary formats)

---

## Related Primitives

- **Parser**: Registered and retrieved by this registry
- **Coordinator** (Vertical 1.2): Uses registry to find parser for `source_type`

---

**Last Updated**: 2025-10-23
**Maturity**: Spec complete, ready for implementation
