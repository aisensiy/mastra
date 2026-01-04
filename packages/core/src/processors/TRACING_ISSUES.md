# Processor Tracing Issues

This document captures findings from investigating processor span tracing issues.

## Background

Processors can implement up to 5 different phase methods:

| Phase        | Method                | When Called                                            |
| ------------ | --------------------- | ------------------------------------------------------ |
| input        | `processInput`        | Once at start, before first LLM call                   |
| inputStep    | `processInputStep`    | At each step of the agentic loop, before each LLM call |
| outputStep   | `processOutputStep`   | After each LLM response, before tool execution         |
| outputStream | `processOutputStream` | For each stream chunk (streaming only)                 |
| outputResult | `processOutputResult` | Once at end, after final response                      |

Processors are collected and combined into workflows via `combineProcessorsIntoWorkflow()` in `agent.ts`.

## Issue 1: Workflow-level spans instead of processor-level spans

### Current Behavior

When processors are combined into a workflow, the runner creates ONE span for the entire workflow execution:

```
- input processor: test-agent-input-processor (ONE span for whole workflow)
  └── (all individual processor executions happen invisibly inside)
```

### Expected Behavior

Each processor should have its own span:

```
- input processor: validator (span per processor)
  └── validator-agent AGENT_RUN
- input processor: summarizer (span per processor)
  └── summarizer-agent AGENT_RUN
```

### Root Cause

1. `combineProcessorsIntoWorkflow()` wraps ALL processors into ONE workflow with ID like `test-agent-input-processor`
2. Runner methods (`runInputProcessors`, `runOutputProcessors`, etc.) create ONE span for the whole workflow
3. Inside the workflow, `createStep` for processors executes the processor method but doesn't create its own span
4. Span creation happens at the wrong level (workflow instead of individual processor)

### Impact

- Span names show workflow ID (`test-agent-input-processor`) instead of processor ID (`validator`)
- No visibility into individual processor execution within the workflow
- Inconsistent with regular (non-workflow) processor behavior which gets individual spans per processor
- Internal agent spans (from processors that call agents) can't be properly nested under their processor span

### Affected Phases

All 4 non-streaming phases are affected:

- `input` phase
- `inputStep` phase
- `outputStep` phase
- `outputResult` phase

## Issue 2: Empty spans for phases with no implementing processors

### Current Behavior

The runner creates spans for step phases (`inputStep`, `outputStep`) even when NO processors in the workflow implement that phase method.

For example, if you have processors that only implement `processInput` and `processOutputResult`:

- `runProcessInputStep` still creates a span for `input step processor: test-agent-input-processor`
- The workflow executes, each step checks if it has `processInputStep`, finds it doesn't, and just passes through
- The span wraps this no-op operation

### Expected Behavior

No span should be created if no processors implement the requested phase.

### Root Cause

1. All processors are combined into one workflow regardless of which phases they implement
2. Runner methods for step phases (`runProcessInputStep`, `runProcessOutputStep`) iterate over ALL processors
3. For workflow processors, they ALWAYS create a span and execute the workflow
4. The workflow handles missing methods gracefully (just passes through), but the span is still created

### Code References

In `runner.ts`, `runProcessInputStep`:

```javascript
// Line 804-824: For workflows, ALWAYS creates span
if (isProcessorWorkflow(processorOrWorkflow)) {
  const result = await this.executeWorkflowAsProcessorWithSpan(...);  // Always runs
}

// Line 829-832: For regular processors, correctly skips
const processMethod = processor.processInputStep?.bind(processor);
if (!processMethod) {
  continue;  // Skip if method doesn't exist
}
```

In `workflow.ts`, `createStep` for processors:

```javascript
// Line 587-640: Checks if method exists, passes through if not
case 'inputStep': {
  if (processor.processInputStep) {
    // ... do the work
  }
  return { ...passThrough, messages }; // Just pass through if not implemented
}
```

## Potential Solutions

### Solution A: Move span creation into workflow steps

Instead of creating one span around the workflow, have each processor step in the workflow create its own span.

**Pros:**

- Individual processor visibility
- Correct span names (processor ID, not workflow ID)
- Internal agent spans nest correctly

**Cons:**

- Requires changes to `createStep` for processors
- Need to pass tracing context through workflow execution

### Solution B: Don't combine processors into workflows

Keep processors as individual items and iterate over them directly in the runner.

**Pros:**

- Simplest conceptually
- Matches current regular processor behavior

**Cons:**

- May break workflow-specific features
- Loses benefits of workflow composition

### Solution C: Track implemented phases when combining

When `combineProcessorsIntoWorkflow` creates the workflow, annotate it with metadata about which phases are actually implemented. Runner can then skip span creation for unimplemented phases.

**Pros:**

- Fixes Issue 2 (empty spans)
- Minimal changes to existing flow

**Cons:**

- Doesn't fix Issue 1 (workflow-level vs processor-level spans)

### Solution D: Separate processor lists by phase

Maintain separate lists for each phase:

- `inputProcessors` - processors implementing `processInput`
- `inputStepProcessors` - processors implementing `processInputStep`
- etc.

**Pros:**

- Clean separation
- Easy to skip empty phases
- Could combine per-phase into smaller workflows

**Cons:**

- More complex processor resolution
- A processor implementing multiple phases would appear in multiple lists

## Related Files

