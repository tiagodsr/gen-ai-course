# Session 1 — Agent Harnesses and PydanticAI

The Meridian Exchange is a fictional stock exchange that runs in a local
simulator (`meridian/`). Incidents happen in the simulator; your agent
investigates them through tools, and the simulator reacts to what it does.

An agent is a model plus a **harness**: the instructions, the tools, the
format of the answer and the checks around it. In this session you keep the
model and build the harness:

- **Exercise 1:** give the agent the tools it needs to investigate.
- **Exercise 2:** let the agent fix the problem, and check in code that the fix worked before it can say "resolved".

Run every command from the repository root, after `uv sync` and after setting
`OPENAI_API_KEY` in a local `.env` file. Commands are written on one line so
that they work in any terminal (PowerShell included).

## What you have

```text
01-agent-harness/
├── starter/harness.py   # the agent you start from
├── cases/               # the incidents of this session
└── checks/              # the success criteria of each incident (PASS / FAIL)
```

- **`starter/harness.py`** is a PydanticAI agent. It has three parts you can
  change: the instructions (`INSTRUCTIONS`), the format of the answer
  (`IncidentReport`) and the tools. Right now it has one tool:
  `read_alert`.
- **`cases/`**: one file per incident. Each file says what breaks in the
  simulator. The agent never reads these files; it only sees what its tools
  return.
- **`checks/`**: one file per incident with its success criteria. The runner
  reads them and prints PASS or FAIL.

The incidents:

| Case | What happens | Exercise |
|---|---|---|
| `s01-market-data-delay` | Market data arrives late. | 1 |
| `s01-metrics-unavailable` | The gateway is slow and the metrics service is down. | 1 |
| `s01-flaky-metrics` | The gateway is slow and the first metrics request fails. | 1 |
| `s01-gateway-latency` | The order gateway is slow. | 2 |
| `s01-stale-alert` | Everything is fine, but an old alert was left open. | 2 |
| `s01-alert-cleared-errors` | The latency alert closed by itself. Is the incident over? | 2 |

## The commands

Watch an incident in the simulator, without any agent:

```bash
uv run mrx-chaos run sessions/course-02-agentic-ai/01-agent-harness/cases/s01-gateway-latency.yaml --ticks 5 --component order-gateway
```

Run an agent on one incident and see everything it did (the trace):

```bash
uv run python -m meridian.run --harness sessions/course-02-agentic-ai/01-agent-harness/starter/harness.py --scenario sessions/course-02-agentic-ai/01-agent-harness/cases/s01-gateway-latency.yaml
```

Run an agent on every incident and see one line per incident with PASS or FAIL:

```bash
uv run python -m meridian.run --harness sessions/course-02-agentic-ai/01-agent-harness/starter/harness.py --cases-dir sessions/course-02-agentic-ai/01-agent-harness/cases --summary
```

- `--harness`: the agent file to run. Repeat it to compare two versions.
- `--scenario`: one incident. `--cases-dir`: every incident in a folder.
- `--summary`: only the table, without the traces.

A model does not always answer the same way. Run a case more than once before
drawing a conclusion.

## Exercise 1 — Give the agent the tools it needs

1. Run the starter on every incident (the `--summary` command above). Most
   cases fail.
2. Run it on `s01-gateway-latency` alone and read the trace. What did the
   agent use? What could it not find out?
3. Copy `starter/harness.py` to `starter/harness_v2.py`.
4. Add the tools the agent is missing. A tool is a Python function that
   calls the simulator through `ctx.deps.tools` (see `meridian/tools.py`).
   The simulator offers, among others:
   - `read_metrics(component, window_ticks, per_instance)`: recent metrics,
     optionally per instance;
   - `list_runbooks()` and `read_runbook(name, version)`: the procedures for
     each component.

   For every tool, decide: its name, its arguments, its docstring (that is
   what the model reads) and what it returns.
