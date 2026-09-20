# AgentsVille-Trip-Planner

A Multi-Agent Travel Assistant System

## Skills Learned

- Iterative Prompt Development
- Implementing Role-Based Prompting
- Feedback Loop Design
- Persona-Based Prompt Design
- Implementing Prompt Chains
- Systematic Prompt Refinement
- CoT and ReAct
- Agentic Reasoning Frameworks
- Prompt Chain Design
- Implementing Feedback Loops

## Skills Tested

- General Prompt Design
- Agent Reasoning and Tool Use
- Structured Output Validation


## QA

### Questions

1) Why the output is not always the same, In both agents? Sometimes, in the first agent it generates the itinerary passing all evals, other times not. In the second agent, sometimes it follows the right path, other times fail.

2) I remember in some lesson, we left the agent to detect which tools they need, so when shall I leave it to the agent, and when I need to force it to use a specific tool?

3) In COT, shall I specify a flow in the prompt, or to only add "think step-by-step" and leave the agent discover the steps needed? We could have added in the prompt, to first get weather data based on dates, then get activities based on dates, then filter activities based on interests, ....... etc... So, shall we need to specify the detailed steps or to leave the agent discover?

4) Can you also please provide the model answer for the 2 prompts, after my answer passes, to learn from?

### Answers

- [1] Two different causes here, and they need different fixes. The wording changes between runs because decoding is stochastic: at each step the model samples from a probability distribution over the next token, and two runs take different draws. Set temperature=0 in the helper you use to call the model and most of that drift disappears (near-deterministic, not bit-identical, since floating-point non-associativity on the serving side can still occasionally flip a token). But sampling doesn't explain why the evals pass sometimes and fail other times. That happens when a requirement lives in the evaluator but not in your prompt, so the model has to guess and gets it right maybe 70% of the time. The fix is mechanical: each time an eval fails, read its failure message and add the exact constraint it checks to ITINERARY_AGENT_SYSTEM_PROMPT as an explicit, testable rule ("total_cost must equal the arithmetic sum of activity prices, do not estimate", "use only activities present in the provided list, copy name and price verbatim"). Run it five times rather than once, that tells you which evals are ambiguous (fail 2/5) versus flat wrong (fail 5/5). The second agent has an extra failure surface, the action space itself, so its variance shows up as wrong tool, malformed ACTION JSON, or looping on evals instead of exiting, which you fix structurally with one worked example turn and a mechanical exit condition rather than by touching temperature.

- [2] and [3] are the same underlying rule, so I'll answer them together. Force the tool call when the step is mandatory on every path regardless of what the agent observes, and let the agent choose when the right step depends on what it just saw. So run_evals_tool before final_answer_tool gets forced, and final_answer_tool as the only exit gets forced, because there is no scenario where submitting an unevaluated plan is correct; but which day to fetch, or whether the calculator is even needed, depends on which evals came back failing, so you let the agent sequence those. The key thing to internalize is that a prompt "MUST" is a strong prior, not a guarantee, and the only truly reliable enforcement is a gate in your orchestration code (if tool_name == "final_answer_tool" and not last_eval_passed: reject and continue). If a step must always happen, put it in the code and stop asking the model. That same distinction settles the chain-of-thought question: specify the explicit flow for the itinerary agent, because the procedure encodes your requirements (weather-per-day, exact cost math, interest matching) that the model can't infer from "plan a trip", and order the steps by dependency so weather is resolved before it filters activity selection. But do not hand the revision agent a fixed step list, because the correct sequence of tool calls isn't known until runtime; for that agent you specify the THINK-ACT-OBSERVE cycle, the invariants, and the exit condition, then let it sequence the calls itself. The general rule to carry forward: fixed known-in-advance procedure means prescribe the steps, procedure that depends on runtime observations means prescribe the cycle and constraints and let the agent order it.

- [4] I can't hand over a reference solution prompt, and there's a real reason beyond policy: there isn't a single correct one. Both prompts are graded on whether they contain the required elements, not on matching specific wording, so a "model answer" would just be one arbitrary point in a large valid space and copying it would skip the part of the exercise that actually transfers. What I can give you is the checklist your prompt is graded against. For ITINERARY_AGENT_SYSTEM_PROMPT: a role, day-by-day chain-of-thought guidance that explicitly involves weather and activities and interests, the output format (ideally injected dynamically via TravelPlan.model_json_schema() so it can't drift), and references to the VacationInfo inputs plus the budget and date-range constraints. For ITINERARY_REVISION_AGENT_SYSTEM_PROMPT: the role and iterative task, the named THINK-ACT-OBSERVE cycle, a description of every tool and its arguments, the exact ACTION JSON format with an example, and the evaluation and exit rules. The genuinely useful exercise once you pass is to diff your own failing version against your passing one, because that delta is the list of constraints that were actually doing the work, and then read the reviewer feedback, which is written against your actual prompt rather than a generic solution.