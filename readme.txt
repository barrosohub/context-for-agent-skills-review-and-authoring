# Agent Skills — Technical Reference

**Format version:** Agent Skills Open Standard  
**Maintained at:** agentskills.io  
**Origin:** Created by Anthropic, released as open standard, adopted by 30+ agent products.  
**Canonical spec:** agentskills.io/specification  
**Reference implementation:** github.com/agentskills/agentskills  
**Example skills library:** github.com/anthropics/skills

---

## 1. Definition

An Agent Skill is a directory containing a required `SKILL.md` file plus optional supporting files (scripts, templates, references, assets). Skills give AI agents access to procedural knowledge, domain expertise, and repeatable workflows they can load on demand.

Skills solve a fundamental problem: agents are capable but lack the context they need to do real work reliably. Skills provide that context in a portable, version-controlled, auditable format.

---

## 2. Directory Structure

```text
skill-name/
├── SKILL.md          # REQUIRED — metadata + instructions
├── scripts/          # OPTIONAL — executable code (Python, Bash, JS)
├── references/       # OPTIONAL — detailed documentation, specs
├── assets/           # OPTIONAL — templates, images, schemas, data files
└── ...               # Any additional files or directories
```

The directory name must match the `name` field in `SKILL.md` frontmatter.

---

## 3. SKILL.md Format

Every `SKILL.md` consists of YAML frontmatter (between `---` delimiters) followed by a Markdown body.

### 3.1 Frontmatter Fields (Open Standard)

| Field | Required | Constraints | Purpose |
|---|---|---|---|
| `name` | Yes | 1-64 chars. Lowercase a-z, digits 0-9, hyphens - only. No leading/trailing/consecutive hyphens. Must match parent directory name. | Short identifier for the skill. |
| `description` | Yes | 1-1024 chars. Non-empty. | What the skill does AND when to use it. Primary signal for auto-discovery and activation. |
| `license` | No | Short string or reference to bundled file. | License applied to the skill. |
| `compatibility` | No | 1-500 chars. | Environment requirements (OS, tools, network, target agent). |
| `metadata` | No | Map of string keys to string values. | Arbitrary additional properties. |
| `allowed-tools` | No | Space-delimited list. Experimental. | Pre-approved tools the skill may use. |

### 3.2 Client-Specific Extensions

Different agents extend the frontmatter with proprietary fields.

**Claude Code extensions:**

| Field | Type | Purpose |
|---|---|---|
| `disable-model-invocation` | boolean | `true` = only user can invoke via `/`, prevents auto-trigger |
| `user-invocable` | boolean | `false` = hide from `/` menu, only model can invoke |
| `argument-hint` | string | Autocomplete hint shown after `/skill-name`, e.g. `[issue-number]` |
| `model` | string | Override which model runs this skill |
| `context` | string | `fork` = run in isolated subagent with separate context window |
| `agent` | string | Subagent type: Explore, Plan, general-purpose, or custom |
| `hooks` | object | Lifecycle hooks scoped to the skill |

Claude Code string substitutions available in body:

- `$ARGUMENTS` — full argument string
- `$ARGUMENTS[N]` or `$N` — Nth argument
- `${CLAUDE_SESSION_ID}` — current session ID
- `${CLAUDE_SKILL_DIR}` — absolute path to the skill directory
- `` !`shell command` `` — dynamic context injection (runs before sending to model)

**OpenAI Codex extensions (via `agents/openai.yaml`):**

- `allow_implicit_invocation` — controls auto-trigger (default: `true`)

### 3.3 Name Validation Rules

**Valid:** `pdf-processing`, `data-analysis`, `code-review`, `my-skill-v2`  
**Invalid:** `PDF-Processing` (uppercase)  
`-pdf` (starts with hyphen)  
`pdf-` (ends with hyphen)  
`pdf--processing` (consecutive hyphens)  
`my skill` (spaces)  
`my_skill` (underscores)

### 3.4 Description Best Practices

The `description` field is the most important field — it determines whether agents activate the skill.

**Good:**

> Extracts text and tables from PDF files, fills PDF forms, and merges multiple PDFs. Use when working with PDF documents or when the user mentions PDFs, forms, or document extraction.

**Poor:**

> Helps with PDFs.

**Guidelines:**

