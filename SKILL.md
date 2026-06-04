---
name: self-healing-funnel
description: >
  Autonomous funnel health monitor that takes a funnel (chart link, chart ID, or list of steps)
  as input, detects conversion drop-offs, diagnoses root causes, audits existing interventions,
  and creates targeted experiments with a full analysis notebook. Use this skill whenever someone
  asks to check funnel health, detect conversion drops, run a self-healing analysis, find funnel
  bottlenecks, or automate funnel optimization in any Amplitude project. Also trigger when someone
  says "self-healing funnel", "funnel drop-off", "conversion decline", "checkout friction",
  "funnel anomaly detection", or asks to "find and fix funnel problems". The user should provide
  a funnel to analyze (Amplitude chart link, chart ID, or ordered step names with a project ID).
  Works against any Amplitude project.
---

# 🩹 The Self-Healing Funnel

An autonomous detection-diagnosis-action loop for Amplitude funnels. This skill turns
Amplitude from a passive analytics tool into an active product optimization engine.

**The thesis:** Amplitude should detect funnel drop-offs, diagnose root causes, audit what
has already been tried, identify intervention gaps, create targeted experiments, and document
the entire loop in a notebook -- without manual analysis.

## When to use this skill

- "Run self-healing on this funnel: [link]"
- "Check funnel health for Add to Cart -> Checkout -> Purchase"
- "Here's our onboarding funnel [link], find drop-offs and propose fixes"
- "Self-healing funnel on project [your-project-id], steps: Sign Up, KYC, First Deposit"

## Input: The Funnel

This skill takes a **funnel** as input. Provide it in one of three ways:

1. **An Amplitude chart URL or ID** (preferred, most common).
   Example: "Run self-healing on https://app.amplitude.com/analytics/[your-org]/chart/[chart-id]"
   Use `get_from_url` or `query_chart` to extract the event sequence and project ID.

2. **A list of step names + project ID**.
   Example: "Project [your-project-id], funnel: Add to Cart -> View Cart -> Checkout -> Complete Purchase"
   Use these event names directly in the funnel definition.

3. **Just a project ID** (fallback). If you only give a project and say "find the funnel",
   the skill will use `search` to find existing funnel charts in the project, pick the one
   with the highest view count, and confirm the steps before proceeding.

If no funnel is provided, the skill will ask:
"Which funnel should I analyze? You can paste an Amplitude chart link, or list the steps
(e.g., 'Sign Up -> Activate -> Purchase') along with the project name or ID."

## Prerequisites

This skill requires:

