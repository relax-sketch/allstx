# LittleWhiteBox Three Extension Refactor Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 将 `LittleWhiteBox`、`vectors-enhanced`、`WestWorld` 合并为一个以 `LittleWhiteBox` 为壳的 SillyTavern 扩展，只保留 ena-planner 依赖链、vectors 强化文件检索、WestWorld 导演/演员系列，并统一通过预设条目控制注入位置。

**Architecture:** `LittleWhiteBox` 继续作为唯一扩展根目录和唯一 manifest，新增 `modules/vectors/` 与 `modules/director/` 两个模块。抽象 `core/preset-entry-registry.js` 作为通用 PromptManager 命名条目服务，`vectorsResults` 与 `westworldDirector` 都写入预设条目，用户通过预设顺序控制位置。保留三套设置 namespace：`LittleWhiteBox`、`vectors_enhanced`、`westworld`，不做一次性迁移。

**Tech Stack:** Native ES modules, SillyTavern extension API, PromptManager, Node `--test`, npm scripts in `LittleWhiteBox/package.json`, static import checks, encoding byte checks.

---

## Source Context

- 架构来源：`重构落地方案.md`
- 聊天记录来源：`d--Github-All-stx---allinone.md`
- 已确认决策：
  - `LittleWhiteBox` 只保留 ena-planner 依赖链和壳层必需基础设施。
  - `vectors-enhanced` 保留强化文件/向量检索，删除楼层消息总结。
  - `WestWorld` 保留导演切拍/演员系列，删除世界书与去重。
  - 注入统一为预设条目：`vectorsResults` 与 `westworldDirector`。
  - 不根据 PowerShell 乱码判断文件损坏；涉及中文文件时以 UTF-8 字节读取和 BOM 检查为准。
  - 不做浏览器验证。

## File Map

**Create**
- `LittleWhiteBox/core/preset-entry-registry.js`: 通用 PromptManager 命名条目服务。
- `LittleWhiteBox/core/preset-entry-registry.test.js`: registry 的纯单元测试。
- `LittleWhiteBox/modules/vectors/`: 从 `vectors-enhanced/` 迁入并精简后的 vectors 模块。
- `LittleWhiteBox/modules/director/`: 从 `WestWorld/` 迁入并精简后的 director 模块。
- `LittleWhiteBox/scripts/refactor-audit.mjs`: 依赖闭包、待删除文件、BOM、乱码扫描脚本。
- `LittleWhiteBox/scripts/check-extension-imports.mjs`: 静态 import 路径存在性检查。

**Modify**
- `LittleWhiteBox/index.js`: 移除非 ena-planner 依赖链模块 import/init/settings 绑定；接入 `initVectors`、`initDirector`；保留 `story-summary`、`story-outline`、`streaming-generation`、`variables/var-commands.js`、`iframe-renderer`。
- `LittleWhiteBox/manifest.json`: 保持唯一 `generate_interceptor`，函数名继续为 `xiaobaixGenerateInterceptor`，由 vectors 模块安装。
- `LittleWhiteBox/settings.html`: 移除已删除模块的设置块，新增 vectors/director 设置入口或 tab 容器。
- `LittleWhiteBox/style.css`: 合并必要的 `vectors-enhanced/style.css` 与 `WestWorld/style.css`，保持原前缀，避免全局重命名。
- `LittleWhiteBox/package.json`: 增加 refactor/audit/import-check/test 脚本。
- `LittleWhiteBox/core/server-storage.js`: 只删除本次移除模块专用 storage export，保留 `EnaPlannerStorage`、`CommonSettingStorage`、`VectorStorage` 以及 story-summary/story-outline 依赖。
- `LittleWhiteBox/bridges/call-generate-service.js`: 保持 `window.WestWorld.*` 兼容契约，必要时把 director 模块内部 API 挂到同名全局。

**Do Not Modify Unless The Task Says So**
- `SillyTavern/`
- `JS-Slash-Runner/`
- 根目录参考文件 `预设下载与预览与设置.js`、`明月秋青脚本-v0.2.0-fix10.json`
- `WestWorld/txtToWorldbook/core/constants.js` 的 BOM 状态；如果后续必须修改该文件，保留 UTF-8 BOM。

---

## Task 1: Add Audit And Import-Check Scripts

**Files:**
- Create: `LittleWhiteBox/scripts/refactor-audit.mjs`
- Create: `LittleWhiteBox/scripts/check-extension-imports.mjs`
- Modify: `LittleWhiteBox/package.json`

- [ ] **Step 1: Create the refactor audit script**

Create `LittleWhiteBox/scripts/refactor-audit.mjs` with this behavior:
- Scan the full `allinone` workspace for reporting so sibling projects remain visible during the refactor.
- Classify files using the `KEEP_HINTS` / `DROP_HINTS` lists.
- Preserve BOM detection in the JSON output.
- Split replacement-character findings into:
  - `scopeGarbled`: files under `keep` or `drop`
  - `reviewGarbled`: files under `review`
- Fail only when `scopeGarbled.length > 0`.
- Keep reporting `reviewGarbled` without failing on unrelated vendored or parent-level files.
- Use `fileURLToPath(import.meta.url)` for path resolution on Windows.

Implementation skeleton:

```js
/* eslint-env node */
import fs from 'node:fs';
import path from 'node:path';
import { fileURLToPath } from 'node:url';

const __filename = fileURLToPath(import.meta.url);
const ROOT = path.resolve(path.dirname(__filename), '..');
const SOURCE_ROOT = path.resolve(ROOT, '..');

const KEEP_HINTS = [
  'LittleWhiteBox/index.js',
  'LittleWhiteBox/manifest.json',
  'LittleWhiteBox/settings.html',
  'LittleWhiteBox/style.css',
  'LittleWhiteBox/core/',
  'LittleWhiteBox/bridges/',
  'LittleWhiteBox/shared/',
  'LittleWhiteBox/widgets/button-collapse.js',
  'LittleWhiteBox/modules/ena-planner/',
  'LittleWhiteBox/modules/story-summary/',
  'LittleWhiteBox/modules/story-outline/',
  'LittleWhiteBox/modules/streaming-generation.js',
  'LittleWhiteBox/modules/iframe-renderer.js',
  'LittleWhiteBox/modules/variables/var-commands.js',
  'LittleWhiteBox/libs/',
];

const DROP_HINTS = [
  'LittleWhiteBox/modules/scheduled-tasks/',
  'LittleWhiteBox/modules/message-preview.js',
  'LittleWhiteBox/modules/immersive-mode.js',
  'LittleWhiteBox/modules/template-editor/',
  'LittleWhiteBox/modules/fourth-wall/',
  'LittleWhiteBox/modules/control-audio.js',
  'LittleWhiteBox/modules/novel-draw/',
  'LittleWhiteBox/modules/tts/',
  'LittleWhiteBox/modules/assistant/',
  'LittleWhiteBox/modules/variables/variables-panel.js',
  'LittleWhiteBox/modules/variables/varevent-editor.js',
  'LittleWhiteBox/modules/variables/state2/',
  'vectors-enhanced/src/core/memory/MemoryService.js',
  'vectors-enhanced/src/ui/components/MemoryUI.js',
];

function toPosix(filePath) {
  return filePath.split(path.sep).join('/');
}

function walk(dir) {
  const out = [];
  for (const entry of fs.readdirSync(dir, { withFileTypes: true })) {
    if (entry.name === '.git' || entry.name === 'node_modules') continue;
    const full = path.join(dir, entry.name);
    if (entry.isDirectory()) out.push(...walk(full));
    else out.push(full);
  }
  return out;
}

function readUtf8(file) {
  const bytes = fs.readFileSync(file);
  return {
    bytes,
    text: bytes.toString('utf8'),
    bom: bytes.length >= 3 && bytes[0] === 0xef && bytes[1] === 0xbb && bytes[2] === 0xbf,
  };
}

function classify(relative) {
  if (KEEP_HINTS.some((hint) => relative === hint || relative.startsWith(hint))) return 'keep';
  if (DROP_HINTS.some((hint) => relative === hint || relative.startsWith(hint))) return 'drop';
  return 'review';
}

const files = walk(SOURCE_ROOT)
  .filter((file) => /\.(js|mjs|json|html|css|md|txt)$/i.test(file))
  .map((file) => {
    const relative = toPosix(path.relative(SOURCE_ROOT, file));
    const { text, bom } = readUtf8(file);
    return {
      relative,
      classification: classify(relative),
      bom,
      replacementChars: (text.match(/\uFFFD/g) || []).length,
      bytes: fs.statSync(file).size,
    };
  });

const scopeGarbled = files.filter((file) => file.replacementChars > 0 && file.classification !== 'review');
const reviewGarbled = files.filter((file) => file.replacementChars > 0 && file.classification === 'review');
const bomFiles = files.filter((file) => file.bom);
const grouped = files.reduce((acc, file) => {
  acc[file.classification] ||= [];
  acc[file.classification].push(file.relative);
  return acc;
}, {});

console.log(JSON.stringify({
  root: SOURCE_ROOT,
  totalFiles: files.length,
  keepCount: grouped.keep?.length || 0,
  dropCount: grouped.drop?.length || 0,
  reviewCount: grouped.review?.length || 0,
  bomFiles,
  scopeGarbled,
  reviewGarbled,
  dropFiles: grouped.drop || [],
}, null, 2));

if (scopeGarbled.length > 0) {
  process.exitCode = 1;
}
```

