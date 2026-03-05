# Plan: kusto-flow-run Skill

## Context

Need a new skill to query and analyze Power Platform flow run history from Kusto. It consumes `flowResourceKusto` output from `/kusto-flow-summary` and produces detailed run analysis: event breakdowns, status summaries, orphaned runs, duration percentiles, and ASCII timecharts.

## Files to Create

```
C:\Users\chenlu\.claude\skills\kusto-flow-run\
├── SKILL.md              (~370 lines)
└── references/
    └── queries.kql       (2 queries: Q1 RunCount, Q2 FlowRunEvents)
```

## Query Design (references/queries.kql)

**Q1: RunCount** — Count WorkflowRunStart events in 28d range to check if >5K.

```kql
TraceEvents
| where serviceName == 'PowerAutomate.WorkflowsManagement' or serviceName == _serviceName
| where env_time >= _runStartTime and env_time <= _runEndTime
| where customDimensions has _flowName and customDimensions has _environmentId
| where eventName == 'WorkflowRunStart'
| count
```

**Q2: FlowRunEvents** — All 4 event types, parse customDimensions fields.

```kql
TraceEvents
| where serviceName == 'PowerAutomate.WorkflowsManagement' or serviceName == _serviceName
| where env_time >= _runStartTime and env_time <= _runEndTime
| where customDimensions has _flowName and customDimensions has _environmentId
| where eventName == 'WorkflowRunStart'
    or eventName == 'WorkflowRunEnd'
    or eventName == 'WorkflowRunDispatched'
    or eventName == 'WorkflowRunStartThrottled'
| parse kind=regex flags=U customDimensions with '"flowRunSequenceId": "'flowRunSequenceId: string'"'
| parse kind=regex flags=U customDimensions with '"kind": "'flowKind: string'"'
| parse kind=regex flags=U customDimensions with '"durationInMilliseconds": 'durationInMilliseconds: long','
| parse kind=regex flags=U customDimensions with '"status": "'status: string'"'
| parse kind=regex flags=U customDimensions with '"statusCode": "'statusCode: string'"'
| parse kind=regex flags=U customDimensions with '"flowId": "'flowId: string'"'
| parse kind=regex flags=U customDimensions with '\\\\"flowDisplayName\\\\":\\\\"'flowDisplayName: string'\\\\"'
| project env_time, eventName, flowRunSequenceId, status, statusCode, correlationId, activityId, flowKind, durationInMilliseconds, flowId, flowDisplayName, serviceName
| order by env_time desc
```

Key notes:
- Only `WorkflowRunEnd` events have populated `status`, `statusCode`, and `durationInMilliseconds` values
- `durationInMilliseconds` is parsed as `long` (no quotes in JSON)
- `flowDisplayName` uses escaped-quote regex (`\\\\"`) because it's nested JSON

## SKILL.md Workflow

### Step 1: Obtain flowResourceKusto
Parse from session's `/kusto-flow-summary` output. Extract: `_environmentId`, `_flowName`, `_flowId`, `_serviceName`, `_cluster`, `_database`. If unavailable → stop, ask user to run `/kusto-flow-summary`.

### Step 2: Load queries.kql
Split by `//== Q<N>:` markers.

### Step 3: Build Preamble
Combine flowResourceKusto + override time variables:
```kql
let _runStartTime = ago(28d);
let _runEndTime = now();
```

### Step 4: Execute Q1 (RunCount)
Run count query. If count > 5000 → `adjustedDays = floor(28 * 5000 / count)` (min 1d), update `_runStartTime`, report adjustment to user.

### Step 5: Execute Q2 (FlowRunEvents)
Run full query with (possibly adjusted) time range.

### Step 6: Client-Side Processing
Process result set into 18 sections:

| # | Section | Source Filter | Group By | Sort |
|---|---------|--------------|----------|------|
| 1 | WorkflowRunStart Events | eventName='WorkflowRunStart' | — | env_time desc |
| 2 | WorkflowRunEnd Events | eventName='WorkflowRunEnd' | — | env_time desc |
| 3 | WorkflowRunStartThrottled Events | eventName='WorkflowRunStartThrottled' | — | env_time desc |
| 4 | WorkflowRunDispatched Events | eventName='WorkflowRunDispatched' | — | env_time desc |
| 5 | Start + End Combined | Start or End | — | env_time desc |
| 6 | Orphaned Runs (Start w/o End) | Start where flowRunSequenceId NOT IN End's set | — | env_time desc |
| 7 | Summary by status | End | status | count desc |
| 8 | Summary by statusCode | End | statusCode | count desc |
| 9 | Summary by status+statusCode | End | status,statusCode | count desc |
| 10 | Status per day | End | day,status | day asc |
| 11 | StatusCode per day | End | day,statusCode | day asc |
| 12 | Status+StatusCode per day | End | day,status,statusCode | day asc |
| 13 | Latest run per status | End | status | arg_max(env_time) |
| 14 | Latest run per statusCode | End | statusCode | arg_max(env_time) |
| 15 | Latest run per status+statusCode | End | status,statusCode | arg_max(env_time) |
| 16 | Top 10 slowest runs | End | — | durationMs desc, top 10 |
| 17 | Duration P50/P99 per day | End | day | day asc |
| 18 | Timecharts | Generated from #10-12, #17 | — | ASCII bar charts |

- Event listing sections (1-6): cap display at 100 rows with note "Showing 100 of N rows"
- Summary sections (7-17): display all rows

### Step 7: Generate ASCII Timecharts
For per-day sections (10-12, 17), produce horizontal bar charts:
```
Day          | Chart                                    | Value
-------------|------------------------------------------|------
2026-02-28   |########                                  |   160
2026-03-01   |##############################            |   600
```
Max bar width 40 chars using `#`. Largest value gets 40 chars, others proportional. One chart per distinct series value (e.g., per status).

### Step 8: Save to File
Write to `{_flowName}-flow-run.md` in current working directory. Structure:
- Header (env ID, flow name, query range, total events, date)
- Event Listings (sections 1-6)
- Status Summaries (sections 7-9)
- Per-Day Summaries with inline timecharts (sections 10-12)
- Latest Runs (sections 13-15)
- Duration Analysis (sections 16-17 with timecharts)

### Step 9: Declare Session Values
Output summary counts: totalEvents, runStartCount, runEndCount, throttledCount, dispatchedCount, orphanedCount, queryRange, rangeAdjusted, outputFile.

## Error Handling
- Missing flowResourceKusto → ask user to run `/kusto-flow-summary`
- Query timeout → retry once with 180s
- Zero results → report no events found, suggest checking time range
- Auth failure → remind about Microsoft corp tenant

## Skill Registration
Add to settings.json system prompt list (follows existing kusto-flow-summary pattern). The YAML frontmatter `name` and `description` handle auto-discovery.

## Verification
1. Run `/kusto-env-summary <env-guid>` → `/kusto-flow-summary <flow-id>` → `/kusto-flow-run`
2. Verify output file generated with all 18 sections
3. Verify timecharts render correctly in markdown
4. Test with a high-volume flow to verify count>5K date adjustment logic

## Unresolved Questions
None.
