# Code-to-Req: AI Code Architecture Audit Engine

> Turn any codebase into a professional architecture audit report — with diagrams, knowledge graphs, and structured requirements.

**What makes this different from just asking AI?** This skill produces a standardized, multi-document audit report — not chat responses. It includes architecture diagrams (Mermaid), knowledge graphs (JSON + visual), health scores, and actionable requirements. The output is ready for stakeholder presentations, not just developer notes.

## Output Example

```
📦 docs/code-analysis/
├── README.md                  ← Report entry (health score + key findings)
├── project-profile.md         ← Project profile (tech stack, scale, modules)
├── architecture-diagrams.md   ← 5 Mermaid diagrams (system, modules, data flow, sequence, ER)
├── knowledge-graph.md         ← Knowledge graph (entities, relations, JSON for rendering)
├── feature-map.md             ← Feature map (user-facing capabilities)
├── reading-plan.md            ← Smart reading plan
├── requirements-draft.md      ← Requirements draft
├── tech-proposal.md           ← Technical proposal
└── features/                  ← Feature drill-down (generated on demand)
    ├── {feature}.md           ← User stories + acceptance criteria + data models
    └── {feature}/
        └── req-{name}.md     ← Function-level call chains + modification points
```

## Install

```bash
# Clone to your Claude Code skills directory
git clone https://github.com/fernandexjulian943-alt/code-to-req.git ~/.claude/skills/code-to-req
```

Then use `/code-to-req` in any Claude Code session.

## Usage

```bash
# Full flow: scan → diagrams → analysis → multi-role discussion → report
/code-to-req <requirement description>

# Quick scan: project profile + architecture diagrams + knowledge graph + feature map
/code-to-req scan

# Scan + plan: generate a smart reading plan for specific analysis
/code-to-req plan <requirement description>

# Drill into a specific feature (user stories, acceptance criteria, data models)
/code-to-req --feature "user auth"

# Drill into a specific requirement (function-level call chains, modification points)
/code-to-req --feature "user auth" --req "token refresh"

# Analyze a project in a different directory
/code-to-req --path /path/to/project scan
```

## Demo: claw-code Project Analysis

See `examples/claw-code/` for a complete analysis of a Rust CLI project (~1000 files):

- **Scan**: [Project Profile](examples/claw-code/project-profile.md) | [Architecture Diagrams](examples/claw-code/architecture-diagrams.md) | [Knowledge Graph](examples/claw-code/knowledge-graph.md) | [Feature Map](examples/claw-code/feature-map.md)
- **Feature drill**: [Context Management](examples/claw-code/features/context-management.md) | [Session Resume](examples/claw-code/features/session-resume.md)
- **Req drill**: [Token Estimation Implementation](examples/claw-code/features/context-management/req-token-estimation.md)

## Recommended Models

| Scenario | Model | Notes |
|----------|-------|-------|
| **Best quality** | Claude Opus 4.6+ | Deep multi-role discussion, accurate architecture analysis |
| **Cost-effective** | Claude Sonnet 4.6 | Full flow works, slightly less depth in discussions |
| **China (primary)** | DeepSeek V4 Pro | Strong code understanding, good Chinese output, low cost |
| **China (backup)** | Qwen 3.6 Plus | Best Chinese comprehension, slightly weaker on code |
| **Not recommended** | Haiku / small models | Insufficient for multi-role analysis |

No API key configuration needed — the skill uses whatever model your Claude Code session is running.

## Features

- **Three-layer drill-down**: Scan (skeleton) → Feature (requirements) → Req (implementation details)
- **Smart scaling**: Adapts strategy based on project size (< 100 to 1000+ files)
- **5 architecture diagrams**: System layers, module dependencies, data flow, API sequence, ER diagram
- **Knowledge graph**: Entities + relationships in both Mermaid and JSON format (for sigma.js/D3)
- **Health scoring**: A/B/C/D ratings across 5 dimensions with quantified metrics
- **Multi-role analysis**: PM + Architect + Developer perspectives with cross-discussion
- **Anti-hallucination**: All outputs verified via grep/sed before writing (line numbers, counts, logic direction)
- **Large project support**: Intelligent filtering, sampling, and batched reading

## How It Works

```
Phase 0: Scale assessment → choose scanning strategy
Phase 1: Code scanning (no AI, pure static analysis)
Phase 1.5: Diagram + knowledge graph generation
Phase 2: Smart reading plan (AI-guided)
Phase 3: Progressive code reading + understanding
Phase 4: Multi-role collaboration (PM / Architect / Developer)
Phase 5: Report generation
```

## License

MIT
