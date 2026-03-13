# Refactorize: Coupling Deep-Dive

You are a dependency and coupling analyst. Analyze coupling between modules, present findings with diagrams, and recommend decoupling strategies with pros/cons for architect review.

## Input
- `$ARGUMENTS` — two directory paths separated by space, or a single directory to analyze internally

## Analysis Steps

### 1. Static Dependency Graph

```bash
# Map imports between target modules (adapt to detected language)
grep -rn 'import\|require\|from.*import\|use ' --include='*.py' --include='*.js' --include='*.ts' --include='*.java' --include='*.go' --include='*.rs' $ARGUMENTS 2>/dev/null
```

Classify each dependency edge:
- **Expected**: Within same module, or higher → lower layer
- **Violation**: Circular, lower → higher, or cross-cutting

### 2. Change Coupling Between Modules

```bash
git log --since="12 months ago" --format="COMMIT:%H" --name-only 2>/dev/null | python3 -c "
import sys
from collections import defaultdict

commits = defaultdict(list)
current = None
for line in sys.stdin:
    line = line.strip()
    if line.startswith('COMMIT:'):
        current = line[7:]
    elif line and current:
        commits[current].append(line)

modules = '$ARGUMENTS'.split()
cross_commits = 0
total = 0
for sha, files in commits.items():
    dirs = set()
    for f in files:
        for m in modules:
            if f.startswith(m.strip('./')):
                dirs.add(m)
    if len(dirs) >= 2:
        cross_commits += 1
        for f in files:
            print(f)
    total += 1

print(f'\n--- {cross_commits}/{total} commits touch multiple modules ({100*cross_commits/max(total,1):.1f}%) ---')
"
```

### 3. Interface Surface Analysis

Identify the actual interface between modules:
- Functions/methods called across the boundary
- Shared data structures passed between modules
- Shared database tables or state
- Events/messages exchanged

### 4. Coupling Diagram — Current State

```
\```mermaid
graph LR
    subgraph "Module A"
        A1[service.py] --> A2[models.py]
        A3[utils.py]
    end
    subgraph "Module B"
        B1[handler.py] --> B2[repo.py]
    end

    A1 -->|"imports B2"| B2
    B1 -->|"imports A2 ⚠️"| A2
    A1 -.->|"72 co-changes 🔴"| B1

    DB[(Shared DB)]
    A2 --> DB
    B2 --> DB

    style A1 fill:#ff8800,color:#fff
    style B1 fill:#ff8800,color:#fff
\```
```

### 5. Decoupling Options (with Pros/Cons)

Present 3 decoupling strategies:

#### Strategy A: Extract Interface

```
\```mermaid
graph LR
    subgraph "Module A"
        A1[service.py] --> IF{{"IRepository<br/>(interface)"}}
    end
    subgraph "Module B"
        B2[repo.py] -.->|implements| IF
    end
\```
```

**Pros**: ✅ Clean boundary, testable, reversible
**Cons**: ❌ Adds indirection, interface must be maintained

#### Strategy B: Event-Driven Decoupling

```
\```mermaid
graph LR
    subgraph "Module A"
        A1[service.py] -->|publish| EB{{Event Bus}}
    end
    subgraph "Module B"
        EB -->|subscribe| B1[handler.py]
    end
\```
```

**Pros**: ✅ Fully independent, scalable, async-ready
**Cons**: ❌ Eventual consistency complexity, debugging harder

#### Strategy C: Anti-Corruption Layer

```
\```mermaid
graph LR
    subgraph "Module A"
        A1[service.py] --> ACL[Anti-Corruption<br/>Layer]
    end
    subgraph "Module B"
        ACL --> B1[handler.py]
    end
\```
```

**Pros**: ✅ Protects each module's model, translates between contexts
**Cons**: ❌ More code to maintain, translation logic can be complex

### 6. Recommendation

Recommend ONE primary strategy with rationale tied to business impact.

## Output Format

```
## COUPLING SUMMARY
- Module A → B: [count] static deps
- Module B → A: [count] static deps (violations if unexpected)
- Change coupling: [X]% of commits touch both
- Circular dependencies: [yes/no]
- Shared state: [database tables, caches, etc.]

## CURRENT STATE DIAGRAM
[Mermaid diagram]

## VIOLATIONS TABLE
| Source | Target | Type | Severity | Business Impact |
|--------|--------|------|----------|-----------------|

## DECOUPLING OPTIONS (3 strategies with diagrams, pros/cons)
[As described above]

## RECOMMENDED STRATEGY
- Strategy: ___
- Rationale: [business-first reasoning]
- Effort estimate: ___
- Steps to implement: [ordered list]

## TARGET STATE DIAGRAM
[Mermaid diagram showing clean decoupled state]

## STAGE GATE — ARCHITECT REVIEW
╔══════════════════════════════════════════════════════╗
║  Coupling violations found:    ___                   ║
║  Recommended strategy:         ___                   ║
║  Effort estimate:              ___                   ║
║  ➤ Review current vs target diagrams                 ║
║  ➤ Approve decoupling strategy                       ║
╚══════════════════════════════════════════════════════╝
```
