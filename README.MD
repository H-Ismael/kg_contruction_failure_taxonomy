# You Cannot Rush a Knowledge Graph: A Multi-Layer Failure Taxonomy for KG Construction

Knowledge graph construction is one of those things that cannot be rushed out -- even with the illusion of help from modern LLMs. Unlike a classification model you can retrain overnight or a prompt you can iterate in minutes, a knowledge graph is a **committed structure**. Every node you create, every edge you draw, every merge you approve becomes load-bearing scaffolding for every query, every inference, and every downstream decision that follows. It is not easily retractable, and it is not time-compressible.

This article presents a **multi-layer, nested failure taxonomy** for knowledge graph construction -- a systematic catalogue of everything that can go wrong when you extract entities, resolve coreferences, assign types, link nodes, and govern a schema at scale. It is a compilation that draws from both the **pre-LLM era** (where entity resolution, deduplication, and ontology management were already hard, well-studied problems) and the **post-LLM era** (where new failure modes like hallucinated triples, schema drift, and prompt sensitivity have been added to an already long list).

The taxonomy is not academic. It was built to be **directly implementable**: each failure mode maps to a concrete check -- some enforced by Pydantic validators, some by regex, some by graph algorithms, and some, only when necessary, by the LLM itself. It is designed to generate **dynamic sanity checks** that fire during construction and **static sanity checks** that sweep the graph during maintenance.

---

## Why Knowledge Graphs Resist Shortcuts

There is a tempting narrative in the LLM era: point a large language model at your documents, ask it to extract entities and relations, and you have a knowledge graph. The reality is different, and the reasons are structural.

**Errors compound, they don't average out.** In a vector database, a bad embedding for one document is a local problem -- it makes that document harder to retrieve, but it doesn't corrupt the representation of other documents. In a knowledge graph, a wrongly merged entity poisons every edge connected to it. A mistyped node cascades into invalid relations. A hallucinated triple becomes indistinguishable from a real one once it sits in the store.

**Merges are irreversible by default.** When you merge two entity nodes because they look like the same person, you inherit all edges from both. If the merge was wrong, separating them requires knowing which edges belonged to which original node -- information that is lost unless you explicitly preserved it. Most pipelines do not.

**The schema is the contract.** Every predicate you define, every type hierarchy you establish, becomes a public API for every downstream consumer. Schema drift -- where the extraction pipeline silently starts producing slightly different labels or structures -- breaks queries, dashboards, and reasoning chains in ways that are hard to diagnose.

**LLMs add new failure modes on top of old ones.** The pre-LLM failure catalogue (boundary errors, coreference mistakes, deduplication failures, temporal reasoning gaps) is not replaced by LLMs -- it is *extended*. You still have all the old problems, plus hallucination, prompt sensitivity, schema drift, and the tendency of helpful models to invent plausible-sounding relations that are not supported by the source text.

---

## The Anatomy of a KG Failure

Before diving into the taxonomy, it helps to understand how failures nest. A single bad extraction can trigger a cascade:

```
[Source text]
  "Le directeur de Zabala nous a confirmé que le projet est annulé."

[Span Extraction]
  mention: "Zabala" ✓
  mention: "le projet" -- but which project? (coreference failure)

[Entity Typing]
  "Zabala" → ORGANIZATION ✓
  "le projet" → ... missing link to antecedent (cascading error)

[Relation Extraction]
  CONFIRMED(director_of_Zabala, project_cancelled) 
  -- but "annulé" is the status, not a relation (modality miss)
  -- and the director is an unnamed PERSON, not "Zabala" (directionality error)

[Graph State]
  Node: Zabala (ORGANIZATION)
  Edge: Zabala --CONFIRMED--> project_cancelled  ← wrong subject, wrong semantics
```

One extraction. Four nested failures. Each one individually documented in the taxonomy, each one addressable by a different check strategy.

---

## The Taxonomy: 13 Domains, 35+ Failure Modes