- [ ] **Step 2: Create the static import check script**

Create `LittleWhiteBox/scripts/check-extension-imports.mjs` with this behavior:
- Check relative `import ... from`, dynamic `import()`, and `export ... from` specifiers.
- Validate only extension-internal relative imports that resolve inside `LittleWhiteBox/`.
- Skip relative imports that walk out of the extension root toward the SillyTavern host checkout, but report them in a non-failing `skippedExternalImports` list for visibility.
- Walk the tree only once.
- Exit non-zero only when unresolved internal imports remain.

Implementation skeleton:

```js
/* eslint-env node */
import fs from 'node:fs';
import path from 'node:path';
import { fileURLToPath } from 'node:url';

const __filename = fileURLToPath(import.meta.url);
const ROOT = path.resolve(path.dirname(__filename), '..');
const importPattern = /^\s*import\s+(?:[^'"]+\s+from\s+)?['"]([^'"]+)['"];?/gm;
const dynamicImportPattern = /import\(\s*['"]([^'"]+)['"]\s*\)/gm;
const exportFromPattern = /^\s*export\s+(?:[^'"]+?\s+from\s+)?['"]([^'"]+)['"];?/gm;

function walk(dir) {
  const out = [];
  for (const entry of fs.readdirSync(dir, { withFileTypes: true })) {
    if (entry.name === '.git' || entry.name === 'node_modules' || entry.name === 'dist') continue;
    const full = path.join(dir, entry.name);
    if (entry.isDirectory()) out.push(...walk(full));
    else if (/\.(js|mjs)$/i.test(entry.name)) out.push(full);
  }
  return out;
}

function existsImport(fromFile, specifier) {
  const resolved = path.resolve(path.dirname(fromFile), specifier);
  const relativeToRoot = path.relative(ROOT, resolved);
  const isInsideRoot = relativeToRoot && !relativeToRoot.startsWith('..') && !path.isAbsolute(relativeToRoot);
  if (!isInsideRoot) {
    return { kind: 'external' };
  }
  const candidates = [
    resolved,
    `${resolved}.js`,
    `${resolved}.mjs`,
    path.join(resolved, 'index.js'),
    path.join(resolved, 'index.mjs'),
  ];
  return {
    kind: candidates.some((candidate) => fs.existsSync(candidate)) ? 'ok' : 'missing',
  };
}

const files = walk(ROOT);
const failures = [];
const skippedExternalImports = [];
for (const file of files) {
  const text = fs.readFileSync(file, 'utf8');
  for (const pattern of [importPattern, dynamicImportPattern, exportFromPattern]) {
    pattern.lastIndex = 0;
    let match;
    while ((match = pattern.exec(text))) {
      const specifier = match[1];
      if (!specifier.startsWith('.')) continue;
      const result = existsImport(file, specifier);
      if (result.kind === 'external') {
        skippedExternalImports.push({
          file: path.relative(ROOT, file).split(path.sep).join('/'),
          specifier,
        });
      } else if (result.kind === 'missing') {
        failures.push({
          file: path.relative(ROOT, file).split(path.sep).join('/'),
          specifier,
        });
      }
    }
  }
}

if (failures.length) {
  console.error(JSON.stringify({ failures, skippedExternalImports }, null, 2));
  process.exit(1);
}

console.log(JSON.stringify({
  checkedFiles: files.length,
  unresolvedInternalImports: failures.length,
  skippedExternalImports,
}, null, 2));
```

- [ ] **Step 3: Update package scripts**

Modify `LittleWhiteBox/package.json` scripts so the object includes these entries:

```json
"audit:refactor": "node scripts/refactor-audit.mjs",
"check:imports": "node scripts/check-extension-imports.mjs",
"test:preset-entry-registry": "node --test core/preset-entry-registry.test.js",
"test:refactor": "npm run audit:refactor && npm run check:imports && npm run test:preset-entry-registry && npm run test:ena-planner && npm run test:story-summary:runtime"
```

Keep the existing script entries. Do not remove `lint`, `test:ena-planner`, or `test:story-summary:runtime`.

- [ ] **Step 4: Run the new audit**

Run:

```powershell
npm run audit:refactor
```

Expected:

```text
JSON prints totalFiles, keepCount, dropCount, reviewCount, bomFiles, scopeGarbled, reviewGarbled, dropFiles.
Exit code 0.
scopeGarbled is [].
WestWorld/txtToWorldbook/core/constants.js appears in bomFiles with bom true.
```

- [ ] **Step 4b: Run the new import check**

Run:

```powershell
npm run check:imports
```

Expected:

```text
Exit code 0.
JSON prints checkedFiles, unresolvedInternalImports, skippedExternalImports.
unresolvedInternalImports is 0.
skippedExternalImports may be non-empty because LittleWhiteBox source imports SillyTavern host files outside the extension root.
```

- [ ] **Step 5: Commit**

From `LittleWhiteBox/`, run:

```powershell
git status --short
git add scripts/refactor-audit.mjs scripts/check-extension-imports.mjs package.json
git commit -m "chore: add refactor audit scripts"
```

Expected: commit succeeds in the `LittleWhiteBox` repository. If the working tree contains unrelated user changes, add only the three files above.

---

## Task 2: Add Generic Preset Entry Registry

**Files:**
- Create: `LittleWhiteBox/core/preset-entry-registry.js`
- Create: `LittleWhiteBox/core/preset-entry-registry.test.js`

- [ ] **Step 1: Write the failing registry tests**

Create `LittleWhiteBox/core/preset-entry-registry.test.js`:

```js
import test from 'node:test';
import assert from 'node:assert/strict';
import {
  createPresetEntryPrompt,
  ensurePresetEntry,
  setPresetEntryContent,
  clearPresetEntryContent,
  getPresetEntryStatus,
} from './preset-entry-registry.js';

function createPromptManager() {
  const activeCharacter = { id: 'char-a' };
  const serviceSettings = {
    prompts: [{ identifier: 'chatHistory', name: 'Chat History' }],
    prompt_order: [{
      character_id: 'char-a',
      order: [{ identifier: 'chatHistory', enabled: true }],
    }],
  };
  return {
    activeCharacter,
    serviceSettings,
    getPromptOrderForCharacter(character) {
      return serviceSettings.prompt_order.find((entry) => entry.character_id === character.id)?.order || [];
    },
    getPromptOrderEntry(character, identifier) {
      return this.getPromptOrderForCharacter(character).find((entry) => entry.identifier === identifier) || null;
    },
  };
}

test('createPresetEntryPrompt creates an extension prompt entry', () => {
  const prompt = createPresetEntryPrompt({
    identifier: 'vectorsResults',
    name: 'Vectors Results',
    role: 'system',
    content: 'hello',
  });

  assert.equal(prompt.identifier, 'vectorsResults');
  assert.equal(prompt.name, 'Vectors Results');
  assert.equal(prompt.role, 'system');
  assert.equal(prompt.content, 'hello');
  assert.equal(prompt.extension, true);
  assert.equal(prompt.injection_position, 0);
});

test('ensurePresetEntry creates prompt and prompt_order reference after chatHistory', () => {
  const promptManager = createPromptManager();
  const result = ensurePresetEntry(promptManager, {
    identifier: 'vectorsResults',
    name: 'Vectors Results',
  });

  assert.equal(result.ok, true);
  assert.equal(result.changed, true);
  assert.equal(promptManager.serviceSettings.prompts.some((prompt) => prompt.identifier === 'vectorsResults'), true);
  assert.deepEqual(promptManager.serviceSettings.prompt_order[0].order.map((entry) => entry.identifier), [
    'chatHistory',
    'vectorsResults',
  ]);
});

test('setPresetEntryContent updates content without duplicating order entries', () => {
  const promptManager = createPromptManager();

  setPresetEntryContent(promptManager, 'first', {
    identifier: 'vectorsResults',
    name: 'Vectors Results',
  });
  const result = setPresetEntryContent(promptManager, 'second', {
    identifier: 'vectorsResults',
    name: 'Vectors Results',
  });

  const order = promptManager.serviceSettings.prompt_order[0].order;
  assert.equal(result.ok, true);
  assert.equal(result.contentLength, 'second'.length);
  assert.equal(promptManager.serviceSettings.prompts.find((prompt) => prompt.identifier === 'vectorsResults').content, 'second');
  assert.equal(order.filter((entry) => entry.identifier === 'vectorsResults').length, 1);
});

test('clearPresetEntryContent empties content and reports reason', () => {
  const promptManager = createPromptManager();
  setPresetEntryContent(promptManager, 'text', {
    identifier: 'westworldDirector',
    name: 'WestWorld Director',
  });

  const result = clearPresetEntryContent(promptManager, 'no-results', {
    identifier: 'westworldDirector',
    name: 'WestWorld Director',
  });

  assert.equal(result.ok, true);
  assert.equal(result.cleared, true);
  assert.equal(result.reason, 'no-results');
  assert.equal(promptManager.serviceSettings.prompts.find((prompt) => prompt.identifier === 'westworldDirector').content, '');
});

test('getPresetEntryStatus reports missing and ready entries', () => {
  const promptManager = createPromptManager();

  assert.deepEqual(getPresetEntryStatus(promptManager, { identifier: 'vectorsResults' }), {
    ok: true,
    exists: false,
    activeEnabled: false,
    contentLength: 0,
    orderCount: 1,
  });

  setPresetEntryContent(promptManager, 'abc', {
    identifier: 'vectorsResults',
    name: 'Vectors Results',
  });

  assert.deepEqual(getPresetEntryStatus(promptManager, { identifier: 'vectorsResults' }), {
    ok: true,
    exists: true,
    activeEnabled: true,
    contentLength: 3,
    orderCount: 1,
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run:

```powershell
npm run test:preset-entry-registry
```

Expected:

```text
FAIL
Cannot find module .../core/preset-entry-registry.js
```

- [ ] **Step 3: Implement the registry**

Create `LittleWhiteBox/core/preset-entry-registry.js`:

```js
export const DEFAULT_PRESET_ENTRY_INJECTION_POSITION = 0;
export const ABSOLUTE_PRESET_ENTRY_INJECTION_POSITION = 1;
export const DEFAULT_PRESET_ENTRY_DEPTH = 4;
export const DEFAULT_PRESET_ENTRY_ORDER = 100;

function clampInteger(value, fallback, min, max) {
    const parsed = Number(value);
    if (!Number.isFinite(parsed)) return fallback;
    return Math.max(min, Math.min(max, Math.trunc(parsed)));
}

function findPrompt(settings, identifier) {
    return Array.isArray(settings?.prompts)
        ? settings.prompts.find((prompt) => prompt?.identifier === identifier)
        : null;
}

function insertOrderReference(order, identifier, enabled = true) {
    if (!Array.isArray(order)) return false;
    if (order.some((entry) => entry?.identifier === identifier)) return false;

    const reference = { identifier, enabled };
    const chatHistoryIndex = order.findIndex((entry) => entry?.identifier === 'chatHistory');
    if (chatHistoryIndex >= 0) {
        order.splice(chatHistoryIndex + 1, 0, reference);
    } else {
        order.push(reference);
    }
    return true;
}

function ensureOrderReferences(promptManager, identifier) {
    const settings = promptManager?.serviceSettings;
    if (!settings) return { changed: false, activeEnabled: false, orderCount: 0 };

    settings.prompt_order = Array.isArray(settings.prompt_order) ? settings.prompt_order : [];

    let changed = false;
    for (const list of settings.prompt_order) {
        if (!list || !Array.isArray(list.order)) continue;
        changed = insertOrderReference(list.order, identifier, true) || changed;
    }

    if (promptManager?.activeCharacter) {
        let activeOrder = [];
        if (typeof promptManager.getPromptOrderForCharacter === 'function') {
            activeOrder = promptManager.getPromptOrderForCharacter(promptManager.activeCharacter);
        }

        if (!Array.isArray(activeOrder) || activeOrder.length === 0) {
            const characterId = promptManager.activeCharacter.id;
            settings.prompt_order.push({
                character_id: characterId,
                order: [{ identifier, enabled: true }],
            });
            changed = true;
        } else {
            changed = insertOrderReference(activeOrder, identifier, true) || changed;
        }
    }

    const activeEntry = typeof promptManager?.getPromptOrderEntry === 'function'
        ? promptManager.getPromptOrderEntry(promptManager.activeCharacter, identifier)
        : null;

    return {
        changed,
        activeEnabled: activeEntry?.enabled === true,
        orderCount: settings.prompt_order.filter((list) => Array.isArray(list?.order)).length,
    };
}

export function createPresetEntryPrompt(options = {}) {
    const injectionPosition = Number.isFinite(Number(options.injectionPosition))
        ? Number(options.injectionPosition)
        : DEFAULT_PRESET_ENTRY_INJECTION_POSITION;
    const prompt = {
        identifier: options.identifier,
        name: options.name || options.identifier,
        role: options.role || 'system',
        content: String(options.content || ''),
        system_prompt: false,
        position: 0,
        injection_position: injectionPosition,
        injection_trigger: [],
        forbid_overrides: false,
        extension: true,
    };

    if (injectionPosition === ABSOLUTE_PRESET_ENTRY_INJECTION_POSITION) {
        prompt.injection_depth = clampInteger(options.depth, DEFAULT_PRESET_ENTRY_DEPTH, 0, 999);
        prompt.injection_order = clampInteger(options.order, DEFAULT_PRESET_ENTRY_ORDER, -10000, 10000);
    }

    return prompt;
}