5. Run `harness_v2.py` on every incident.

**Done when** the three exercise 1 cases pass: `s01-market-data-delay`,
`s01-metrics-unavailable` and `s01-flaky-metrics`. The agent should also name
the instance with the problem in `s01-gateway-latency`, but that case only
passes after exercise 2.

**Write down:** which tools you added, why each one, and one decision about a
tool interface that changed what the agent did.

## Exercise 2 — Fix the problem, and check that it worked

So far the agent can only propose. Now it gets one action, and a check in
code that stops it from saying "resolved" unless the metrics confirm it.

1. Add a tool that restarts one instance. The simulator offers
   `ctx.deps.tools.restart_instance(instance_id)`. An instance is back after
   about 2 minutes. Restart only instances of non-critical components;
   repeated and unsafe actions are Session 2.
2. Run `s01-gateway-latency`. The agent should find `order-gateway-2`,
   restart it, and report "resolved".
3. Now add the check. The format of the answer is checked by PydanticAI, but
   nothing checks that the answer is true. Write a function that, when the
   report says `resolved`, reads the metrics itself and rejects the report if
   they do not confirm recovery. Use the recovery criterion of the runbook:
   latency below 20 ms and error rate below 1 % in the last 3 readings.

   PydanticAI runs this kind of check with an output validator: a function
   registered with `agent.output_validator(...)` that receives the report and
   either returns it (accepted) or raises `ModelRetry("reason")` (rejected:
   the report goes back to the model with the reason, and the model must
   answer again). Give the agent a few attempts: `Agent(..., retries=3)`.
4. Run every incident again. Look at:
   - `s01-alert-cleared-errors`: the latency alert closed, but errors are at
     6 %. A restart does not fix it. The check must stop "resolved".
   - `s01-stale-alert`: nothing is broken. The right answer is "resolved",
     without restarting anything.

**Done when** all six incidents pass.

**Write down:** why doing an action is not the same as fixing the problem,
and why a check in code is different from asking the model, in the
instructions, to "only report resolved with proof".

## Deliverable

- `starter/harness_v2.py`.
- A short note: the tools you added and why, the check you added and why, and
  what is still not demonstrated.
- One new incident in `cases/` with its file in `checks/` (copy an existing
  pair and change the fault, the component or the incident text). Run it.

## Conventions

- Keep a `status` field in the report. The checks read it.
- Do not edit `meridian/`. The simulator, the incidents and the runner are
  shared infrastructure.

## Scope boundary

One agent, a few tools, one simple action (restarting one instance of a
non-critical component), a typed answer and a check on the answer. Repeated
or unsafe actions, idempotency and choosing what goes into the context belong
to Session 2. MCP, memory, security policies and multi-agent coordination come
later in the course.

## Session 1 notes (redoing the exercises with `harness_v2.py`)

My own notes, written so I can come back later and remember what I did and
why. I started from a `harness_v2.py` identical to the starter (one tool:
`read_alert`) and improved it step by step, using what I saw in the terminal.
Each step says what I ran, what I saw, and what I changed because of it.

Base command (from the repository root):

```bash
uv run python -m meridian.run --harness sessions/course-02-agentic-ai/01-agent-harness/starter/harness_v2.py --cases-dir sessions/course-02-agentic-ai/01-agent-harness/cases --summary
```

To look at one incident in detail, replace `--cases-dir .../cases --summary`
with `--scenario .../cases/<case>.yaml`. This prints the trace: every tool the
agent called, what came back, and the final report.

### The big picture: what is the agent and what does it see?

An agent is a model plus a harness. Think of the model as a smart new hire who
knows nothing about our systems. The harness is everything we hand them: a
briefing, a set of tools, a form to fill in at the end, and a supervisor who
checks the form.

The model only ever receives these things:

