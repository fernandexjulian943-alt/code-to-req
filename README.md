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
├── reading-plan.md            ← Smart reading plan
├── requirements-draft.md      ← Requirements draft
└── tech-proposal.md           ← Technical proposal
```

## Install

**Option A: Plugin marketplace** (if your marketplace includes this skill)
```
/plugin install code-to-req
```

**Option B: Manual install**
```bash
# Clone to your Claude Code skills directory
git clone https://github.com/YOUR_ORG/code-to-req.git ~/.claude/skills/code-to-req
```

## Usage

```bash
# Full flow: scan → diagrams → analysis → multi-role discussion → report
/code-to-req <requirement description>

# Quick scan: project profile + architecture diagrams + knowledge graph
/code-to-req scan

# Scan + plan: generate a smart reading plan for specific analysis
/code-to-req plan <requirement description>
```

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

- **Smart scaling**: Adapts strategy based on project size (< 100 to 1000+ files)
- **5 architecture diagrams**: System layers, module dependencies, data flow, API sequence, ER diagram
- **Knowledge graph**: Entities + relationships in both Mermaid and JSON format (for sigma.js/D3)
- **Health scoring**: A/B/C/D ratings across 5 dimensions
- **Multi-role analysis**: PM + Architect + Developer perspectives with cross-discussion
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
