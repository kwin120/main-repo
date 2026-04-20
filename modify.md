# TestRunner.vue — Modification Guide

## Overview

This document describes all changes needed to add four features to the existing `TestRunner.vue`:

1. Test list with checkboxes (left column)
2. Board profile dropdown with Add / Save buttons
3. Board profile persistence via Electron `userData`
4. Output window with per-test status, stdout/stderr, timing, and overall progress

The layout changes from a single-column flow to a **two-column split**: a narrow left control panel and a wider right output panel.

---

## Required changes to `electron-preload/index.ts`

Before touching the Vue file, add two new IPC methods to the preload bridge. The board profiles need to be read/written via the main process since `fs` is not available in the renderer.

```typescript
contextBridge.exposeInMainWorld('engineApi', {
  listTests:   () => ipcRenderer.invoke('engine-list-tests'),
  runTest:     (t)  => ipcRenderer.invoke('engine-run-test', t),
  // NEW:
  loadProfiles: () => ipcRenderer.invoke('profiles-load'),
  saveProfiles: (data: unknown) => ipcRenderer.invoke('profiles-save', data),
})
```

## Required changes to `electron-main/index.ts`

Add two IPC handlers that read/write a `board-profiles.json` file in `app.getPath('userData')`.

```typescript
import { app, ipcMain } from 'electron'
import path from 'path'
import fs from 'fs'

const PROFILES_PATH = path.join(app.getPath('userData'), 'board-profiles.json')

ipcMain.handle('profiles-load', () => {
  try {
    if (!fs.existsSync(PROFILES_PATH)) return {}
    return JSON.parse(fs.readFileSync(PROFILES_PATH, 'utf-8'))
  } catch {
    return {}
  }
})

ipcMain.handle('profiles-save', (_event, data: unknown) => {
  fs.writeFileSync(PROFILES_PATH, JSON.stringify(data, null, 2), 'utf-8')
})
```

The JSON structure written to disk:

```json
{
  "Z790-P DDR4": ["power_state", "user_account"],
  "X670E-F":     ["power_state"]
}
```

---

## Changes to `TestRunner.vue`

### 1. Update type definitions (`<script setup>`)

Add the two new API methods to the `EngineApi` type:

```typescript
// BEFORE
type EngineApi = {
  listTests: () => Promise<TestMeta[]>
  runTest: (payload: { name: string; params: Record<string, string> }) => Promise<RunResult>
}

// AFTER
type EngineApi = {
  listTests:    () => Promise<TestMeta[]>
  runTest:      (payload: { name: string; params: Record<string, string> }) => Promise<RunResult>
  loadProfiles: () => Promise<Record<string, string[]>>
  saveProfiles: (data: Record<string, string[]>) => Promise<void>
}
```

Also extend `RunResult` to include timing:

```typescript
// BEFORE
type RunResult = {
  status: string
  stdout?: string
  stderr?: string
}

// AFTER
type RunResult = {
  status: string
  stdout?: string
  stderr?: string
  durationMs?: number   // added — populated client-side before pushing to results
}
```

---

### 2. Add new reactive state (`<script setup>`)

Add these refs below the existing ones:

```typescript
// Board profile state
const profiles        = ref<Record<string, string[]>>({})  // { boardName: [testName, ...] }
const selectedBoard   = ref<string>('')
const newBoardName    = ref<string>('')
const addingBoard     = ref<boolean>(false)

// Per-test run-state for the output panel
type TestRunState = 'idle' | 'running' | 'pass' | 'fail' | 'error'
const runStates  = ref<Record<string, TestRunState>>({})
const runTimes   = ref<Record<string, number>>({})   // milliseconds

// Progress counters (derived from runStates)
const totalSelected  = computed(() => selected.value.filter(Boolean).length)
const totalDone      = computed(() =>
  Object.values(runStates.value).filter(s => s === 'pass' || s === 'fail' || s === 'error').length
)
const totalPass      = computed(() => Object.values(runStates.value).filter(s => s === 'pass').length)
const totalFail      = computed(() => Object.values(runStates.value).filter(s => s === 'fail' || s === 'error').length)
const progressPct    = computed(() =>
  totalSelected.value === 0 ? 0 : Math.round((totalDone.value / totalSelected.value) * 100)
)
```

---

### 3. Update `getEngineApi()` dev mock

Add stub implementations for the two new methods so the dev server still works without Electron:

```typescript
// Inside the DEV branch of getEngineApi(), after the existing runTest stub:
async loadProfiles() {
  return {}
},
async saveProfiles(_data: Record<string, string[]>) {
  // no-op in dev
},
```

---

### 4. Add board profile functions (`<script setup>`)

```typescript
async function loadProfiles() {
  try {
    const api = getEngineApi()
    profiles.value = await api.loadProfiles()
    const names = Object.keys(profiles.value)
    if (names.length > 0 && !selectedBoard.value) {
      selectedBoard.value = names[0]
      applyBoardProfile()
    }
  } catch (err) {
    console.error('Failed to load profiles', err)
  }
}

function applyBoardProfile() {
  if (!selectedBoard.value) return
  const testNames = profiles.value[selectedBoard.value] ?? []
  selected.value = tests.value.map(t => testNames.includes(t.name))
}

async function saveProfile() {
  if (!selectedBoard.value) return
  profiles.value[selectedBoard.value] = tests.value
    .filter((_, i) => selected.value[i])
    .map(t => t.name)
  try {
    await getEngineApi().saveProfiles(profiles.value)
  } catch (err) {
    console.error('Failed to save profile', err)
  }
}

function addBoard() {
  const name = newBoardName.value.trim()
  if (!name || profiles.value[name]) return
  profiles.value[name] = []
  selectedBoard.value = name
  newBoardName.value = ''
  addingBoard.value = false
  selected.value = tests.value.map(() => false)
}
```

---

### 5. Update `loadTests()` to also load profiles

```typescript
// BEFORE — at the end of the try block:
initializeFormState(tests.value)

// AFTER — call loadProfiles after tests are ready:
initializeFormState(tests.value)
await loadProfiles()
```

---

### 6. Update `runSelected()` to track per-test state and timing

Replace the existing `runSelected` function:

```typescript
async function runSelected() {
  if (!hasSelection.value) return

  running.value  = true
  error.value    = null
  results.value  = []
  runStates.value = {}
  runTimes.value  = {}

  try {
    const api   = getEngineApi()
    const queue = tests.value
      .map((test, index) => ({ test, index }))
      .filter(({ index }) => selected.value[index])

    for (const { test, index } of queue) {
      runStates.value[test.name] = 'running'
      const t0 = Date.now()

      try {
        const result = await api.runTest({ name: test.name, params: values.value[index] ?? {} })
        const ms = Date.now() - t0
        runStates.value[test.name] = result.status === 'pass' ? 'pass' : 'fail'
        runTimes.value[test.name]  = ms
        results.value.push({ name: test.name, data: { ...result, durationMs: ms } })
      } catch (err) {
        const ms = Date.now() - t0
        runStates.value[test.name] = 'error'
        runTimes.value[test.name]  = ms
        results.value.push({
          name: test.name,
          data: { status: 'error', stdout: '', stderr: err instanceof Error ? err.message : String(err), durationMs: ms },
        })
      }
    }
  } finally {
    running.value = false
  }
}
```

---

### 7. Add `runSingle(testName)` for the "Run One" button

```typescript
async function runSingle(testName: string) {
  const idx = tests.value.findIndex(t => t.name === testName)
  if (idx === -1) return

  running.value = true
  runStates.value[testName] = 'running'
  const t0 = Date.now()

  try {
    const api    = getEngineApi()
    const result = await api.runTest({ name: testName, params: values.value[idx] ?? {} })
    const ms     = Date.now() - t0
    runStates.value[testName] = result.status === 'pass' ? 'pass' : 'fail'
    runTimes.value[testName]  = ms

    const existing = results.value.findIndex(r => r.name === testName)
    const entry    = { name: testName, data: { ...result, durationMs: ms } }
    if (existing >= 0) results.value[existing] = entry
    else               results.value.push(entry)
  } catch (err) {
    runStates.value[testName] = 'error'
    runTimes.value[testName]  = Date.now() - t0
  } finally {
    running.value = false
  }
}
```

---

### 8. Replace the entire `<template>`

Replace everything inside `<template>` with the following. The layout is a two-column grid — left column is the control panel, right column is the output panel.

