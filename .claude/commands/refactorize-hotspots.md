# Refactorize: Hotspot Deep-Dive

You are a code hotspot analyst. Drill into a specific hotspot file or module to identify internal extraction opportunities and present findings with diagrams for architect review.

## Input
- `$ARGUMENTS` — file path or directory to analyze

## Analysis Steps

### 1. Function-Level Hotspot X-Ray (Tornhill Method)

Analyze git history at function granularity to find internal hotspots:

```bash
# All revisions of the file
git log --oneline --follow -- "$ARGUMENTS" 2>/dev/null | head -50

# Diff hunks mapped to functions
git log --since="6 months ago" -p -- "$ARGUMENTS" 2>/dev/null | grep -E '^@@.*@@' | head -50
```

Parse diff hunks to map changes to individual functions. Report which functions change most.

### 2. Complexity Analysis

```bash
# Indentation depth distribution (complexity proxy)
awk '{match($0, /^[[:space:]]*/); print RLENGTH}' "$ARGUMENTS" | sort -n | uniq -c | sort -rn | head -20

# Function/method boundaries and lengths
grep -n 'def \|function \|func \|fn \|public \|private \|protected ' "$ARGUMENTS" 2>/dev/null
```

### 3. Responsibility Analysis

Read the file and identify:
- **Distinct responsibilities**: Group methods by domain concept
- **Data clumps**: Parameters/fields that always appear together
- **Feature envy**: Methods using more data from other classes than their own
- **God class signals**: >7 instance variables, >20 methods, >500 LOC

### 4. Responsibility Cluster Diagram

Present a Mermaid diagram showing the internal structure:

```
\```mermaid
graph TB
    subgraph "Current: MonolithicClass (2,340 LOC)"
        subgraph "🔵 Cluster A: User Auth"
            A1[login] --> A2[validate_token]
            A1 --> A3[refresh_session]
        end
        subgraph "🟢 Cluster B: Profile Mgmt"
            B1[update_profile] --> B2[validate_email]
            B1 --> B3[upload_avatar]
        end
        subgraph "🟠 Cluster C: Notifications"
            C1[send_email] --> C2[format_template]
            C1 --> C3[queue_notification]
        end
        A1 -.->|"tangled"| B1
        B1 -.->|"tangled"| C1
    end
\```
```

### 5. Extraction Plan with Target Diagram

```
\```mermaid
graph TB
    subgraph "After Extraction"
        subgraph "AuthService (480 LOC)"
            A1[login] --> A2[validate_token]
            A1 --> A3[refresh_session]
        end
        subgraph "ProfileService (620 LOC)"
            B1[update_profile] --> B2[validate_email]
            B1 --> B3[upload_avatar]
        end
        subgraph "NotificationService (340 LOC)"
            C1[send_email] --> C2[format_template]
            C1 --> C3[queue_notification]
        end
        A1 -->|"clean interface"| B1
        B1 -->|"clean interface"| C1
    end
\```
```

### 6. Pros/Cons of Extraction

For each proposed extraction:

**Pros**:
- ✅ [Business benefit — e.g., "team X can deploy independently"]
- ✅ [Technical benefit — e.g., "reduces file from 2,340 to 900 LOC"]
- ✅ [Quality benefit — e.g., "enables focused unit testing"]

**Cons**:
- ❌ [Cost — e.g., "requires updating 14 callers"]
- ❌ [Risk — e.g., "shared database transaction must be redesigned"]
- ❌ [Effort — e.g., "estimated 2-3 days of work + characterization tests"]

## Output Format

```
## FILE PROFILE
- Path, LOC, function count, last modified, commit count (12mo)
- Business context: which feature/team does this serve?

## INTERNAL HOTSPOTS
| Function | Changes (6mo) | LOC | Complexity | Priority |
|----------|--------------|-----|------------|----------|
| ...      | ...          | ... | ...        | 🔴/🟠/🟡 |

## RESPONSIBILITY CLUSTER DIAGRAM
[Mermaid diagram showing current tangled state]

## TARGET STATE DIAGRAM
[Mermaid diagram showing clean extracted state]

## EXTRACTION PLAN (priority order)
For each extraction:
  1. Name & scope
  2. Methods and fields that move
  3. Callers to update
  4. Seam type (Feathers)
  5. Pros/Cons
  6. Estimated effort
  7. Characterization tests needed

## STAGE GATE — ARCHITECT REVIEW
[Review checklist for sign-off]
```