The full taxonomy (available as a [structured JSON](../raw_data/kg_construction_failure_taxo.json) and as a [detailed technical reference](./kg_failure_taxonomy.md)) covers 13 domains:

| # | Domain | What can go wrong |
|---|---|---|
| 1 | **Mention Detection** | Partial spans, missed entities, spurious mentions |
| 2 | **Entity Typing** | Wrong coarse type, wrong fine-grained type |
| 3 | **Entity Normalisation & Linking** | Duplicate nodes, over-merging, alias ambiguity, NIL failures, granularity mismatch |
| 4 | **Coreference Resolution** | Pronoun ambiguity, nominal mismatch, event splits, abstract reference errors, split antecedents, cataphora |
| 5 | **Relation Extraction** | Wrong predicate, inverted direction, negation/speculation missed, hallucinated relations |
| 6 | **Event Extraction** | Status collapse (planned stored as completed), missing participants |
| 7 | **Temporal Reasoning** | Missing timestamps, wrong validity intervals |
| 8 | **Ontology & Schema** | Predicate duplication, instance/class/attribute confusion |
| 9 | **Deduplication & Consistency** | Duplicate edges, contradictory merges |
| 10 | **Provenance, Conflict & Confidence** | Missing evidence trails, suppressed conflicts, miscalibrated scores |
| 11 | **Chunking & Context** | Cross-chunk entity splits, cross-chunk relation loss |
| 12 | **LLM-Specific Failures** | Hallucinated facts, schema drift, prompt sensitivity |
| 13 | **Evaluation & Operations** | Metric illusions, irreversible merges, silent drift |

Each failure mode is documented with its **typical causes**, **downstream impact**, and **resolution strategies** -- creating a decision tree that maps directly to code.

---

## Two Rhythms of Checking: Dynamic and Static

Not all checks belong in the same place. The taxonomy distinguishes two complementary rhythms:

### Dynamic Checks -- Construction-Time Guards

These fire **synchronously** in the extraction pipeline, *before* a triple reaches the graph store. They must be fast, deterministic where possible, and they block bad data at the gate.

```
[LLM Output] → [JSON Parse] → [Pydantic Validation] → [Regex Filters] → [Dedup Hash] → [Graph Insert]
                                      ↓ reject                ↓ reject          ↓ skip
                                 [Retry with repair]     [Flag for review]   [Aggregate evidence]
```

**Examples of dynamic checks:**

- A Pydantic model rejects a relation where `modality` is `"factual"` but the evidence span contains negation cues like *"n'est plus"* or *"no longer"*.
- A regex filter strips punctuation from mention boundaries before they can create malformed nodes.
- An edge hash prevents the same `(subject, predicate, object)` triple from being inserted twice when overlapping chunks produce duplicate extractions.
- A type-constraint validator rejects `WORKS_AT(ORGANIZATION, PERSON)` because the directionality is inverted.

### Static Checks -- Maintenance-Time Sweeps

These run **asynchronously** on a schedule -- after a batch ingestion, nightly, or on-demand. They operate on the full graph and can afford to be expensive.

```
[Persisted Graph] → [Cycle Detection] → [Duplicate Clustering] → [Temporal Consistency]
                          ↓                      ↓                         ↓
                   [Flag ontology issue]   [Merge candidates]      [Stale fact alerts]
```

**Examples of static checks:**

- Connected-component analysis on a similarity graph of entity names surfaces duplicate clusters that slipped past construction-time dedup.
- Temporal overlap detection finds two edges asserting contradictory facts about the same entity at the same time (e.g., a person simultaneously `WORKS_AT` two competing organisations with overlapping validity intervals).
- Schema audits detect predicate proliferation -- seven different predicates that all mean "is a competitor of".
- Regression suites compare extraction output on frozen benchmarks before and after a model upgrade, prompt change, or schema modification.

---

## Four Resolution Strategies