- **Claude Code** -- the Claude desktop CLI ([install guide](https://docs.claude.ai/en/docs/claude-code/overview))
- **Amplitude MCP connector** -- connects Claude to your Amplitude project. Set it up at Settings > Integrations > MCP in your Amplitude org, or follow the [Amplitude MCP docs](https://www.amplitude.com/blog/amplitude-mcp).

The skill uses these MCP tools:
- `query_chart` / `get_from_url` (to read existing funnel charts)
- `query_amplitude_data` (funnel queries)
- `render_chart` / `save_chart_edits`
- `search` (for existing guides, experiments, charts)
- `create_experiment`
- `create_notebook`

## Workflow

Execute these phases in order. Pace MCP calls (one at a time) to avoid rate limiting.

---

### Phase 1: Resolve the Funnel

Turn the user's input into a concrete funnel definition (project ID + ordered event list).

**If the user provided a chart URL or ID:**

Call `query_chart` with the chart ID (or `get_from_url` with the URL). Extract:
- `definition.app` -> project ID
- `definition.params.events` -> ordered list of step event names
- The chart's existing data gives you a baseline to work from immediately

**If the user provided step names + project:**

Validate the events exist by calling `query_amplitude_data` in Mode 1 with the step names
as `eventSearchTerms`. Confirm they have volume > 0. If any step has zero volume, flag it
and ask the user for an alternative.

**If the user only provided a project (fallback):**

Call `search` with `appIds: [projectId]`, `entityTypes: ["CHART"]`, `queries: ["funnel", "conversion"]`.
Pick the chart with the highest view count. Call `query_chart` to get its definition.
Confirm with the user: "I found this funnel: [steps]. Should I analyze this one?"

**Output of this phase:** A project ID and an ordered list of 3-5 funnel step event names.

---

### Phase 2: Detect

Query the funnel conversion over time to identify declining steps.

**Step 2.1: Create the trend chart**

Call `render_chart` with a funnel definition:
```
type: "funnels"
metric: "OVER_TIME"
interval: 7 (weekly)
range: "Last 12 Weeks"
mode: "ordered"
conversionSeconds: 2592000 (30 days)
```

**Step 2.2: Analyze the trend data**

From the CSV response, extract weekly conversion rates. Calculate:
- **Trailing 4-week average** (weeks 5-8 from the end)
- **Recent 2-week average** (most recent 2 complete weeks)
- **Relative change**: (recent - trailing) / trailing

Flag the funnel as declining if relative change < -15%.

**Step 2.3: Identify the bottleneck step**

Also create a static funnel conversion bar chart to visualize per-step drop-off:
```
metric: "CONVERSION"
range: "Last 30 Days"
```

This produces the classic staircase visualization. Identify the step with the largest
absolute drop-off (lowest step-to-step conversion rate). This is the target step.

**Step 2.4: Save both charts**

Call `save_chart_edits` to persist both charts (trend + bar) for the notebook.

---

### Phase 3: Diagnose

For the identified bottleneck step, run segment breakdowns to understand who is affected.

**Step 3.1: Platform breakdown**

Create a 2-step funnel (problem step -> next step) with:
- `metric`: "OVER_TIME", `interval`: 7
- `groupBy`: [{"type": "user", "value": "platform", "group_type": "User"}]

**Step 3.2: Device type breakdown**

Same funnel with:
- `groupBy`: [{"type": "user", "value": "device_type", "group_type": "User"}]

**Step 3.3: Interpret the diagnosis**

Determine whether the decline is:
- **Universal** (all segments declining equally): points to UX/content issue in the step itself
- **Segment-specific** (one platform or device much worse): points to technical/rendering issue
- **New-user-concentrated**: points to onboarding or first-experience problem

Save diagnosis charts via `save_chart_edits`.

---

### Phase 4: Audit Existing Interventions

This is the critical "intelligence" phase. Before proposing anything new, understand
what has already been tried and why the problem persists.

**Step 4.1: Search for existing guides**

Call `search` with:
- `appIds`: [projectId]
- `entityTypes`: ["GUIDE", "SURVEY"]
- `queries`: [terms related to the bottleneck step, e.g. "checkout", "shipping", "cart"]

**Step 4.2: Search for existing experiments**

Call `search` with:
- `appIds`: [projectId]
- `entityTypes`: ["EXPERIMENT", "FLAG"]
- `queries`: [same terms as above]

**Step 4.3: Analyze the gap**

For each existing intervention found, categorize what friction driver it addresses:
- **Trust/security** (SSL badges, return policies, social proof)
- **Incentives** (discounts, loyalty, free shipping)
- **CTA/visual** (button color, copy, placement)
- **Payment friction** (one-click, saved cards, payment options)
- **Form friction** (field count, autocomplete, progress indicators)
- **Cost transparency** (shipping costs, taxes, fees shown upfront)
- **Navigation/UX** (progress bars, back buttons, save-for-later)

Identify which categories are covered and which are NOT. The uncovered categories
form the hypothesis for the new experiment.

**Step 4.4: Write the audit summary**

Produce a plain-English gap analysis:
- How many guides and experiments exist targeting this area
- What friction drivers they collectively address
- What friction drivers remain unaddressed
- Why the problem likely persists despite existing interventions

---

### Phase 5: Hypothesize & Create Experiment

**Step 5.1: Formulate the hypothesis**

Based on the gap analysis, write a specific, testable hypothesis:
"Users drop off at [step] because of [unaddressed friction driver]. Addressing this
via [specific intervention] will improve [step] conversion by [expected magnitude]."

**Step 5.2: Create the experiment**

Call `create_experiment` with:
- `projectIds`: [projectId]
- `key`: "self-healing-" + kebab-case description (e.g., "self-healing-shipping-form-streamline")
- `name`: "Self-Healing: [Description]"
- `description`: Include the detection summary, diagnosis, audit findings, and hypothesis.
  This makes the experiment self-documenting.
- `variants`: control (current experience) + treatment (with the proposed change).
  Include a payload on the treatment variant describing the feature flags needed.

---

### Phase 6: Build the Notebook

Create a comprehensive notebook documenting the full loop. This is the primary deliverable.

**Notebook structure:**

Row 1: Title and intro (rich_text, width 12)
```markdown
# 🩹 Self-Healing Funnel: [Bottleneck Step] Drop-off Analysis

*Auto-generated on [date] by the Self-Healing Funnel skill*

This notebook documents an automated detection-diagnosis-action loop.
The system identified a conversion decline, diagnosed the root cause,
audited existing interventions, and proposed a targeted experiment.
```

Row 2: Detection summary (rich_text, width 12)
```markdown
## 🔍 Detection: Conversion Decline Identified

[Plain-English summary of the trend: X% to Y% over Z weeks,
which step is the bottleneck, what the drop-off rate is]
```

Row 3: Two charts side by side
- Funnel bar chart (CONVERSION) -- width 6
- Funnel trend chart (OVER_TIME) -- width 6

This pairing is important: the bar chart shows WHERE the problem is (which step),
the trend chart shows HOW BAD it's getting (trajectory over time).

Row 4: Diagnosis summary (rich_text, width 12)
```markdown
## 🔬 Diagnosis: Who Is Affected?

[Summary of segment findings: universal vs. segment-specific,
what this implies about the root cause]
```

Row 5: Diagnosis charts side by side (width 6 each)

Row 6: Audit (rich_text, width 12)
```markdown
## 📋 Audit: What Has Already Been Tried?

[List existing guides and experiments, what they target,
the gap analysis, why the problem persists]
```

Row 7: Hypothesis and experiment (rich_text, width 12)
```markdown
## 🧪 Hypothesis & Experiment

**Hypothesis:** [The specific testable hypothesis]

**Experiment:** [Link to experiment, variant descriptions, metrics]
```

Row 8: Results (rich_text, width 12)

If there is a completed experiment in the project targeting a related funnel step,
pull its results via `query_experiment` and present them as a proxy/early signal:
```markdown
## 📊 Early Signal

Using [experiment name] as an analogous reference (both target checkout friction):

| Metric | Control | Treatment | Lift | Significance |
|--------|---------|-----------|------|-------------|
| [metric] | X% | Y% | +Z% relative | p < threshold |

[Interpretation and what this implies for the new experiment]
```

If no analogous experiment exists, use a forward-looking placeholder:
```markdown
## 📊 Results

Results will populate after the experiment accumulates 7+ days of data.
The notebook can be refreshed at that point to include the analysis.
```

Row 9: Summary (rich_text, width 12)
```markdown
## 🩹 Summary

**Detected:** [one-line summary]
**Diagnosed:** [one-line summary]
**Audited:** [one-line summary of existing coverage and gap]
**Created:** [experiment name and link]
**Early signal:** [one-line summary or "pending"]

---
*This notebook was generated autonomously by the Self-Healing Funnel skill.*
```

Call `create_notebook` with the assembled rows.

---

### Phase 7: Output Summary

Present the user with a concise summary of everything created:

```
🩹 Self-Healing Funnel Analysis Complete

Charts created:
- [Funnel bar chart name + link]
- [Funnel trend chart name + link]
- [Platform diagnosis chart + link]
- [Device diagnosis chart + link]

Experiment created:
- [Experiment name + link]

Notebook:
- [Notebook name + link]

Key finding: [one-sentence summary of the bottleneck and proposed fix]
```

If the user wants a Slack digest, draft one to the channel they specify.

---

## Configuration & Thresholds

These defaults work well for most funnels. Adjust based on your data volume and funnel characteristics:

| Parameter | Default | Notes |
|-----------|---------|-------|
| Trend window | Last 12 Weeks | Longer for slow-moving funnels |
| Trend interval | Weekly (7) | Daily for high-volume funnels |
| Decline threshold | -15% relative | Lower for stable, high-volume funnels |
| Conversion window | 30 days (2592000s) | Shorter for same-session funnels |
| Funnel mode | ordered | Use "sequential" only if the funnel is strict |
