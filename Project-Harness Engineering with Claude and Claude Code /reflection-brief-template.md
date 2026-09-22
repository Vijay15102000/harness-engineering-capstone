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
   → the loop in claims_intake/loop.py continues when the model returns tool_use and stops when it returns end_turn. Terminal tools such as route_to_adjuster and escalate_to_human mark the claims as terminal and exit the loop. In the latest run, 4 of 8 claims were routed and 4 were incomplete, showing hopw the loop handles both terminal and non-terminal outcomes

2. **Anti-pattern.** Name one anti-pattern `test_antipatterns.py` checks for. What would break in your run if the loop used it?
   → tests/test_antipatterns.py(29 passed in evidence/system1-agentic-loop/pytest.txt) flage ignoring stop_reason treating the first tool result as done, or treating end_turn as "keep going". If run_loop stopped after the first tool, claim_05_auto_collision.jsonl would never reach the later tool_use turns or route_to_adjuster. if it ignored end_turn, the loop would spin past a finished model run and turn blow wll clock/token caps. 

3. **Tool design.** Pick two tools with overlapping inputs. How do the descriptions prevent misrouting? What did a structured tool error let the agent do that a generic string would not?
   → classify_claim and assess_severity both consume structured claim facts, but the schemas split "what kind of loss" vs"how bad".route_to_adjuster vs escalate_to_human both end loop; description seperate confident routing from low confidence/injury/humanreviwe. A structured tool error(types fixed in toops.py) lets the next turn repair one argument insted of dumping a blob. That is why claim_01 could continue tool_use turns instead of dying on a string exception(outcome+error never appears in summary.md)

4. **Your numbers.** Quote the turn count and cost for one claim. How does it differ from the README sample, and why?
   → claim_05_autocollision used 14826 input tokens, poroduced 908 output tokens , took 4 turns and cost an esti,mate pof $0.0194. Four claims were incomplete : claim_01, claim_02, claim_06, claim_08; only claim_02 and claim_08 fit the roughly 2-turn,~6k inpuit token pattern

### System 2 — Context strategy

5. **The reduction.** From `budget.json`: baseline tokens, assembled tokens, reduction %. Which section dominates the assembled context, and why keep it verbatim?
   → evidence/system2_context_strategy/budget.json and the task 3 run :baseline 38708 and assembled 16779 56.5% reduction . The active selection dominates assembled tockens ~16.8k tockens beuause it is the live issue. it stays byte-exact so Q4/Q5 - style question still hit the raw card/status strings.

6. **Summarize vs preserve.** State the rule for what gets summarized vs kept byte-exact, citing your per-section token numbers.
   → Resolved segments are summarized the active segment is preserved byte-exact. Run numbers; resolved/refund 12334-374,resolved/subscription 11475-430, active kept 16779 , case_facts ~204. eval.jsonl still answers Q1-Q6 on the assembled context. the compresser tests in pytest.txt/S2 long require that the active segment is not compressed. 

7. **Facts block.** Compare `eval.jsonl` to `eval_control.jsonl`. Which question regressed, and what does that prove?
   → main eval : 6/6 pass. Control(case-facts stripped):Q1 unexpected pass, *Q6 expected fail*.That is in the task 3 control block and eval_control_jsonl. Q6 needs the structured payment-method status from the fcats block. the regression proves case_facts carry answerability that compressed pros doesnot

### System 3 — Claude Code config

8. **Path-scoped rules.** Quote the glob frontmatter from one rule file. Why is it better than a directory-level CLAUDE.md for cross-cutting conventions?
   → The rule applies to "**/*.test.tsx " and "**/*.test.ts" foles anywhere in the repo. A glob is better than directory-level CLAUDE.md becquse it follows the file type so the convention applies even when tests are scattered across different directories . 

9. **Forked skill.** Quote the `context: fork` and `allowed-tools` lines. What does running forked + read-only buy you? What breaks without it?
   → the Skill uses context:fork and a read-only allowed-tools list. Forking keeps intermediate work isolated from the oarent session, while the read-only tols limit the skill's ability to modify the repository.without them, the skill cloud pollute the parfent context or make unintended changes.

