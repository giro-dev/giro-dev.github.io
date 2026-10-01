---
title: "Orchestrating LLM agents in Java: a tool-calling loop you can test"
date: 2026-08-31T00:00:00Z
description: "A hands-on tutorial on building an agent loop in Java 21 and Spring Boot: typed tools, step and token budgets, timeouts and cancellation, and deterministic tests with a fake model."
---

## Problem

Every "build an agent" tutorial stops at the interesting part:

```java
String answer = chatClient.prompt(question).call().content();
```

That is not an agent — it is one request. An agent is a **loop**: the model proposes a tool call, your code executes it, the result goes back into the conversation, and the model decides whether to call another tool or answer. The loop is where all the engineering lives, and it is where a demo turns into an incident:

1. **It does not terminate.** The model calls `searchRules` eleven times with the same arguments and the request dies on the gateway timeout, having burnt 300k tokens.
2. **It cannot be tested.** The only way to know whether the loop handles a malformed tool call is to run it against a paid, non-deterministic API.
3. **It has no blast radius.** A tool named `deleteCampaign` is exposed to a probabilistic caller with no confirmation, no authorization check and no audit trail.
4. **It is invisible.** When a user complains that "the assistant went weird", there is no record of which tools ran with which arguments.

This post builds that loop from scratch: a small, explicit, **testable** orchestrator with typed tools, hard budgets, cancellation and a fake model for tests. The examples come from patterns I used in [AI-game-master](https://github.com/giro-dev), where an agent narrates a tabletop session while calling tools that query rules and mutate campaign state — a domain where a hallucinated write is immediately visible to five players.

Baseline: Java 21 (LTS), Spring Boot 3.5.x, Spring AI 1.0.x. The design applies unchanged to a raw HTTP client against OpenAI or Gemini.

## Background: the loop, precisely

Every tool-calling provider implements the same state machine. Written out, without a framework:

```text
messages := [system, user]
repeat:
    response := model(messages, tools)          # 1 network call
    if response has no tool calls:
        return response.text                    # terminal state
    messages += response                        # the assistant's tool-call turn
    for each call in response.toolCalls:
        result := execute(call)                 # your code, your rules
        messages += toolMessage(call.id, result)
until budget exhausted
```

Four properties of that pseudocode matter more than the model you plug into it:

- **The conversation grows monotonically.** Each iteration appends the assistant turn *and* one message per tool result. Cost per iteration grows with the transcript, so iteration 8 is far more expensive than iteration 1 — which is why a step budget alone is not a cost budget.
- **`execute` is ordinary code.** It is a method call with validation, authorization and error handling. The model chooses *which* one; it does not get to choose *whether* your rules apply.
- **Tool results must be paired by id.** Every `toolCalls[i].id` needs exactly one tool message with the same id, or the provider rejects the next request.
- **The terminal state is "no tool calls".** Everything else — repeated calls, empty arguments, a tool that throws — is a case *you* decide how to handle.

Spring AI (like the OpenAI SDK) can run this loop for you. Convenient for a prototype, and the reason production agents are unobservable and unbounded: the default `ToolCallingManager` will happily iterate until the model stops. This tutorial keeps the loop in application code so that budgets, retries, redaction and audit are all ours.

## Step 1 — Dependencies and a bounded chat client

```xml
<dependency>
  <groupId>org.springframework.ai</groupId>
  <artifactId>spring-ai-starter-model-openai</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
  <groupId>io.github.resilience4j</groupId>
  <artifactId>resilience4j-spring-boot3</artifactId>
</dependency>
```

```properties
spring.ai.openai.api-key=${OPENAI_API_KEY}
spring.ai.openai.chat.options.model=gpt-4o-mini
spring.ai.openai.chat.options.temperature=0.2

# The loop drives tool execution, not the framework
spring.ai.openai.chat.options.internal-tool-execution-enabled=false

spring.threads.virtual.enabled=true
```

That third property is the important one. With internal tool execution disabled, a `call()` returns the assistant's *tool-call request* instead of silently executing it — the model becomes a pure function `(messages, tools) -> response` and the loop becomes ours.

Define the port your loop talks to, rather than depending on the client directly:

```java
public interface ChatModelPort {
    /** One round trip. No retries, no tool execution, no loop. */
    AgentResponse exchange(List<Message> messages, List<ToolSpec> tools);
}

public record AgentResponse(String text, List<ToolCall> toolCalls, TokenUsage usage) {
    boolean isTerminal() { return toolCalls.isEmpty(); }
}

public record ToolCall(String id, String name, String argumentsJson) { }
public record TokenUsage(int prompt, int completion) {
    int total() { return prompt + completion; }
}
```

One interface, three benefits: the loop is unit-testable with a fake, the provider is swappable (OpenAI ↔ Gemini), and retry/timeout policy lives in one adapter instead of being sprinkled through the orchestration.

## Step 2 — Typed tools with an explicit contract

A tool is not "a method the model can call". It is a **capability declaration** with four parts: a name, a JSON schema for its arguments, an executor, and a risk classification.

```java
public interface AgentTool<A> {
    String name();
    String description();          // the model reads this — it is prompt, not docs
    Class<A> argumentType();
    Risk risk();                   // READ, WRITE, DESTRUCTIVE
    String execute(A args, ToolContext ctx);
}

public enum Risk { READ, WRITE, DESTRUCTIVE }
```

A read-only tool, with arguments as a record so the schema is generated, not hand-written:

```java
@Component
class SearchRulesTool implements AgentTool<SearchRulesTool.Args> {

    record Args(
        @JsonPropertyDescription("Rules question in natural language") String query,
        @JsonPropertyDescription("Max passages, 1-5") Integer limit) { }

    private final RuleSearchService rules;

    @Override public String name()        { return "search_rules"; }
    @Override public String description() { return "Search the campaign rulebook for passages relevant to a question."; }
    @Override public Class<Args> argumentType() { return Args.class; }
    @Override public Risk risk()          { return Risk.READ; }

    @Override
    public String execute(Args args, ToolContext ctx) {
        int limit = args.limit() == null ? 3 : Math.clamp(args.limit(), 1, 5);
        var passages = rules.search(args.query(), limit);
        return passages.isEmpty()
                ? "No matching rule found."
                : passages.stream().map(p -> p.id() + ": " + p.text()).collect(joining("\n"));
    }
}
```

Three conventions that pay off immediately:

- **`description()` is part of the prompt.** Say what the tool is *for* and when not to use it. "Search the rulebook" gets called for arithmetic; "Search the campaign rulebook for passages relevant to a rules question; do not use for character state" does not.
- **Clamp, do not trust.** `limit=10000` is a valid JSON integer. Validate inside `execute` — the schema is a hint to the model, not a guarantee.
- **Return text the model can use.** A stack trace or a raw JSON blob wastes tokens and invites hallucination. Return a compact, labelled result — and prefix ids (`p-412:`) so the model can cite them and you can verify the citation.

For a `WRITE` tool, the executor is where the *real* authorization happens:

```java
@Override
public String execute(ApplyDamage.Args args, ToolContext ctx) {
    var character = characters.find(args.characterId())
            .orElseThrow(() -> new ToolInputException("Unknown character: " + args.characterId()));

    if (!ctx.actorCanEdit(character.campaignId())) {
        throw new ToolForbiddenException("Actor may not modify this campaign.");
    }
    if (args.amount() < 0 || args.amount() > 999) {
        throw new ToolInputException("amount must be between 0 and 999");
    }
    return characters.applyDamage(character.id(), args.amount(), ctx.idempotencyKey()).summary();
}
```

The `ToolContext` carries the *authenticated* actor, tenant and an idempotency key derived from the tool-call id — never from the model's arguments. Rule of thumb: **the model chooses the verb, your code owns the subject**. If a tool can be invoked with an id the caller has no right to touch, the tool is broken regardless of how good the prompt is.

## Step 3 — The loop, with budgets

Now the orchestrator. It is about sixty lines, and every one of them is a policy decision you want to be able to point at.

```java
@Service
public class AgentRunner {

    private final ChatModelPort model;
    private final ToolRegistry tools;
    private final AgentBudget defaults;   // maxSteps, maxTokens, wallClock

    public AgentResult run(AgentRequest request) {
        var budget = defaults.mergedWith(request.budget());
        var messages = new ArrayList<Message>();
        messages.add(new SystemMessage(request.systemPrompt()));
        messages.add(new UserMessage(request.userMessage()));

        var trace = new AgentTrace(request.conversationId());
        var deadline = Instant.now().plus(budget.wallClock());
        int tokens = 0;

        for (int step = 1; step <= budget.maxSteps(); step++) {
            if (Instant.now().isAfter(deadline)) {
                return trace.exhausted(Reason.DEADLINE);
            }

            var response = model.exchange(messages, tools.specsFor(request.allowedTools()));
            tokens += response.usage().total();
            trace.recordModelCall(step, response, tokens);

            if (tokens > budget.maxTokens()) {
                return trace.exhausted(Reason.TOKEN_BUDGET);
            }
            if (response.isTerminal()) {
                return trace.answered(response.text());
            }

            messages.add(new AssistantMessage(response.text(), response.toolCalls()));

            for (ToolCall call : response.toolCalls()) {
                var outcome = executeGuarded(call, request.toolContext(), trace);
                messages.add(new ToolResponseMessage(call.id(), call.name(), outcome.payload()));
            }
        }
        return trace.exhausted(Reason.STEP_BUDGET);
    }
}
```

What the structure buys you:

| Decision | Why it is in the loop |
|---|---|
| `maxSteps` | Terminates the cheap-per-call, infinite-in-aggregate case |
| `maxTokens`, accumulated | The real cost ceiling; steps get more expensive as the transcript grows |
| `wallClock` deadline | An HTTP caller has a timeout; the loop must give up first, with a usable partial answer |
| `request.allowedTools()` | Per-caller tool exposure — a read-only endpoint must not even *see* the write tools |
| `trace` | One audit record per run: every model call, tool call, argument set and outcome |
| Explicit `Reason` | "Exhausted" is not an error; the caller decides whether to show partial output or escalate |

Return the reason to the caller as data. An agent that quietly stops looks identical to one that answered, and that ambiguity ends up as a support ticket.

### Guarded tool execution

A tool that throws must **not** break the loop. The model can often recover if the failure is described to it:

```java
private ToolOutcome executeGuarded(ToolCall call, ToolContext ctx, AgentTrace trace) {
    var started = Instant.now();
    try {
        var tool = tools.byName(call.name())
                .orElseThrow(() -> new ToolInputException("Unknown tool: " + call.name()));
        var args = json.readValue(call.argumentsJson(), tool.argumentType());

        String payload = CompletableFuture
                .supplyAsync(() -> tool.execute(args, ctx), toolExecutor)
                .orTimeout(tool.timeout().toMillis(), MILLISECONDS)
                .join();

        trace.recordTool(call, Outcome.OK, Duration.between(started, Instant.now()));
        return ToolOutcome.ok(truncate(payload, 4_000));

    } catch (JsonProcessingException | ToolInputException e) {
        trace.recordTool(call, Outcome.BAD_INPUT, e.getMessage());
        return ToolOutcome.ok("ERROR: invalid arguments — " + e.getMessage() + ". Fix the arguments and retry once.");

    } catch (ToolForbiddenException e) {
        trace.recordTool(call, Outcome.FORBIDDEN, e.getMessage());
        return ToolOutcome.ok("ERROR: not permitted. Do not retry; explain the limitation to the user.");

    } catch (CompletionException e) {
        trace.recordTool(call, Outcome.FAILED, e.getMessage());
        return ToolOutcome.ok("ERROR: the tool is unavailable. Answer without it or state that you cannot.");
    }
}
```

The error strings are **prompt engineering with a stack trace behind it**. "Fix the arguments and retry once" recovers a malformed call; "Do not retry" prevents a permission denial from consuming the whole step budget. What must never leak into the payload is internal detail — table names, SQL, hostnames, other tenants' ids — because everything you put in a tool message is context the model may repeat verbatim to the user.

`truncate` is not cosmetic either: one tool that returns a 200 KB document blows the token budget in a single step and evicts the system prompt on providers that trim from the front.

## Step 4 — Break repetition loops explicitly

The most common real failure is not an infinite loop; it is the model calling the same tool with the same arguments three times in a row and then apologising. Detect it — cheaply:

```java
class RepetitionGuard {
    private final Map<String, Integer> seen = new HashMap<>();

    boolean isRepeat(ToolCall call) {
        String key = call.name() + '|' + normalize(call.argumentsJson());
        return seen.merge(key, 1, Integer::sum) > 2;
    }
}
```

In the loop, a repeat is short-circuited without touching the tool:

```java
if (repetition.isRepeat(call)) {
    messages.add(new ToolResponseMessage(call.id(), call.name(),
        "ERROR: this exact call was already made and returned the same result. "
      + "Use the previous result, or answer with the information you have."));
    continue;
}
```

Two extra guards worth having from day one:

- **Idempotent replay for `READ` tools.** Cache by the same key within a run: the second identical call costs nothing and the transcript stays consistent.
- **A hard cap on `DESTRUCTIVE` calls per run** (usually one, behind an explicit user confirmation). Budgets bound cost; this bounds damage.

## Step 5 — Make it deterministic in tests

This is the payoff for Step 1's port. A scripted fake replaces the provider entirely:

```java
class ScriptedChatModel implements ChatModelPort {

    private final Deque<AgentResponse> script;
    final List<List<Message>> received = new ArrayList<>();

    ScriptedChatModel(AgentResponse... responses) { this.script = new ArrayDeque<>(List.of(responses)); }

    @Override
    public AgentResponse exchange(List<Message> messages, List<ToolSpec> tools) {
        received.add(List.copyOf(messages));               // assert on what the model was shown
        if (script.isEmpty()) throw new AssertionError("Model called more times than scripted");
        return script.poll();
    }

    static AgentResponse callsTool(String name, String argsJson) {
        return new AgentResponse("", List.of(new ToolCall("call-" + name, name, argsJson)), new TokenUsage(100, 20));
    }
    static AgentResponse answers(String text) {
        return new AgentResponse(text, List.of(), new TokenUsage(120, 30));
    }
}
```

Now the loop's behaviour is ordinary JUnit 5, and the interesting cases are the failure ones:

```java
@Test
void executes_tool_then_answers_with_its_result() {
    var model = new ScriptedChatModel(
            callsTool("search_rules", """
                    {"query":"grappling","limit":2}"""),
            answers("Grappling requires an athletics check."));

    var result = runner.run(request().withTools("search_rules").build());

    assertThat(result.status()).isEqualTo(Status.ANSWERED);
    assertThat(result.text()).contains("athletics check");
    assertThat(result.trace().toolCalls()).singleElement()
            .satisfies(c -> assertThat(c.outcome()).isEqualTo(Outcome.OK));
    // the tool result really reached the second model call
    assertThat(model.received.get(1)).last()
            .extracting(Message::getText).asString().contains("p-412");
}

@Test
void stops_at_step_budget_and_reports_the_reason() {
    var model = new ScriptedChatModel(IntStream.range(0, 3)
            .mapToObj(i -> callsTool("search_rules", "{\"query\":\"q" + i + "\"}"))
            .toArray(AgentResponse[]::new));

    var result = runner.run(request().withBudget(AgentBudget.ofSteps(3)).build());

    assertThat(result.status()).isEqualTo(Status.EXHAUSTED);
    assertThat(result.reason()).isEqualTo(Reason.STEP_BUDGET);
}

@Test
void malformed_arguments_are_reported_to_the_model_not_thrown() {
    var model = new ScriptedChatModel(
            callsTool("search_rules", "{\"limit\": \"not-a-number\"}"),
            answers("Could you rephrase the rules question?"));

    var result = runner.run(request().build());

    assertThat(result.status()).isEqualTo(Status.ANSWERED);
    assertThat(model.received.get(1)).last()
            .extracting(Message::getText).asString().contains("ERROR: invalid arguments");
}

@Test
void repeated_identical_calls_are_short_circuited() {
    var model = new ScriptedChatModel(
            callsTool("search_rules", "{\"query\":\"grappling\"}"),
            callsTool("search_rules", "{\"query\":\"grappling\"}"),
            callsTool("search_rules", "{\"query\":\"grappling\"}"),
            answers("Grappling requires an athletics check."));

    runner.run(request().build());

    assertThat(searchRulesSpy.invocations()).isEqualTo(2);   // third never reached the tool
}

@Test
void write_tools_are_invisible_to_a_read_only_request() {
    var model = new ScriptedChatModel(answers("done"));

    runner.run(request().withTools("search_rules").build());

    assertThat(capturedToolSpecs()).extracting(ToolSpec::name).containsExactly("search_rules");
}
```

Five tests, no API key, milliseconds per run — and they cover the failures that actually happen in production. Build the tool-data with Object Mother builders (`request()`, `campaign()`), the same way you would for any other service: an agent test with twelve lines of literal JSON setup stops being read after a month.

For the tools themselves, test them as plain components — `execute(args, ctx)` is a method call, so authorization and clamping deserve their own tests without any model in sight:

```java
@Test
void apply_damage_rejects_an_actor_from_another_campaign() {
    var ctx = toolContext().actor(anotherGm()).build();

    assertThatThrownBy(() -> applyDamage.execute(new Args("char-1", 5), ctx))
            .isInstanceOf(ToolForbiddenException.class);
}
```

Two things still need a real provider, and only two: that your tool **schemas** are accepted, and that the prompt makes the model pick the right tool. Keep those in a separate, tagged suite (`@Tag("llm")`) that runs nightly with a handful of golden scenarios and asserts on the *trace* — "called `search_rules` before answering", not on exact wording. Never in the PR pipeline.

## Step 6 — Observe and cancel

An agent run is a distributed transaction with a probabilistic coordinator. Instrument it like one, using the Observation API so each step is a child span:

```java
var observation = Observation.createNotStarted("agent.run", registry)
        .lowCardinalityKeyValue("agent", request.agentName())      // bounded → metric tag
        .highCardinalityKeyValue("conversation.id", request.conversationId());

return observation.observe(() -> loop(request));
```

Per step, record `agent.step`, the tool name (low cardinality), the outcome and the duration; on completion record `agent.tokens` and the stop reason. The four signals worth alerting on:

- **Steps per run, p95.** Rising p95 is a prompt or tool-description regression, visible before users complain.
- **Tokens per run, p95.** Your cost SLO, and the earliest sign of an unbounded transcript.
- **Tool outcome rate by name.** A climbing `BAD_INPUT` rate on one tool means its schema or description is misleading the model.
- **Exhaustion rate by reason.** `STEP_BUDGET` climbing means loops; `DEADLINE` climbing means a slow tool or provider.

Cancellation deserves its own mention because it is trivial to get wrong. If the HTTP caller disconnects, the loop should stop *before* the next model call — that is the expensive one:

```java
if (Thread.currentThread().isInterrupted()) {
    return trace.exhausted(Reason.CANCELLED);
}
```

On virtual threads with `spring.threads.virtual.enabled=true`, a Servlet container that interrupts the request thread makes this a one-line check per iteration. Do not, however, interrupt mid-`WRITE`: let the current tool finish, then stop. A cancelled agent that leaves a half-applied mutation is worse than one that runs a second too long.

## Common pitfalls

- **Letting the framework run the loop.** Convenient until you need a budget, an audit trail or a repetition guard. Disable internal tool execution and own the ten lines.
- **A step budget without a token budget.** Steps are linear; tokens are quadratic in the transcript. Only one of those shows up on the invoice.
- **Trusting ids from tool arguments.** Actor, tenant and idempotency key come from the authenticated context. Always.
- **Throwing tool exceptions out of the loop.** A described failure is recoverable; an exception is a 500 and a lost transcript.
- **Returning raw payloads to the model.** Truncate, label, and strip internals — tool messages are context the model may echo to the user.
- **Exposing every tool to every request.** Scope the tool list per caller and per endpoint; an unavailable tool cannot be misused.
- **Testing against the real provider.** Non-deterministic, slow, paid, and it still misses the malformed-arguments path. Script a fake and keep provider tests nightly.
- **Ids and prompts as low-cardinality metric tags.** Conversation ids belong on the span, not on the meter.
- **No repetition guard.** The signature failure of real agents is a stuck triple call, and it costs three full-transcript round trips before anyone notices.

## Outcome

An agent is not a model feature; it is a control loop with a stochastic component in the middle. Treat it like any other orchestrator and the engineering becomes familiar:

1. Put the provider behind a **one-round-trip port** — no retries, no loops, no tool execution.
2. Declare tools as **typed capabilities** with a description written for the model and validation written for you.
3. Own the loop, with **step, token and wall-clock budgets** and an explicit stop reason returned as data.
4. Make tool failures **messages, not exceptions**, and tell the model whether to retry.
5. Add a **repetition guard** and a hard cap on destructive calls.
6. Test the loop with a **scripted fake**: budgets, malformed arguments, repetition and tool scoping, all deterministic.
7. **Observe** steps, tokens, tool outcomes and exhaustion reasons; cancel before the next model call.

The result is boring in the best sense: a bounded, audited, unit-tested component whose cost you can predict and whose failure modes you have already seen in CI.
