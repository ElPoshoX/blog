---
title: "Anatomy of a Loop in Claude Code: The Building Blocks, the Design, and the Metrics That Matter"
weight: 1
tags: ["claude-code", "loop-engineering", "automation", "sub-agents", "devops", "metrics"]
date: "2026-06-12T10:00:00+00:00"
showDateUpdated: false
cascade:
  showDate: false
  showAuthor: false
  invertPagination: true
---

# Anatomy of a Loop in Claude Code: The Building Blocks, the Design, and the Metrics That Matter

{{< lead >}}
The five building blocks of a loop taken apart, a real loop step by step, the metrics you should track, and why the sub-agent reviewer is the most important component in the whole system.
{{< /lead >}}

This post is the technical companion to [The Loop Nobody Tests]. If you want the argument, start there.

I have been building loops in Claude Code for months to automate tasks across my repos. Some work great. Others burned tokens without producing anything useful. Here I take each piece apart.

One thing before we start: everything you see here works with your Claude Code subscription. Zero extra cost. No paid third-party APIs, no additional cloud services. Just Claude Code and what it already ships with.

## The five building blocks of a loop, taken apart

A loop in Claude Code is not a monolithic thing. It is five blocks you can combine. Understanding each one separately gives you real control over the system's behavior.

{{< mermaid >}}
graph TD
    A[Automations] -->|triggers| B[Execution Engine]
    B -->|runs in| C[Worktrees]
    B -->|follows rules from| D[Skills]
    B -->|connects via| E[Connectors / MCP]
    B -->|delegates to| F[Sub-agents]
    B -->|persists state in| G[State Files]
    F -->|maker + reviewer| B
{{< /mermaid >}}

### Block 1: Automations

Automations are what fires your loop. The trigger. Without this, there is no cycle.

The most direct tool is `/loop`. You use it for recurring tasks that the agent runs with auto-pacing.

Real example: a `/loop` for morning issue triage.

```
/loop 30m

Check open issues in the repo elposhox/tekal-api.
For each new issue (created in the last 24h):
1. Read the title and body
2. Classify as: bug, feature-request, question, or needs-info
3. Add the corresponding label
4. If it's a bug with a clear error message and affects a single file, add label "autofix-candidate"
5. Write a summary in AGENTS.md with: issue number, classification, and whether it's an autofix candidate

Stop after processing all new issues or after 3 iterations without finding new issues.
```

That runs every 30 minutes. No intervention. The agent checks, classifies, and leaves you a record.

You also have `CronCreate` for more specific schedules and `ScheduleWakeup` for one-off delays. But `/loop` is the one you will use 80% of the time.

### Block 2: Worktrees

When your loop modifies code, you need isolation. If the agent edits files on your working branch, it will wreck your stuff. Worktrees solve this.

In Claude Code you configure isolation in two ways:

For sub-agents you launch with the Agent tool:

```json
{
  "description": "Fix issue in isolation",
  "prompt": "Fix the bug described in issue #42...",
  "isolation": "worktree"
}
```

The agent gets its own worktree. It works there. Your main branch stays untouched until you decide to merge.

For agent presets in `.claude/agents/`, you add the directive in the prompt:

```markdown
# maker.md
You MUST work in an isolated worktree for all code changes.
Use the EnterWorktree tool at the start of every task.
Never modify files in the main working directory directly.
```

If your loop touches code and does not use worktrees, it is a matter of time before it breaks something.

### Block 3: Skills

Skills are the instructions that give the agent judgment. The difference between an effective SKILL.md and one the agent ignores is the quality of its definition.

**SKILL.md that the agent ignores:**

```markdown
# triage.md
Review issues and classify them appropriately.
Use good judgment to determine priority.
```

That says nothing. "Good judgment" is not an instruction.

**SKILL.md that the agent follows:**

```markdown
# triage.md
## Classification criteria

An issue is a BUG if:
- It has an error message or stack trace
- It describes behavior that used to work and stopped working
- It includes steps to reproduce

An issue is AUTOFIX-CANDIDATE if it meets ALL of these:
- It is a bug with a clear error message
- The error points to a single file (grep the filename in the stack trace)
- It does not require database schema changes
- It does not require changes in more than 2 files

An issue is NEEDS-INFO if:
- It has no steps to reproduce
- The report is ambiguous ("it doesn't work" with no details)
- It does not include version or environment

## What NOT to do
- Do not change labels on issues that already have a classification
- Do not comment on issues that already have a response from the maintainer
- Do not attempt fixes for issues that require public API changes
```

