# Csound Debugger Extension — PR #2519 Notes

Personal notes on the development process, PR lifecycle, and lessons learned.
Use this as a reference when preparing the next PR (UDO debugging support).

---

## What Was Built

A **per-k-cycle debug callback** for the Csound debugger, exposed in the C API
and wrapped for the WASM/browser environment.

**In plain terms:** when the Csound debugger is active (`kperf_debug` mode),
a user-supplied function is called after every k-cycle, once all instruments
have run and before audio output is sent. Inside the callback, the caller can
use the existing debugger API (`csoundDebugGetInstrInstances`,
`csoundDebugGetVariables`) to inspect live instrument state in real time.

**Motivation:** Building a web-based Csound debugger at papad.dev required a
hook that fires every k-cycle in the browser. Without it there was no clean
way to know *when* to query instrument variables from JavaScript.

---

## Files Changed

### C Core
| File | What changed |
|---|---|
| `include/csdebug.h` | Added `debug_cb_t` typedef; declared `csoundSetDebugCallback` / `csoundRemoveDebugCallback` with full Doxygen docs |
| `include/csoundCore.h` | Added `debug_cb` and `debug_cb_data` fields to `CSOUND_` struct |
| `Top/csound.c` | Initialized both new fields to NULL in the global `cenviron_` initializer |
| `Top/csound_debug.c` | Wired the callback into `kperf_debug()`; implemented `csoundSetDebugCallback` / `csoundRemoveDebugCallback`; made `csoundDebuggerInit()` idempotent (early return if already initialized) |

### WASM / Browser
| File | What changed |
|---|---|
| `wasm/src/csound_wasm.c` | Added `csoundWasiJsDebugCallback` JS import declaration; added `csoundSetDebugCallbackWasi()` which calls `csoundDebuggerInit()` + sets the callback |
| `wasm/src/exports.json` | Added `csoundSetDebugCallbackWasi`, `csoundSetDebugCallback`, `csoundRemoveDebugCallback` and other debugger functions to the WASM export list |
| `wasm/browser/src/modules/performance.js` | Added `csoundSetDebugCallbackWasi` JS wrapper |
| `wasm/browser/src/libcsound.js` | Added import and export of `csoundSetDebugCallbackWasi` |
| `wasm/browser/src/module.js` | Registered `csoundWasiJsDebugCallback` JS function in WASM env; posts `{ debugCallback: true }` message |
| `wasm/browser/src/events.js` | Added `"debugCallback"` event to the allowed event list; added `triggerDebugCallback()` method |
| `wasm/browser/src/mains/messages.main.js` | Handles incoming `{ debugCallback: true }` message and fires the JS event |
| `wasm/browser/src/mains/sab.main.js` | Added `enableDebugCallback()` convenience method |
| `wasm/browser/src/mains/vanilla.main.js` | Same |
| `wasm/browser/src/mains/worklet.singlethread.main.js` | Same |
| `wasm/browser/index.d.ts` | Added `enableDebugCallback(): Promise<void>` to `CsoundObj`; added `"debugCallback"` to `PublicEvents` type |

### Tests
| File | What changed |
|---|---|
| `tests/c/csound_debug_callback_test.cpp` | New file — 5 Google Test cases covering: set/remove without crash, fires in debug mode, silent without debugger init, stops after remove, skips stopped k-cycles |
| `tests/c/CMakeLists.txt` | Registered the new test file |

---

## Public API Summary

### C API (`include/csdebug.h`)

```c
/* Callback type */
typedef void (*debug_cb_t)(CSOUND *csound, void *userdata);

/* Register a per-k-cycle debug callback.
   Requires csoundDebuggerInit() to have been called first.
   Fires inside kperf_debug() only — NOT in the normal kperf() loop. */
PUBLIC void csoundSetDebugCallback(CSOUND *csound,
                                   debug_cb_t cb, void *userdata);

/* Remove the callback */
PUBLIC void csoundRemoveDebugCallback(CSOUND *csound);
```

### Browser / WASM API

```js
// Enable the debug callback (auto-inits the debugger, idempotent)
// Must be called before csound.start()
await csound.enableDebugCallback();

// Subscribe to the per-k-cycle event
csound.on("debugCallback", () => {
  // read instrument variables here via csoundDebugGetInstrInstances etc.
});
```

---

## How the PR Went — Timeline

### Round 1 — Initial submission
- Submitted as `test_csound_debugger_in_wasm` targeting `develop`
- Had `kcycle_cb` wired into **both** `kperf()` and `kperf_debug()`

**Feedback from vlazzarini (main maintainer):**
> "I would prefer if we did not add callbacks to kperf(). We have deliberately
> removed them in 7.0 to streamline the code. I think it's ok for kperf_debug()
> but not here."

