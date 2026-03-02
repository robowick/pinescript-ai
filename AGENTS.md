# PineScript v6 — Agent Rules

When generating, reviewing, or modifying PineScript code, follow every rule below. These rules are extracted from the PineScript v6 specification and TradingView's compiler requirements.

## Script Structure

- First non-empty line MUST be `//@version=6`
- Every script MUST have exactly ONE of: `indicator()`, `strategy()`, or `library()`
- Never combine `indicator()` and `strategy()` in the same script
- Always include a descriptive title string in the declaration: `indicator("My RSI Divergence")`
- Parentheses, brackets, and string quotes must be balanced

## Deprecated Patterns (NEVER use in v6)

These will cause compilation errors or warnings on TradingView:

| Deprecated | Replacement |
|---|---|
| `security()` | `request.security()` |
| `study()` | `indicator()` |
| `transp=` parameter | `color.new(color, transparency)` where transparency is 0-100 |
| `iff()` | Ternary: `condition ? valueIfTrue : valueIfFalse` |
| `tostring()` | `str.tostring()` |
| `tonumber()` | `str.tonumber()` |
| `input.integer()` | `input.int()` |
| `input.resolution()` | `input.timeframe()` |
| `input.symbol()` | `input.symbol()` |

## Things That Do NOT Exist

LLMs commonly hallucinate these — never use them:

- `plot.style_dashed` — DOES NOT EXIST. Use `plot.style_line` with `linewidth` for visual distinction.
- `plot.style_dotted` — DOES NOT EXIST. Use `plot.style_circles` or `plot.style_cross` instead.
- Do not invent function signatures. If unsure whether a function exists, check `data/pinescript-docs/reference-functions.json` in this repo.

## v6 Type System

- `bool x = na` is INVALID — use `bool x = bool(na)`
- `int x = na` needs explicit cast — use `int x = int(na)`
- `float x = na` is acceptable (float is the default na type)
- Type qualifiers: `const`, `input`, `simple`, `series` — understand the qualifier hierarchy
- `method` keyword is used for defining methods on UDTs (user-defined types)

## v6-Specific Traps

- `fill()` cannot mix `hline` and `plot` references — both arguments must be the same type
- `input.int()` and `input.float()` use `defval=`, NOT `def=`
- `request.security()` without `barmerge.lookahead_off` causes repainting on historical bars
- `barstate.isrealtime`-dependent logic behaves differently in historical vs live context

## TradingView Limits

- Maximum **64** plot-type calls per script (`plot`, `plotshape`, `plotchar`, `plotarrow`, `plotcandle`, `plotbar`, `hline`, `fill`, `bgcolor`, `barcolor`)
- Maximum **40** `request.*` calls per script
- Compiled script size limit is approximately **70K characters**
- When approaching these limits, suggest splitting into a `library()` + `indicator()` pattern

## Repainting Awareness

Always flag potential repainting when you see:

- `request.security()` without explicit `lookahead=barmerge.lookahead_off`
- Logic that depends on `barstate.isrealtime` or `barstate.isconfirmed`
- Using `close` in `request.security()` on a higher timeframe without confirming bar close
- `timenow` in calculations (changes on every tick)

## Best Practices

- Use `input()` functions for all configurable parameters with proper `defval`
- Use `var` for variables that should persist across bars (not reset each bar)
- Prefer built-in `ta.*` functions over manual calculations (e.g. `ta.rsi()` not rolling your own)
- Handle `na` values explicitly — use `na()` checks or `nz()` for defaults
- Handle first-bar edge cases where historical data may not exist yet
- Use `math.max()`, `math.min()`, `math.abs()` — not raw operators for safety
- Include meaningful plot colors and styles to distinguish overlapping outputs
- Use `tooltip` parameter in `plotshape()` and `label.new()` for interactive data

## Response Format

When generating PineScript code:

1. Put the complete code block FIRST, then explanation after
2. Use ` ```pinescript ` fenced code blocks
3. Always generate complete, compilable scripts — not fragments
4. When modifying existing code, update the existing script rather than rewriting from scratch

## Function Reference Namespaces

PineScript v6 organizes functions into namespaces. The primary ones:

- `ta.*` — Technical analysis: `ta.rsi`, `ta.sma`, `ta.ema`, `ta.macd`, `ta.crossover`, `ta.crossunder`, `ta.atr`, `ta.bb`, `ta.stoch`, `ta.pivot`, `ta.highest`, `ta.lowest`, `ta.change`, `ta.rma`, `ta.wma`, `ta.vwma`
- `strategy.*` — Backtesting: `strategy.entry`, `strategy.close`, `strategy.exit`, `strategy.position_size`, `strategy.equity`
- `request.*` — External data: `request.security`, `request.financial`, `request.seed`, `request.currency_rate`
- `math.*` — Math: `math.abs`, `math.round`, `math.max`, `math.min`, `math.pow`, `math.sqrt`, `math.log`, `math.exp`
- `str.*` — Strings: `str.tostring`, `str.format`, `str.contains`, `str.replace`, `str.split`
- `array.*` — Arrays: `array.new`, `array.push`, `array.pop`, `array.get`, `array.set`, `array.size`, `array.sort`
- `color.*` — Colors: `color.new`, `color.from_gradient`, `color.rgb`
- `input.*` — User inputs: `input.int`, `input.float`, `input.bool`, `input.string`, `input.color`, `input.timeframe`, `input.symbol`
- `chart.*` — Chart info: `chart.point.new`
- `map.*` — Maps: `map.new`, `map.put`, `map.get`, `map.keys`, `map.values`
- `matrix.*` — Matrices: `matrix.new`, `matrix.get`, `matrix.set`, `matrix.mult`

## RAG Data Available in This Repo

This repository contains pre-indexed PineScript v6 documentation at `data/pinescript-docs/`:

- `reference-functions.json` — 441 function signatures with examples
- `docs-chunks.json` — 438 documentation chunks covering concepts, types, execution model
- `example-scripts.json` — 285 complete working PineScript examples
- `bm25-index.json` — Search index (used by the app's RAG engine)

When unsure about a function's exact signature or parameters, consult `reference-functions.json` before generating code.

## Pine Transpiler (Sibling Repo)

The `pine-transpiler` at `../pine-transpiler/` is an AST-based PineScript-to-JavaScript transpiler. It provides a `canTranspilePineScript(code)` function useful for validation — if code parses through the transpiler's lexer + parser, it is structurally valid PineScript.

### Transpiler Limitations to Be Aware Of

The transpiler is indicators-only. These features are NOT supported by the transpiler:

- `strategy.*` functions (no backtesting)
- `request.security`, `request.financial`, and all `request.*` data functions
- `matrix.*`, `map.*`, `polyline.*` data structures
- `line.new`, `label.new`, `box.new`, `table.new` (parsed but produce no-op stubs)
- External library imports (parsed but not resolved)

These limitations affect transpiler validation only — they do NOT mean you shouldn't use these features in PineScript. TradingView supports them natively. The transpiler just can't validate scripts that rely on them.
