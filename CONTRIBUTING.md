# Contributing to Refactorizer

Thanks for your interest in improving Refactorizer!

## How to Contribute

### Reporting Issues

- Open an issue describing the problem, the codebase characteristics (language, size, mono/polyrepo), and what you expected vs. what happened.
- Include the stage where the issue occurred (Stage 1-6).

### Improving the Skills

The skills live in `.claude/commands/`:

| File | Purpose |
|------|---------|
| `refactorize.md` | Main 6-stage decomposition pipeline |
| `refactorize-hotspots.md` | Function-level hotspot deep-dive |
| `refactorize-coupling.md` | Module coupling analysis |

When modifying skills:

1. **Test on a real codebase** — clone a well-known open-source project (e.g., [JHipster](https://github.com/jhipster/generator-jhipster), [Sentry](https://github.com/getsentry/sentry)) and run the skill end-to-end.
2. **Keep the business-first framing** — every recommendation should tie back to a business outcome.
3. **Preserve stage gates** — architect review checkpoints between stages are a core feature.
4. **Include diagrams** — Mermaid diagrams at each stage for visual review.
5. **Present options, not mandates** — the 5-option framework gives architects choice.

### Adding New Skills

New skills should follow the pattern:
- Named `refactorize-<aspect>.md`
- Include input specification, analysis steps, diagram output, pros/cons, and a stage gate
- Focus on a single concern (don't duplicate the main pipeline)

Ideas for new skills:
- `/refactorize-database` — Shared database decomposition analysis
- `/refactorize-api` — API surface analysis for service boundary identification
- `/refactorize-tests` — Test coverage and characterization test gap analysis
- `/refactorize-events` — Event flow mapping for event-driven decomposition

### Adding Examples

Add example outputs in `examples/` showing the skill run against real open-source projects. Include:
- The project name and version
- Which skill was run
- The full output

## Development Setup

```bash
git clone https://github.com/YOUR_USER/refactorizer.git
cd refactorizer

# Symlink into a test project
ln -s $(pwd)/.claude/commands/refactorize.md /path/to/test-project/.claude/commands/

# Open Claude Code in the test project and run /refactorize
```

## Code of Conduct

Be respectful, constructive, and focused on making the tool better for everyone.