Every failure mode in the taxonomy maps to one or more resolution strategies. The key insight is that **no single strategy covers all failure types** -- and choosing the wrong one for a given failure mode creates a false sense of safety.

### 1. Pydantic / Schema Validation

**Best for:** structural invariants, type safety, mandatory fields, enum enforcement, hierarchical consistency.

Pydantic models are the first line of defence. They are deterministic, self-documenting, and they catch the most common failures -- wrong types, missing provenance, schema violations -- at near-zero latency.

```python
class ExtractedRelation(BaseModel):
    subject: str
    predicate: Literal[
        "WORKS_AT", "COMPETITOR_OF", "MENTIONS",
        "DECIDED_BY", "FOLLOWED_UP", "OBJECTED_TO",
    ]
    object: str
    modality: Literal["factual", "negated", "speculative", "planned"]
    evidence_span: str
    source_doc_id: str
```

This single model prevents: predicate invention (closed `Literal` set), missing provenance (`source_doc_id` required), modality flattening (`modality` is mandatory, not optional), and free-form hallucination (every triple must cite its `evidence_span`).

**Limitation:** Pydantic cannot tell you that an evidence span *doesn't actually support* the extracted relation. That requires a different strategy.

### 2. Regex / Pattern Rules

**Best for:** surface normalisation, negation cue detection, spurious mention filtering, temporal expression extraction, canonical form computation.

Regex rules are transparent, unit-testable, and fast. They handle the mechanical aspects of text processing that don't require understanding.

```python
NEGATION_CUES = re.compile(
    r"\b(ne\s+\w+\s+pas|n'est plus|not|never|no longer|"
    r"might not|won't|wouldn't|didn't)\b",
    re.IGNORECASE,
)

def flag_negation(evidence: str, declared_modality: str) -> bool:
    """Returns True if evidence contains negation but modality claims factual."""
    return declared_modality == "factual" and bool(NEGATION_CUES.search(evidence))
```

A simple regex like this catches a dangerous class of errors -- relations stored as factual when the source text explicitly negates them -- with zero LLM cost and perfect determinism.

**Limitation:** Regex breaks on unseen patterns, complex negation scoping, and double negatives. It is a necessary first pass, not a sufficient one.

### 3. Algorithmic (Graph-Theoretic)

**Best for:** cycle detection, deduplication, connected-component clustering, temporal consistency, contradiction detection, schema constraint validation.

These are the checks that operate on the graph *as a structure*, independent of the text that produced it.

```python
import hashlib
import networkx as nx

def detect_ontology_cycles(type_hierarchy: nx.DiGraph) -> list[list[str]]:
    """Cycles in the type hierarchy indicate modelling errors."""
    return list(nx.simple_cycles(type_hierarchy))

def edge_fingerprint(subj: str, pred: str, obj: str) -> str:
    """Deterministic dedup hash for triple identity."""
    return hashlib.sha256(f"{subj}|{pred}|{obj}".encode()).hexdigest()
```

Cycle detection in the ontology layer, connected-component analysis for entity deduplication, temporal interval overlap checks -- these are problems with well-known polynomial-time algorithms. They are provably correct, scalable, and they catch structural invariant violations that no amount of LLM reasoning can reliably detect.

**Limitation:** Algorithms need well-defined properties to check. They cannot tell you whether "the CEO" and "our main contact" refer to the same person -- that requires semantic reasoning.

### 4. LLM-Based Verification

**Best for:** semantic disambiguation, paraphrase detection, conflict summarisation, ambiguity flagging, coreference resolution, abstract reference identification.

LLMs are the strategy of last resort -- reserved for problems that are inherently semantic and cannot be reduced to patterns, schemas, or graph properties.

The critical discipline is: **use LLMs as constrained classifiers over candidates, never as free generators.** An LLM asked "extract all relations from this text" will hallucinate. An LLM asked "given these 3 candidate predicates and this evidence span, which one applies, or none?" is far more reliable.