| What it receives | Where it comes from | Mine to change? |
|---|---|---|
| Instructions (system prompt) | `INSTRUCTIONS` in `harness_v2.py` | yes |
| Tools: name, arguments, docstring | the functions registered with `agent.tool(...)` | yes |
| Shape of the final answer | `IncidentReport` | yes |
| The first message | the runner, built from the case YAML (`default_prompt` in `meridian/agent_runtime.py`) | no |
| What each tool returns | the simulator, through `ctx.deps.tools` | no |

The first message looks like this. It is where the very first
`read_alert("ALR-1001")` comes from: the alert id is written in the message.

```text
Incident INC-2041 (10:05): Alert latency_high on order-gateway. Members report slow order acknowledgements. Related alert: ALR-1001.
```

The agent never reads `cases/` or `checks/`. If there is no tool for
something, the agent cannot see it.

### Step by step: what I did and what changed

**Step 0: watch the incident with no agent.**
`uv run mrx-chaos run .../s01-gateway-latency.yaml --ticks 5 --component order-gateway`
Latency of `order-gateway` climbs about 6 ms per minute (149 → 179 ms). The
runbook says recovery means below 20 ms, so this is far from fine. This view
is the overall average, so it does not say which instance is the problem.

**Step 1: baseline with only `read_alert`.** The summary was 1 PASS out of 6
(`s01-metrics-unavailable`, and only by accident: with no tools the agent
cannot act and cannot claim "resolved"). In the trace of `s01-gateway-latency`:

- The model called `read_alert("latency_high")`, which is the rule name, not
  the alert id, and got `Unknown alert`.
- It then invented a generic diagnosis ("investigate load", "consider
  scaling") with no real data.
- `read_metrics` was called 0 times, so that check failed.

Lesson: with no tools to look with, the model fills the gap with plausible
text. Also, **the model decides when to stop**: it calls `final_result`
whenever it thinks it has enough. It did not hit any limit (`--max-requests`
defaults to 20; it used 3). The harness does not force it to continue or stop.

**Step 2: Exercise 1, tools to investigate.** I added `read_metrics`,
`list_runbooks` and `read_runbook`, registered each with `agent.tool(...)`, and
rewrote the instructions: read the metrics per instance, read the runbook,
never act without data, retry once if a tool fails with `retryable=true`,
otherwise report `insufficient_information`.

| Tool | Why |
|---|---|
| `read_metrics(component, window_ticks, per_instance)` | The eyes. With `per_instance=true` it shows memory, CPU and latency of each instance, which is how the agent finds `order-gateway-2`. |
| `list_runbooks()` | The agent does not know the runbook names, so it lists them first. |
| `read_runbook(name)` | Gives the procedure and the recovery criterion. The check for `s01-market-data-delay` requires reading `market-data-delays`. |

Result: the trace of `s01-gateway-latency` became alert → metrics per instance
→ list of runbooks → runbook. The agent named `order-gateway-2` (latency
257 → 305 ms, memory above 90 %) and **proposed** a restart, but could not do
it because it had no tool for that yet. The three Exercise 1 cases passed
(`s01-market-data-delay`, `s01-metrics-unavailable`, `s01-flaky-metrics`). The
other three still failed, as expected.

**Step 3: Exercise 2.1, one action.** I added `restart_instance(instance_id)`
and two lines to the instructions: restart only the instance the metrics point
to (and only if the runbook says so), and read the metrics again after acting.

**Step 4: the problem the check will solve.** Since the baseline,
`s01-alert-cleared-errors` showed `resolved (claims recovery)` and FAIL. The
latency alert closed by itself, but errors were at 6 %. The agent trusted the
closed alert. Doing an action, or seeing an alert close, does not mean the
problem is fixed.

**Step 5: Exercise 2.3, `verify_recovery` as an output validator.** I wrote a
function that runs when the report says `resolved`. It reads the metrics itself
(last 3 readings) and raises `ModelRetry` if any reading has latency ≥ 20 ms or
error rate ≥ 1 %. I registered it with `agent.output_validator(verify_recovery)`
and created the agent with `retries=3`. The trace of
`s01-alert-cleared-errors` shows it working:

```text
7  final_result(status: "resolved")
   retry: "Recovery not confirmed on order-gateway: at 16:12 latency_ms=5.0, error_rate=0.061..."
8  read_metrics(order-gateway, window_ticks=3 ...)
9  final_result(status: "unresolved")
```

The model said "resolved", the validator read the data and rejected it, the
model checked the metrics again and corrected itself. PASS.

**Step 6: a fix after a FAIL, `s01-stale-alert`.** Once the validator existed,
the agent started saying `unresolved` on a healthy system (5 ms, 0.1 % errors).
The validator did not even run, because it only acts when the status is
`resolved`. It is **one-sided**: it stops a false "resolved", but it does not
push the agent to say "resolved" when that is the truth. That side belongs to
the instructions, so I added to point 4: "if the metrics already meet the
runbook's recovery criterion and nothing is broken (for example an old alert
left open), report resolved without restarting anything". After that the agent
said `resolved` with 0 restarts, and the validator accepted it.

