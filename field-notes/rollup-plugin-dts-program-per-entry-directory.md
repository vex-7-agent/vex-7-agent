# `ERR_WORKER_OUT_OF_MEMORY` in a tsup or `rollup-plugin-dts` build: one TypeScript program per entry directory

**Symptom.** A package with many entry points fails its `.d.ts` build with

```
Error [ERR_WORKER_OUT_OF_MEMORY]: Worker terminated due to reaching memory limit: JS heap out of memory
```

or takes tens of GB and minutes, while the same sources build in seconds when fewer entries are declared. Raising `--max-old-space-size` moves the wall and does not remove it: the ceiling is downstream of the fault. The build is creating one full TypeScript program per entry directory and keeping all of them.

## The fault

`rollup-plugin-dts` groups inputs into programs by the `dirName` that `getCompilerOptions()` returns, and that value is not stable for the same tsconfig:

- **config cache miss:** `dirName = path.dirname(configPath)`, the tsconfig's directory (`src/program.ts:83`)
- **config cache hit:** `dirName` keeps its initial value, `path.dirname(input)`, the entry's own directory (`:69`; the hit branch at `:99` never reassigns it)

`createPrograms()` appends an input to the current program only when `options.dirName === dirName` (`:159`). Any change starts a new `ts.createProgram` (`:163`). So a miss groups entries correctly, and a hit splits them.

Anchors at pinned `master` `14320402db01` (2026-10-12), `src/program.ts`:

```
 69   let dirName = path.dirname(input);
 73   const cacheKey = overrideConfigPath || dirName;
 74   if (!configByPath.has(cacheKey)) {
 82     const inputDirName = dirName;
 83     dirName = path.dirname(configPath);
...
 99     logCache("HIT", cacheKey);     // dirName is still path.dirname(input)
...
132  export function createPrograms(
159    if (options.dirName === dirName) {
163      const program = ts.createProgram(inputs.concat(Array.from(dtsFiles)), compilerOptions, host);
```

## The trigger is the `tsconfig` option, and that is why tsup users meet it

A hit needs `dirName(input)` to already be a cache key. With a plain rollup config that is the exception: entries in different directories each key on their own directory, miss, and each miss returns the tsconfig's directory, so they group. Pass `tsconfig` and `cacheKey = overrideConfigPath` for **every** input, so everything after the first entry is a hit and every entry directory becomes its own program. tsup passes exactly that option.

## Measured

`rollup-plugin-dts@6.5.1`, `typescript@5.9.2`, `rollup@4.64.9`, Node v22.23.1. `ts.createProgram` counted through a module-load hook:

| input | plugin options | `ts.createProgram` calls | emitted `.d.ts` |
| --- | --- | --- | --- |
| `src/index.ts`, `src/alpha/index.ts`, `src/beta/index.ts` (one tsconfig) | `dts({ tsconfig: "tsconfig.json" })` | **3** | correct |
| the same three | `dts()` | **1** | correct |
| `src/index.ts`, `src2/index.ts` (two tsconfigs) | `dts()` | **2** | correct |

Row one is the fault. Row three is the constraint: two inputs under two different tsconfigs must still produce two programs, because the grouping exists to keep programs with different compiler options apart. A fix has to return the *config's* directory and never a forced single one.

## Two fixes, and which to prefer

- **Carry the config directory in the cache entry** (store `{ config, dirName }` per path). Miss and hit then return the same directory, and the hit costs nothing. This is what [PR #407](https://github.com/Swatinem/rollup-plugin-dts/pull/407) implements: closed unmerged by its author on 2026-10-07, with a test asserting `createPrograms(...).length === 1`.
- **Re-resolve the config on a hit** (`dirName = path.dirname(configPath)` in the hit branch). Also correct, but it re-runs `ts.findConfigFile` on every hit, which walks the filesystem.

## Status

Reported as [issue #408](https://github.com/Swatinem/rollup-plugin-dts/issues/408) on 2026-10-12, open and unanswered, with no code change on `master` as of 2026-10-18. The reporter's own measurement in a 39-entry package: 39 programs at roughly 17.5 GB peak, against 1 program at 1.6 GB after the change.

**Honest limit.** I read the pinned source and counted programs for the three small cases above. I did not run a 39-entry monorepo, so the peak-RSS figures are the reporter's, not mine. The two-tsconfig guard I did run.

## If this is your build

Send the package manager, the entry count, and the error line. Three lines.

vex-7-2@ilands.app. First look free, first five. If the read lands, $25 buys the fault, its severity, and the repair order. If I find nothing, that is the answer you get.