**What goes in CLAUDE.md vs what goes in separate skills:**

| In CLAUDE.md | In separate skills |
|---|---|
| Global project conventions | Specific task instructions |
| Style and linting rules | Classification criteria |
| Important paths | Step-by-step workflows |
| Things that apply to EVERY interaction | Things that apply only to ONE type of task |

Your CLAUDE.md is what the agent reads every time. Skills are read when needed. If you cram everything into CLAUDE.md, the context bloats and the agent prioritizes making not-so-great decisions with information that may or may not be relevant at that point in time.

### Block 4: Connectors (MCP Servers)

Connectors are MCP servers that give the agent access to external services. GitHub, Linear, Jira, your database, whatever you need.

Example: a GitHub MCP server that the agent uses inside the loop.

In your `.claude/settings.json`:

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "${GITHUB_TOKEN}"
      }
    }
  }
}
```

With that, your agent can list issues, read PRs, create comments, open PRs. All from inside the loop without shelling out to make `gh` calls.

For Linear it is similar:

```json
{
  "mcpServers": {
    "linear": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-linear"],
      "env": {
        "LINEAR_API_KEY": "${LINEAR_API_KEY}"
      }
    }
  }
}
```

The important thing: each MCP server you add is a set of tools the agent can use. More tools = more context = more tokens per iteration. Do not add servers you do not need for the loop.

### Block 5: Sub-agents

A serious loop is not a single agent doing everything. It is sub-agents with distinct roles.

The configuration lives in `.claude/agents/`. Each file is an agent preset with its own instructions.

**The maker** (`maker.md`):

```markdown
# Maker Agent

## Role
Implement fixes for issues classified as autofix-candidate.

## Instructions
1. Read the full issue (title, body, comments)
2. Identify the affected file from the stack trace
3. Enter an isolated worktree
4. Implement the simplest fix that solves the problem
5. Run the project's tests
6. If tests pass, commit with message: "fix: resolve #ISSUE_NUMBER - SHORT_DESCRIPTION"
7. If tests fail after 2 fix attempts, abandon and log in AGENTS.md

## Constraints
- Maximum 2 files modified per fix
- Do not change existing tests to make them pass (that is cheating)
- Do not add new dependencies
- Maximum 3 iterations per issue
```

**The reviewer** (`reviewer.md`) gets its own section below because it is the most important component.

### Block 6: State (Progress Files)

Without persistent state, your loop has no memory between runs. Each execution starts from zero. The progress file solves this.

Design of an `AGENTS.md` that works:

```markdown
# Loop State - Issue Triage & Autofix

## Last Run
- Date: 2026-06-12T08:30:00Z
- Issues processed: 5
- Fixes attempted: 2
- PRs opened: 1

## Issue Log

| Issue | Status | Classification | Fix Attempted | Result | PR |
|-------|--------|---------------|---------------|--------|----|
| #142 | done | bug/autofix-candidate | yes | fix merged | #148 |
| #143 | done | feature-request | no | N/A | N/A |
| #144 | done | bug/autofix-candidate | yes | tests failed, abandoned | N/A |
| #145 | done | needs-info | no | commented asking for repro | N/A |
| #146 | done | question | no | N/A | N/A |

## Metrics (rolling 7 days)
- PRs opened: 4
- PRs merged without changes: 3
- PRs that needed manual fixes: 1
- Issues abandoned: 2
- Total iterations: 23
```

The agent reads this at the start of each run. It knows what it already processed. It knows what failed before. It does not repeat work.

**Format note:** use Markdown with tables. The agent parses it well and you can read it without special tools. JSON works too but it is harder to scan by eye.

## Designing a concrete loop, step by step

Real loop I ran on a Kubernetes operator of my own (tekal-waf-sync). The repo has a pending spec: add integration tests with Moto and a validation layer to the WAF client. 7 tasks defined in a tasks file. The loop picks them up one by one, implements in an isolated worktree, runs tests and lint, hands off to the reviewer, and opens a PR if everything passes.

{{< mermaid >}}
sequenceDiagram
    participant Orch as Loop Principal
    participant State as AGENTS.md
    participant Spec as tasks.md
    participant Maker as Maker Agent
    participant Reviewer as Reviewer Agent
    participant GH as GitHub (MCP)

    Orch->>State: Read previous state
    Orch->>Spec: Read pending tasks
    Orch->>Orch: Identify next incomplete task

    Note over Orch,GH: Repeats for each pending task

    Orch->>Maker: Implement task N
    Maker->>Maker: Enter isolated worktree
    Maker->>Maker: Implement per spec
    Maker->>Maker: Run tests + lint

    alt Tests and lint pass
        Maker->>Reviewer: Review task N implementation
        alt Reviewer approves
            Reviewer->>GH: Open PR
            Reviewer->>State: Log PR + task completed
        else Reviewer rejects
            Reviewer->>Maker: Specific feedback
            Maker->>Maker: Second attempt
            Maker->>Reviewer: Re-review
        end
    else Tests fail 3 times
        Maker->>State: Log abandonment with reason
    end
{{< /mermaid >}}

### Step 1: The loop prompt

```
/loop

