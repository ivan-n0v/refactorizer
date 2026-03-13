# Refactorizer

Data-driven monolith decomposition skills for Claude Code.

## Skills

| Command | Purpose |
|---------|---------|
| `/refactorize` | Full 6-stage decomposition pipeline with architect review gates |
| `/refactorize-hotspots <file>` | Function-level hotspot deep-dive with extraction recommendations |
| `/refactorize-coupling <mod1> [mod2]` | Module coupling analysis with 3 decoupling strategies |

## Approach

**Business-first**: Every recommendation ties to a business outcome (velocity, reliability, autonomy).

**Data-driven**: Uses git history + static analysis — no opinions without evidence.

**Architect-reviewed**: Stage gates between each phase for human sign-off.

**5 options, not 1 mandate**: Presents conservative → aggressive options with pros/cons and comparison matrix.

## Requirements

- Git repo with 6+ months history (behavioral analysis)
- Python 3 (inline scripts, no packages)
- `cloc` recommended (falls back to `wc -l`)