- Describe both what the skill does and when to use it.
- Include specific keywords that help agents identify relevant tasks.
- Test trigger accuracy with eval queries (half should-trigger, half should-not).

### 3.5 Minimal Example

```yaml
---
name: code-review
description: Reviews code for bugs, security issues, and style violations. Use when asked to review, audit, or check code quality.
---
```

```markdown
# Code Review

## Steps
1. Read the files or diff provided
2. Check for bugs, security vulnerabilities, and style issues
3. Report findings with file paths and line numbers
4. Suggest fixes with code snippets
```

### 3.6 Full Example (All Optional Fields)

```yaml
---
name: pdf-processing
description: Extract PDF text, fill forms, merge files. Use when handling PDFs.
license: Apache-2.0
compatibility: Requires Python 3.11+, pdfplumber, and pikepdf packages.
allowed-tools: Bash(python:*) Read Write
metadata:
  author: example-org
  version: "1.0"
  category: document-processing
---
```

```markdown
# PDF Processing

## When to use this skill
Use this skill when the user needs to:
- Extract text or tables from PDF files
- Fill in PDF form fields
- Merge or split PDF documents

## How to extract text
1. Run the extraction script:
   `scripts/extract.py --input <file.pdf> --output <output.txt>`
2. The script uses pdfplumber for text and tabula for tables.

## How to fill forms
See [the forms reference](references/FORMS.md) for field mappings.

## Edge cases
- Scanned PDFs: use OCR fallback via `scripts/ocr.py`
- Password-protected: prompt user for password, pass via `--password` flag
```

---

## 4. Progressive Disclosure

Skills use a three-tier loading model to keep agents fast while providing deep context on demand:

| Tier | What Loads | When | Token Cost | Content |
|---|---|---|---|---|
| 1. Catalog | `name` + `description` only | Session startup (all skills) | ~50-100 tokens per skill | Enough to know when skill is relevant |
| 2. Instructions | Full `SKILL.md` body | When skill is activated | `<5,000` tokens recommended | Step-by-step instructions, examples |
| 3. Resources | `scripts/`, `references/`, `assets/` | When instructions reference them | Varies | Executable code, detailed docs, templates |

An agent with 20 installed skills pays ~1,000-2,000 tokens at startup (catalog only). Full instructions load only for skills actually used.

**Budget constraint (Claude Code):** Skill descriptions share a budget of 2% of context window (fallback: 16,000 characters). Excess skills are excluded. Check with `/context`. Override via `SLASH_COMMAND_TOOL_CHAR_BUDGET` env var.

**Recommendations:**

- Keep `SKILL.md` body under 500 lines.
- Move detailed reference material to `references/` files.
- Keep file references one level deep from `SKILL.md`. Avoid deeply nested chains.

---

## 5. Skill Lifecycle

### 5.1 Discovery

At session startup, agents scan predefined directories for subdirectories containing `SKILL.md`.

**Scanning rules:**

- Look for subdirectories containing a file named exactly `SKILL.md` (prefer uppercase, some accept lowercase).
- Skip `.git/`, `node_modules/`, `__pycache__/`, etc.
- Optionally respect `.gitignore`.
- Reasonable bounds: max depth 4-6 levels, max 2,000 directories scanned.

### 5.2 Activation

Two activation patterns:

**Model-driven (automatic):** Agent reads descriptions in context, determines relevance, loads full `SKILL.md` when a task matches.

**User-explicit (manual):** User types a slash command (`/skill-name` in Claude Code, Copilot) or dollar reference (`$skill-name` in Codex) to force-activate a skill.

### 5.3 Execution

Agent follows the Markdown instructions in `SKILL.md`, optionally loading referenced files or executing bundled scripts as needed.

### 5.4 Context Management

- Protect skill content from context compaction/pruning — exempt from truncation.
- Deduplicate — track activated skills per session, skip re-injection.
- Subagent delegation (advanced) — run skill in isolated subagent, return summary to main conversation.

---

## 6. Discovery Paths by Agent

### 6.1 Claude Code

| Scope | Path |
|---|---|
| Personal (all projects) | `~/.claude/skills/<name>/SKILL.md` |
| Project | `<project>/.claude/skills/<name>/SKILL.md` |
| Nested (monorepo) | `<subdir>/.claude/skills/<name>/SKILL.md` |
| Plugin | `<plugin>/skills/<name>/SKILL.md` (namespaced `plugin:skill`) |
| Enterprise | Admin-deployed managed settings |
| `--add-dir` | Skills in added directories, with live change detection |

