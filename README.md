# PRIME

**Proactive detection of O-RAN / RRC misconfigurations via implicit inter-parameter constraints extracted from 3GPP specifications.**
PRIME reads the 3GPP RRC specifications (TS 38.331 for NR and TS 36.331 for LTE) and extracts the *implicit* dependencies between configuration parameters that the specifications imply but never state as a single explicit rule. Violating one of these dependencies produces a valid-looking configuration that misbehaves once deployed (handover oscillation, coverage holes, UEs that never sleep). PRIME turns these dependencies into a machine-checkable **constraints database** that an operator can validate any configuration against before it reaches the RAN.

This repository releases two artifacts from the paper:

1. **Constraints database** — 433 implicit constraints extracted by PRIME from TS 38.331 and TS 36.331.
2. **Partial oracle** — 94 known or specification-stated implicit constraints, used as ground truth to measure extraction recall.

---

## Pipeline in brief

PRIME is an offline pipeline that produces a reusable artifact:

```
TS 38.331 / TS 36.331
        │
        ▼
 Parameter Extractor   →  12,090 leaf parameters
        │
        ▼
 Semantic Annotator    →  per-parameter semantic labels (cross-reference augmented)
        │
        ▼
 Pair Generator        →  category-consistent candidate pairs (from ~73M naive pairs)
        │
        ▼
 Constraint Extractor  →  433 constraints  ►  constraints database  (this repo)
```

The database is the output. Runtime enforcement is left to the network operator: an
online controller validates a proposed configuration against the database and can reject
the request, return it with a suggested fix, or accept it and raise an alarm for review.

---

## Constraint taxonomy

Every constraint belongs to one of five structural categories.

| ID | Category         | What it captures                                              | Example |
|----|------------------|---------------------------------------------------------------|---------|
| C1 | Opposite-action  | Two rules on the same metric and state, opposite directions   | A1 vs A2 thresholds on serving RSRP |
| C2 | Cross-state      | The same quantity bounded in two different RRC states         | Idle-mode selection vs connected-mode maintenance |
| C3 | Timing           | Two timers/counters whose durations must be ordered           | time-to-trigger vs T310 |
| C4 | Cross-metric     | One decision governed by thresholds on different metrics      | RSRP-based vs RSRQ-based threshold for one action |
| C5 | Sequential-stage | One stage produces an output a later stage requires (presence)| Measurement config omits a gap a later stage needs |

C1–C4 are value relations (a checkable inequality). C5 is a presence relation (co-presence of two IEs).

---

## Repository layout

<!-- TODO: adjust to your actual file names / formats -->
```
PRIME/
├── README.md
├── constraints/
│   └── constraints.json        # 433 extracted constraints
├── oracle/
│   └── partial_oracle.json     # 94 ground-truth constraints
├── schema/
│   ├── constraint.schema.json  # JSON schema for a constraint record
│   └── oracle.schema.json      # JSON schema for an oracle entry
└── LICENSE
```

---

## Constraints database

`constraints/constraints.json` is a list of constraint records. Value constraints (C1–C4)
and presence constraints (C5) share most fields but differ in how the relation is expressed.

**Value constraint (C1–C4):**

```json
{
  "id": "C1-0001",
  "category": "C1",
  "param_a": "a1-Threshold",
  "param_b": "a2-Threshold",
  "source_spec": "TS 38.331",
  "direct_inequality": "a1-Threshold >= a2-Threshold",
  "violation": "a1-Threshold < a2-Threshold",
  "description": "A1 and A2 trigger on serving-cell RSRP in opposite directions; if a1-Threshold is set below a2-Threshold the two events overlap over a range of RSRP values and the cell oscillates."
}
```

**Presence constraint (C5):**

