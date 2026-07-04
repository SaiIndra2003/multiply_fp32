# Takehome: Improving an AI Agent's Pass Rate on a Hardware Design Task

## Background

 We build evaluation tasks for AI coding agents. An AI agent is given a hardware design problem — a specification and a skeleton of SystemVerilog code — and must write a complete, functionally correct implementation. The agent's solution is then graded by a hidden testbench that checks for bit-exact correctness. It is like an exam problem for a student, where the student is given a problem statement and a skeleton code, and must write the complete solution. The professor has an automated grading system to grade the student's solution.

Your job in this takehome is to **analyze why an AI agent is failing a specific task, then modify the specification to make the problem easier, thereby improving its pass rate**.

You are given a task where an AI agent (like Claude Sonnet or Haiku) achieves approximately **10% pass rate** across 20 independent attempts. Your goal is to modify the prompt/specification so that the agent achieves a pass rate **between 40% and 70%** (out of at least 10 runs).

---

## Section 0: Setup (~15 min)

Install the following if you don't already have them:

- [github.com](http://github.com) account
- [Docker Desktop](https://www.docker.com/products/docker-desktop)
- [Cursor IDE](https://cursor.com) (recommended, but any IDE works)
- [Git](https://git-scm.com/downloads)
- [Python 3.10+](https://www.python.org/downloads/)
- [uv](https://docs.astral.sh/uv/getting-started/installation/) (Python package manager)
- [Icarus Verilog](https://steveicarus.github.io/iverilog/usage/installation.html) (`brew install icarus-verilog` on Mac, `apt install iverilog` on Linux)
- HUD account (you should have received an invite — let us know if not). Traces and job results live at [hud.ai](https://hud.ai).

**Windows users:** You will need WSL. Follow the instructions in: [WSL Guide](https://docs.google.com/document/d/1LF0nSO5fTD7e_OC6GThE4AWRrnbPHl7Sz-8rnBxjEiM/edit?usp=sharing) (sections 3.1–3.5).

You will also need an **Anthropic API key** (for calling the AI agent). Email your liaison to receive one.

---

## Section 1: Understanding the Task (~30 min)

### What the agent sees

Fork the task repository to your own GitHub account, then clone your fork:

1. Go to [https://github.com/phinitylabs/multiply_fp32](https://github.com/phinitylabs/multiply_fp32) and click **Fork**. Make sure the fork has **public** visibility. Remember to **uncheck** `Copy main branch only` when creating the fork.
2. Clone your fork:

```bash
git clone https://github.com/<your-github-username>/multiply_fp32.git
cd multiply_fp32
```

Forking gives you your own copy of the repo (with all branches included) where you can push your spec modifications.

The task is an **IEEE-754 FP32 floating-point multiplier**. The agent receives:

| File | Purpose |
|------|---------|
| `prompt.txt` | High-level instructions telling the agent what to do |
| `doc/spec.md` | Detailed specification of the multiplier's behavior |
| `sources/multiply_fp32.sv` | Empty module skeleton — the agent must fill this in |

The agent does **not** see the test or the golden solution. It reads the prompt and spec, writes SystemVerilog code, and can run its own tests. After it finishes, a hidden testbench grades its implementation by running 100 random FP32 multiplications and checking for bit-exact correctness against Python's IEEE-754 math.

### Branch structure

The repo has three important branches:

| Branch | What's on it |
|--------|-------------|
| `multiply_fp32_detailed_baseline` | The starting point: empty skeleton + full spec. **This is what the agent sees.** |
| `multiply_fp32_detailed_test` | Same as baseline, plus the hidden cocotb testbench (`tests/test_multiply_fp32.py`). Used for grading. |
| `multiply_fp32_detailed_golden` | Same as baseline, but with the complete, correct implementation in `sources/multiply_fp32.sv`. |

To look at the golden solution:

```bash
git checkout multiply_fp32_detailed_golden
```

To look at the hidden test:

```bash
git checkout multiply_fp32_detailed_test
cat tests/test_multiply_fp32.py
```

To return to the baseline (what the agent starts with):

```bash
git checkout multiply_fp32_detailed_baseline
```

Take time to read `doc/spec.md`, `sources/multiply_fp32.sv`, and the golden solution. Understand what the multiplier does and how the 7-stage FSM pipeline works.

---

## Section 2: Running Tests Locally (~10 min)

You can run the hidden testbench locally to verify that the golden solution passes and the baseline fails.

First, checkout the test branch (which has the testbench):

```bash
git checkout multiply_fp32_detailed_test
```

Run the test against the **baseline** code (should fail — the module has no internal logic, so the testbench will timeout waiting for a response):

```bash
cd tests
uv run pytest test_multiply_fp32.py --log-cli-level=INFO
```

You should see a failure. Now, temporarily swap in the golden solution and re-run:

```bash
cp ../golden/multiply_fp32.sv ../sources/multiply_fp32.sv
rm -rf sim_build __pycache__
uv run pytest test_multiply_fp32.py --log-cli-level=INFO
```

This should pass. Restore the baseline before continuing:

```bash
git checkout -- ../sources/multiply_fp32.sv
cd ..
```

---

## Section 3: Analyzing Agent Behavior (1 hour)

Here are the results of 20 runs of Claude Sonnet 4 attempting this task with the current spec:

**[View the 20 runs here](https://hud.ai/jobs/09836173-f7f4-4cd5-9a9f-8247caea5d20)**

The agent achieves approximately **10% pass rate** — it solves the task correctly in very few of the 20 attempts.

Open 1 or more of the **failing** runs and carefully read through the agent's work. Pay attention to:

- What approach does the agent take?
- Where does it go wrong?
- What does it get right vs. wrong?
- Does it test its own code? What does it miss?
- Are there patterns across multiple failures?

Understanding *why* the agent fails is the key to this takehome. The agent is not randomly broken — there are specific, identifiable reasons it produces incorrect implementations. Your analysis of these reasons will directly inform how you modify the spec. We strongly recommend working backwards--look at the final implementation the agent created at the end of the trace, and compare this to the golden solution. You can find the final implementation in HUD by scrolling to the bottom of the trace and looking through the code window in the final cell (on the left side of the transcript).

---

## Section 4: Modifying the Specification (~1–2 hrs)

Your goal is to modify `doc/spec.md` (and/or `prompt.txt`) so that the agent's pass rate improves to **between 40% and 70%**.

The spec currently provides detailed guidance for each pipeline stage. You may modify any part of it — add detail, remove detail, restructure it, add hints, change wording — whatever you believe will help the agent produce correct implementations more often, based on your analysis in Section 3.

A few principles to keep in mind:

- The agent is an AI model, not a human engineer. What helps a human read a spec may not help (or may even hurt) an agent.
- The agent can look things up, write its own tests, and iterate. Consider what information it truly needs vs. what it can figure out on its own.
- Don't give the agent the solution directly (e.g., don't paste the golden code into the spec). The goal is to guide it, not hand it the answer.

Each time you make a change, you should test it by running on HUD (Section 5).

---

## Section 5: Running on HUD (~30 min)

This framework uses **HUD v6**. Each problem runs in its own Docker image. The container serves a v6 control channel (`hud serve` on port 8765); the agent connects over SSH to edit files in the workspace. When the agent finishes, hidden tests grade the workspace via patch + pytest.

### One-time framework setup

Clone the evaluation framework and install its dependencies:

```bash
git clone https://github.com/phinitylabs/verilog-coding-template.git
cd verilog-coding-template
uv sync
```

Set your API keys. Your HUD API key can be created on [hud.ai](https://hud.ai) — go to your dashboard, then click "Phinity Labs" in the bottom left. Then click settings, this opens the project settings page, then go to the "API Keys" tab and create a new API key.

> **Note:** If you previously created an API key from the legacy HUD platform, that key will **not** work. Create a new key by following the above steps.

```bash
hud set HUD_API_KEY=<your HUD key>
hud set ANTHROPIC_API_KEY=<your Anthropic key from liaison>
```

**Register the problem.** Open `src/hud_controller/problems/basic.py` and append:

```python
PROBLEM_REGISTRY.append(
    ProblemSpec(
        id="multiply_fp32_detailed",
        description="""Complete the implementation in sources/multiply_fp32.sv based on the specification in doc/spec.md.
The RTL code should be synthesizable. Make sure the filename remains sources/multiply_fp32.sv.
This environment uses Icarus Verilog for testing. Avoid using SystemVerilog Assertion (SVA) property/sequence syntax.
""",
        difficulty="hard",
        base="multiply_fp32_detailed_baseline",
        test="multiply_fp32_detailed_test",
        golden="multiply_fp32_detailed_golden",
        test_files=["tests/test_multiply_fp32.py"],
    )
)
```

The branch names (`base`, `test`, `golden`) must exactly match the branches in your fork. Since forking preserves all branches, these already exist.

**Point the Dockerfile at your fork.** Open `Dockerfile` and find the `REPO_URL` line (near line 103):

```dockerfile
ARG REPO_URL=https://github.com/hud-evals/example-verilog-codebase.git
```

Change it to:

```dockerfile
ARG REPO_URL=https://github.com/<your-github-username>/multiply_fp32.git
```

Since your fork is public, Docker can clone it without any authentication token.

### After each spec modification

**1. Update the branches.** From your fork (`multiply_fp32`), commit your spec change and push to both the baseline and test branches:

```bash
cd /path/to/multiply_fp32
git checkout multiply_fp32_detailed_baseline
# (make your spec edits to doc/spec.md)
git add doc/spec.md
git commit -m "modify spec"
git push origin multiply_fp32_detailed_baseline

git checkout multiply_fp32_detailed_test
git cherry-pick <commit-hash-from-above>
git push origin multiply_fp32_detailed_test
```

**2. Rebuild the Docker image.** Back in the framework repo, first increment the cache-buster in the `Dockerfile` so Docker pulls fresh code from your fork (find the `ENV random=random6` line near `REPO_URL` and change the number, e.g. `random7`, `random8`, etc.). Then build:

```bash
cd /path/to/verilog-coding-template
uv run utils/imagectl3.py verilog_ -b --ids multiply_fp32_detailed
```

**3. Validate** (confirm golden passes, baseline fails):

```bash
uv run utils/imagectl3.py verilog_ -v --ids multiply_fp32_detailed
```

**4. Generate tasks and run** (10 independent attempts):

```bash
uv run utils/imagectl3.py verilog_ -j --ids multiply_fp32_detailed
```

This writes `tasks.py` with one row per problem. Then run the local eval driver:

```bash
uv run python run_eval.py --ids multiply_fp32_detailed --agent claude \
  --model claude-sonnet-4-5 \
  --max-steps 100 \
  --group-size 10
```

`run_eval.py` starts a fresh Docker container per rollout, runs the agent, grades each attempt, and prints progress plus a job URL like `https://hud.ai/jobs/<job-id>`. Open that link to inspect individual rollouts.

Iterate until you achieve 40–70% pass rate.

---

## Section 6: Submission

Once you have achieved a pass rate between 40% and 70%, email your liaison with:

1. **Your modified `doc/spec.md`**

2. **Your HUD results link** showing the pass rate

3. **A written analysis** containing:

   **A. Root Cause Analysis:**
   Pick 1 or more failing runs from the [original 20-run evaluation](https://hud.ai/jobs/09836173-f7f4-4cd5-9a9f-8247caea5d20). For each, describe:
   - What the agent did wrong (the specific bug in its implementation)
   - Why it went wrong (what caused the agent to make that mistake)

   **B. Faulty Assumptions / Missed Insights:**
   What flawed reasoning, misconceptions, or missing domain knowledge led the agent to produce incorrect code? (e.g., did it misunderstand a concept in the spec? Did it use a Verilog pattern incorrectly? Did it skip a step?)

   **C. Prompt Modifications:**
   What did you change in the spec and why? Connect your changes to your analysis — explain how each modification addresses a specific failure mode you observed in the agent's behavior.

---

This takehome should take approximately **5–8 hours total**. The most important part is the analysis — we want to see that you can diagnose agent failures and reason about how to guide an AI model to produce correct hardware implementations.
