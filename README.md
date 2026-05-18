# Redline

Redline is a Claude Code plugin for processing security incidents at **Argo Financial**, a fintech that handles payments, KYC, and customer financial data under GDPR, PCI-DSS, GLBA, and state breach notification laws.

When a security incident happens, the on-call engineer is under pressure to do three things fast: figure out what happened, scrub PII before any of it lands in a doc or a Slack channel, and surface the regulatory exposure so legal and compliance can start their own clocks. Redline runs that pipeline from a single command.

---

## What it currently does

```
Raw incident log
       ↓
┌─────────────────────┐
│  incident-reviewer   │  Extract → Mask PII → Flag compliance
└─────────┬───────────┘
          ↓
┌─────────────────────┐
│  🛑 Human Checkpoint │  Review masked output + flags → approve or stop
└─────────┬───────────┘
          ↓
┌─────────────────────┐
│  incident-reporter   │  Format report → Draft Slack summary → Save
└─────────────────────┘
```

The pipeline produces **two outputs** today, both saved to `incidents/`:

1. **Incident report** (`INC-YYYY-MM-DD-###.md`) — formal write-up with timeline, affected systems, compliance flags, action items, and lessons learned. Audience: engineering leadership and compliance.
2. **Slack summary** (`INC-YYYY-MM-DD-###-slack.txt`) — five-line summary for the incident channel. Audience: the broader engineering org.

Both are generated from the same reviewed, masked incident data. Both go through a human checkpoint before anything is written.

---

## The exercise — adding an executive update

Right now, the reporter produces a leadership-friendly **executive summary** as a section *inside* the formal report. Leadership rarely reads the full report — they want the four lines that matter, in their inbox or Slack, fast.

In this exercise, you will:

1. **Add a third output to the reporter agent** — a standalone "Executive Update" file that contains only status, business impact, customer impact, and the next action.
2. **Update the skill to open that file** when the pipeline finishes, so the executive update is the first thing the engineer sees after a run.

Two small edits. One new artifact. No code, no framework, no DSL — just plain English instructions added to two markdown files.

---

## Making the changes

### Change 1 — Add the executive update output

Open `agents/incident-reporter.md`. Scroll to the bottom of the existing outputs section (after Output 2 — Slack Summary) and add this block:

```
## Output 3 — Executive Update

EXECUTIVE UPDATE
=========================================
Status:       [Resolved / Monitoring / Ongoing]
Impact:       [one sentence, business impact only]
Customer:     [number affected, what was exposed]
Next Action:  [single most important next step]
=========================================

Save the executive update as a separate file:
incidents/INC-[YYYY-MM-DD]-[###]-exec.md
```

Save the file. That's it. The reporter agent will pick up the new output block the next time the pipeline runs.

### Change 2 — Open the executive update at the end of the pipeline

Open `skills/incident/SKILL.md`. Find Step 4 (the existing `open` step that opens the full incident report). Update it to also open the new executive update file:

```
### 📂 Step 4 — Open Reports
After the incident-reporter completes, open the saved files for the user.

Run: open [path to saved executive update file]
```

Save the file. Done.

---

## Running the exercise

You have two sample incident logs in `samples/` to test with. The recommended one for this exercise is `unauthorized-access.txt` — it's the cleanest end-to-end run and the executive update format makes the most sense against it.

**Before the change** — run the pipeline and observe the two output files generated in `incidents/`:

```
/redline:incident samples/unauthorized-access.txt
```

**Make the two edits above.**

**After the change** — run the same command again and observe:

- Three output files are now generated in `incidents/`
- The executive update opens automatically alongside the full report
- The same masked, reviewed data now serves three audiences instead of two

Compare the outputs from before and after. Notice that you changed the behavior of a production-grade pipeline by editing two markdown files. No agent code was rewritten. No new tool was registered. The plain-English instructions in the reporter agent and the skill are the contract.

---

## Setup

Prerequisites:

- [Claude Code](https://docs.claude.com/en/docs/claude-code) installed and authenticated

Clone and launch:

```bash
git clone https://github.com/tarumfraz/redline.git
cd redline
claude
```

The plugin is auto-discovered from the repo root.

---

## Usage

```
/redline:incident <path-to-incident-log>
```

Example:

```
/redline:incident samples/unauthorized-access.txt
```

The plugin will:

- Read and mask the incident log
- Present masked output and compliance flags for your review
- After approval, generate the report files in `incidents/`
- Open the report(s) automatically

---

## File structure

```
redline/
├── plugin.json                     # Plugin metadata
├── agents/
│   ├── incident-reviewer.md        # Agent 1: extraction, PII masking, compliance flagging
│   └── incident-reporter.md        # Agent 2: report formatting, Slack summary, exec update
├── hooks/
│   └── hooks.json                  # Pre-run file validation, output directory check
├── skills/
│   └── incident/
│       └── SKILL.md                # Pipeline orchestrator
├── samples/                        # Sample incident logs for testing
│   ├── unauthorized-access.txt
│   ├── payment-processing-outage.txt
│   └── vendor-breach.txt
└── incidents/                      # Output directory for generated reports
```

---

## Sample incident files

Three sample logs are included in `samples/` for testing:

- `unauthorized-access.txt` — hardcoded credentials in public GitHub repo, 1,200 customer records exposed, GDPR and state breach law triggers
- `payment-processing-outage.txt` — bad deploy causes 50-minute payment service outage, SLA breach, PCI-DSS exposure
- `vendor-breach.txt` — third-party KYC vendor ransomware attack, biometric data at risk, GDPR 72-hour clock running

---

## Architecture decisions

**Why two agents instead of three?**
The reviewer agent handles both extraction and PII masking in a single pass. Splitting them into separate agents would require re-reading the same source material twice with no additional reasoning value, just a redundant hop and an extra failure point.

**Why a human in the loop before formatting?**
In regulated environments, a hallucinated compliance flag or a missed PII instance has legal consequences. The checkpoint ensures a human validates the masked output and flags before any report is generated or distributed.

**Why local file input?**
This is v1. The architecture supports MCP integration with Slack, PagerDuty, Jira, and observability tools for automated incident data ingestion. Local file input keeps the plugin portable and dependency-free for initial adoption.

---

## Author

Tarum Fraz
