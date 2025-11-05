# CRITICAL ARCHITECTURAL FIX: Separation of Concerns

**Date:** 2025-11-04
**Issue:** IPiiScrubber conflates two independent responsibilities
**Impact:** HIGH - Affects all interface design and implementation
**Status:** Must fix before porting code to new repo

---

## The Problem: Conflated Concerns

### Current Design (WRONG)

```python
class IPiiScrubber(ABC):
    """
    ❌ PROBLEM: This does TWO unrelated things:
    1. Normalization (collapse variations → canonical key)
    2. PII Extraction (extract entities for app use)
    """
    @abstractmethod
    def scrub(self, prompt: str) -> ScrubResult:
        pass

@dataclass
class ScrubResult:
    clean_prompt: str      # Canonical key (normalization)
    pii_extracted: dict    # Entities (extraction)
```

**Why This Is Wrong:**

| Concern | Purpose | Example |
|---------|---------|---------|
| **Normalization** | Collapse variations into canonical key | "cancel 1234" → "cancel_order" |
| **PII Extraction** | Extract entities for application use | "cancel 1234" → {"order_id": "1234"} |

**These are INDEPENDENT:**
- You might want normalization WITHOUT extraction (privacy-first)
- You might want extraction WITHOUT normalization (exact-match cache)
- You might want BOTH (most common)
- You might want NEITHER (pass-through)