export function ensurePresetEntry(promptManager, options = {}) {
    const settings = promptManager?.serviceSettings;
    const identifier = String(options.identifier || '');
    if (!identifier) return { ok: false, reason: 'identifier-missing' };
    if (!settings || typeof settings !== 'object') {
        return { ok: false, reason: 'prompt-manager-settings-missing' };
    }

    settings.prompts = Array.isArray(settings.prompts) ? settings.prompts : [];

    const injectionPosition = Number.isFinite(Number(options.injectionPosition))
        ? Number(options.injectionPosition)
        : DEFAULT_PRESET_ENTRY_INJECTION_POSITION;
    let changed = false;
    let prompt = findPrompt(settings, identifier);

    if (!prompt) {
        prompt = createPresetEntryPrompt({
            ...options,
            identifier,
            injectionPosition,
        });
        settings.prompts.push(prompt);
        changed = true;
    } else {
        const updates = {
            name: options.name || prompt.name || identifier,
            role: options.role || 'system',
            system_prompt: false,
            extension: true,
        };
        Object.entries(updates).forEach(([key, value]) => {
            if (prompt[key] !== value) {
                prompt[key] = value;
                changed = true;
            }
        });
        if (!Number.isFinite(Number(prompt.injection_position))) {
            prompt.injection_position = injectionPosition;
            changed = true;
        }
        if (options.content !== undefined && prompt.content !== String(options.content || '')) {
            prompt.content = String(options.content || '');
            changed = true;
        }
    }

    const orderResult = ensureOrderReferences(promptManager, identifier);
    changed = orderResult.changed || changed;

    return {
        ok: true,
        changed,
        prompt,
        identifier,
        activeEnabled: orderResult.activeEnabled,
        orderCount: orderResult.orderCount,
    };
}

export function setPresetEntryContent(promptManager, content = '', options = {}) {
    const ensured = ensurePresetEntry(promptManager, options);
    if (!ensured.ok) return ensured;

    const value = String(content || '');
    const changed = ensured.prompt.content !== value;
    ensured.prompt.content = value;

    return {
        ...ensured,
        changed: ensured.changed || changed,
        contentLength: value.length,
    };
}

export function clearPresetEntryContent(promptManager, reason = '', options = {}) {
    const result = setPresetEntryContent(promptManager, '', options);
    return {
        ...result,
        cleared: result.ok === true,
        reason: String(reason || ''),
    };
}

