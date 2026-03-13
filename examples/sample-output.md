# Example Output: /refactorize on a Django E-Commerce Monolith

> This is a representative example of what `/refactorize` produces when run against a ~200K LOC Django monolith.

---

## STAGE 1: BUSINESS CONTEXT & CODEBASE CENSUS

### Business Context
- **Pain points**: Releases take 2 weeks, payment bugs affect all modules, 3 teams blocked by shared `core/` module
- **Teams**: 3 (Platform, Payments, Growth)
- **Deploy frequency**: Bi-weekly
- **Lead time**: ~10 days

### Codebase Profile
- **Total LOC**: 187,432 (Python: 142K, JavaScript: 38K, HTML: 7K)
- **Top-level modules**: 12 directories
- **Build system**: Django, pip, webpack
- **Architecture**: Layered monolith trending toward ball-of-mud

### Architecture Overview

```mermaid
graph TB
    subgraph "Django Monolith"
        AUTH[auth/<br/>8,200 LOC]
        USERS[users/<br/>12,400 LOC]
        ORDERS[orders/<br/>34,600 LOC]
        PAYMENTS[payments/<br/>18,900 LOC]
        CATALOG[catalog/<br/>22,100 LOC]
        SHIPPING[shipping/<br/>11,300 LOC]
        NOTIFY[notifications/<br/>6,800 LOC]
        CORE[core/<br/>28,100 LOC]
    end

    AUTH --> CORE
    USERS --> CORE
    ORDERS --> CORE
    ORDERS --> PAYMENTS
    ORDERS --> SHIPPING
    ORDERS --> CATALOG
    PAYMENTS --> CORE
    CATALOG --> CORE
    SHIPPING --> CORE
    NOTIFY --> CORE

    DB[(PostgreSQL)] --- CORE
    REDIS[(Redis)] --- ORDERS
    STRIPE[Stripe API] --- PAYMENTS
    BROWSER[Users] --> AUTH
    BROWSER --> ORDERS
    BROWSER --> CATALOG
```

### Stage Gate 1

```
╔══════════════════════════════════════════════════════════════╗
║  STAGE GATE 1: CODEBASE CENSUS COMPLETE                     ║
╠══════════════════════════════════════════════════════════════╣
║                                                              ║
║  Total LOC:        187,432                                   ║
║  Languages:        Python (76%), JavaScript (20%), HTML (4%) ║
║  Top-level modules: 12                                       ║
║  Build system:     Django + webpack                          ║
║  Architecture:     Layered monolith (ball-of-mud tendencies) ║
║                                                              ║
║  Business context:                                           ║
║  • Pain points:    Slow releases, payment bugs cascade,      ║
║                    teams blocked by shared core               ║
║  • Teams:          3 (Platform, Payments, Growth)            ║
║  • Deploy freq:    Bi-weekly                                 ║
║                                                              ║
╚══════════════════════════════════════════════════════════════╝
```

---

## STAGE 2: HOTSPOT ANALYSIS

| # | File | Commits (12mo) | LOC | Score | Business Impact |
|---|------|----------------|-----|-------|-----------------|
| 1 | orders/processor.py | 142 | 2,340 | 🔴 CRITICAL | Blocks every sprint |
| 2 | payments/service.py | 98 | 1,870 | 🔴 CRITICAL | 3 prod incidents in 6mo |
| 3 | core/models.py | 87 | 1,640 | 🔴 CRITICAL | All 3 teams conflict here |
| 4 | orders/views.py | 76 | 1,200 | 🟠 HIGH | Merge conflicts weekly |
| 5 | catalog/search.py | 54 | 980 | 🟠 HIGH | Performance bottleneck |
| 6 | users/permissions.py | 48 | 720 | 🟡 MEDIUM | Growing complexity |
| 7 | shipping/calculator.py | 41 | 560 | 🟡 MEDIUM | New carrier integration |

### Hotspot Heatmap