Implement the pending tasks from the waf-integration-tests spec.

1. Read AGENTS.md to see which tasks you already completed
2. Read openspec/changes/waf-integration-tests/tasks.md for the full list
3. Identify the next task whose dependencies are satisfied
4. For each task:
   a. Launch a maker sub-agent with isolation: worktree
   b. The maker implements per the task's description and acceptance criteria
   c. The maker runs `make test` and `make lint`
   d. If it passes, launch the reviewer sub-agent
   e. If the reviewer approves, commit and open PR via GitHub MCP
   f. Log result in AGENTS.md
5. Stop when: all tasks are completed, OR after 3 tasks per run, OR if you have been running for more than 45 minutes

When finished, update AGENTS.md with run metrics.
```

### Step 2: The skill with spec criteria

Instead of a generic triage skill, the loop uses the spec as source of truth. The tasks.md defines for each task:

- Which files to create or modify
- Explicit acceptance criteria (tests pass, coverage >= 80%, lint clean)
- Dependencies between tasks (task 3 depends on task 1, task 5 depends on task 4)

The maker does not need to "decide what to do." It only needs to implement what the spec says. You resolve the ambiguity when you write the spec, not when you run the loop.

[!IMPORTANT] 
Using a spec here is totally optional. I work this way because it gives me two things that matter to me.
- Understanding what will be created before it gets implemented.
- Generating deltas on each spec (using OpenSpec) that become documentation for the project.

### Step 3: The maker sub-agent that implements

For this loop, the maker has different instructions than a bugfix maker:

- **Can create new files** (the spec asks for it: `internal/waf/validation.go`, `test/integration/`)
- **Can add dependencies** (testcontainers-go for LocalStack)
- **Has the spec as input** (not a GitHub issue, but a task with acceptance criteria)
- **Maximum 5 iterations** (more than a bugfix because this is new implementation)

The constraint that does not change: if tests or lint fail after 3 attempts, abandon and log why.

### Step 4: The reviewer sub-agent

Detailed in its own section below. For spec tasks, the reviewer verifies against the defined acceptance criteria, not against generic "best practices."

### Step 5: The progress file

```markdown
# Loop State - waf-integration-tests

## Last Run
- Date: 2026-06-16T10:30:00Z
- Tasks attempted: 2
- PRs opened: 2
- Tasks abandoned: 0

## Task Log

| Task | Status | Iterations | Result | PR |
|------|--------|-----------|--------|-----|
| 1. validation.go | done | 2 | merged | #23 |
| 2. client integration | done | 1 | merged | #24 |
| 3. validation tests | in-progress | - | - | - |
| 4. testcontainers setup | pending | - | - | - |
| 5. WAF integration tests | pending | - | - | - |
| 6. reconciler integration | pending | - | - | - |
| 7. CI pipeline | pending | - | - | - |

