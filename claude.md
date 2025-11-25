# Under the hood of Claude Code

Over the last few months, it feels like everyone has been talking about Claude Code - Anthropic's take on agentic (and largely asynchronous) code editing[^1]. Claude Code is installed via a Node package that runs locally, so every bash command or file you edit happens right on your laptop. It calls out to the regular Sonnet or Opus APIs behind the scenes. This local executable gives us a rare chance to introspect exactly what it's doing once you give it a goal.

There's a lot to be learned by looking at LLM prompts from other products. The bigger companies can get more empirical about testing prompts than you can when you're first starting off. It's like an A/B test for model behavior instead of for UI design. They can tweak a prompt, measure the acceptance/rejection rate of all their users, and further tweak it from there.

Models have gotten pretty good at refusing to reveal their system prompts. Regardless, there have been routine "leaks" of these prompts that give us the ability to see what makes a successful prompt specification for a cloud hosted LLM.[^2] Claude Code gives us even more context here because all the prompts are just specified in the Javascript package. We can see exactly how they're injected.

What follows is a dive into the architecture as it exists today, based on my read of the code. See my process section below for some caveats.

## The Architecture

There are three main parts of the architecture.

**Agent**: This controls the main loop of the agent control flow. It's a multi-turn conversation where your initial prompt leads to a cascade of tool calls & tool responses. This happens on loop until the model runs out of things to do or you add a new message into the console box that's injected into the conversation flow.

When tool calls happen, they happen in a single `async call()` function call. The entire scope of these functions is a single input-output execution - just like you have in a normal function within a programming language.[^3]

**Sub-Agent**: Sometimes there are jobs that are too complex for a single tool argument. In these cases Claude can spin off sub-agents that can run multiple loops. These can draft, revise, perform research, etc. They always deliver their results back to the main conversation flow when finished. It's basically just a way of creating isolated conversation scopes that only should perform one task and do it well.

There are token length savings to this approach as well since you don't have to keep all the tool calls permanently in the turn-by-turn flow of the main conversation. You can just let the sub-agent's output represent the whole process that it took to get there.

**Tool Calls**: At the lowest level of the stack, you have the prompts and implementation for the tool calls themselves. Each tool has its own brief description accompanied with a much longer prompt that instructs the model how to call it.

## Prompt Explorer

I built out a small webapp where you can see the prompts that Claude uses to inform its tool calling. You can also go to the full tool if you'd rather view it in fullscreen.

Since a lot of these prompts are populated dynamically at runtime with your project context, I've tried my best to do a manual analysis of the source code for these variables and specify what they'll actually become when you run them. Some of these prompts will contextually hide or show elements of the prompt depending on your settings. I've indicated where these variables switch behavior with the `[start conditional]` section headers.

There's a ton of detail in some of these prompts. I find that level of detail roughly aligns with how flexible the tool call is. The `bash` tool call has a ton of specification for what functions are typically safe to call without side-effects, help specify some reasonable timeouts, define sandboxing behavior, and similar. Which makes sense: bash can do anything so best to use it intentionally. `glob` for instance is a much simpler request so the prompt can be proportionally shorter.

The agent also has access to some additional tools over what is displayed here - but they only include brief prompts or none at all. Presumably they're obvious enough (like directory construction) that they don't need the additional context of a prompt to run successfully.

## Conditioning behavior

Almost every complex prompt specifies behavior _to do_ and behavior _not to do_. These prompts both explain behavior at a high level and give some examples. Some of them almost feel like they're encoding an RCA worth of content in the guidance.

Despite our best efforts with RLHF, there's nothing to say that the model will actually follow these constraints. But that's not really in scope of the prompts themselves. The main pressure of improving this alignment is in the training and post-training stages. Instead these prompts throw down the gauntlet for how we want the model to behave - suspecting that the model will become better able to follow them as more generations of architectures evolve.[^4]

### Constraints ("NEVER", "MUST") for critical safety issues