export function getPresetEntryStatus(promptManager, options = {}) {
    const settings = promptManager?.serviceSettings;
    const identifier = String(options.identifier || '');
    if (!identifier) return { ok: false, reason: 'identifier-missing' };
    if (!settings || typeof settings !== 'object') {
        return { ok: false, reason: 'prompt-manager-settings-missing' };
    }

    const prompt = findPrompt(settings, identifier);
    const activeEntry = typeof promptManager?.getPromptOrderEntry === 'function'
        ? promptManager.getPromptOrderEntry(promptManager.activeCharacter, identifier)
        : null;

    return {
        ok: true,
        exists: !!prompt,
        activeEnabled: activeEntry?.enabled === true,
        contentLength: String(prompt?.content || '').length,
        orderCount: Array.isArray(settings.prompt_order)
            ? settings.prompt_order.filter((list) => Array.isArray(list?.order)).length
            : 0,
    };
}
```

- [ ] **Step 4: Run registry tests**

Run:

```powershell
npm run test:preset-entry-registry
```

Expected:

```text
PASS
```

- [ ] **Step 5: Commit**

```powershell
git add core/preset-entry-registry.js core/preset-entry-registry.test.js package.json
git commit -m "feat: add preset entry registry"
```

---

## Task 3: Trim LittleWhiteBox Shell To Ena Dependency Chain

**Files:**
- Modify: `LittleWhiteBox/index.js`
- Modify: `LittleWhiteBox/settings.html`
- Modify: `LittleWhiteBox/style.css`
- Modify: `LittleWhiteBox/core/server-storage.js`

- [ ] **Step 1: Record current import baseline**

Run:

```powershell
npm run audit:refactor
```

Expected:

```text
Exit code 0.
dropFiles includes scheduled-tasks, message-preview, immersive-mode, template-editor, fourth-wall, control-audio, novel-draw, tts, assistant, variables-panel, varevent-editor, variables/state2.
```

- [ ] **Step 2: Modify `index.js` imports**

In `LittleWhiteBox/index.js`, remove imports for:

```js
import { initTasks } from "./modules/scheduled-tasks/scheduled-tasks.js";
import { initMessagePreview, addHistoryButtonsDebounced } from "./modules/message-preview.js";
import { initImmersiveMode } from "./modules/immersive-mode.js";
import { initTemplateEditor } from "./modules/template-editor/template-editor.js";
import { initFourthWall, fourthWallCleanup } from "./modules/fourth-wall/fourth-wall.js";
import { initVariablesPanel, cleanupVariablesPanel } from "./modules/variables/variables-panel.js";
import { initControlAudio } from "./modules/control-audio.js";
import { initVariablesCore, cleanupVariablesCore } from "./modules/variables/variables-core.js";
import { initVareventEditor, cleanupVareventEditor } from "./modules/variables/varevent-editor.js";
import { initNovelDraw, cleanupNovelDraw } from "./modules/novel-draw/novel-draw.js";
import { initTts, cleanupTts } from "./modules/tts/tts.js";
import { initAssistant, cleanupAssistant } from "./modules/assistant/assistant.js";
```

Keep these imports:

```js
import { extension_settings } from "../../../extensions.js";
import { saveSettingsDebounced, eventSource, event_types, getRequestHeaders } from "../../../../script.js";
import { EXT_ID, extensionFolderPath } from "./core/constants.js";
import { executeSlashCommand } from "./core/slash-command.js";
import { EventCenter } from "./core/event-manager.js";
import { initButtonCollapse } from "./widgets/button-collapse.js";
import { initStreamingGeneration } from "./modules/streaming-generation.js";
import {
    initRenderer,
    cleanupRenderer,
    processExistingMessages,
    clearBlobCaches,
    renderHtmlInIframe,
    shrinkRenderedWindowFull
} from "./modules/iframe-renderer.js";
import { initVarCommands, cleanupVarCommands } from "./modules/variables/var-commands.js";
import "./modules/story-summary/story-summary.js";
import "./modules/story-outline/story-outline.js";
import { initEnaPlanner, cleanupEnaPlanner } from "./modules/ena-planner/ena-planner.js";
```

- [ ] **Step 3: Modify `index.js` default settings**

Replace the `extension_settings[EXT_ID] = extension_settings[EXT_ID] || { ... }` defaults with the reduced shape:

```js
extension_settings[EXT_ID] = extension_settings[EXT_ID] || {
    enabled: true,
    storySummary: { enabled: true },
    storyOutline: { enabled: false },
    enaPlanner: { enabled: false },
    useBlob: false,
    wrapperIframe: true,
    renderEnabled: true,
    maxRenderedMessages: 5,
};
```

If code below still reads `variablesCore`, replace those checks with `true` only where the code path is part of `var-commands`, `streaming-generation`, or `iframe-renderer`. Do not keep the variable panel UI.

- [ ] **Step 4: Modify module init arrays**

Both module init arrays in `LittleWhiteBox/index.js` must reduce to:

```js
const moduleInits = [
    { condition: extension_settings[EXT_ID].enaPlanner?.enabled, init: initEnaPlanner },
    { condition: true, init: initStreamingGeneration },
    { condition: true, init: initButtonCollapse },
];
```

For the later array that uses `settings`, use:

```js
const moduleInits = [
    { condition: settings.enaPlanner?.enabled, init: initEnaPlanner },
    { condition: true, init: initStreamingGeneration },
    { condition: true, init: initButtonCollapse },
];
```

- [ ] **Step 5: Modify settings checkbox bindings**

Reduce the settings binding array to:

```js
const toggleBindings = [
    { id: 'xiaobaix_story_summary_enabled', key: 'storySummary' },
    { id: 'xiaobaix_story_outline_enabled', key: 'storyOutline' },
    { id: 'xiaobaix_ena_planner_enabled', key: 'enaPlanner', init: initEnaPlanner },
];
```

Keep the existing behavior for:

```js
if (key === 'storySummary') {
    $(document).trigger('xiaobaix:storySummary:toggle', [enabled]);
}
if (key === 'storyOutline') {
    $(document).trigger('xiaobaix:storyOutline:toggle', [enabled]);
}
```

- [ ] **Step 6: Remove deleted module storage exports**

In `LittleWhiteBox/core/server-storage.js`, remove these exports if no kept file imports them:

```js
export const TasksStorage = new StorageFile('LittleWhiteBox_Tasks.json');
export const NovelDrawStorage = new StorageFile('LittleWhiteBox_NovelDraw.json', { debounceMs: 800 });
export const AssistantStorage = new StorageFile('LittleWhiteBox_Assistant.json', { debounceMs: 800 });
export const TtsStorage = new StorageFile('LittleWhiteBox_TTS.json', { debounceMs: 800 });
```

Keep:

```js
export const StoryOutlineStorage = new StorageFile('LittleWhiteBox_StoryOutline.json');
export const EnaPlannerStorage = new StorageFile('LittleWhiteBox_EnaPlanner.json', { debounceMs: 800 });
export const CommonSettingStorage = new StorageFile('LittleWhiteBox_CommonSettings.json', { debounceMs: 1000 });
export const VectorStorage = new StorageFile('LittleWhiteBox_Vectors.json', { debounceMs: 3000 });
```

- [ ] **Step 7: Remove settings HTML blocks**

In `LittleWhiteBox/settings.html`, remove controls with these IDs:

```text
scheduled_tasks_enabled
xiaobaix_template_enabled
xiaobaix_immersive_enabled
xiaobaix_fourth_wall_enabled
xiaobaix_audio_enabled
xiaobaix_variables_panel_enabled
xiaobaix_variables_core_enabled
xiaobaix_variables_mode
xiaobaix_novel_draw_enabled
xiaobaix_novel_draw_open_settings
xiaobaix_tts_enabled
xiaobaix_tts_open_settings
xiaobaix_assistant_open_settings
```

Keep controls with these IDs:

```text
xiaobaix_story_summary_enabled
xiaobaix_story_outline_enabled
xiaobaix_ena_planner_enabled
```

- [ ] **Step 8: Run import check**

Run:

```powershell
npm run check:imports
```

Expected:

```text
checked N module files; all relative imports resolve
```

- [ ] **Step 9: Run kept module tests**

Run:

```powershell
npm run test:ena-planner
npm run test:story-summary:runtime
```

Expected:

```text
PASS for ena-planner dice tests.
[story-summary-runtime] finishes without failed status.
```

- [ ] **Step 10: Commit**

```powershell
git add index.js settings.html style.css core/server-storage.js
git commit -m "refactor: trim shell to ena dependency chain"
```

---

## Task 4: Move Vectors Into LittleWhiteBox And Preserve Its Namespace

**Files:**
- Create: `LittleWhiteBox/modules/vectors/**`
- Modify: `LittleWhiteBox/index.js`
- Modify: `LittleWhiteBox/style.css`
- Modify: `LittleWhiteBox/manifest.json`

- [ ] **Step 1: Copy source files**

From repository root, run:

```powershell
Copy-Item -LiteralPath '..\vectors-enhanced\index.js' -Destination 'modules\vectors\index.js'
Copy-Item -LiteralPath '..\vectors-enhanced\settings-modular.html' -Destination 'modules\vectors\settings.html'
Copy-Item -LiteralPath '..\vectors-enhanced\style.css' -Destination 'modules\vectors\style.css'
Copy-Item -LiteralPath '..\vectors-enhanced\src' -Destination 'modules\vectors\src' -Recurse
```

Run from `LittleWhiteBox/`. Expected: files appear under `LittleWhiteBox/modules/vectors/`.

- [ ] **Step 2: Fix vectors SillyTavern import depth**

In `LittleWhiteBox/modules/vectors/index.js`, replace imports that previously targeted SillyTavern with one additional `../` level because the file moved from extension root to `modules/vectors/`.

Use this mapping:

```text
../../../chats.js                  -> ../../../../../chats.js
../../../constants.js              -> ../../../../../constants.js
../../../extensions.js             -> ../../../../../extensions.js
../../../openai.js                 -> ../../../../../openai.js
../../../popup.js                  -> ../../../../../popup.js
../../../slash-commands/           -> ../../../../../slash-commands/
../../../textgen-settings.js       -> ../../../../../textgen-settings.js
../../../world-info.js             -> ../../../../../world-info.js
../../../../script.js              -> ../../../../../../script.js
```

Do not change local `./src/...` imports.

- [ ] **Step 3: Rename the global interceptor installation**

In `LittleWhiteBox/modules/vectors/index.js`, replace:

```js
window['vectors_rearrangeChat'] = rearrangeChat;
```

with:

```js
window.xiaobaixGenerateInterceptor = rearrangeChat;
window.vectors_rearrangeChat = rearrangeChat;
```

Keep `window.vectors_rearrangeChat` during the transition so old references do not break.

- [ ] **Step 4: Export init and cleanup functions**

At the bottom of `LittleWhiteBox/modules/vectors/index.js`, add:

```js
export async function initVectors() {
  window.xiaobaixGenerateInterceptor = rearrangeChat;
  window.vectors_rearrangeChat = rearrangeChat;
}

export function cleanupVectors() {
  if (window.xiaobaixGenerateInterceptor === rearrangeChat) {
    delete window.xiaobaixGenerateInterceptor;
  }
  if (window.vectors_rearrangeChat === rearrangeChat) {
    delete window.vectors_rearrangeChat;
  }
}
```

- [ ] **Step 5: Hook vectors into LittleWhiteBox shell**

In `LittleWhiteBox/index.js`, add:

```js
import { initVectors, cleanupVectors } from "./modules/vectors/index.js";
```

Add the default setting under `extension_settings[EXT_ID]`:

```js
vectors: { enabled: true },
```

Add to both module init arrays before `initEnaPlanner`:

```js
{ condition: extension_settings[EXT_ID].vectors?.enabled, init: initVectors },
```

and:

```js
{ condition: settings.vectors?.enabled, init: initVectors },
```

When disabling vectors from settings, call `cleanupVectors()`.

- [ ] **Step 6: Keep manifest interceptor name**

Verify `LittleWhiteBox/manifest.json` contains:

```json
"generate_interceptor": "xiaobaixGenerateInterceptor"
```

Do not add `vectors_rearrangeChat` to `manifest.json`.

- [ ] **Step 7: Import vectors CSS**

At the end of `LittleWhiteBox/style.css`, append the content of `LittleWhiteBox/modules/vectors/style.css` under this comment:

```css
/* vectors-enhanced module styles */
```

Keep original `.vectors_*` and `#vectors_enhanced_*` selectors.

- [ ] **Step 8: Run import check**

Run:

```powershell
npm run check:imports
```

Expected:

```text
all relative imports resolve
```

- [ ] **Step 9: Commit**

```powershell
git add index.js manifest.json style.css modules/vectors
git commit -m "feat: move vectors into LittleWhiteBox"
```

---

## Task 5: Remove Vectors Floor Summary And Switch Injection To Preset Entry

**Files:**
- Modify: `LittleWhiteBox/modules/vectors/index.js`
- Delete: `LittleWhiteBox/modules/vectors/src/core/memory/MemoryService.js`
- Delete: `LittleWhiteBox/modules/vectors/src/ui/components/MemoryUI.js`
- Modify: `LittleWhiteBox/modules/vectors/src/ui/settingsManager.js`
- Modify: `LittleWhiteBox/modules/vectors/settings.html`

- [ ] **Step 1: Remove memory imports**

In `LittleWhiteBox/modules/vectors/src/ui/settingsManager.js`, remove imports of:

```js
import { MemoryUI } from './components/MemoryUI.js';
import { MemoryService } from '../core/memory/MemoryService.js';
```

Remove the full `initializeMemoryUI()` method and all calls to it.

- [ ] **Step 2: Remove memory settings and summary listener**