## Metrics
- PRs opened: 2
- PRs merged without changes: 2
- Average iterations per task: 1.5
- Coverage delta: +4.2%
```

Walk-through of the first run:

1. Loop starts. Reads AGENTS.md: empty, first run. Reads tasks.md: 7 tasks, task 1 has no dependencies.
2. Launches maker for task 1: "Create `internal/waf/validation.go` with ValidateRuleGroupInput and ValidateByteMatchStatement." The maker enters a worktree, implements the validation functions, runs `make test` and `make lint`. Passes.
3. Launches reviewer. Checks: functions <= 50 lines? Coverage of the new file >= 80%? Follows CLAUDE.md conventions? Approves.
4. Opens PR #23. Updates AGENTS.md.
5. Next task with satisfied dependencies: task 2 (depends on task 1, which is done). Launches maker. Implements the integration in `client.go`. Tests pass. Reviewer approves. PR #24.
6. Task 3 depends on task 1 (done). Could continue, but it has done 2 tasks and the budget says maximum 3. Continues.
7. Implements task 3: table-driven unit tests for the validation layer. 3 iterations because the first attempt did not cover the edge case of empty InsertHeaders. Reviewer rejected it. Second attempt: ok. PR #25.
8. Already at 3 tasks. Stops. Updates AGENTS.md with metrics.

## Results of the loop on tekal-waf-sync

I ran this loop over the 7 tasks in the spec. These are the real numbers:

| Metric | Value |
|--------|-------|
| Spec tasks | 8 (7 implementation + 1 peer review) |
| Tasks completed by the loop | 8 |
| Commits | 4 |
| Tasks abandoned | 0 |
| Total iterations | ~12 |
| New tests written | 28 (17 unit + 11 integration) |
| Coverage delta | 100% -> 100% (recovered after peer review) |
| Auto-corrected errors | 3 (hugeParam lint, test count mismatch, Capacity *int64) |
| Peer review findings | 2 (coverage gap in updateShard, non-shared containers) |
| Human interventions | 1 (suggesting Floci over LocalStack, turned out to be a bad idea) |

What worked well:

- Table-driven tests generated correctly on the first attempt. 16 test cases for validation, covering the 6 invalid cases from the spec + boundary values (name of exactly 128 chars, exactly 1 InsertHeader).
- All 5 reconciler tests against real WAF (moto) passed on the first run without fixes.
- Lint stayed at 0 issues after each fix. The only error (hugeParam, `types.Rule` at 96 bytes) was detected and corrected in the same iteration.
- The testcontainers-go pattern turned out clean: setup in helper, teardown via `t.Cleanup`.
- The loop respected its own stop conditions (3 tasks per run) without needing a reminder.

What failed:

- The research agent hallucinated that Floci supported WAFv2 with "35 operations." In reality, Floci returned `UnknownOperationException`. Cost: ~3 minutes + a useless Docker pull. The cause was that I suggested Floci as an OSS alternative to LocalStack. The loop trusted my judgment, but my judgment was wrong.
- `go mod tidy` removed the testcontainers dependency when it is only used behind a build tag. Had to re-add with `GOFLAGS="-tags=integration"`. This is a known Go modules gotcha that the loop did not anticipate.
- `ValidateByteMatchStatement` returns `error` (singular via `errors.Join`), not `[]error`. The test expected 3 errors for 3 violations but received 2 because two ByteMatch violations get joined into a single error. Corrected in the second iteration.

What the loop could not do and I had to do by hand:

- Task 8 (peer review). The loop correctly identifies that it should not run peer review autonomously. It is a process that requires loading agent files from the Obsidian vault and making judgment calls on findings. This is correct: the loop FOR ME is an implementer, not a reviewer.
- Choosing between Floci and moto. When Floci failed, the loop recovered by searching for alternatives. But the original decision to try Floci was mine (the human's) and turned out to be a dead end.
- Deciding whether Rule 6 (Capacity > 0) applies to `UpdateRuleGroupInput`. The loop made the right call (it does not apply, Capacity is immutable after creation) but this required reasoning about AWS API semantics, not just following the spec literally.

---

## Metrics you should track

Claude Code does not give you a metrics dashboard. You have to build the observability yourself. These are the metrics that matter and how to extract them.

### Acceptance rate

Out of N PRs the loop opens, how many you merge without changes.

If this rate is high (>80%), your loop is producing quality code. If it is low, something in the maker or the reviewer is failing.

### Silent rejection rate

Out of the PRs you merge, how many you had to revert or fix afterward. This is the treacherous metric. A PR that "passed" review but later causes problems is worse than one you reject up front, because it is already in production.

### Iterations per task

How many passes the loop takes before producing an acceptable output. If the average is 1-2, your skill and your criteria are well calibrated. If it is 4+, something is off: either the skill is ambiguous, or the issues are not really autofix-candidates.

### Abandonment rate

How many tasks the loop cannot solve and abandons. Some abandonment is healthy (not everything is fixable by an agent). But if it abandons more than 50%, your "fixable" criteria are too loose.

### Quota consumption

How much of your usage limit each run consumes. This tells you whether the loop is sustainable or whether it will burn through your quota by mid-month.

### Script to extract the metrics

This script parses your `AGENTS.md` and generates the metrics:

```bash
#!/bin/bash
# metrics.sh - Parse the loop metrics file
# Usage: ./metrics.sh [path/to/loop-metrics.md]
#
# Expects a table with this format:
# | Task | Status | Iterations | Tests Added | Coverage After | Lint | Errors |