Some constraints are non-negotiable because they'll completely break the user experience:

> "NEVER use git commands with the -i flag (like git rebase -i or git add -i) since they require interactive input which is not supported." - _Bash tool_

This is the kind of constraint that probably came from user reports. Even a simple `git rebase -i` would hang forever waiting for user input that will never come. Running all requests with a subprocess timeout can mitigate this stalling, but it's better not to need to learn that lesson every time if the initial exec can otherwise run a more targeted command.

### Deterministic constraints

Some constraints enforce good engineering practices by creating artificial dependencies:

> "You must use your "Read" tool at least once in the conversation before editing. This tool will error if you attempt an edit without reading the file." - _Edit tool_

This combines deterministic validation with dynamic function calls. It helps prevent the classic LLM mistake of confidently editing code it hasn't actually seen. By making the edit tool throw an error, they turned a best practice into a hard requirement.

### Edge cases that typically break systems

Some prompts target very specific but critical details:

> "When editing text from Read tool output, ensure you preserve the exact indentation (tabs/spaces) as it appears AFTER the line number prefix." - _Edit tool_

This helps prevents Python and YAML files from breaking due to indentation changes. Since edits typically happen in a targeted way (per line or for a group of lines), encouraging some of this contextual awareness is probably required to get outputs that really execute.

> "Always quote file paths that contain spaces with double quotes (e.g., cd "path with spaces/file.txt")" - _Bash tool_

Same with this one. I imagine the model isn't too great at properly creating the right escape patterns to ignore common filename edge cases. Better to just quote the whole string.

### Recovery strategies when things go wrong

The prompts even encode recovery strategies for when automated processes interfere:

> "If the commit fails due to pre-commit hook changes, retry the commit ONCE to include these automated changes." - _Bash tool_

This handles the common case where pre-commit hooks automatically format code or fix linting issues. Without this guidance, Claude might get stuck in a loop or give up entirely when the commit fails the first time.

> "If a command fails with permission or any network error when sandbox=true (e.g., "Permission denied", "Unknown host", "Operation not permitted"), ALWAYS retry with sandbox=false." - _Bash tool_

Similar to the sandbox. It's a "try this before anything else".

## Task Decomposition

If you've used Claude Code, you've probably seen the task list that it creates up front. It analyzes the task definition that you've given it, does a bit of additional research, then creates a todo list that it will work against. There's nothing stopping it from adding to this tool list over time - it's just another function call that the model can access. But a lot of the guidance for the model is about working off of this task list.

### Task identification and breakdown

The prompts teach Claude to recognize when something is genuinely complex versus when it's just a simple request:

> "Complex multi-step tasks - When a task requires 3 or more distinct steps or actions" - _TodoWrite tool_

Rather than creating a todo list for every single request ("1. Read the user's question 2. Think about it 3. Respond"), it only kicks in when there's actual complexity to manage.

### Progress tracking with specific state management

The system enforces focus through explicit state management:

> "Mark tasks as in_progress BEFORE beginning work. Ideally you should only have one todo as in_progress at a time" - _TodoWrite tool_

For very complex initial queries, LLMs are sometimes not sure where to start or try to solve too many problems at the same time. This ends up leaving the project in a half-completed state, where a ton of unit tests might break and getting it back into a working state is even harder than the original request. In all software engineering, acute changes are better than big breaking ones.

This prompt is encouraging the model to implement a single-threaded execution model on top of what could otherwise be a chaotic multi-tasking system. The model should commit to finishing what it starts before moving on to the next thing.

### Verification and completion criteria defined

The prompts are strict about what counts as "done":

> "ONLY mark a task as completed when you have FULLY accomplished it" - _TodoWrite tool_

This fights against agent tendency for a model to be overly optimistic about completion. And especially for the model to declare success after editing the file but without actually validating changes are working. Without this constraint, models might mark tasks as done when they're 90% finished, leaving users with broken implementations. The prompt forces the model to actually validate its work before claiming success.