```json
{
  "id": "C5-0007",
  "category": "C5",
  "param_a": "measObjectNR (inter-frequency)",
  "referenced_entity": "measGapConfig",
  "source_spec": "TS 38.331",
  "relation": "co-presence",
  "violation": "measObjectNR present AND measGapConfig absent",
  "description": "An inter-frequency measurement object requires a measurement gap; without it the UE cannot measure the configured frequency."
}
```

**Field reference**

| Field | Applies to | Meaning |
|-------|-----------|---------|
| `id` | all | Stable identifier, `<category>-<number>`. |
| `category` | all | One of C1–C5. |
| `param_a` | all | First parameter (or the requiring IE, for C5). |
| `param_b` | C1–C4 | Second parameter in the value relation. |
| `referenced_entity` | C5 | The IE whose presence is required. |
| `source_spec` | all | Specification the parameters are declared in. |
| `direct_inequality` | C1–C4 | The relation that must hold. May embed a spec constant (e.g. `t310 > n311 * 20ms`) evaluated as a formula. |
| `relation` | C5 | Presence relation type (currently `co-presence`). |
| `violation` | all | The condition that flags a misconfiguration (negation of the constraint). |
| `description` | all | Human-readable explanation of the dependency and its failure mode. |

---

## Partial oracle

`oracle/partial_oracle.json` is a benchmark of 94 implicit constraints that a correct
extractor *should* recover. It is used in the paper to measure recall. It is **partial** by
construction: it captures known, specification-stated, and constructed cases, not the full
space of implicit dependencies, so recall against it is a lower bound on coverage.

Each entry is drawn from one of three sources:

- **Conformance** — requirements the specifications state explicitly, which a configuration can violate by omission.
- **Real-world** — misconfigurations documented in published 4G/5G measurement studies (`reference` cites the source).
- **Synthetic** — cases instantiated directly from the taxonomy by turning a category signature into a concrete parameter relationship.

```json
{
  "id": "RW-01",
  "description": "5G SCell add threshold set below the remove threshold creates an add/remove overlap loop.",
  "parameters": ["b1-Threshold", "a2-Threshold"],
  "category": "C1",
  "source": "real-world",
  "reference": "TODO: citation",
  "notes": "theta_B1 must be > theta_A2"
}
```

<!-- TODO: confirm the source-wise breakdown of the 94 entries (conformance / real-world / synthetic) -->

---

## Using the constraints

The database is a data artifact, not a runtime component. PRIME does not ship an enforcement
controller; building one is the operator's responsibility. A minimal check is deterministic
and needs no language model at runtime:

```python
import json

db = json.load(open("constraints/constraints.json"))

def check(config: dict, db: list) -> list:
    """Return the constraints a proposed config violates.
    config maps parameter/IE names to values (or presence flags)."""
    violations = []
    for c in db:
        if c["category"] == "C5":
            requiring = c["param_a"]
            required  = c["referenced_entity"]
            if config.get(requiring) is not None and config.get(required) is None:
                violations.append(c)
        else:
            # C1–C4: evaluate c["violation"] over the proposed values.
            # Bind param_a / param_b (and any embedded spec constant) and test the relation.
            if evaluate(c["violation"], config):   # your evaluator
                violations.append(c)
    return violations
```

An operator can then reject the request, return it with the violated constraint and the
direction the offending parameter must move, or accept and raise an alarm, according to
their own policy.

---

## Citation

<!-- TODO: replace with the final BibTeX entry -->
```bibtex
@inproceedings{prime,
  title     = {PRIME: ...},
  author    = {...},
  booktitle = {...},
  year      = {...}
}
```

---

## License

<!-- TODO: choose a license. A common split for research artifacts is data under
     CC BY 4.0 and any code under MIT. Note that the constraints are derived from
     3GPP specifications; check 3GPP redistribution terms for the descriptive text. -->
TBD.

---

## Disclaimer

The constraints are extracted automatically and validated against a partial oracle; they are
a research artifact, not a certified configuration-safety tool. Validate against your own
deployment requirements before relying on them.