**Legacy:** `.claude/commands/*.md` still works. If both a command and a skill share the same name, the skill takes precedence. Since v2.1.3, slash commands and skills are a unified system.

**Invocation:** `/skill-name [args]` (manual) or automatic via description match.

**Docs:** `code.claude.com/docs/en/skills`

### 6.2 VSCode / GitHub Copilot

| Scope | Path |
|---|---|
| Project (native) | `<project>/.github/skills/<name>/SKILL.md` |
| Project (cross-tool) | `<project>/.claude/skills/<name>/SKILL.md` |
| Project (cross-tool) | `<project>/.agents/skills/<name>/SKILL.md` |
| Personal (native) | `~/.copilot/skills/<name>/SKILL.md` |
| Personal (cross-tool) | `~/.claude/skills/<name>/SKILL.md` |
| Personal (cross-tool) | `~/.agents/skills/<name>/SKILL.md` |

**Cross-tool compatibility:** Copilot explicitly reads `.claude/skills/`, `.agents/skills/`, `CLAUDE.md`, `AGENTS.md`, and `.claude/rules/`.

**Invocation:** `/skill-name` in chat, or automatic via description match, or `/create-skill` to scaffold.

**Docs:**

- `code.visualstudio.com/docs/copilot/customization/agent-skills`
- `docs.github.com/en/copilot/concepts/agents/about-agent-skills`

### 6.3 OpenAI Codex

| Scope | Path |
|---|---|
| Repository | `<project>/.agents/skills/<name>/SKILL.md` (walks up to git root) |
| User | `~/.agents/skills/<name>/SKILL.md` |
| Admin | `/etc/codex/skills/<name>/SKILL.md` |
| System | Bundled with Codex |

**Note:** Codex uses `AGENTS.md` (not `CLAUDE.md`) for project instructions. Override with `AGENTS.override.md`.

**Invocation:** `$skill-name` (explicit) or implicit by description match.

**Docs:** `developers.openai.com/codex/skills/`

### 6.4 Other Adopters

| Agent | Docs |
|---|---|
| Cursor | `cursor.com/docs/context/skills` |
| Gemini CLI | `geminicli.com/docs/cli/skills/` |
| JetBrains Junie | `junie.jetbrains.com/docs/agent-skills.html` |
| Goose (Block) | `block.github.io/goose/docs/guides/context-engineering/using-skills/` |
| Roo Code | `docs.roocode.com/features/skills` |
| Amp | `ampcode.com/manual#agent-skills` |
| OpenHands | `docs.openhands.dev/overview/skills` |
| Databricks | `docs.databricks.com/aws/en/assistant/skills` |
| Snowflake | `docs.snowflake.com/en/user-guide/cortex-code/extensibility#extensibility-skills` |
| Spring AI | `spring.io/blog/2026/01/13/spring-ai-generic-agent-skills/` |

---

## 7. Cross-Agent Compatibility Matrix

### 7.1 Skill Discovery Paths

| Path | Claude Code | VSCode/Copilot | OpenAI Codex |
|---|---|---|---|
| `.claude/skills/` | native | reads | — |
| `.github/skills/` | — | native | — |
| `.agents/skills/` | — | reads | native |
| `~/.claude/skills/` | native | reads | — |
| `~/.copilot/skills/` | — | native | — |
| `~/.agents/skills/` | — | reads | native |

### 7.2 Instruction Files

| File | Claude Code | VSCode/Copilot | OpenAI Codex |
|---|---|---|---|
| `CLAUDE.md` | native | reads | — |
| `AGENTS.md` | — | reads | native |
| `.github/copilot-instructions.md` | — | native | — |
| `.claude/rules/` | native | reads | — |

### 7.3 Maximum Portability Strategy

When creating Agent Skills (`SKILL.md`), always:

- Create the skill in `.agents/skills/<skill-name>/SKILL.md` (canonical source — widest cross-agent reach: Claude Code, Copilot, Codex, Gemini CLI)
- Create symlinks in `.claude/skills/`, `.agent/skills/`, pointing to `../../.agents/skills/<skill-name>`

**Recommended project-level layout:**