> "Run tests and build process, addressing any failures or errors that occur" - _TodoWrite_

Examples within this prompt also often cite the validation criteria that the user is looking for. If you've requested the model to loop until your unit tests are complete, the examples help make sure that this will actually be factored into the todo list for the model. From there, if the model is obeying the todo list, it helps ensure that changes are validated to work before the agent finally exits.

## Cutting down on excessive code

Claude 3.7 was widely thought to be overly proactive based on either its agentic finetuning or its human feedback. You'd ask for one code file to be changed and it would change ten files and give you a new Readme.

My sense is 4.0 is better out of the gate because of how it's trained. But this section of the initial system prompt probably doesn't hurt:

> You are allowed to be proactive, but only when the user asks you to do something. You should strive to strike a balance between:
>
> - Doing the right thing when asked, including taking actions and follow-up actions
> - Not surprising the user with actions you take without asking
>
> For example, if the user asks you how to approach something, you should do your best to answer their question first, and not immediately jump into taking actions.
>
> Do not add additional code explanation summary unless requested by the user. After working on a file, just stop, rather than providing an explanation of what you did.

## All the different thinks

Looks like Simon Willison found this one as well. The CLI uses a regex match to figure out the intensity of thinking. They're trying to make this as user friendly as possible (for people that don't know the magic incantation of the ultrathink keyword).

The _best_ approach here would be a classifier calibrated both on what the user is asking and how difficult the initial question seems to be. But having a simplified heuristic probably works well enough and saves on compute[^5]. But that's not feasible unless you have a local ML accelerator or unless they develop some lightweight cloud model for this purpose.[^6] We can see `o3-pro` doing some version of this, where it can choose how long it spends researching. Sometimes that's one minute and sometimes that's 25. Without a doubt the next generation or architectures are going to make more of these choices themselves versus having a fixed reasoning budget.

```javascript
Bk6 = {
    english: {
        HIGHEST: [{
            pattern: "think harder",
            needsWordBoundary: !0
        }, {
            pattern: "think intensely",
            needsWordBoundary: !0
        }, {
            pattern: "think longer",
            needsWordBoundary: !0
        }, {
            pattern: "think really hard",
            needsWordBoundary: !0
        }, {
            pattern: "think super hard",
            needsWordBoundary: !0
        }, {
            pattern: "think very hard",
            needsWordBoundary: !0
        }, {
            pattern: "ultrathink",
            needsWordBoundary: !0
        }],
        MIDDLE: [{
            pattern: "think about it",
            needsWordBoundary: !0
        }, {
            pattern: "think a lot",
            needsWordBoundary: !0
        }, {
            pattern: "think deeply",
            needsWordBoundary: !0
        }, {
            pattern: "think hard",
            needsWordBoundary: !0
        }, {
            pattern: "think more",
            needsWordBoundary: !0
        }, {
            pattern: "megathink",
            needsWordBoundary: !0
        }],
        BASIC: [{
            pattern: "think",
            needsWordBoundary: !0
        }],
        NONE: []
    },
}
```

## My analysis process

Despite having a repo, the original source for Claude Code isn't released. Instead they publish their js package to npm in a minimized format (`cli.js`).

Since the package code is minimized, it removes all of the readable variable names and some of the clearness of the control flow. But it's not too hard to introspect these minimized files especially when you know you're searching for prompt strings that are specified inside. I started by finding the obvious strings, then tracing their wrapper object format and function calls to see where they're being formatted.

String variables that were more than 5 lines were especially valuable. These would usually be templated strings using the Javascript backtick format, which allows for function calls to other composable prompts. These in turn revealed embedded names of tools which are declared as global variables:

```javascript
var EC = "Bash";
```

From there you can search around until you find the consolidated payload that defines the tool names jointly with metadata about them:

```javascript
var _9 = {
    name: EC,
    async description({
        description: A
    }) {
        return A || "Run shell command"
    },
    async prompt() {
        return oo0()
    },
    isConcurrencySafe(A) {
        return this.isReadOnly(A)
    },
    isReadOnly(A) {
        let {
            command: B
        } = A;
        return ("sandbox" in A ? !!A.sandbox : !1) || Ik(B).every((D) => {
            for (let I of Jw6)
                if (I.test(D)) return !0;
            return !1
        })
    },
    inputSchema: zF1() ? Cw6 : sM2,
    userFacingName(A) {
        if (!A) return "Bash";
        return ("sandbox" in A ? !!A.sandbox : !1) ? "SandboxedBash" : "Bash"
    },
    isEnabled() {
        return !0
    },
    async checkPermissions(A, B) {
        if ("sandbox" in A ? !!A.sandbox : !1) return {
            behavior: "allow",
            updatedInput: A
        };
        return gAA(A, B)
    },
    async validateInput(A) {
        let B = fAA(A, dA(), U9(), YX());
        if (B.behavior !== "allow") return {
            result: !1,
            message: B.message,
            errorCode: 1
        };
        return {
            result: !0
        }
    },
    renderToolUseMessage(A, {
        verbose: B
    }) {
        ...
    },
    renderToolUseRejectedMessage() {
        return RD.createElement(Y6, null)
    },
    renderToolUseProgressMessage() {
        return RD.createElement($0, {
            height: 1
        }, RD.createElement(P, {
            color: "secondaryText"
        }, "Running…"))
    },
    renderToolUseQueuedMessage() {
        return RD.createElement($0, {
            height: 1
        }, RD.createElement(P, {
            color: "secondaryText"
        }, "Waiting…"))
    },
    renderToolResultMessage(A, B, {
        verbose: Q
    }) {
        return RD.createElement(Fc, {
            content: A,
            verbose: Q
        })
    },
    mapToolResultToToolResultBlockParam({
        interrupted: A,
        stdout: B,
        stderr: Q,
        isImage: D
    }, I) {
        ...
    },
    async *call(A, {
        abortController: B,
        getToolPermissionContext: Q,
        readFileState: D,
        options: {
            isNonInteractiveSession: I
        },
        setToolJSX: G
    }) {
        ...
    }
}
```

There's also a longtail of other commands that don't have fully baked out prompts. These cover commands like creating a new directory or removing files.

```javascript
var kk6 = O0(() => [Wx2, Qf2, Df2, Zf2, Ff2, Vv2, jw1, wg2, wv2, Nv2, qv2, Ab2, Ib2, zv2, Qb2, Sg2, Gb2, Wb2, Cb2, jb2, tx2, $w1, Qw, _g2, fb2, ib2, Ug2, ...!Bg() ? [Tv2, bv2()] : [], ...process.env.ENABLE_BACKGROUND_TASKS ? [xb2] : [], ...[], ...[]]),
```

Some other random architectural notes:

- They use extensive templating of prompts that reference other tools by a global variable, which is a nice touch to keep things in sync.
- Seems to use `vercel/ai` for some API communication, based on this error message. It also provides the ability to route through Bedrock/Vertex or a custom model with `$ANTHROPIC_MODEL`.

## What we can learn

All the large frontier labs are betting big on agentic code. In addition to the initial training process, these prompt specifications are also what is currently required for the models to work well across thousands of different codebases and environments.[^7]

There's a tension in AI tool design: giving the model enough freedom to be genuinely helpful while constraining it enough to be reliably safe. The prompts here try to walk that tightrope by being extremely specific about failure modes while staying flexible about success paths.

To me they also show that the difference between a proof-of-concept and a production system isn't just the infrastructure. It's the accumulated wisdom of what actually works when real users start doing real work. It drives home the importance of having community feedback when constructing these practical rules. You can only code into prompts what you can actually observe.

The next time you're writing prompts for your own agent architecture, remember that every constraint in Claude Code probably came from someone's 3am debugging session or frustrated support issue. That's the kind of user empathy that separates good AI tools from great ones.

---
