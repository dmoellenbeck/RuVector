# RuVector Local Setup (rvlite Cypher Fix)

Until PR #270 is merged and a new ruvector npm release includes the fix,
the following manual steps are required to enable rvlite Cypher with
multi-row property queries.

## Prerequisites

- Node.js 22.5+
- Rust toolchain (`rustup`, `cargo`)
- `wasm-pack` (`cargo install wasm-pack`)

## Step 1: Build the fixed WASM

```bash
cd ~/dev_workspace/claude-coach/RuVector
git checkout fix/cypher-multirow-properties

# Build WASM (takes ~2 min first time)
wasm-pack build crates/rvlite --target web --out-dir ../../pkg-rvlite
```

Output: `RuVector/pkg-rvlite/` with `rvlite_bg.wasm` + `rvlite.js`

## Step 2: Install ruvector globally

```bash
npm install -g ruvector@latest
npm install --prefix $(npm root -g)/ruvector rvlite
```

## Step 3: Replace WASM + JS glue in global ruvector

```bash
RVLITE_WASM="$(npm root -g)/ruvector/node_modules/rvlite/dist/wasm"

rm -f "$RVLITE_WASM/rvlite_bg.wasm" "$RVLITE_WASM/rvlite.js"
cp ~/dev_workspace/claude-coach/RuVector/pkg-rvlite/rvlite_bg.wasm "$RVLITE_WASM/"
cp ~/dev_workspace/claude-coach/RuVector/pkg-rvlite/rvlite.js "$RVLITE_WASM/"
```

## Step 4: Fix rvlite package.json exports

The rvlite npm package does not export `./package.json`, which causes
`require.resolve('rvlite/package.json')` to fail:

```bash
node -e "
const fs = require('fs');
const p = '$(npm root -g)/ruvector/node_modules/rvlite/package.json';
const pkg = JSON.parse(fs.readFileSync(p));
pkg.exports['./package.json'] = './package.json';
fs.writeFileSync(p, JSON.stringify(pkg, null, 2));
console.log('Fixed rvlite exports');
"
```

## Step 5: Patch MCP server for ESM rvlite import

The ruvector MCP server uses `require('rvlite')` (CJS) but rvlite v0.2.x
is ESM-only. The `rvlite_cypher` case in `mcp-server.js` must be patched.

File: `$(npm root -g)/ruvector/bin/mcp-server.js`

Find the `case 'rvlite_cypher':` block and replace it with:

```javascript
case 'rvlite_cypher': {
  try {
    if (!globalThis._rvliteCypherEngine) {
      const fs = require('fs');
      const path = require('path');
      const rvliteBase = path.join(__dirname, '..', 'node_modules', 'rvlite');
      const wasmPath = path.join(rvliteBase, 'dist', 'wasm', 'rvlite_bg.wasm');
      if (!fs.existsSync(wasmPath)) {
        return { content: [{ type: 'text', text: JSON.stringify({
          success: false,
          error: 'rvlite WASM not found at ' + wasmPath,
          hint: 'Install with: npm install rvlite'
        }, null, 2) }] };
      }
      const wasmModulePath = 'file://' + path.join(rvliteBase, 'dist', 'wasm', 'rvlite.js');
      const rvliteWasm = await import(wasmModulePath);
      const wasmBuffer = fs.readFileSync(wasmPath);
      rvliteWasm.initSync(wasmBuffer);
      globalThis._rvliteCypherEngine = new rvliteWasm.CypherEngine();
    }
    const cypher = globalThis._rvliteCypherEngine;
    const results = cypher.execute(args.query);
    return { content: [{ type: 'text', text: JSON.stringify({
      success: true,
      query_type: 'cypher',
      results,
      row_count: results.rows ? results.rows.length : 0
    }, null, 2) }] };
  } catch (e) {
    return { content: [{ type: 'text', text: JSON.stringify({
      success: false,
      error: e.message
    }, null, 2) }], isError: true };
  }
}
```

## Step 6: Configure MCP server in Claude Code

```bash
claude mcp add ruvector -- ruvector mcp start
```

## Step 7: Verify

Restart Claude Code, then test:

```
> Use rvlite_cypher: CREATE (:Test {value: 42})
> Use rvlite_cypher: MATCH (t:Test) RETURN t.value
```

Expected: `{ rows: [{ "t.value": 42 }], row_count: 1 }`

## What changes after PR #270 is merged

Once a new ruvector npm release includes the fix, only these steps remain:

```bash
npm install -g ruvector@latest
claude mcp add ruvector -- ruvector mcp start
```

Steps 1-5 (WASM build, file replacement, patching) become unnecessary.
The MCP server ESM fix (Step 5) should ideally also be included upstream.