```text
.agents/skills/my-skill/SKILL.md          # canonical source
.claude/skills/my-skill -> ../../.agents/skills/my-skill
.agent/skills/my-skill  -> ../../.agents/skills/my-skill
```

**Optional additional symlink for native Copilot path:**

```text
.github/skills/my-skill -> ../../.agents/skills/my-skill
```

**Example commands:**

```bash
mkdir -p .agents/skills .claude/skills .agent/skills
mkdir -p .agents/skills/my-skill

ln -s ../../.agents/skills/my-skill .claude/skills/my-skill
ln -s ../../.agents/skills/my-skill .agent/skills/my-skill

# Optional for native Copilot path
ln -s ../../.agents/skills/my-skill .github/skills/my-skill
```

At user level:

```text
~/.agents/skills/my-skill/SKILL.md        # canonical source
~/.claude/skills/my-skill -> ~/.agents/skills/my-skill
~/.agent/skills/my-skill  -> ~/.agents/skills/my-skill
```

Optional:

```text
~/.copilot/skills/my-skill -> ~/.agents/skills/my-skill
```

---

## 8. Name Collision Resolution

| Rule | Behavior |
|---|---|
| Project-level vs user-level | Project wins (more specific scope overrides broader scope) |
| Same scope, same name | First-found or last-found (agent-dependent). Log warning. |
| Skill vs legacy command (Claude Code) | Skill wins |

---

## 9. Scripts

Place executable code in `scripts/`. Design for agentic use:

- No interactive prompts — agents cannot respond to stdin prompts.
- Good `--help` output — agents read this to understand usage.
- Structured output to stdout — JSON preferred for machine parsing.
- Diagnostics to stderr — keeps stdout clean for results.
- Idempotent operations — safe to re-run.
- Self-contained or clearly documented dependencies.
- Helpful error messages with actionable guidance.

**Common patterns:**

- Python scripts with inline dependencies (PEP 723 `# /// script` blocks)
- Bash scripts for system operations
- One-off tools via `uvx`, `npx`, `bunx`

---

## 10. References and Assets

### `references/` — On-Demand Documentation

- `REFERENCE.md` — detailed technical reference
- `FORMS.md` — form templates or structured data formats
- Domain-specific files (`finance.md`, `legal.md`, etc.)

Keep individual files focused. Agents load these only when instructions reference them.

### `assets/` — Static Resources

- Templates (document templates, config templates)
- Images (diagrams, examples)
- Data files (lookup tables, schemas, sample data)

### File References in SKILL.md

Use relative paths from the skill root:

```markdown
See [the reference guide](references/REFERENCE.md) for details.
Run: `scripts/extract.py --input file.pdf`
```

Agents resolve relative paths against the skill directory.

---

## 11. Validation

Use the `skills-ref` CLI tool from `github.com/agentskills/agentskills/tree/main/skills-ref`:

```bash
# Install
pip install skills-ref   # or: uvx skills-ref

# Validate a skill
skills-ref validate ./my-skill

# Read properties as JSON
skills-ref read-properties ./my-skill

# Generate XML prompt catalog
skills-ref to-prompt ./skills-directory
```

**Lenient validation guidance for implementers:**

- Name doesn't match directory → warn, load anyway
- Name exceeds 64 chars → warn, load anyway
- Description missing or empty → skip the skill, log error
- YAML completely unparseable → skip the skill, log error

---

## 12. Evaluation (Testing Skills)

### Trigger Accuracy Testing

1. Create ~20 eval queries (half should-trigger, half should-not).
2. Test trigger rate across multiple runs.
3. Use train/validation split (60/40) to avoid overfitting descriptions.
4. Iterate: broaden for missed triggers, narrow for false triggers.

### Functional Evaluation

Store test cases in `evals/evals.json`:

```json
[
  {
    "prompt": "Extract text from invoice.pdf",
    "expected_output": "Structured text with line items",
    "files": ["test-data/invoice.pdf"],
    "assertions": [
      "Output contains extracted text",
      "Tables are properly formatted"
    ]
  }
]
```

Run with-skill vs without-skill. Grade assertions as PASS/FAIL with evidence. Aggregate to `benchmark.json`.

**Docs:** `agentskills.io — Evaluating Skills`

---

## 13. Implementation Guide for Agent Developers

**Full guide:** `agentskills.io/integrate-skills`

### Step 1: Discover Skills

Scan project-level and user-level directories. Load only `name` + `description` per skill.