```python
def llm_disambiguate(
    mention: str,
    candidates: list[dict],
    context: str,
) -> str | None:
    """
    Ask the LLM to pick among candidates or abstain.
    Returns candidate ID or None (= NIL / insufficient evidence).
    """
    prompt = f"""Given the mention "{mention}" in context:
    ---
    {context}
    ---
    Which of these entities does it refer to?
    {json.dumps(candidates, indent=2)}

    Respond with the entity ID, or "NONE" if there is insufficient evidence.
    """
    response = call_llm(prompt)
    if response.strip() == "NONE":
        return None
    return response.strip()
```

**Limitation:** Non-deterministic, expensive, and prone to hallucination. LLM-based checks must always be auditable (log the prompt, the response, and the decision) and should never be the sole gatekeeper for high-stakes operations like entity merges.

---

## Pre-LLM Meets Post-LLM

This failure taxonomy is a **compilation of two eras**.

The pre-LLM knowledge graph community spent decades studying entity resolution, ontology alignment, deduplication, temporal reasoning, and schema governance. These problems did not go away when GPT arrived. They are still there, still hard, and the solutions that work -- normalised forms, constraint validation, graph algorithms, fuzzy matching -- are still the right answers.

What the LLM era added is a new *source* of extractions (the LLM itself) and a new *class* of failures specific to that source:

| Era | Failure type | Example |
|---|---|---|
| **Pre-LLM** | Boundary errors | NER model cuts "Jean-Pierre" into "Jean" and "Pierre" |
| **Pre-LLM** | Coreference failure | Pronoun "il" resolved to wrong antecedent |
| **Pre-LLM** | Deduplication miss | "IBM" and "I.B.M." stored as separate nodes |
| **Pre-LLM** | Temporal gap | "CEO since 2019" stored without validity interval |
| **Post-LLM** | Hallucinated triple | LLM invents a relation not stated in the text |
| **Post-LLM** | Schema drift | LLM outputs `"works_for"` instead of `"WORKS_AT"` on the next run |
| **Post-LLM** | Prompt sensitivity | Minor prompt rewording changes 15% of extraction outputs |
| **Post-LLM** | Helpfulness bias | LLM "helpfully" infers a relation from world knowledge rather than the source |
| **Both eras** | Over-merge | Two entities with the same name but different identities are merged |
| **Both eras** | Negation miss | "does not work at Acme" stored as `WORKS_AT(person, Acme)` |

The taxonomy treats both eras with equal seriousness. An LLM can introduce hallucinated triples, but a regex can catch the negation cue that the LLM missed. A Pydantic model can enforce the closed predicate set that prevents schema drift. A graph algorithm can detect the cycle that neither the LLM nor the regex would notice. **Defence in depth requires all four layers.**

---

## From Taxonomy to Implementation: The Three Phases

### Phase 1 -- Construction-Time Guards

These are the dynamic checks that run synchronously in the extraction pipeline.

**What to build first:**

1. **Pydantic models for every extraction output.** No triple enters the graph without passing through a typed, validated model. This alone catches wrong types, missing fields, schema violations, and directional errors.

2. **Regex filters as pre-validators.** Boundary cleanup, negation detection, spurious mention blacklists, canonical form normalisation. These are cheap, deterministic, and they reduce the load on downstream checks.

3. **Edge hashing at insert time.** A SHA-256 hash on `(subject_id, predicate, object_id)` prevents the most common form of duplication -- overlapping chunks producing identical triples.

4. **LLM output validation loop.** Parse the raw LLM response, validate against the Pydantic schema, and if validation fails, send the error message back to the LLM for a repair attempt. Cap retries at 2-3 to avoid infinite loops.

### Phase 2 -- Maintenance Sweeps

These are the static checks that run asynchronously.

**What to schedule:**

1. **Graph-algorithm sweeps.** Cycle detection on the ontology layer. Connected-component analysis for duplicate entity clusters. Temporal consistency checks (overlapping validity intervals on the same relation). Contradiction detection (conflicting attribute values on merged nodes).

