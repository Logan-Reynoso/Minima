# Minima: estimation methodology

This document explains how the **current hackathon prototype** calculates its displayed impact estimates. It is based on the implementation in `server.mjs`; it is not a measurement study or an independently validated energy model.

## 1. Approximate token count

`countTokens(text)` estimates tokens from whitespace-delimited words:

```text
tokens ~= max(1, ceil(word_count * 1.33))
```

This is not the NVIDIA model's tokenizer and will differ from actual billed usage.

## 2. Predicted output size

`predictOutputTokens` assigns heuristic output sizes based on prompt structure. For original prompts it uses broad-scope, category, and constraint checks; for rewritten prompts it looks for word limits or output formats and otherwise uses defaults. These are projections, not sampled results.

## 3. Energy model

The estimator combines a projected total token count, a call-weight factor, and fixed energy coefficients:

```text
total_tokens   = input_tokens + projected_output_tokens
call_weight    = max(1, model_calls + 0.35 * tool_calls)
scenario_scale = 0.65 if optimized else 1

low_Wh    = (total_tokens / 1000) * 0.35 * call_weight * scenario_scale
likely_Wh = (total_tokens / 1000) * 1.25 * call_weight * scenario_scale
high_Wh   = (total_tokens / 1000) * 5.50 * call_weight * scenario_scale
```

The `0.65` optimization multiplier and `0.35/1.25/5.50` Wh coefficients are assumptions built into the prototype, not measured device efficiency values.

The app then derives rough carbon, water, and everyday-use equivalents from these outputs using additional fixed factors.

## 4. Projected savings

The backend uses `buildSavings` to compare the *likely* values for original and optimized scenarios:

```text
projected_savings_Wh = max(0, original_likely_Wh - optimized_likely_Wh)
projected_savings_pct = round(100 * projected_savings_Wh / original_likely_Wh)
```

The percentage is therefore the result of comparing **two modeled scenarios**, not the difference between two electricity-meter measurements.

## 5. Major limitations

1. Token counts are approximated rather than read from NVIDIA API usage metadata.
2. Original and rewritten output lengths are heuristic projections, sometimes much larger or smaller than real outputs.
3. The energy coefficients are not calibrated to specific GPU hardware, batching, load, or data-center efficiency.
4. The optimized comparison does not fully account for optimizer model calls, retries, or fallback overhead. In fact, a pre-flight rewrite may consume inference resources before any final-answer call.
5. The application does not verify that an optimized answer has equivalent quality, completeness, or user utility to the original request.
6. Carbon and water equivalents add further assumptions and should not be interpreted as precise emissions accounting.

## 6. Suggested next evaluation

- Select and publish a fixed benchmark set of broad, bounded, and already-efficient prompts.
- Capture actual input/output token usage from responses if the endpoint supplies it.
- Record each optimizer and final-answer call, including retries, to compare total workflow cost.
- Compare answer quality and task success against the original requests.
- Present assumptions and ranges rather than a single universal savings percentage.
- Avoid asserting actual energy reduction unless validated with appropriate hardware-level energy measurement.

This transparency is part of the engineering story: Minima is a prompt-interception and optimization **prototype**, with an impact-visualization model that should be refined through measurement.