```html
<template>
  <div class="runner">
    <header class="hero">
      <div>
        <p class="eyebrow">ASRR Diagnostics</p>
        <h1>Test Runner</h1>
      </div>
      <button class="ghost-button" type="button" :disabled="loading" @click="loadTests">
        {{ loading ? 'Loading...' : 'Reload Tests' }}
      </button>
    </header>

    <div v-if="loading" class="panel status-panel"><p>Loading tests...</p></div>

    <div v-else class="content">
      <div v-if="error" class="panel load-error">
        <h2>Unable to load tests</h2>
        <p>{{ error }}</p>
      </div>

      <div v-if="tests.length > 0" class="two-col">

        <!-- ── LEFT: control panel ── -->
        <div class="panel control-panel">

          <!-- Board profile selector -->
          <p class="section-label">Board Profile</p>
          <div class="board-row">
            <select v-if="!addingBoard" v-model="selectedBoard" @change="applyBoardProfile">
              <option v-for="name in Object.keys(profiles)" :key="name" :value="name">{{ name }}</option>
              <option v-if="Object.keys(profiles).length === 0" disabled value="">— no profiles —</option>
            </select>
            <input
              v-else
              v-model="newBoardName"
              class="board-name-input"
              placeholder="Board name..."
              @keyup.enter="addBoard"
              @keyup.escape="addingBoard = false"
            />
            <button class="icon-btn" title="Add new board" @click="addingBoard ? addBoard() : (addingBoard = true)">
              {{ addingBoard ? '✓' : '+' }}
            </button>
            <button class="icon-btn icon-btn-save" title="Save current selection" :disabled="!selectedBoard" @click="saveProfile">
              ↓
            </button>
          </div>

          <hr class="divider" />

          <!-- Test list -->
          <p class="section-label">Tests</p>
          <div class="test-list">
            <label
              v-for="(test, idx) in tests"
              :key="test.name"
              class="test-item"
              :class="{ checked: selected[idx] }"
            >
              <input v-model="selected[idx]" type="checkbox" />
              <span class="test-name">{{ test.name }}</span>
              <span class="run-badge" :class="runStates[test.name] ?? 'idle'">
                {{ runStates[test.name] ?? 'idle' }}
              </span>
            </label>
          </div>

          <hr class="divider" />

          <!-- Action buttons -->
          <div class="action-row">
            <button
              class="ghost-button small"
              type="button"
              :disabled="running || !hasSelection"
              @click="runSingle(tests.find((_, i) => selected[i])?.name ?? '')"
            >
              Run One
            </button>
            <button
              class="primary-button small"
              type="button"
              :disabled="running || !hasSelection"
              @click="runSelected"
            >
              {{ running ? 'Running…' : 'Run All Selected' }}
            </button>
          </div>
        </div>

        <!-- ── RIGHT: output panel ── -->
        <div class="output-col">

          <!-- Progress summary -->
          <div class="panel progress-panel">
            <p class="section-label">Overall Progress</p>
            <div class="stat-row">
              <div class="stat">
                <span class="stat-val">{{ totalDone }} / {{ totalSelected }}</span>
                <span class="stat-lbl">Tests done</span>
              </div>
              <div class="stat">
                <span class="stat-val pass-color">{{ totalPass }}</span>
                <span class="stat-lbl">Passed</span>
              </div>
              <div class="stat">
                <span class="stat-val fail-color">{{ totalFail }}</span>
                <span class="stat-lbl">Failed</span>
              </div>
            </div>
            <div class="progress-track">
              <div class="progress-fill" :style="{ width: progressPct + '%' }" />
            </div>
            <p class="progress-pct">{{ progressPct }}%</p>
          </div>

          <!-- Output log -->
          <div class="panel log-panel">
            <p class="section-label">Output Log</p>
            <div v-if="tests.filter((_, i) => selected[i]).length === 0" class="log-empty">
              No tests selected.
            </div>
            <div v-else class="log-list">
              <div
                v-for="test in tests.filter((_, i) => selected[i])"
                :key="test.name"
                class="log-item"
              >
                <div class="log-header">
                  <span class="run-badge" :class="runStates[test.name] ?? 'idle'">
                    {{ runStates[test.name] ?? 'idle' }}
                  </span>
                  <span class="log-name">{{ test.name }}</span>
                  <span v-if="runTimes[test.name] != null" class="log-time">
                    {{ (runTimes[test.name] / 1000).toFixed(1) }}s
                  </span>
                  <span v-else class="log-time">—</span>
                  <!-- Run single button inside log row -->
                  <button
                    class="run-one-btn"
                    :disabled="running"
                    @click="runSingle(test.name)"
                  >▶</button>
                </div>
                <template v-if="results.find(r => r.name === test.name) as logResult">
                  <pre v-if="logResult.data.stdout" class="log-body stdout">{{ logResult.data.stdout }}</pre>
                  <pre v-if="logResult.data.stderr"  class="log-body stderr">{{ logResult.data.stderr }}</pre>
                </template>
              </div>
            </div>
          </div>

        </div>
      </div>

      <div v-if="tests.length === 0 && !error" class="panel status-panel">
        <p>No tests found.</p>
      </div>
    </div>
  </div>
</template>
```