### Step 2: Parse SKILL.md

Extract YAML frontmatter (between `---` delimiters). Everything after closing `---` is the body. Handle malformed YAML gracefully (unquoted colons, etc.).

### Step 3: Disclose Catalog to Model

Inject skill catalog into system prompt or tool description:

```xml
<available_skills>
  <skill>
    <name>pdf-processing</name>
    <description>Extract PDF text, fill forms, merge files.</description>
    <location>/home/user/.claude/skills/pdf-processing/SKILL.md</location>
  </skill>
</available_skills>
```

### Step 4: Activate on Demand

**File-read activation:** Model calls its read tool with the `SKILL.md` path. Simplest approach.

**Dedicated tool activation:** Register an `activate_skill` tool. Returns body content, optionally wrapped:

```xml
<skill_content name="pdf-processing">
  [SKILL.md body here]

  Skill directory: /home/user/.claude/skills/pdf-processing
  <skill_resources>
    <file>scripts/extract.py</file>
    <file>references/FORMS.md</file>
  </skill_resources>
</skill_content>
```

### Step 5: Manage Context

Protect skill content from compaction. Deduplicate activations. Optionally delegate to subagents.

---

## 14. Anthropic Example Skills Library

**Repository:** `github.com/anthropics/skills`

| Skill | Description | Bundled Resources |
|---|---|---|
| `algorithmic-art` | Generate algorithmic art | `templates/` |
| `brand-guidelines` | Follow brand guidelines | — |
| `canvas-design` | Design canvas layouts | — |
| `claude-api` | Build with Claude API/SDK | `reference/` |
| `doc-coauthoring` | Co-author documents | — |
| `docx` | Create/edit Word documents | source-available |
| `frontend-design` | Build production-grade UIs | — |
| `internal-comms` | Write internal communications | `examples/` |
| `mcp-builder` | Build MCP servers | `reference/` |
| `pdf` | Create/edit PDFs | source-available |
| `pptx` | Create PowerPoint presentations | `scripts/` |
| `skill-creator` | Create new skills | `scripts/`, `evals/` |
| `slack-gif-creator` | Create Slack GIFs | `core/`, `examples/` |
| `theme-factory` | Generate UI themes | `themes/` |
| `web-artifacts-builder` | Build web artifacts | `scripts/` |
| `webapp-testing` | Test web applications | `scripts/`, `examples/` |
| `xlsx` | Create/edit spreadsheets | `scripts/` |

**Template for new skills:** `github.com/anthropics/skills/tree/main/template`

---

## 15. Key Links Index

### Specification and Standard

- Spec: `agentskills.io/specification`
- What are skills: `agentskills.io/what-are-skills`
- Integration guide: `agentskills.io/integrate-skills`
- Standard repo: `github.com/agentskills/agentskills`
- Validation CLI (`skills-ref`): `github.com/agentskills/agentskills/tree/main/skills-ref`

### Example Skills and Authoring

- Anthropic skills library: `github.com/anthropics/skills`
- Best practices: `platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices`
- Evaluating skills: `agentskills.io/skill-creation/evaluating-skills`
- Optimizing descriptions: `agentskills.io/skill-creation/optimizing-descriptions`
- Using scripts: `agentskills.io/skill-creation/using-scripts`

### Agent-Specific Documentation

- Claude Code: `code.claude.com/docs/en/skills`
- Claude Platform: `platform.claude.com/docs/en/agents-and-tools/agent-skills/overview`
- VSCode/Copilot: `code.visualstudio.com/docs/copilot/customization/agent-skills`
- GitHub Copilot: `docs.github.com/en/copilot/concepts/agents/about-agent-skills`
- OpenAI Codex: `developers.openai.com/codex/skills/`
- Codex `AGENTS.md`: `developers.openai.com/codex/guides/agents-md/`
- OpenAI Skills Catalog: `github.com/openai/skills`

### Community and Adoption

- Cursor: `cursor.com/docs/context/skills`
- Gemini CLI: `geminicli.com/docs/cli/skills/`
- JetBrains Junie: `junie.jetbrains.com/docs/agent-skills.html`
- Goose: `block.github.io/goose/docs/guides/context-engineering/using-skills/`
- Roo Code: `docs.roocode.com/features/skills`
- Amp: `ampcode.com/manual#agent-skills`