**By forcing them into one interface, we:**
1. ❌ Violate Single Responsibility Principle
2. ❌ Force users to implement both even if they only need one
3. ❌ Make the naming confusing ("PII Scrubber" but it's really a normalizer?)
4. ❌ Prevent independent composition

---

## The Solution: Two Separate Interfaces

### Correct Design

```python
# ========================================
# Interface 1: Normalization (Core L0 Logic)
# ========================================

class INormalizer(ABC):
    """
    Collapses query variations into a canonical key for caching.

    PRIMARY JOB: Turn 10,000 variations into 1 cache key

    EXAMPLE:
        Input:  "cancel order 12345"
        Output: "cancel_order"

    IMPLEMENTATIONS:
    - PassThroughNormalizer: Returns input unchanged (exact match)
    - RegexNormalizer: Basic keyword detection
    - BERTNormalizer: Fine-tuned intent classifier
    - CustomNormalizer: User-provided domain logic
    """

    @abstractmethod
    def normalize(self, prompt: str) -> str:
        """
        Returns canonical key for cache lookup.

        Args:
            prompt: User query (may contain variations, PII, etc.)

        Returns:
            Canonical key (e.g., "cancel_order", "track_shipment")

        Performance:
            - PassThroughNormalizer: <0.1ms
            - RegexNormalizer: <1ms
            - BERTNormalizer: <10ms
        """
        pass


# ========================================
# Interface 2: Entity Extraction (Optional)
# ========================================

class IEntityExtractor(ABC):
    """
    Extracts structured entities from user query.

    PURPOSE: Provide entities for application to use in API calls

    EXAMPLE:
        Input:  "cancel order 12345"
        Output: {"order_id": "12345"}

    IMPLEMENTATIONS:
    - NoOpExtractor: Returns empty dict (no extraction)
    - PresidioExtractor: Microsoft Presidio (automatic PII detection)
    - SpacyExtractor: spaCy NER (custom trained)
    - CustomExtractor: User-provided regex/rules
    """

    @abstractmethod
    def extract(self, prompt: str) -> Dict[str, Any]:
        """
        Returns extracted entities.

        Args:
            prompt: User query

        Returns:
            Dictionary of entities (e.g., {"order_id": "12345", "email": "user@example.com"})

        Performance:
            - NoOpExtractor: <0.1ms
            - PresidioExtractor: <50ms (optimized)
            - SpacyExtractor: <10ms
        """
        pass


# ========================================
# Gateway Flow (Correct)
# ========================================

class L0L3Gateway:
    def __init__(
        self,
        llm_client,
        normalizer: INormalizer = None,           # ← Separate
        entity_extractor: IEntityExtractor = None # ← Separate
    ):
        self.llm = llm_client
        self.normalizer = normalizer or PassThroughNormalizer()
        self.extractor = entity_extractor or NoOpExtractor()

    def chat(self, prompt: str) -> Response:
        # Step 1: Normalize (for cache key)
        canonical_key = self.normalizer.normalize(prompt)

        # Step 2: Extract entities (for app use)
        entities = self.extractor.extract(prompt)

        # Step 3: Cache lookup
        if cached := self.l0.get(canonical_key):
            return Response(
                text=cached.value,
                entities=entities  # Return entities separately
            )

        # Step 4: LLM call (if cache miss)
        response_text = self.llm.call(prompt)

        return Response(
            text=response_text,
            entities=entities
        )
```

---

## Use Cases & Combinations

### Use Case 1: Privacy-First (Normalization Only)

**Scenario:** You want high cache hit rate but don't want to extract/store PII

```python
gateway = L0L3Gateway(
    llm=client,
    normalizer=RegexNormalizer(),  # Collapse variations
    entity_extractor=NoOpExtractor()  # Don't extract PII
)

result = gateway.chat("cancel order 12345")
# result.entities == {}  (no PII extracted)
```

**Flow:**
```
"cancel order 12345" → RegexNormalizer → "cancel_order" → L0.get("cancel_order")
                     ↘ NoOpExtractor → {}
```

---

### Use Case 2: Entity Extraction Without Normalization

**Scenario:** You want exact-match caching but need entities for app logic

```python
gateway = L0L3Gateway(
    llm=client,
    normalizer=PassThroughNormalizer(),  # No normalization (exact match)
    entity_extractor=PresidioExtractor()  # Extract PII
)

result = gateway.chat("cancel order 12345")
# result.entities == {"order_id": "12345"}
```

**Flow:**
```
"cancel order 12345" → PassThroughNormalizer → "cancel order 12345" (unchanged)
                     ↘ PresidioExtractor → {"order_id": "12345"}
```

---

### Use Case 3: Both (Most Common)

**Scenario:** High cache hit rate AND entity extraction

```python
gateway = L0L3Gateway(
    llm=client,
    normalizer=RegexNormalizer(),     # Collapse variations
    entity_extractor=PresidioExtractor()  # Extract PII
)

result = gateway.chat("cancel order 12345")
# Canonical key: "cancel_order"
# Entities: {"order_id": "12345"}
```

**Flow:**
```
"cancel order 12345" → RegexNormalizer → "cancel_order" → L0.get("cancel_order")
                     ↘ PresidioExtractor → {"order_id": "12345"}
```

---

### Use Case 4: Neither (Pass-Through)

**Scenario:** You handle everything upstream (just want L0 orchestration)

```python
gateway = L0L3Gateway(
    llm=client,
    normalizer=PassThroughNormalizer(),  # No normalization
    entity_extractor=NoOpExtractor()     # No extraction
)
```

---

## Implementation Examples

### PassThroughNormalizer (Default)

```python
class PassThroughNormalizer(INormalizer):
    """
    No normalization. Returns input unchanged.

    USE CASE: Exact-match caching (traditional approach)

    PERFORMANCE: <0.1ms (instant)
    """
    def normalize(self, prompt: str) -> str:
        return prompt
```

### RegexNormalizer

```python
class RegexNormalizer(INormalizer):
    """
    Basic keyword-based normalization.

    USE CASE: Simple intent detection for known domains

    PERFORMANCE: <1ms (fast regex)
    """
    def normalize(self, prompt: str) -> str:
        lower = prompt.lower()

        # Domain-specific rules
        if 'cancel' in lower and 'order' in lower:
            return 'cancel_order'
        elif 'track' in lower or 'shipment' in lower:
            return 'track_shipment'
        elif 'return' in lower:
            return 'return_item'
        else:
            return 'unknown_intent'
```

### BERTNormalizer

```python
class BERTNormalizer(INormalizer):
    """
    Fine-tuned BERT classifier for intent detection.

    USE CASE: High-accuracy normalization (production)

    PERFORMANCE: <10ms (model inference)
    """
    def __init__(self, model_url: str):
        self.model_url = model_url
        self.client = requests.Session()

    def normalize(self, prompt: str) -> str:
        response = self.client.post(
            f"{self.model_url}/predict",
            json={"text": prompt}
        )
        return response.json()["intent"]
```

---

### NoOpExtractor (Default)

```python
class NoOpExtractor(IEntityExtractor):
    """
    No entity extraction. Returns empty dict.

    USE CASE: Privacy-first (don't extract PII)

    PERFORMANCE: <0.1ms (instant)
    """
    def extract(self, prompt: str) -> Dict[str, Any]:
        return {}
```

### PresidioExtractor

```python
class PresidioExtractor(IEntityExtractor):
    """
    Microsoft Presidio for automatic PII detection.

    USE CASE: Automatic entity extraction (emails, phones, SSNs, etc.)

    PERFORMANCE: <50ms p99 (optimized)
    """
    def __init__(self):
        self.analyzer = AnalyzerEngine()

    def extract(self, prompt: str) -> Dict[str, Any]:
        results = self.analyzer.analyze(text=prompt, language='en')

        entities = {}
        for result in results:
            entity_type = result.entity_type
            value = prompt[result.start:result.end]
            entities[entity_type.lower()] = value

        return entities
```

### CustomExtractor (User-Provided)

```python
class EcommerceExtractor(IEntityExtractor):
    """
    Custom extractor for e-commerce domain.

    USE CASE: Domain-specific entity extraction (order IDs, SKUs, etc.)

    PERFORMANCE: <1ms (fast regex)
    """
    def extract(self, prompt: str) -> Dict[str, Any]:
        entities = {}

        # Extract order ID
        if match := re.search(r'order\s*#?(\d+)', prompt, re.IGNORECASE):
            entities['order_id'] = match.group(1)

        # Extract SKU
        if match := re.search(r'SKU:?\s*([A-Z0-9-]+)', prompt):
            entities['sku'] = match.group(1)

        return entities
```

---

## Naming Comparison

| Old Name (Wrong) | New Name (Correct) | Responsibility |
|------------------|-------------------|----------------|
| `IPiiScrubber` | `INormalizer` | Collapse variations → canonical key |
| `IPiiScrubber` | `IEntityExtractor` | Extract entities for app use |
| `ScrubResult.clean_prompt` | `normalize() → str` | Canonical key |
| `ScrubResult.pii_extracted` | `extract() → dict` | Entities |
| `NoOpScrubber` | `PassThroughNormalizer` + `NoOpExtractor` | No-op for both |
| `PresidioScrubber` | `RegexNormalizer` + `PresidioExtractor` | Separate concerns |

---

## Response Object (Updated)

```python
@dataclass
class Response:
    """
    Gateway response with text and entities.

    DESIGN: Text and entities are separate concerns
    """
    text: str                  # LLM response or cached value
    entities: Dict[str, Any]   # Extracted entities (from IEntityExtractor)
    source: str                # "L0_CACHE", "L1_CACHE", "L3_LLM"
    latency_ms: float          # Total latency
```

---

## Migration Path

### Old Code (WRONG)

```python
from constraint_cache import L0L3Gateway, NoOpScrubber, PresidioScrubber

# Option 1: NoOp (nothing)
gateway = L0L3Gateway(
    llm=client,
    pii_scrubber=NoOpScrubber()  # ❌ Confusing name
)

# Option 2: Presidio (both normalization + extraction)
gateway = L0L3Gateway(
    llm=client,
    pii_scrubber=PresidioScrubber()  # ❌ Forces both concerns together
)
```

### New Code (CORRECT)

```python
from constraint_cache import L0L3Gateway, PassThroughNormalizer, NoOpExtractor
from constraint_cache import RegexNormalizer, PresidioExtractor

# Option 1: No normalization, no extraction (pass-through)
gateway = L0L3Gateway(
    llm=client,
    normalizer=PassThroughNormalizer(),  # ✅ Clear intent
    entity_extractor=NoOpExtractor()     # ✅ Separate concerns
)

# Option 2: Normalization + extraction (independent)
gateway = L0L3Gateway(
    llm=client,
    normalizer=RegexNormalizer(),      # ✅ For cache key
    entity_extractor=PresidioExtractor()  # ✅ For app logic
)

# Option 3: Normalization only (privacy-first)
gateway = L0L3Gateway(
    llm=client,
    normalizer=RegexNormalizer(),      # ✅ High cache hit rate
    entity_extractor=NoOpExtractor()   # ✅ No PII extraction
)
```

---

## Impact on Existing Code

### Files That Need Changes

| File | Change Required | Impact |
|------|----------------|--------|
| `IMPLEMENTATION_GUIDE_V1.md` | **REWRITE** interfaces | HIGH |
| `ARCHITECTURE_V2.md` | Update diagrams (archived anyway) | LOW |
| `ARCHITECTURE_SCALE.md` | Update terminology | MEDIUM |
| `README.md` | Update quickstart examples | HIGH |

### Architecture Diagrams (Updated)

**OLD (Wrong):**
```
User Query → IPiiScrubber → ScrubResult {clean_prompt, pii} → L0.get(clean_prompt)
             ↑ Conflated concerns
```

**NEW (Correct):**
```
User Query → INormalizer → canonical_key → L0.get(canonical_key)
          ↘ IEntityExtractor → entities (for app use)
          ↑ Independent concerns
```

---

## Benefits of Separation

### 1. Single Responsibility Principle ✅
Each interface does ONE thing:
- `INormalizer`: Collapse variations
- `IEntityExtractor`: Extract entities

### 2. Independent Composition ✅
Users can mix and match:
- Regex normalization + Presidio extraction
- BERT normalization + custom extraction
- Normalization only (privacy-first)
- Extraction only (exact-match cache)

### 3. Clear Naming ✅
- "Normalizer" clearly indicates intent collapse
- "EntityExtractor" clearly indicates extraction
- No confusion about "PII Scrubber" doing both

### 4. Performance Transparency ✅
Users can see latency for each concern:
- Normalization: <1ms (regex) or <10ms (BERT)
- Extraction: <50ms (Presidio) or <1ms (custom)

### 5. Testability ✅
Each concern has independent tests:
- Test normalization: "cancel 1234" → "cancel_order"
- Test extraction: "cancel 1234" → {"order_id": "1234"}

---

## Action Items (Before Migration)

### Immediate (Must Fix)
- [ ] Update IMPLEMENTATION_GUIDE_V1.md with new interfaces
- [ ] Rewrite interface definitions (INormalizer, IEntityExtractor)
- [ ] Rewrite L0L3Gateway to accept both separately
- [ ] Update all code examples (PassThrough, Regex, Presidio, etc.)

### Before New Repo
- [ ] Update ARCHITECTURE_SCALE.md terminology
- [ ] Update README.md quickstart examples
- [ ] Update all diagrams to show separation

### Documentation
- [ ] Add section: "When to use normalizer vs extractor"
- [ ] Add performance comparison table
- [ ] Add use case matrix (4 combinations)

---

## Conclusion

**This is a critical architectural fix that must happen BEFORE we port code to the new repository.**

**The Fix:**
1. Split `IPiiScrubber` into two interfaces:
   - `INormalizer` (collapse variations → canonical key)
   - `IEntityExtractor` (extract entities for app use)

2. Update L0L3Gateway to accept both separately:
   - `normalizer=PassThroughNormalizer()` (default)
   - `entity_extractor=NoOpExtractor()` (default)

3. Update all examples and documentation

**Benefits:**
- ✅ Single Responsibility Principle
- ✅ Independent composition
- ✅ Clear naming
- ✅ Performance transparency
- ✅ Better testability

**Timeline:**
- Update IMPLEMENTATION_GUIDE_V1.md: 2 hours
- Update ARCHITECTURE_SCALE.md: 1 hour
- Update README.md: 30 minutes
- Total: ~3.5 hours

**Ready to proceed with the fix?**