In `LittleWhiteBox/modules/vectors/index.js`, remove:

```js
export const MEMORY_EXTENSION_TAG = '4_memory';
```

Remove the `settings.memory` default block.

Remove the full `disableWorldInfoEntries(worldName, entries)` function.

Remove the `document.addEventListener('vectors:vectorize-summary', async (event) => { ... })` block.

- [ ] **Step 3: Delete memory files**

Run:

```powershell
Remove-Item -LiteralPath 'modules\vectors\src\core\memory\MemoryService.js'
Remove-Item -LiteralPath 'modules\vectors\src\ui\components\MemoryUI.js'
```

Expected: only those two files are deleted.

- [ ] **Step 4: Remove memory HTML**

In `LittleWhiteBox/modules/vectors/settings.html`, remove the full block with ID:

```text
vectors_enhanced_memory
```

Remove the button/control with ID:

```text
memory_summarize_btn
```

- [ ] **Step 5: Replace `setExtensionPrompt` injection**

At the top of `LittleWhiteBox/modules/vectors/index.js`, remove `setExtensionPrompt` from the SillyTavern import list and add:

```js
import { promptManager } from "../../../../../openai.js";
import {
  clearPresetEntryContent,
  setPresetEntryContent,
} from "../../core/preset-entry-registry.js";
```

Add constants near `EXTENSION_PROMPT_TAG`:

```js
export const VECTORS_PRESET_ENTRY_IDENTIFIER = 'vectorsResults';
export const VECTORS_PRESET_ENTRY_NAME = 'Vectors Results';
```

Replace the successful injection call:

```js
setExtensionPrompt(
  EXTENSION_PROMPT_TAG,
  insertedText,
  settings.position,
  settings.depth,
  settings.include_wi,
  settings.depth_role,
);
```

with:

```js
setPresetEntryContent(promptManager, insertedText, {
  identifier: VECTORS_PRESET_ENTRY_IDENTIFIER,
  name: VECTORS_PRESET_ENTRY_NAME,
  role: settings.depth_role || 'system',
});
```

Replace the empty-result clear call:

```js
setExtensionPrompt(EXTENSION_PROMPT_TAG, '', settings.position, settings.depth, settings.include_wi, settings.depth_role);
```

with:

```js
clearPresetEntryContent(promptManager, 'vectors-no-results', {
  identifier: VECTORS_PRESET_ENTRY_IDENTIFIER,
  name: VECTORS_PRESET_ENTRY_NAME,
  role: settings.depth_role || 'system',
});
```

- [ ] **Step 6: Run import and syntax checks**

Run:

```powershell
npm run check:imports
node --check modules/vectors/index.js
node --check modules/vectors/src/ui/settingsManager.js
```

Expected:

```text
all relative imports resolve
No output from node --check commands.
```

- [ ] **Step 7: Commit**

```powershell
git add modules/vectors
git commit -m "refactor: inject vectors through preset entry"
```

---

## Task 6: Move Director Into LittleWhiteBox And Preserve WestWorld API Contract

**Files:**
- Create: `LittleWhiteBox/modules/director/**`
- Modify: `LittleWhiteBox/index.js`
- Modify: `LittleWhiteBox/style.css`
- Modify: `LittleWhiteBox/bridges/call-generate-service.js` only if API lookup breaks

- [ ] **Step 1: Copy director source**

From `LittleWhiteBox/`, run:

```powershell
New-Item -ItemType Directory -Force -Path 'modules\director'
Copy-Item -LiteralPath '..\WestWorld\index.js' -Destination 'modules\director\index.js'
Copy-Item -LiteralPath '..\WestWorld\style.css' -Destination 'modules\director\style.css'
Copy-Item -LiteralPath '..\WestWorld\txtToWorldbook' -Destination 'modules\director\txtToWorldbook' -Recurse
```

Expected: `modules/director/index.js` and `modules/director/txtToWorldbook/**` exist.

- [ ] **Step 2: Fix director SillyTavern import depth**

In `LittleWhiteBox/modules/director/index.js`, replace:

```js
import * as scriptApi from '../../../../script.js';
import { extension_settings, renderExtensionTemplateAsync } from '../../../extensions.js';
import { promptManager } from '../../../../scripts/openai.js';
import { INJECTION_POSITION } from '../../../../scripts/PromptManager.js';
```

with:

```js
import * as scriptApi from '../../../../../../script.js';
import { extension_settings, renderExtensionTemplateAsync } from '../../../../../extensions.js';
import { promptManager } from '../../../../../../scripts/openai.js';
import { INJECTION_POSITION } from '../../../../../../scripts/PromptManager.js';
```

In `modules/director/txtToWorldbook/**`, adjust SillyTavern imports by adding two more `../` segments only for imports that leave the extension and point to SillyTavern files. Leave local `./` and `../services/` imports unchanged.

- [ ] **Step 3: Replace director prompt manager service usage**

In `LittleWhiteBox/modules/director/index.js`, replace imports from:

```js
} from './txtToWorldbook/services/directorPromptManagerService.js';
```

with imports from:

```js
} from '../../core/preset-entry-registry.js';
```

Use options:

```js
const DIRECTOR_PRESET_ENTRY = {
  identifier: 'westworldDirector',
  name: 'WestWorld Director',
  role: 'system',
};
```

Map old calls:

```text
ensureDirectorPromptManagerEntry(promptManager, options) -> ensurePresetEntry(promptManager, { ...DIRECTOR_PRESET_ENTRY, ...options })
setDirectorPromptManagerContent(promptManager, content, options) -> setPresetEntryContent(promptManager, content, { ...DIRECTOR_PRESET_ENTRY, ...options })
clearDirectorPromptManagerContent(promptManager, reason, options) -> clearPresetEntryContent(promptManager, reason, { ...DIRECTOR_PRESET_ENTRY, ...options })
getDirectorPromptManagerStatus(promptManager, options) -> getPresetEntryStatus(promptManager, { ...DIRECTOR_PRESET_ENTRY, ...options })
```

- [ ] **Step 4: Export init and cleanup**

At the bottom of `LittleWhiteBox/modules/director/index.js`, ensure these exports exist:

```js
export async function initDirector() {
  await initTxtToWorldbookBridge?.();
  window.WestWorld = window.WestWorld || {};
  window.StoryWeaver = window.WestWorld;
  return window.WestWorld;
}

export function cleanupDirector() {
  if (window.StoryWeaver === window.WestWorld) {
    delete window.StoryWeaver;
  }
  delete window.WestWorld;
}
```

If `initTxtToWorldbookBridge` is already called by the copied file, keep one call path only.

- [ ] **Step 5: Hook director into shell**

In `LittleWhiteBox/index.js`, add:

```js
import { initDirector, cleanupDirector } from "./modules/director/index.js";
```

Add setting default:

```js
director: { enabled: true },
```

Add director to both module init arrays before `initEnaPlanner`:

```js
{ condition: extension_settings[EXT_ID].director?.enabled, init: initDirector },
```

and:

```js
{ condition: settings.director?.enabled, init: initDirector },
```

When disabling director from settings, call `cleanupDirector()`.

- [ ] **Step 6: Import director CSS**

At the end of `LittleWhiteBox/style.css`, append the content of `LittleWhiteBox/modules/director/style.css` under:

```css
/* WestWorld director module styles */
```

Keep `.ttw-*` selectors.

- [ ] **Step 7: Verify public API contract statically**

Run:

```powershell
rg -n "getDirectorPromptForLittleWhiteBox|prepareDirectorPromptForInput|clearPreparedDirector|window\\.WestWorld|window\\.StoryWeaver" modules\director bridges\call-generate-service.js modules\ena-planner\ena-planner.js
```

Expected:

```text
modules/director has getDirectorPromptForLittleWhiteBox exposed through window.WestWorld.
modules/ena-planner/ena-planner.js still calls window.WestWorld or window.StoryWeaver.
bridges/call-generate-service.js still accepts window.WestWorld, window.WestWorldTxtToWorldbook, or window.StoryWeaver.
```

- [ ] **Step 8: Run import and syntax checks**

Run:

```powershell
npm run check:imports
node --check modules/director/index.js
```

Expected:

```text
all relative imports resolve
No syntax errors.
```

- [ ] **Step 9: Commit**

```powershell
git add index.js style.css modules/director bridges/call-generate-service.js
git commit -m "feat: move director into LittleWhiteBox"
```

---

## Task 7: Remove Director Worldbook And Dedup Features

**Files:**
- Modify: `LittleWhiteBox/modules/director/**`
- Delete worldbook/dedup-only files listed below

- [ ] **Step 1: Remove worldbook and dedup service imports**

In `LittleWhiteBox/modules/director/txtToWorldbook/main.js`, remove imports for:

```js
createMergeService
createCategoryPersistenceService
createExportNameService
createMemoryQueueActionsService
createProcessingStateService
createRepairService
createWorldbookRuntimeService
createPackagePolicyService
createFeatureServices
createFeatureServicesConfig
createFeatureBindings
createRerollBridge
createRuntimeActionsFacade
createEntryConfigModals
createHistoryView
createSearchModal
createReplaceModal
createHelpModal
createStartButtonView
createStopButtonView
createWorldbookViewRuntime
```

Keep imports for:

```js
createDirectorService
createDirectorTelemetryService
createChapterExperienceView
createPromptEditorView
Logger
createErrorHandler
ModalFactory
EventDelegate
```

- [ ] **Step 2: Delete worldbook/dedup service files**

Run from `LittleWhiteBox/`:

```powershell
$files = @(
  'modules\director\txtToWorldbook\services\worldbookService.js',
  'modules\director\txtToWorldbook\services\worldbookRuntimeService.js',
  'modules\director\txtToWorldbook\services\parserService.js',
  'modules\director\txtToWorldbook\services\exportFormatService.js',
  'modules\director\txtToWorldbook\services\exportNameService.js',
  'modules\director\txtToWorldbook\services\importMergeService.js',
  'modules\director\txtToWorldbook\services\importExportService.js',
  'modules\director\txtToWorldbook\services\categoryPersistenceService.js',
  'modules\director\txtToWorldbook\services\categoryLightService.js',
  'modules\director\txtToWorldbook\services\entryConfigService.js',
  'modules\director\txtToWorldbook\services\rerollService.js',
  'modules\director\txtToWorldbook\services\repairService.js',
  'modules\director\txtToWorldbook\services\taskStateService.js',
  'modules\director\txtToWorldbook\services\apiService.js',
  'modules\director\txtToWorldbook\services\promptService.js',
  'modules\director\txtToWorldbook\services\processingStateService.js',
  'modules\director\txtToWorldbook\services\processingService.js',
  'modules\director\txtToWorldbook\services\memoryQueueActionsService.js',
  'modules\director\txtToWorldbook\services\replaceAndCleanService.js',
  'modules\director\txtToWorldbook\services\nameNormalizationService.js',
  'modules\director\txtToWorldbook\services\mergeService.js',
  'modules\director\txtToWorldbook\services\mergeWorkflowService.js'
)
foreach ($file in $files) {
  if (Test-Path -LiteralPath $file) { Remove-Item -LiteralPath $file }
}
```

- [ ] **Step 3: Delete worldbook/dedup UI files**

Run:

```powershell
$files = @(
  'modules\director\txtToWorldbook\ui\worldbookView.js',
  'modules\director\txtToWorldbook\ui\categoryEditorModal.js',
  'modules\director\txtToWorldbook\ui\categoryListView.js',
  'modules\director\txtToWorldbook\ui\defaultEntriesView.js',
  'modules\director\txtToWorldbook\ui\entryConfigModals.js',
  'modules\director\txtToWorldbook\ui\historyView.js',
  'modules\director\txtToWorldbook\ui\rerollModals.js',
  'modules\director\txtToWorldbook\ui\searchModal.js',
  'modules\director\txtToWorldbook\ui\replaceModal.js',
  'modules\director\txtToWorldbook\ui\helpModal.js',
  'modules\director\txtToWorldbook\ui\apiModeView.js',
  'modules\director\txtToWorldbook\ui\modelActionsView.js',
  'modules\director\txtToWorldbook\ui\messageChainView.js',
  'modules\director\txtToWorldbook\ui\progressView.js',
  'modules\director\txtToWorldbook\ui\startButtonView.js',
  'modules\director\txtToWorldbook\ui\stopButtonView.js',
  'modules\director\txtToWorldbook\ui\memoryQueueView.js',
  'modules\director\txtToWorldbook\ui\chapterRegexView.js',
  'modules\director\txtToWorldbook\ui\mergeModals.js'
)
foreach ($file in $files) {
  if (Test-Path -LiteralPath $file) { Remove-Item -LiteralPath $file }
}
```

- [ ] **Step 4: Stop director from reading worldbook memory queue**

In kept director state code, replace reads of:

```js
state.memory.queue
```

with a director-owned slice:

```js
state.experience.chapterScript.beats
```

and:

```js
state.experience.currentBeatIndex
```

The final director state object must keep these fields:

```js
experience: {
  chapterOutline: '',
  chapterScript: { beats: [] },
  currentBeatIndex: 0,
},
settings: {
  directorMode: true,
}
```

- [ ] **Step 5: Verify removed files are not imported**

Run:

```powershell
npm run check:imports
rg -n "worldbookService|worldbookRuntimeService|replaceAndCleanService|mergeWorkflowService|memory\\.queue|createWorldbookViewRuntime|createMemoryQueueView" modules\director
```

Expected:

```text
check:imports passes.
rg prints no references to deleted services/UI and no director runtime reads of memory.queue.
```

- [ ] **Step 6: Verify ena/director API names remain**

Run:

```powershell
rg -n "getDirectorPromptForLittleWhiteBox|prepareDirectorPromptForInput|clearPreparedDirector" modules\director modules\ena-planner bridges
```

Expected:

```text
All three public API names are still present.
```

- [ ] **Step 7: Commit**

```powershell
git add modules/director
git commit -m "refactor: keep only director features"
```

---

## Task 8: Add Unified Settings Surface

**Files:**
- Modify: `LittleWhiteBox/settings.html`
- Modify: `LittleWhiteBox/index.js`
- Modify: `LittleWhiteBox/style.css`

- [ ] **Step 1: Add settings tabs**

In `LittleWhiteBox/settings.html`, add this container after the kept LittleWhiteBox enable controls:

```html
<div id="xiaobaix_unified_modules" class="xiaobaix-unified-modules">
  <div class="xiaobaix-module-tabs" role="tablist">
    <button type="button" class="xiaobaix-module-tab active" data-xb-module-tab="ena">Ena Planner</button>
    <button type="button" class="xiaobaix-module-tab" data-xb-module-tab="vectors">Vectors</button>
    <button type="button" class="xiaobaix-module-tab" data-xb-module-tab="director">Director</button>
    <button type="button" class="xiaobaix-module-tab" data-xb-module-tab="preset">Preset Entries</button>
  </div>
  <div class="xiaobaix-module-panel active" data-xb-module-panel="ena">
    <button type="button" id="xiaobaix_ena_planner_open_settings" class="menu_button">Open Ena Planner</button>
  </div>
  <div class="xiaobaix-module-panel" data-xb-module-panel="vectors">
    <label class="checkbox_label">
      <input type="checkbox" id="xiaobaix_vectors_enabled">
      Enable vectors retrieval
    </label>
    <div id="xiaobaix_vectors_settings_mount"></div>
  </div>
  <div class="xiaobaix-module-panel" data-xb-module-panel="director">
    <label class="checkbox_label">
      <input type="checkbox" id="xiaobaix_director_enabled">
      Enable director
    </label>
    <div id="xiaobaix_director_settings_mount"></div>
  </div>
  <div class="xiaobaix-module-panel" data-xb-module-panel="preset">
    <p class="notes">Preset entries are created as <code>vectorsResults</code> and <code>westworldDirector</code>. Position them in the active preset prompt order.</p>
  </div>
</div>
```