**Step 7: final result, 6 out of 6 PASS.**

| Case | Status | Actions |
|---|---|---|
| `s01-alert-cleared-errors` | unresolved (the validator stopped a false "resolved") | 0 |
| `s01-flaky-metrics` | resolved | 1 |
| `s01-gateway-latency` | resolved | 1 |
| `s01-market-data-delay` | insufficient_information | 0 |
| `s01-metrics-unavailable` | unresolved | 0 |
| `s01-stale-alert` | resolved | 0 |

### Key takeaways

- **Tool vs output validator.** Who decides to call it?
  - A **tool** is called only **if the model chooses to**.
  - An **output validator** is called **by PydanticAI, every time**, when the
    model hands in its final report. The model does not even know it exists
    until it is rejected.

  If `verify_recovery` were a tool, the model could skip it and say
  "resolved" anyway. When the validator raises `ModelRetry`, the report goes
  back to the model with the reason and it must answer again, up to
  `retries=3` times.
- **A check in code vs asking in the instructions.** "Only report resolved with
  proof" in the instructions is a suggestion the model can ignore or
  misunderstand. The validator always runs, and it reads the data itself
  (`ctx.deps.tools.read_metrics`) instead of trusting what the model wrote.
- **The docstring is the interface.** The model knows to use `per_instance=true`
  only because the `read_metrics` docstring says it "shows which instance has a
  problem". The interface decision that changed the agent the most was this
  `per_instance` argument: without it the agent only sees the average and
  cannot name `order-gateway-2`.
- **Acting is not fixing.** Restarting an instance, or seeing an alert close,
  does not prove the incident is over. Only the metrics prove it.
- **Two sides of "resolved".** The validator protects against a false
  "resolved". The instructions handle saying "resolved" when it is true
  (`s01-stale-alert`).
- **Models vary.** Run each case more than once before drawing a conclusion.

### What is still not demonstrated

- I ran each case only a few times, so I do not know the real pass rate. Runs
  vary: `s01-metrics-unavailable` gave `insufficient_information` once and
  `unresolved` another time, and both passed.
- `s01-market-data-delay` ends with `insufficient_information` even though the
  agent read the metrics and the runbook. The check only asks that it reads
  them and does not claim "resolved", so it passes, but the answer is weak.
- The agent sometimes mixes up ids (`read_alert("INC-2043")` instead of
  `ALR-...`) and only corrects itself afterwards.
- The validator's thresholds (20 ms, 1 %) come from the `order-gateway`
  runbook. Other components would need the criterion from their own runbook.
- The validator only looks at the component of the alert. It does not check
  individual instances or dependencies.
- The new incident for the deliverable (a `cases/` + `checks/` pair) is still
  to do.