2. **Schema health audits.** Predicate distribution analysis -- are any predicates suspiciously rare or suspiciously common? Unused predicate detection. Type-distribution anomaly detection.

3. **Calibration and drift monitoring.** Run the extraction pipeline on a frozen benchmark after every model or prompt change. Compute regression deltas. Set up change-point alerts on key metrics (mention recall, relation precision, dedup rate).

4. **Sample-based recall audits.** Pull a random sample of source documents, run manual annotation, and compare against the graph. This is the only way to catch `missed_mention` failures -- by definition, they are not in the graph to be found algorithmically.

### Phase 3 -- LLM-Assisted Triage

LLM-based checks are the most expensive and least deterministic. They belong in the third phase, not the first.

**Where to use LLMs in maintenance:**

1. **Semantic deduplication.** Group paraphrased evidence spans that support the same triple. This consolidates the provenance layer without losing information.

2. **Conflict summarisation.** When two sources assert contradictory facts about the same entity, generate a human-readable summary for the reviewer rather than silently picking one.

3. **Ambiguity flagging.** Surface cases where entity linking, typing, or coreference confidence is below threshold and a human decision is needed. The LLM's role is to *explain* the ambiguity, not to resolve it.

4. **Drift explanation.** After a regression is detected by the algorithmic checks, use an LLM to generate a natural-language summary of what changed and why, for the team's review.

---

## A Practical Example: The Negation Trap

Consider this real-world failure pattern from sales transcripts:

> **Transcript:** *"Nous ne travaillons plus avec Ayming depuis 2023."*
> ("We no longer work with Ayming since 2023.")

Without the taxonomy, a naive extraction pipeline produces:

```
WORKS_WITH(our_company, Ayming) -- modality: factual
```

This is the `negation_speculation_miss` failure (Domain 5, Relation Extraction). The taxonomy prescribes a layered defence:

**Layer 1 -- Regex (dynamic):** The negation cue `"ne...plus"` is detected by pattern matching. The check fires *before* the triple is validated, flagging a mismatch between the detected negation and the `"factual"` modality.

**Layer 2 -- Pydantic (dynamic):** The `ModalRelation` model requires a `modality` field. If the LLM outputs `"factual"`, but the regex layer has already flagged negation cues, the model validator raises a `ValueError`. The triple is rejected.

**Layer 3 -- LLM (dynamic, retry):** The LLM is re-prompted with the error message: *"Evidence span contains negation cues but modality is 'factual'. Please re-classify."* On retry, the LLM corrects to `"negated"`.

**Layer 4 -- Temporal (dynamic):** The temporal extraction pass picks up `"depuis 2023"` and assigns a `valid_to: 2023` on the `WORKS_WITH` relation, turning it into a historically bounded, negated fact.

**Result:** The graph correctly stores:

```
WORKS_WITH(our_company, Ayming) -- modality: negated, valid_to: 2023
```

Four layers. Four different strategies. Each one catches something the others miss.

---

## Why This Matters for Teams

The taxonomy is not just a technical artefact -- it is a **communication tool**.

**For incident response:** When a graph anomaly is discovered, the taxonomy provides a shared language. Instead of "the graph has wrong data", the team can say "we have a `directionality_error` on the `DECIDED_BY` predicate -- the subject and object types are inverted." This immediately tells the developer which validator to check, which regex to audit, and which LLM prompt to review.

**For onboarding:** New team members can read the taxonomy to understand the full surface area of things that can go wrong -- before they encounter them in production. Each failure mode comes with its causes, its impact, and its resolution strategy.

**For prevention:** When adding a new entity type, predicate, or extraction step, the team can walk through the taxonomy and ask: "which of these failure modes does this change introduce or exacerbate?" This turns the taxonomy into a **design review checklist**.

