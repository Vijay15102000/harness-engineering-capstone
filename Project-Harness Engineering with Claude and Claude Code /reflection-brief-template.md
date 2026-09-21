# Reflection Brief — Harness Engineering Capstone

*Name:* Vijay Kumar K S
*Date:* 2026-09-21

Replace each `→` with your answer. **Every answer cites at least one artifact from your own runs** — a run ID, file path, token count, claim outcome, or test count. Uncited answers do not pass. 3–6 sentences each unless noted. Paste short artifact snippets where they help.

**Environment**

- Model(s):claude-haiku-4-5-20251001 (System 1 traces) ; course claude defaults for Systems 2-4.
- OS / Python: Linux Project Workspace, Python 3.13
- Approx. API spend: 1 summary.md run 20260916_100855; System 2-4 used recorder/local fixtures where noted

---

## Part 1 — Per-system

### System 1 — Agentic loop

1. **Loop control.** Quote the `stop_reason` sequence from one trace. Name the file and function that decides continue-vs-stop, and how.
   → continue-vs-stop is decided in claims_intake/loop.py, function run_loop. After client.message.create, it branches on response.stop_reason:tool_use executed tools and continues;end_turn stops. Terminal trool route_to_adjauster/escalate_to_human set terminal_called and exit. evidence/system1-agentic-loop/traces/claim_05_auto_collission.jsonl shows tools_use then end_turn. summary.md(run 20260916_100855) marks that claim routed. six other rows in the same rummary.md are incomplete because the model ended after facts without a terminal tool

2. **Anti-pattern.** Name one anti-pattern `test_antipatterns.py` checks for. What would break in your run if the loop used it?
   → tests/test_antipatterns.py(29 passed in evidence/system1-agentic-loop/pytest.txt) flage ignoring stop_reason treating the first tool result as done, or treating end_turn as "keep going". If run_loop stopped after the first tool, claim_05_auto_collision.jsonl would never reach the later tool_use turns or route_to_adjuster. if it ignored end_turn, the loop would spin past a finished model run and turn blow wll clock/token caps. 

3. **Tool design.** Pick two tools with overlapping inputs. How do the descriptions prevent misrouting? What did a structured tool error let the agent do that a generic string would not?
   → classify_claim and assess_severity both consume structured claim facts, but the schemas split "what kind of loss" vs"how bad".route_to_adjuster vs escalate_to_human both end loop; description seperate confident routing from low confidence/injury/humanreviwe. A structured tool error(types fixed in toops.py) lets the next turn repair one argument insted of dumping a blob. That is why claim_01 could continue tool_use turns instead of dying on a string exception(outcome+error never appears in summary.md)