- `packages/core/src/processors/runner.ts` - ProcessorRunner with span creation
- `packages/core/src/agent/agent.ts` - `combineProcessorsIntoWorkflow()`
- `packages/core/src/workflows/workflow.ts` - `createStep()` for processors
- `packages/core/src/observability/types/tracing.ts` - EntityType enum

## Test Reference

The integration test `should trace all processor spans including internal agent spans` in `observability/mastra/src/integration-tests.test.ts` expects:

```
// Expected span structure:
// - Test Agent AGENT_RUN (root)
//   - PROCESSOR_RUN (input processor: validator) - has internal agent
//     - validator-agent AGENT_RUN
//       - validator-agent MODEL_GENERATION
//   - Test Agent MODEL_GENERATION (initial model call)
//   - PROCESSOR_RUN (output processor: summarizer) - has internal agent
//     - summarizer-agent AGENT_RUN
//       - summarizer-agent MODEL_GENERATION
```

But currently produces spans with workflow IDs and extra empty step spans.

## Comprehensive Test Suite

A comprehensive test suite has been created at `observability/mastra/src/processor-tracing.test.ts` that documents expected tracing behavior. **All tests currently fail** because they assert expected behavior, not current behavior.

### Test Coverage

| Category                          | Tests | Description                                                                                                               |
| --------------------------------- | ----- | ------------------------------------------------------------------------------------------------------------------------- |
| **Single Processor**              | 2     | Verifies single input/output processor creates span with processor ID in name                                             |
| **Multiple Processors**           | 3     | Verifies each processor gets individual span (not one workflow span)                                                      |
| **Step Processors**               | 4     | Tests all 5 phases: `processInput`, `processInputStep`, `processOutputStep`, `processOutputStream`, `processOutputResult` |
| **Processor with Internal Agent** | 1     | Verifies internal agent span is child of processor span (tracing context propagation)                                     |
| **Workflow as Processor**         | 2     | Tests workflow used directly as input/output processor                                                                    |
| **Entity Types**                  | 2     | Verifies correct entity types: `INPUT_PROCESSOR`, `INPUT_STEP_PROCESSOR`, `OUTPUT_PROCESSOR`, `OUTPUT_STEP_PROCESSOR`     |
| **Processor Executor Attribute**  | 1     | Verifies `processorExecutor` attribute is set on spans                                                                    |
| **Streaming with Processors**     | 1     | Tests processor tracing during streaming                                                                                  |
| **Mixed Configurations**          | 1     | Tests processors implementing different phase combinations                                                                |
| **Span Hierarchy**                | 1     | Verifies processor spans are direct children of agent span                                                                |
| **Memory Processors**             | 4     | Tests built-in memory processors: `MessageHistory`, `WorkingMemory`, alongside custom processors, execution order         |

**Total: 22 tests**

### Key Expected Behaviors

1. **Span names should use processor ID**, not workflow ID:
   - Expected: `input processor: validator`
   - Current: `input processor: test-agent-input-processor`

2. **Each processor should have its own span** when multiple processors are configured

3. **No spans should be created for phases with no implementing processors**

4. **Internal agent spans should be children of processor spans** (requires `tracingContext` propagation)

5. **Memory processors should be traced** alongside custom processors

### Running the Tests

```bash
cd observability/mastra
pnpm test src/processor-tracing.test.ts
```

All 23 tests will fail until the tracing issues are fixed.

### Expected Span Hierarchy

The tests document the expected processor span hierarchy:

```
AGENT_RUN
  └── input_processor (processInput - once at start, child of AGENT_RUN)
  └── MODEL_GENERATION (ONE per agent run)
      └── MODEL_STEP (per LLM API call in agentic loop)
          └── input_step_processor (processInputStep - before each LLM call)
          └── MODEL_CHUNK (per chunk)
          └── TOOL_CALL (if tools are called)
          └── output_step_processor (processOutputStep - after each LLM response)
  └── output_processor (processOutputResult - once at end, child of AGENT_RUN)
```

**Key hierarchy rules:**

- `processInput` and `processOutputResult` spans are **children of AGENT_RUN** (run once at start/end)
- `processInputStep` and `processOutputStep` spans are **children of MODEL_STEP** (run per LLM call)
- This groups each step in the agentic loop: input_step_processors → model response → output_step_processors

### Implementation Note: Step Processor Context Propagation

**Current Issue:** Step processors don't receive MODEL_STEP as their parent because the wrong `tracingContext` is passed.

**In `llm-execution-step.ts`:**

```typescript
// Line 504: MODEL_STEP span is created
modelSpanTracker?.startStep();

// Line 566-580: BUT workflow's tracingContext is passed, not MODEL_STEP's
const processInputStepResult = await processorRunner.runProcessInputStep({
  ...
  tracingContext,  // This is workflow context, NOT MODEL_STEP context!
  ...
});
```

**Fix Required:** After calling `startStep()`, use `modelSpanTracker.getTracingContext()` instead:

```typescript
modelSpanTracker?.startStep();
const stepTracingContext = modelSpanTracker?.getTracingContext() ?? tracingContext;

// Pass stepTracingContext to processors so they become children of MODEL_STEP
await processorRunner.runProcessInputStep({
  ...
  tracingContext: stepTracingContext,
  ...
});
```

The `ModelSpanTracker.getTracingContext()` method (at `observability/mastra/src/model-tracing.ts:73`) correctly returns the MODEL_STEP span when active:

```typescript
getTracingContext(): TracingContext {
  return {
    currentSpan: this.#currentStepSpan ?? this.#modelSpan,
  };
}
```