**For continuous improvement:** When a new failure pattern is discovered in production, it is added to the taxonomy with its domain, causes, impact, and resolution strategy. The taxonomy grows with the system, and every addition triggers a concrete implementation task -- a new validator, a new regex, a new graph check.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                     Source Documents                             │
│              (Transcripts, Emails, CRM Notes)                   │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                   LLM Extraction Pipeline                       │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐   │
│  │ Mention   │→│  Entity   │→│ Relation  │→│    Event      │   │
│  │ Detection │  │  Typing   │  │Extraction │  │  Extraction   │  │
│  └──────────┘  └──────────┘  └──────────┘  └──────────────┘   │
└──────────────────────────┬──────────────────────────────────────┘
                           │ raw LLM output
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│              DYNAMIC SANITY CHECKS (Phase 1)                    │
│                                                                 │
│  ┌─────────────────┐  ┌──────────────┐  ┌─────────────────┐   │
│  │ Pydantic Schema  │  │ Regex Filters │  │  Edge Hashing   │   │
│  │ Validation       │  │ (negation,    │  │  (dedup at      │   │
│  │ (types, enums,   │  │  boundaries,  │  │   insert)       │   │
│  │  provenance,     │  │  blacklists,  │  │                 │   │
│  │  directionality) │  │  normalise)   │  │                 │   │
│  └────────┬────────┘  └──────┬───────┘  └────────┬────────┘   │
│           │ reject/retry      │ flag/reject        │ skip/merge │
│           └───────────────────┴────────────────────┘            │
└──────────────────────────┬──────────────────────────────────────┘
                           │ validated triples
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Graph Store (Neo4j)                           │
└──────────────────────────┬──────────────────────────────────────┘
                           │ persisted graph
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│              STATIC SANITY CHECKS (Phase 2)                     │
│                                                                 │
│  ┌───────────────┐  ┌────────────────┐  ┌──────────────────┐  │
│  │  Graph Algos   │  │ Schema Audits   │  │ Drift Monitoring │  │
│  │  (cycles,      │  │ (predicate      │  │ (regression      │  │
│  │   components,  │  │  distribution,  │  │  suite, change   │  │
│  │   temporal     │  │  type anomaly)  │  │  point alerts)   │  │
│  │   consistency) │  │                 │  │                  │  │
│  └───────────────┘  └────────────────┘  └──────────────────┘  │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │            LLM-Assisted Triage (Phase 3)                  │  │
│  │  (semantic dedup, conflict summaries, ambiguity flags)     │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

---

## The Full Failure Map

For the complete technical reference -- every failure mode with its causes, impact, check timing, resolution strategies, and code sketches -- see the companion document:

**[KG Construction Failure Taxonomy -- Technical Reference](./kg_failure_taxonomy.md)**

For the raw structured taxonomy data (suitable for programmatic consumption):

**[kg_construction_failure_taxo.json](../raw_data/kg_construction_failure_taxo.json)**

---

## Key Takeaways

1. **Knowledge graphs cannot be rushed.** Errors compound, merges are irreversible by default, and schema drift breaks downstream consumers silently. The cost of fixing a corrupted graph is orders of magnitude higher than preventing the corruption.

2. **LLMs are a tool, not a solution.** They add new extraction capabilities and new failure modes in equal measure. Every LLM output must pass through deterministic validation before it touches the graph.

3. **Defence in depth requires four layers.** Pydantic for structure, regex for patterns, algorithms for graph invariants, LLMs for semantics. No single layer is sufficient.

4. **Dynamic and static checks are complementary.** Construction-time guards block the most common failures. Maintenance-time sweeps catch the ones that slip through and detect slow degradation.

5. **The taxonomy is a living document.** Every production failure that is not already in the taxonomy is a gap. Adding it -- with its causes, impact, and resolution strategy -- is as important as fixing the immediate bug.

6. **Shared vocabulary prevents shared blindness.** When the whole team speaks the same language about failure modes, incidents are triaged faster, design reviews are more thorough, and the gap between "it works on my examples" and "it works in production" gets smaller.

---
