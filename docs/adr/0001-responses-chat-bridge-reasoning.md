# ADR-0001: Normalize Responses reasoning objects for the Chat bridge

- Status: Proposed
- Date: 2026-08-31
- Scope: LiteLLM Responses-to-Chat conversion

## Context

LiteLLM exposes `POST /v1/responses`, but a deployment configured as a custom
OpenAI-compatible Chat Completions model is handled by the Responses-to-Chat bridge.
Responses clients can send reasoning as an object, for example:

```json
{"effort":"high","summary":"detailed","context":"all_turns"}
```

The Chat Completions upstream accepts only a reasoning level string. Forwarding the
whole object serializes it into the level field and causes an upstream HTTP 500:

```text
level "{\"context\":\"all_turns\",\"effort\":\"high\",\"summary\":\"detailed\"}" not supported
```

LiteLLM has related upstream changes ([b6344c3](https://github.com/BerriAI/litellm/commit/b6344c319b82d84f6fe8e34a90ab9948ad4feb0d)
and [ee3ecb5](https://github.com/BerriAI/litellm/commit/ee3ecb59941211a36c2adf14850b5add8c8c81f7)),
but those changes preserve summary-bearing objects for native Responses providers. They do not
cover this Chat bridge deployment.

## Decision

In `transform_responses_api_request_to_chat_completion_request`, extract only the effective
`effort` from a reasoning object:

```python
if isinstance(reasoning_param, dict):
    reasoning_effort = reasoning_param.get("effort")
```

This keeps `high`, `xhigh`, and other supported levels, drops bridge-unsupported `summary` and
`context`, and omits `reasoning_effort` when no effort is supplied. String reasoning values remain
unchanged. The change is limited to the Responses-to-Chat bridge; native Responses provider paths
continue to receive their full reasoning object.

## Consequences

- No client change is required.
- Message history, tools, function calls, streaming, and response conversion are unchanged.
- Chat-only upstreams do not provide Responses-style reasoning summaries; the returned
  `reasoning` field may be empty. Native Responses routing is required for that feature.
- `context` is not removed from the input messages; only the unsupported reasoning metadata is
  omitted.
- The regression test covers dicts with `summary/context`, dicts with `xhigh`, strings, and an
  object without `effort`.

## Verification

The production-shaped request that previously failed was replayed against LiteLLM after the
change. It returned HTTP 200 with a completed response and no error; the upstream no longer
reported `level ... not supported`.