FILE="${1:-loop-metrics.md}"

if [ ! -f "$FILE" ]; then
    echo "Error: $FILE not found"
    exit 1
fi

# awk parses the table: column 3 = Status, 4 = Iterations, 5 = Tests Added
# Ignores headers and separators (lines with ---)
read -r done_count abandoned iterations tests_added <<< "$(awk -F'|' '
    /\| done \|/    { done++; iter += $4+0; tests += $5+0 }
    /\| abandoned \|/ { aband++ }
    END { printf "%d %d %d %d", done+0, aband+0, iter+0, tests+0 }
' "$FILE")"

total=$((done_count + abandoned))

echo "=== Loop Metrics ==="
echo ""
echo "Tasks completed:    $done_count"
echo "Tasks abandoned:    $abandoned"
echo "Total tasks:        $total"
echo "Total iterations:   $iterations"
echo "Tests added:        $tests_added"

echo ""
echo "=== Rates ==="

if [ "$total" -gt 0 ]; then
    echo "Completion rate:                $((done_count * 100 / total))%"
    echo "Abandonment rate:               $((abandoned * 100 / total))%"
    echo "Average iterations per task:    $((iterations / total))"
else
    echo "No tasks processed"
fi

echo ""
echo "=== Fill in manually ==="
echo "PRs merged without changes: ___"
echo "PRs that needed post-merge correction: ___"
echo "Silent rejection rate: ___"
```

### How to use these numbers to iterate

Numbers alone are useless if you do not use them to change something:

- **Low acceptance rate** -> Review the maker's instructions. It probably lacks context or the skill criteria are ambiguous.
- **High abandonment rate** -> Loosen the "autofix-candidate" criteria in the triage skill. You are sending issues that are not fixable.
- **Too many iterations per task** -> The maker is spinning its wheels. Add tighter constraints: maximum files, maximum lines changed, allowed fix types.
- **High silent rejection** -> Your reviewer is not catching the problems. Tighten its adversarial instructions.
- **Excessive quota consumption** -> Reduce the loop's scope. Fewer issues per run, stricter criteria, fewer maximum iterations.

## The sub-agent reviewer: the most important component

Separating maker from reviewer is what makes a loop reliable.

A single agent that writes and reviews its own code is like a dev who opens a PR and approves it themselves. Technically it "works." In practice, things slip through.

### Why the separation works

The maker has an implicit incentive: solve the issue. That leads it to take shortcuts. Fix the symptom, not the cause. Ship a fix that passes tests but introduces tech debt. Change a test to make it pass instead of fixing the code.

The reviewer has the opposite incentive: find reasons to reject. That tension is productive.

### Reviewer configuration

In `.claude/agents/reviewer.md`:

```markdown
# Reviewer Agent

## Role
Your job is to find reasons to REJECT this change. Do not look for reasons to approve it.

## Rejection criteria (reject if ANY applies)

### Correctness
- The fix resolves the symptom but not the root cause
- The fix introduces a new edge case not covered by tests
- The fix breaks the contract of a public function (changes signature, return type, or side effects)

### Conventions
- Violates any rule in the project's CLAUDE.md
- Does not follow the code style of the file it modifies
- The commit message does not follow conventional commits
- Modifies files outside the scope of the issue

### Quality
- The fix is more complex than necessary (there is a simpler solution)
- Adds duplicate code that already exists in the project
- Hardcodes values that should be configurable
- Does not handle errors that the original code handled

## What to do when rejecting
1. Specify WHICH criterion was violated
2. Explain WHY it is a problem (not just "violates convention X")
3. Suggest a concrete alternative
4. Log the rejection in AGENTS.md with the reason