- [ ] **Step 2: Add tab behavior**

In `LittleWhiteBox/index.js`, inside the settings initialization function, add:

```js
document.querySelectorAll('[data-xb-module-tab]').forEach((tab) => {
    tab.addEventListener('click', () => {
        const key = tab.getAttribute('data-xb-module-tab');
        document.querySelectorAll('[data-xb-module-tab]').forEach((item) => {
            item.classList.toggle('active', item === tab);
        });
        document.querySelectorAll('[data-xb-module-panel]').forEach((panel) => {
            panel.classList.toggle('active', panel.getAttribute('data-xb-module-panel') === key);
        });
    });
});
```

- [ ] **Step 3: Bind vectors/director toggles**

Add checkbox bindings:

```js
$('#xiaobaix_vectors_enabled')
    .prop('checked', !!settings.vectors?.enabled)
    .on('change', function () {
        settings.vectors ||= {};
        settings.vectors.enabled = !!this.checked;
        saveSettingsDebounced();
        if (settings.vectors.enabled) {
            initVectors();
        } else {
            cleanupVectors();
        }
    });

$('#xiaobaix_director_enabled')
    .prop('checked', !!settings.director?.enabled)
    .on('change', function () {
        settings.director ||= {};
        settings.director.enabled = !!this.checked;
        saveSettingsDebounced();
        if (settings.director.enabled) {
            initDirector();
        } else {
            cleanupDirector();
        }
    });
```

- [ ] **Step 4: Add minimal tab styles**

Append to `LittleWhiteBox/style.css`:

```css
.xiaobaix-unified-modules {
  margin-top: 12px;
}

.xiaobaix-module-tabs {
  display: flex;
  flex-wrap: wrap;
  gap: 6px;
  margin-bottom: 10px;
}

.xiaobaix-module-tab {
  border: 1px solid var(--SmartThemeBorderColor, #666);
  background: var(--SmartThemeBlurTintColor, transparent);
  color: var(--SmartThemeBodyColor, inherit);
  padding: 6px 10px;
  border-radius: 6px;
  cursor: pointer;
}

.xiaobaix-module-tab.active {
  background: var(--SmartThemeQuoteColor, #444);
}

.xiaobaix-module-panel {
  display: none;
}

.xiaobaix-module-panel.active {
  display: block;
}
```

- [ ] **Step 5: Verify syntax**

Run:

```powershell
npm run check:imports
node --check index.js
```

Expected:

```text
all relative imports resolve
No syntax errors.
```

- [ ] **Step 6: Commit**

```powershell
git add settings.html index.js style.css
git commit -m "feat: add unified module settings"
```

---

## Task 9: Delete Non-Kept LittleWhiteBox Modules

**Files:**
- Delete only the audited non-kept files and directories.

- [ ] **Step 1: Verify no kept file imports delete targets**

Run:

```powershell
npm run check:imports
rg -n "scheduled-tasks|message-preview|immersive-mode|template-editor|fourth-wall|control-audio|novel-draw|modules/tts|modules/assistant|variables-panel|varevent-editor|variables/state2" .
```

Expected:

```text
check:imports passes.
rg output is limited to documentation, deleted-file paths, or package audit lists.
No kept runtime file imports these modules.
```

- [ ] **Step 2: Delete modules**

Run from `LittleWhiteBox/`:

```powershell
$targets = @(
  'modules\scheduled-tasks',
  'modules\message-preview.js',
  'modules\immersive-mode.js',
  'modules\template-editor',
  'modules\fourth-wall',
  'modules\control-audio.js',
  'modules\novel-draw',
  'modules\tts',
  'modules\assistant',
  'modules\variables\variables-panel.js',
  'modules\variables\varevent-editor.js',
  'modules\variables\state2'
)
foreach ($target in $targets) {
  if (Test-Path -LiteralPath $target) { Remove-Item -LiteralPath $target -Recurse }
}
```

- [ ] **Step 3: Re-run checks**

Run:

```powershell
npm run check:imports
npm run audit:refactor
npm run test:ena-planner
npm run test:story-summary:runtime
```

Expected:

```text
check:imports passes.
audit:refactor exits 0 with garbled [].
ena-planner tests pass.
story-summary runtime check finishes without failed status.
```

- [ ] **Step 4: Commit**

```powershell
git add -A
git commit -m "refactor: remove non-ena LittleWhiteBox modules"
```

---

## Task 10: Final Static Regression

**Files:**
- No planned source edits unless checks reveal a defect.

- [ ] **Step 1: Run full refactor verification**

Run from `LittleWhiteBox/`:

```powershell
npm run test:refactor
```

Expected:

```text
audit:refactor passes.
check:imports passes.
test:preset-entry-registry passes.
test:ena-planner passes.
test:story-summary:runtime passes.
```

- [ ] **Step 2: Run syntax checks on moved entry points**

Run:

```powershell
node --check index.js
node --check modules/vectors/index.js
node --check modules/director/index.js
node --check bridges/call-generate-service.js
```

Expected:

```text
No output and exit code 0 for each command.
```

- [ ] **Step 3: Confirm manifest and globals**

Run:

```powershell
Get-Content -LiteralPath manifest.json -Raw
rg -n "xiaobaixGenerateInterceptor|vectors_rearrangeChat|window\\.WestWorld|window\\.StoryWeaver|getDirectorPromptForLittleWhiteBox|prepareDirectorPromptForInput" index.js modules bridges
```

Expected:

```text
manifest.json has generate_interceptor xiaobaixGenerateInterceptor.
modules/vectors installs window.xiaobaixGenerateInterceptor.
modules/director exposes window.WestWorld.
ena-planner and call-generate-service still find the WestWorld public API names.
```

- [ ] **Step 4: Check encoding**

Run:

```powershell
npm run audit:refactor
```

Expected:

```text
garbled is [].
Any original BOM file that still exists keeps bom true.
```

- [ ] **Step 5: Commit final fixes**

If Step 1-4 required fixes:

```powershell
git add -A
git commit -m "fix: complete merged extension regression"
```

If no files changed:

```powershell
git status --short
```

Expected:

```text
No output from git status --short, or only unrelated user changes.
```

---

## Self-Review

- Spec coverage:
  - `LittleWhiteBox` as shell: Tasks 3, 4, 6, 8, 9.
  - ena-planner dependency chain retained: Tasks 3, 9, 10.
  - vectors file/vector retrieval retained and floor summary removed: Tasks 4, 5.
  - vectors preset entry insertion and result count compatibility: Task 5 keeps existing query settings and replaces injection destination with `vectorsResults`.
  - WestWorld director/actor retained and worldbook/dedup removed: Tasks 6, 7.
  - no `generate_interceptor` conflict: Tasks 4 and 10.
  - separate namespaces retained: Tasks 4, 6, 8 do not migrate `extension_settings`.
  - encoding safety: Tasks 1 and 10.
  - no browser verification: all verification steps are Node/static/grep-based.

- Placeholder scan:
  - No `TBD`.
  - No `TODO`.
  - No `implement later`.
  - No generic "add validation" without code or command.

- Type/signature consistency:
  - Registry API names are consistent: `ensurePresetEntry`, `setPresetEntryContent`, `clearPresetEntryContent`, `getPresetEntryStatus`.
  - Vectors preset identifier is consistently `vectorsResults`.
  - Director preset identifier is consistently `westworldDirector`.
  - Public director API names remain `getDirectorPromptForLittleWhiteBox`, `prepareDirectorPromptForInput`, `clearPreparedDirector`.