4. **Your numbers.** Quote the turn count and cost for one claim. How does it differ from the README sample, and why?
   → claim_05_auto_collission in summary.md(20260916_100855) completed routed after multiple  tool_use turns and ~14826 input tokens.The same file lists six incomplete claims that stopped after `2 TURNS/~6K TOKENS WITH NO classify_claim. Fulltime_estimated cost ~$0.40. That matches the project guide sample's mix of incomplete rows more that clean 8/8 terminal table.

### System 2 — Context strategy

5. **The reduction.** From `budget.json`: baseline tokens, assembled tokens, reduction %. Which section dominates the assembled context, and why keep it verbatim?
   → evidence/system2_context_strategy/budget.json and the task 3 run :baseline 38708 and assembled 16779 56.5% reduction . The active selection dominates assembled tockens ~16.8k tockens beuause it is the live issue. it stays byte-exact so Q4/Q5 - style question still hit the raw card/status strings.

6. **Summarize vs preserve.** State the rule for what gets summarized vs kept byte-exact, citing your per-section token numbers.
   → Resolved segments are summarized the active segment is preserved byte-exact. Run numbers; resolved/refund 12334-374,resolved/subscription 11475-430, active kept 16779 , case_facts ~204. eval.jsonl still answers Q1-Q6 on the assembled context. the compresser tests in pytest.txt/S2 long require that the active segment is not compressed. 

7. **Facts block.** Compare `eval.jsonl` to `eval_control.jsonl`. Which question regressed, and what does that prove?
   → main eval : 6/6 pass. Control(case-facts stripped):Q1 unexpected pass, *Q6 expected fail*.That is in the task 3 control block and eval_control_jsonl. Q6 needs the structured payment-method status from the fcats block. the regression proves case_facts carry answerability that compressed pros doesnot

### System 3 — Claude Code config

8. **Path-scoped rules.** Quote the glob frontmatter from one rule file. Why is it better than a directory-level CLAUDE.md for cross-cutting conventions?
   → Root evidence/system3-claude-code-config/CLAUD.md uses @import of .claude/standards/files keeping the root short and pushing details oin to imported standards 8s wgar test_us01_claude_md_hierarchy checks. Validator printed ok. 

9. **Forked skill.** Quote the `context: fork` and `allowed-tools` lines. What does running forked + read-only buy you? What breaks without it?
   → .claude/rules.api.md binds API-only globs , .claude/rules/react.md binds component/page globs .claude/rules/tests.md binds test globs. claude_dir_structure.txt and the copied rule files record those rules. Scoped rules stops a react rule from rewriting an APi handler.

10. **Scope.** From the validator output: project-level vs user-level scope. Give one example of each from this config.
    → deploy-check/SKILL.md sets allowed tools to read-only checks. test_us04-deploy_check_skill passed. That limits blast radious if the skill is invoked on a fork.

### System 4 — Orchestration

11. **Push work down.** Defects the SQL query returned vs warm-tier total. Name the indexed query. Why does the model never see the full history?
    → Hot: hot_state.json *643 bytes* 
    warm: SQLite warm.sqlite plus defects_since indexed query so a shift doesnot load the full defect table 
    cold: monthly summary. shift-run_output.txt/scratchpad show Shift C recorder-response completed with ) new defects in tat fixture.

12. **Crash recovery.** The resume-vs-fresh decision and its staleness threshold (`recovery.py`). Why is a fresh start with an injected summary sometimes more reliable than resuming?
    → shift_scratchpad.jsonl is append+fsynch, mid-write reads only complete lines (test_us034_crash_recovery). Fork states copies state into an independentscratchpad without mutating the base(test_us04_fork_scratchpad).resumes uses the last complete scratchpad line plue hot state.

13. **Small state.** Byte size of your `hot_state.json`. Why does the budget matter for a system run once per shift, indefinitely?
    → 643 bytes under the size budget the test enforce. A shift  job that runs forever cannot let hit state grow with every defect or the next process will exceed context and disk for "just the live pointer.warm/cold hold history", hot stays a pointer

---

## Part 2 — Synthesis

*Graded on connecting two or more systems. Cite a named file/artifact from each.*

14. **Three layers.** Point to a file/artifact for each layer and justify.
    → Model: claim_05_auto_collision.jsonl the llm chooses stop_reason.
    → Harness:claims_intake/loop.py runloop plus pytest.txt - deterministic continue/stop.
    → Orchestration:

15. **Deterministic vs prompt.** Cite one behavior guaranteed in code (terminal tool, read-only allowlist, atomic write, byte budget) and one guided by prompt. When is each right?
    → Deterministic : system 1 branches on stop_reason in run_loop system 3 read-only alowed-tools on the forked-skill; System 4 hot state stayed 643 bytes Prompt-guided: which intake tool to call, and what system 2 summarizes vs keeps(budget.json).code is right for safety and budgets; prompt are right when the choice depends on claim language

16. **Context, two faces.** Compare context management in System 2 (intra-session) and System 4 (cross-session) with cited numbers from both. Same principle, different mechanism — how?
    → System 2 is a intra-session : 38708-16779 tokens active kept vervatim. System 4 is cross-session:643-byte hot pointer plue warm defects_since instead of shipping the whole history every shift. Same idea dont reload everything different mechanism.

17. **Reliability you can't see in one run.** Name one behavior a test guarantees that a single successful run would not reveal. Why does it matter before shipping?
    → crash recovery tests guarantee that at torn last line in shift_scratchpad.jsonl does not load as state .one happy shift c run would  not show that. it matters before shipping because a killed shift is the common case.

18. **Blast radius.** Pick one system. What's the blast radius if it misbehaves, and what's the kill switch? Ground it in that system's tools, enforcement points, and state.
    → System 1 : a bad tool call can mis-route c claim;kill switch is escalate_to_human, max_wall_clock_s, and stopping on end_turn/terminal_called_ in run_loop. state is per-claim JSONL, so one fixture cannot wipe another.

---

## Part 3 — Honest assessment

19. **What broke.** One thing that failed first try in your environment, and how you fixed it. (If nothing, what you checked to be sure.)
    → mis spelled the git commands whuke pushing the project to the repository "git add"  Fix: git commit -all to git add .

20. **What you'd change.** One architectural decision you'd make differently, grounded in what you observed.
    → set default low confidence path to human instead of end_turn. I would also iterate --fixture claim_01_kitchen_fire before --all to save budget when the key works. 