```mermaid
graph TB
    subgraph "🔴 Critical Hotspots"
        H1["orders/processor.py<br/>142 commits | 2,340 LOC"]
        H2["payments/service.py<br/>98 commits | 1,870 LOC"]
        H3["core/models.py<br/>87 commits | 1,640 LOC"]
    end
    subgraph "🟠 High Priority"
        H4["orders/views.py<br/>76 commits | 1,200 LOC"]
        H5["catalog/search.py<br/>54 commits | 980 LOC"]
    end
    subgraph "🟡 Watch List"
        H6["users/permissions.py<br/>48 commits | 720 LOC"]
        H7["shipping/calculator.py<br/>41 commits | 560 LOC"]
    end

    style H1 fill:#ff4444,color:#fff
    style H2 fill:#ff4444,color:#fff
    style H3 fill:#ff4444,color:#fff
    style H4 fill:#ff8800,color:#fff
    style H5 fill:#ff8800,color:#fff
    style H6 fill:#ffcc00,color:#000
    style H7 fill:#ffcc00,color:#000
```

**Key finding**: 3 files (1.6% of codebase) account for 38% of all commits. Every team touches `core/models.py` — this is your primary bottleneck.

---

## STAGE 3: COUPLING & DEPENDENCY ANALYSIS

### Unexpected Coupling Pairs

| Co-changes | File A | File B | Expected? |
|-----------|--------|--------|-----------|
| 72 | orders/processor.py | payments/service.py | 🔴 No — should be independent |
| 54 | core/models.py | orders/views.py | 🟡 Partially — shared models |
| 41 | payments/service.py | notifications/email.py | 🔴 No — payment shouldn't know about email |
| 38 | catalog/search.py | orders/processor.py | 🔴 No — search and orders are separate domains |

### Dependency Map

```mermaid
graph LR
    subgraph "orders/"
        O1[processor.py] --> O2[models.py]
        O3[views.py] --> O1
    end
    subgraph "payments/"
        P1[service.py] --> P2[models.py]
    end
    subgraph "core/"
        C1[models.py]
    end
    subgraph "notifications/"
        N1[email.py]
    end

    O1 -.->|"🔴 72 co-changes"| P1
    P1 -->|"direct import ⚠️"| O2
    O1 -->|"imports"| C1
    P1 -->|"imports"| C1
    P1 -.->|"🔴 41 co-changes"| N1

    style O1 fill:#ff4444,color:#fff
    style P1 fill:#ff4444,color:#fff
    style C1 fill:#ff8800,color:#fff
```

**Circular dependency found**: `payments/service.py` imports `orders/models.py` AND `orders/processor.py` imports `payments/service.py`. This circular dependency makes independent deployment impossible.

---

## STAGE 5: FIVE DECOMPOSITION OPTIONS

### Options Comparison

| Criteria | Opt 1: Modular | Opt 2: Strangler | Opt 3: DDD | Opt 4: Events | Opt 5: Full |
|----------|---------------|-----------------|------------|---------------|-------------|
| Risk | 🟢 Low | 🟡 Med | 🟡 Med | 🟠 High | 🔴 V.High |
| Effort | 2-4 weeks | 2-3 months | 4-6 months | 4-8 months | 6-12 months |
| Team autonomy | Partial | Partial | High | High | Full |
| Deploy independence | No | Partial | Yes | Yes | Yes |
| Infra complexity | None | Low | Medium | Medium | High |
| Reversibility | Easy | Easy | Medium | Medium | Hard |
| First value | 1-2 weeks | 4-6 weeks | 8-12 weeks | 6-10 weeks | 12-16 weeks |

### Recommended Approach

**Primary: Option 3 — DDD Bounded Contexts**

Rationale: With 3 teams already aligned to business domains (Platform, Payments, Growth), bounded context extraction directly addresses the core pain: teams blocking each other in shared code. The data confirms natural domain boundaries — orders, payments, and catalog have distinct ownership patterns. The circular dependency between orders↔payments is the critical blocker to resolve first.

**Fallback: Option 2 — Strangler Fig (Payments first)**

Rationale: If full DDD extraction proves too ambitious, extracting payments alone (the module with the most production incidents) delivers 60% of the reliability benefit at 30% of the cost.

---

*This is a truncated example. The full output includes Stages 4 and 6 with ownership maps, execution roadmap, Gantt chart, and metrics dashboard.*
