# 🩹 Self-Healing Funnel

A Claude skill that turns Amplitude into an active product optimization engine. Point it at any funnel and it detects conversion drop-offs, diagnoses root causes, audits what has already been tried, and creates a targeted experiment -- all documented in a single Amplitude notebook.

## What it does

1. **Detects** -- queries your funnel over the last 12 weeks and identifies declining steps
2. **Diagnoses** -- breaks down the drop-off by platform and device to understand who is affected
3. **Audits** -- searches your existing experiments and guides to understand what has already been tried and what gaps remain
4. **Hypothesizes** -- proposes a targeted intervention based on the uncovered gap
5. **Creates** -- builds a draft experiment and a full analysis notebook in Amplitude documenting the entire loop

## Prerequisites

### 1. Claude Code
Install the Claude CLI if you haven't already:
```bash
npm install -g @anthropic-ai/claude-code
```
Full install guide: https://docs.claude.ai/en/docs/claude-code/overview

### 2. Amplitude MCP connector
The skill connects to Amplitude via MCP (Model Context Protocol). To set this up:
- In your Amplitude org, go to **Settings > Integrations > MCP**
- Follow the setup instructions to connect your project
- More details: https://www.amplitude.com/blog/amplitude-mcp

## Installation

1. Copy `SKILL.md` into your Claude skills directory:
```bash
mkdir -p ~/.claude/skills/self-healing-funnel
cp SKILL.md ~/.claude/skills/self-healing-funnel/SKILL.md
```

2. Restart Claude Code (or run `/reload` if already open)

## Usage

Once installed, trigger the skill by describing the funnel you want to analyze:

```
Run self-healing on this funnel: [paste Amplitude chart URL]
```

```
Self-healing funnel on project [your-project-id], steps: Sign Up -> Activate -> Subscribe
```

```
Check funnel health for our onboarding funnel [chart link], find drop-offs and propose fixes
```

The skill will walk through each phase and produce:
- 4 saved charts (trend, conversion bar, platform breakdown, device breakdown)
- 1 draft experiment targeting the identified gap
- 1 notebook documenting the full detection-diagnosis-action loop

## Notes on the SKILL.md file

The YAML block at the top of `SKILL.md` is metadata that Claude Code uses to understand when to trigger the skill automatically. You don't need to edit it. The skill will also work if you trigger it manually with any of the example phrases above.

## Configuration

The skill uses sensible defaults but you can adjust thresholds for your funnel characteristics -- see the **Configuration & Thresholds** table at the bottom of `SKILL.md`.

For high-volume funnels (millions of events per day), consider switching the trend interval from weekly to daily. For slow-moving funnels like annual subscription flows, extend the trend window beyond 12 weeks.