> **Note on `v-for` + `template`:** The `results.find(...)` pattern inside the `<template v-if>` block requires Vue 3.3+. If your Vue version is older, use a computed map instead:
> ```typescript
> const resultMap = computed(() =>
>   Object.fromEntries(results.value.map(r => [r.name, r]))
> )
> ```
> Then replace `results.find(r => r.name === test.name)` with `resultMap[test.name]`.

---

### 9. Replace the entire `<style scoped>` block

Keep all existing styles and add the following new rules. The easiest approach is to append them at the end of the existing `<style scoped>` block.

```css
/* ── Layout ── */
.two-col {
  display: grid;
  grid-template-columns: 260px 1fr;
  gap: 16px;
  align-items: start;
}

.output-col {
  display: grid;
  gap: 16px;
}

/* ── Control panel ── */
.control-panel {
  display: grid;
  gap: 0;
}

.section-label {
  font-size: 0.72rem;
  font-weight: 700;
  letter-spacing: 0.08em;
  text-transform: uppercase;
  color: #64748b;
  margin-bottom: 8px;
}

.board-row {
  display: flex;
  gap: 6px;
  align-items: center;
  margin-bottom: 12px;
}

.board-row select,
.board-name-input {
  flex: 1;
  min-width: 0;
  border: 1px solid #cbd5e1;
  border-radius: 10px;
  padding: 8px 10px;
  font: inherit;
  font-size: 0.9rem;
  background: #fff;
}

.icon-btn {
  flex-shrink: 0;
  width: 34px;
  height: 34px;
  border-radius: 10px;
  border: 1px solid #cbd5e1;
  background: #fff;
  font-size: 1rem;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
}

.icon-btn-save {
  border-color: #0f766e;
  color: #0f766e;
}

.divider {
  border: none;
  border-top: 1px solid rgba(148, 163, 184, 0.22);
  margin: 12px 0;
}

/* ── Test list ── */
.test-list {
  display: grid;
  gap: 6px;
}

.test-item {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 8px 10px;
  border-radius: 10px;
  border: 1px solid rgba(148, 163, 184, 0.28);
  background: rgba(255, 255, 255, 0.6);
  cursor: pointer;
  user-select: none;
}

.test-item.checked {
  border-color: rgba(14, 165, 233, 0.4);
  background: rgba(224, 242, 254, 0.5);
}

.test-name {
  flex: 1;
  font-size: 0.9rem;
  font-family: Consolas, 'Courier New', monospace;
}

/* ── Run state badges ── */
.run-badge {
  display: inline-flex;
  align-items: center;
  border-radius: 999px;
  padding: 2px 9px;
  font-size: 0.75rem;
  font-weight: 700;
  text-transform: uppercase;
}

.run-badge.idle    { background: #f1f5f9; color: #64748b; }
.run-badge.running { background: #fef3c7; color: #92400e; }
.run-badge.pass    { background: #dcfce7; color: #166534; }
.run-badge.fail,
.run-badge.error   { background: #fee2e2; color: #991b1b; }

/* ── Action row ── */
.action-row {
  display: flex;
  gap: 8px;
  justify-content: flex-end;
  flex-wrap: wrap;
}

.primary-button.small,
.ghost-button.small {
  padding: 8px 14px;
  font-size: 0.88rem;
}

/* ── Progress panel ── */
.progress-panel {
  display: grid;
  gap: 10px;
}

.stat-row {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 10px;
}

.stat {
  background: rgba(248, 250, 252, 0.9);
  border-radius: 12px;
  padding: 10px 14px;
  border: 1px solid rgba(148, 163, 184, 0.2);
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.stat-val {
  font-size: 1.4rem;
  font-weight: 700;
  color: #1f2937;
}

.stat-val.pass-color { color: #166534; }
.stat-val.fail-color { color: #991b1b; }

.stat-lbl {
  font-size: 0.78rem;
  color: #64748b;
}

.progress-track {
  height: 6px;
  background: #e2e8f0;
  border-radius: 999px;
  overflow: hidden;
}

.progress-fill {
  height: 100%;
  background: linear-gradient(90deg, #0284c7, #0f766e);
  border-radius: 999px;
  transition: width 0.4s ease;
}

.progress-pct {
  font-size: 0.8rem;
  color: #64748b;
  text-align: right;
}

/* ── Output log ── */
.log-panel {
  display: grid;
  gap: 10px;
}

.log-empty {
  font-size: 0.9rem;
  color: #94a3b8;
  text-align: center;
  padding: 16px 0;
}

.log-list {
  display: grid;
  gap: 10px;
}

.log-item {
  border: 1px solid rgba(148, 163, 184, 0.28);
  border-radius: 12px;
  overflow: hidden;
}

.log-header {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 9px 12px;
  background: rgba(248, 250, 252, 0.9);
}

.log-name {
  flex: 1;
  font-size: 0.9rem;
  font-weight: 700;
  font-family: Consolas, 'Courier New', monospace;
}

.log-time {
  font-size: 0.8rem;
  color: #64748b;
  min-width: 36px;
  text-align: right;
}

.run-one-btn {
  border: none;
  background: transparent;
  color: #64748b;
  cursor: pointer;
  font-size: 0.8rem;
  padding: 2px 4px;
  border-radius: 6px;
  transition: color 0.15s, background 0.15s;
}

.run-one-btn:hover:not(:disabled) {
  color: #0f766e;
  background: rgba(15, 118, 110, 0.08);
}

.run-one-btn:disabled {
  opacity: 0.4;
  cursor: not-allowed;
}

.log-body {
  padding: 10px 14px;
  font-family: Consolas, 'Courier New', monospace;
  font-size: 0.85rem;
  white-space: pre-wrap;
  overflow-x: auto;
  margin: 0;
  line-height: 1.6;
}

.log-body.stdout { background: #eff6ff; color: #1e3a8a; }
.log-body.stderr { background: #fef2f2; color: #991b1b; }

/* ── Responsive ── */
@media (max-width: 820px) {
  .two-col {
    grid-template-columns: 1fr;
  }
}
```