10. **Scope.** From the validator output: project-level vs user-level scope. Give one example of each from this config.
    → project level configuration includes .c.aude.md, .claude/standards/ and .claude/rules/ which are vetrsion controlled and shared with the team.
    user level configuration includes ~/.claude/CLAUDE.md,~/.claude/commands/ and ~/.claude/skills/ which stay on the developer's machine and are not version controlled.

### System 4 — Orchestration

11. **Push work down.** Defects the SQL query returned vs warm-tier total. Name the indexed query. Why does the model never see the full history?
    → The warm tier contains 40 defects, while the indexed defects_since query returns only the required time slice: 4 rows with --since 2026-04-22 and 17 rows with --since 2026-04-01. The model recieves only these filtered results rather than the full 40 row history, keeping the historical retrieval targeted and reducing context usage.

12. **Crash recovery.** The resume-vs-fresh decision and its staleness threshold (`recovery.py`). Why is a fresh start with an injected summary sometimes more reliable than resuming?
    → The recovery logic resumes state only when it is 30 minutes older or less; state older than 30 minute is treated as fresh. A fresh run  with an injected summary avoids carrying potentially stale state, while a valid resource preserves prior partial findings and adds new defects.Ths decision is validsted by the crash-recovery truth tables.

13. **Small state.** Byte size of your `hot_state.json`. Why does the budget matter for a system run once per shift, indefinitely?
    → hot_state.json is 648 bytes under the size budget the test enforce. A shift  job that runs forever cannot let hit state grow with every defect or the next process will exceed context and disk for "just the live pointer.warm/cold hold history", hot stays a pointer

---

## Part 2 — Synthesis

*Graded on connecting two or more systems. Cite a named file/artifact from each.*

14. **Three layers.** Point to a file/artifact for each layer and justify.
    → Model: System1's claim_05_auto_collision.jsonl trace shows the model using tools and eventually routing the clkaim.
    → Harness:claims_intake/loop.py controls the stop_reason loop and terminal-tool handling.
    → Orchestration: The shift pipeline assembles state,decides wheater to resume or startfresh, and manages hot state, cold history and isolated scratchpads.

15. **Deterministic vs prompt.** Cite one behavior guaranteed in code (terminal tool, read-only allowlist, atomic write, byte budget) and one guided by prompt. When is each right?
    → Deterministic : system 1 branches on stop_reason in run_loop system 3 read-only alowed-tools on the forked-skill; System 4 hot state stayed 648 bytes Prompt-guided: which intake tool to call, and what system 2 summarizes vs keeps(budget.json).code is right for safety and budgets; prompt are right when the choice depends on claim language

16. **Context, two faces.** Compare context management in System 2 (intra-session) and System 4 (cross-session) with cited numbers from both. Same principle, different mechanism — how?
    → System 2 is a intra-session : 38708-16779 tokens active kept vervatim. System 4 is cross-session:648-byte hot pointer plue warm defects_since instead of shipping the whole history every shift. Same idea dont reload everything different mechanism.

17. **Reliability you can't see in one run.** Name one behavior a test guarantees that a single successful run would not reveal. Why does it matter before shipping?
    → crash recovery tests guarantee that at torn last line in shift_scratchpad.jsonl does not load as state .one happy shift c run would  not show that. it matters before shipping because a killed shift is the common case.

18. **Blast radius.** Pick one system. What's the blast radius if it misbehaves, and what's the kill switch? Ground it in that system's tools, enforcement points, and state.
    → System 1 : a bad tool call can mis-route c claim;kill switch is escalate_to_human, max_wall_clock_s, and stopping on end_turn/terminal_called_ in run_loop. state is per-claim JSONL, so one fixture cannot wipe another.

---

## Part 3 — Honest assessment

19. **What broke.** One thing that failed first try in your environment, and how you fixed it. (If nothing, what you checked to be sure.)
    → across the five recorded runs , multiple claims repeatedly ended as incomplete rather than reaching a terminal routing/escaltion outcome.This indicates the loop can stop before calling a terminal tool, so would change the harness to route or escalate when the model stops without completing a terminal action. The five run artifact are under exercises/03-dynamic-decomposition/solution/runs/*/summary/md

20. **What you'd change.** One architectural decision you'd make differently, grounded in what you observed.
    → set default low confidence path to human instead of end_turn. I would also iterate --fixture claim_01_kitchen_fire before --all to save budget when the key works. 