**Fix:** Removed the callback from `kperf()` — kept it in `kperf_debug()` only.

---

### Round 2 — After kperf fix
**Feedback from hlolli (WASM maintainer):**
> "Why do we need this in C? In the browser, the worker→main thread boundary
> is the bottleneck anyway, not the C side. My suggestion: implement
> `csound.on("perform")` in pure JS instead."

**Feedback from vlazzarini:**
> "I guess it is already implemented in C when you run the debugger.
> Rename it to `csoundSetDebugCallback()` to make it explicit as a debugging
> tool. Also add a use case and a test."

**Response strategy:**
- Explained that the C callback is needed for cross-platform debugger access
  (not just for the browser event system)
- Mentioned papad.dev as the real-world use case
- Agreed to: rename, add test, keep JS "perform" event as a separate future PR

---

### Round 3 — Rename + test + WASM auto-init
Changes:
- Renamed `csoundSetKcycleCallback` → `csoundSetDebugCallback` everywhere
- Added `csoundDebuggerInit()` auto-call in the WASM wrapper
- Added 5 Google Tests

**hlolli's remaining comments:**
1. `csoundDebuggerInit()` is not idempotent — double-calling could leak state
2. Event name `"kcycle"` is too generic — rename to `"debugCallback"`
3. Add TypeScript declarations in `index.d.ts`

---

### Round 4 — Final fixes
Changes:
- Made `csoundDebuggerInit()` idempotent in `csound_debug.c`
  (note: the guard lives in C core, NOT in `csound_wasm.c` which only sees
  the opaque `CSOUND*` pointer and cannot access internal fields)
- Renamed JS event from `"kcycle"` to `"debugCallback"`
- Added `enableDebugCallback` and `"debugCallback"` to `index.d.ts`

**Status:** Both maintainers satisfied. Awaiting final merge.

---

## Key Lessons for the Next PR

### On the review process
- **Maintainers are collaborative**, not adversarial. Explain your use case early
  (papad.dev link was very helpful).
- **Keep PRs small and focused.** The UDO debugging work was kept in a separate
  branch (`feat/udo-debugging`) and will be a separate PR.
- **Reply to each review comment explicitly** — name what changed and why.

### On the C/WASM architecture
- `csound_wasm.c` only includes the public `csound.h` — it sees `CSOUND*` as
  an opaque pointer. Never access `csound->internal_field` there — it won't
  compile. Internal guards belong in `Top/` files where `csoundCore.h` is
  included.
- The WASM event flow is: C callback → `csoundWasiJsDebugCallback()` (WASM
  import) → `module.js` posts message → `messages.main.js` fires JS event →
  user's `csound.on("debugCallback", ...)` handler.
- `kperf_debug()` is only active when `csoundDebuggerInit()` has been called.
  The WASM wrapper handles this automatically — native C users must call it
  themselves.

### On Git workflow
- **Never use the VS Code "Sync" button on rebased branches.** It tries to
  pull first and causes conflicts. Always use `git push --force` in the
  terminal after a rebase.
- Keep the PR branch (`test_csound_debugger_in_wasm`) clean — one commit on
  top of `develop`.
- Use `git rebase --onto <target> <exclude-up-to> <branch>` to surgically
  move only specific commits onto a new base.

### On building WASM
- The build requires Nix inside WSL Ubuntu.
- Start the Nix daemon manually: `sudo /nix/var/nix/profiles/default/bin/nix-daemon &`
- Run the build from the `wasm/` directory: `bash scripts/compile.sh`
- All `.sh` and `.nix` files on the Windows NTFS mount may have CRLF endings —
  run `sed -i 's/\r//' scripts/compile.sh scripts/nixpkgs-pin.sh` before
  building if you get `bash\r: No such file or directory` errors.
- The compiled `wasm/lib/csound.wasm` is gitignored — it is not committed to
  the repo. CI rebuilds it from source on every PR.

---

## Branch Map

```
origin/develop  ←── PR #2519 targets this
       │
       └── test_csound_debugger_in_wasm   ← active PR branch
                    │
                    └── feat/udo-debugging  ← next PR (UDO debugging support)
```

## Next PR Checklist (UDO debugging)

- [ ] Rebase `feat/udo-debugging` onto `origin/develop` after PR #2519 merges
- [ ] Review what `csoundDebugGetUdoFrames` exposes — ensure it follows the
      same pattern as `csoundDebugGetInstrInstances`
- [ ] Add WASM exports for new UDO debug functions in `exports.json`
- [ ] Add TypeScript declarations for new functions in `index.d.ts`
- [ ] Write Google Tests for UDO frame inspection
- [ ] Rebuild WASM and test in papad.dev
- [ ] In the PR description, reference PR #2519 as the foundation