---

## Summary of all changed symbols

| Symbol | File | Change type | Reason |
|--------|------|-------------|--------|
| `EngineApi` | TestRunner.vue | Modified | Add `loadProfiles` / `saveProfiles` methods |
| `RunResult` | TestRunner.vue | Modified | Add optional `durationMs` field |
| `profiles` | TestRunner.vue | New ref | Stores board → test-name mapping |
| `selectedBoard` | TestRunner.vue | New ref | Currently selected board name |
| `newBoardName` | TestRunner.vue | New ref | Input value for adding a new board |
| `addingBoard` | TestRunner.vue | New ref | Toggles name input visibility |
| `runStates` | TestRunner.vue | New ref | Per-test idle/running/pass/fail |
| `runTimes` | TestRunner.vue | New ref | Per-test elapsed ms |
| `totalSelected` | TestRunner.vue | New computed | Count of checked tests |
| `totalDone` | TestRunner.vue | New computed | Count of finished tests |
| `totalPass` | TestRunner.vue | New computed | Count of passing tests |
| `totalFail` | TestRunner.vue | New computed | Count of failing tests |
| `progressPct` | TestRunner.vue | New computed | Percentage 0–100 |
| `loadProfiles()` | TestRunner.vue | New function | Fetches profiles from main process |
| `applyBoardProfile()` | TestRunner.vue | New function | Syncs checkboxes to selected board |
| `saveProfile()` | TestRunner.vue | New function | Persists current checkbox state |
| `addBoard()` | TestRunner.vue | New function | Adds a new board name entry |
| `runSelected()` | TestRunner.vue | Modified | Now updates `runStates` and `runTimes` |
| `runSingle()` | TestRunner.vue | New function | Runs one test by name |
| `loadTests()` | TestRunner.vue | Modified | Now calls `loadProfiles()` after load |
| `<template>` | TestRunner.vue | Replaced | Two-column layout |
| `<style>` | TestRunner.vue | Appended | New layout and component styles |
| `engineApi` | electron-preload/index.ts | Modified | Expose `loadProfiles` / `saveProfiles` |
| `profiles-load` handler | electron-main/index.ts | New | Read `board-profiles.json` |
| `profiles-save` handler | electron-main/index.ts | New | Write `board-profiles.json` |