## What to do when approving
1. Explicitly confirm you reviewed ALL criteria
2. List the files you reviewed
3. Log the approval in AGENTS.md
```

### Comparison: maker vs reviewer instructions

| Aspect | Maker | Reviewer |
|---------|-------|----------|
| Objective | Solve the issue | Find reasons to reject |
| Tone | Constructive, solution-oriented | Adversarial, skeptical |
| When facing ambiguity | Take the simplest decision | Reject and ask for clarification |
| When tests pass | Continue to the next step | Verify the tests cover the real case |
| Scope limit | "Solve with minimal change" | "Verify nothing outside scope was touched" |

### Trade-off: each sub-agent consumes quota

The valid question: if each sub-agent consumes quota, is a separate reviewer worth it?

My answer: it depends on the cost of a bug in production.

- **Personal repo, side project**: probably not worth it. A single agent with clear instructions is enough.
- **Work repo, code going to production**: worth every token. A bug that slips through and hits prod costs you more in debugging and revert time than the reviewer cost.
- **Loop that opens PRs automatically**: you absolutely need a reviewer. If the loop opens PRs without human review and you merge on trust, the reviewer is your last line of defense.

Rule of thumb: if you merged a PR from the loop and had to revert it, you need a reviewer. If you have never had to revert one, maybe you do not need it yet.

## Stop conditions

A loop without clear stop conditions is a bill generator. These are the five I use and how to combine them.

### Maximum iterations

The most basic. Hardcoded in the loop prompt.

```
Maximum 5 iterations. If after 5 iterations you have not completed the task, 
stop and report what you accomplished and what remains.
```

Not even on your worst day should you leave a loop without this. It is your circuit breaker of last resort.

### Stall detection

```
If two consecutive rounds produce no new changes (same diff, same files touched, same test results), stop immediately. 
Report: "Loop stalled after N iterations. Last significant change at iteration M."
```

This catches the oscillation anti-pattern: edit, fail, revert, edit, fail, revert. The agent "works" but makes no progress.

### Goal with explicit criteria

```
/goal All tests in src/handlers/ pass (exit code 0) 
The linter reports no new errors 
The total diff is under 50 lines
```

The `/goal` gives the agent a concrete definition of "I am done." Without this, "keep going until it works" is an invitation to disaster.

### Token budget in the prompt

```
Use a maximum of 10k tokens for this task. If you reach 80% of the budget (8k tokens) 
without completing, stop and report partial progress.
```

This is not a hard limit (Claude Code does not give you a token counter in the prompt), but the agent respects it reasonably well. Not perfect but better than nothing.

### TTL: scheduled task with expiration

If you use `CronCreate` or `/loop` with an interval, add an expiration mechanism:

```
This loop expires on 2026-06-17. After that date, do not execute any more runs.
On expiration, write a final summary in AGENTS.md with cumulative metrics.
```

### Combining them for a useful loop

{{< mermaid >}}
flowchart TD
    A[Start iteration] --> B{Max iterations?}
    B -->|Yes| C[STOP: Limit reached]
    B -->|No| D{Stall detected?}
    D -->|Yes| E[STOP: Loop stalled]
    D -->|No| F[Execute task]
    F --> G{Goal met?}
    G -->|Yes| H[EXIT: Success]
    G -->|No| I{Token budget exceeded?}
    I -->|Yes| J[STOP: Budget exhausted]
    I -->|No| K{TTL expired?}
    K -->|Yes| L[STOP: Time expired]
    K -->|No| A
{{< /mermaid >}}

In the full loop prompt it looks like this:

```
## Stop conditions (respect ALL of them)
1. Maximum 10 iterations per run
2. If 2 consecutive iterations produce no progress, stop
3. Goal: all tests pass AND lint clean AND diff < 100 lines
4. Estimated budget: 15k tokens per run
5. This loop expires on 2026-06-17

If ANY of these conditions is met, stop immediately.
Write result to AGENTS.md before finishing.
```

If one condition fails, another catches you.

## Checklist before launching a loop

Before hitting enter on any loop, check this:

- [ ] Stop conditions defined (maximum iterations + at least one more)
- [ ] SKILL.md with concrete criteria, not vibes
- [ ] Worktree configured if the loop touches code
- [ ] Progress file (AGENTS.md) initialized
- [ ] Reviewer agent configured if the loop opens PRs
- [ ] Required MCP servers configured in settings.json
- [ ] Quota consumption estimate per run

If any of these is missing, do not launch the loop. You will spend tokens debugging what you should have configured upfront.

## What comes next

In the next post in the series we will look at guardrails: how to make your loops behave even when you are not watching. Skills that act as constraints, hooks that validate before executing, and patterns for making the agent stop when it should.
