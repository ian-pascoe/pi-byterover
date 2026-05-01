# Pi ByteRover Port Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Convert the package from an OpenCode plugin into a Pi-only ByteRover extension/package.

**Architecture:** Keep the existing ByteRover bridge behavior, but replace OpenCode plugin hooks with a Pi extension factory. Split Pi-specific concerns into focused modules for config loading, `.brv/.gitignore` bootstrapping, Pi session message formatting, manual tool registration, and event orchestration.

**Tech Stack:** TypeScript ESM, Pi extension API (`@mariozechner/pi-coding-agent`), `typebox`, `@byterover/brv-bridge`, Zod v4, Vitest, Rolldown, pnpm.

---

## File Structure

- Modify `package.json`: remove OpenCode dependencies, add Pi package metadata and peer deps.
- Modify `README.md`: rewrite installation/configuration docs for Pi.
- Modify `src/config.ts`: rename gitignore markers to Pi, keep config schema/defaults.
- Create `src/config-loader.ts`: read `.pi/byterover.json` then `~/.pi/agent/byterover.json` and validate config.
- Create `src/gitignore.ts`: move `.brv/.gitignore` bootstrap helpers out of `index.ts`.
- Replace `src/messages.ts`: Pi session entry formatter/selectors.
- Create `src/tools.ts`: register `brv_recall`, `brv_search`, `brv_persist` Pi tools.
- Replace `src/index.ts`: Pi extension factory and event handlers.
- Replace `src/index.test.ts`: Pi extension integration tests with mocked `ExtensionAPI` and contexts.
- Replace `src/messages.test.ts`: Pi session message helper tests.
- Create `src/config-loader.test.ts`: config precedence tests.
- Keep `src/lru-cache.ts`, `src/lru-cache.test.ts`, `src/recall.ts`, `src/recall.test.ts`.
- Remove OpenCode-only imports/types from all source and tests.

---

### Task 1: Package Manifest and Config Marker Baseline

**Files:**

- Modify: `package.json`
- Modify: `src/config.ts`
- Test: `pnpm typecheck`

- [ ] **Step 1: Update package metadata and dependencies**

Edit `package.json` so the relevant sections match this shape:

```json
{
  "name": "pi-byterover",
  "description": "Pi ByteRover extension",
  "keywords": ["pi-package", "pi", "byterover", "memory"],
  "files": ["dist", "README.md"],
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "import": "./dist/index.js",
      "default": "./dist/index.js"
    }
  },
  "pi": {
    "extensions": ["./dist/index.js"]
  },
  "dependencies": {
    "@byterover/brv-bridge": "^1.1.0",
    "zod": "^4.3.6"
  },
  "peerDependencies": {
    "@mariozechner/pi-coding-agent": "*",
    "@mariozechner/pi-ai": "*",
    "typebox": "*"
  }
}
```

Keep existing scripts, devDependencies, version, repository, license, author, homepage, bugs, and publishConfig unless they conflict with this snippet. Remove `@opencode-ai/plugin` and `@opencode-ai/sdk`.

- [ ] **Step 2: Rename managed gitignore markers**

In `src/config.ts`, replace:

```ts
export const brvGitignoreBeginMarker = "# BEGIN opencode-byterover";
export const brvGitignoreEndMarker = "# END opencode-byterover";
```

with:

```ts
export const brvGitignoreBeginMarker = "# BEGIN pi-byterover";
export const brvGitignoreEndMarker = "# END pi-byterover";
```

- [ ] **Step 3: Run typecheck to confirm expected failures are only OpenCode-related**

Run:

```bash
pnpm typecheck
```

Expected: failures about missing `@opencode-ai/*` and Pi implementation not yet migrated. Do not try to make this pass in Task 1.

- [ ] **Step 4: Commit**

```bash
git add package.json src/config.ts
git commit -m "chore: prepare package for pi extension"
```

---

### Task 2: Config Loader

**Files:**

- Create: `src/config-loader.ts`
- Create: `src/config-loader.test.ts`
- Modify: `src/config.ts`

- [ ] **Step 1: Write config loader tests**

Create `src/config-loader.test.ts`:

```ts
import { mkdir, mkdtemp, writeFile } from "node:fs/promises";
import { tmpdir } from "node:os";
import { join } from "node:path";
import { afterEach, describe, expect, test } from "vitest";
import { configDefaults } from "./config.js";
import { loadConfig } from "./config-loader.js";

const tempDirs: string[] = [];

const tempDir = async () => {
  const dir = await mkdtemp(join(tmpdir(), "pi-byterover-config-"));
  tempDirs.push(dir);
  return dir;
};

afterEach(async () => {
  const { rm } = await import("node:fs/promises");
  await Promise.all(tempDirs.splice(0).map((dir) => rm(dir, { recursive: true, force: true })));
});

describe("loadConfig", () => {
  test("uses defaults when no config files exist", async () => {
    const cwd = await tempDir();
    const home = await tempDir();

    const result = await loadConfig({ cwd, homeDir: home });

    expect(result.success).toBe(true);
    if (result.success) expect(result.config).toMatchObject(configDefaults);
  });

  test("loads global config when project config is absent", async () => {
    const cwd = await tempDir();
    const home = await tempDir();
    await mkdir(join(home, ".pi", "agent"), { recursive: true });
    await writeFile(
      join(home, ".pi", "agent", "byterover.json"),
      JSON.stringify({ autoRecall: false }),
    );

    const result = await loadConfig({ cwd, homeDir: home });

    expect(result.success).toBe(true);
    if (result.success) expect(result.config.autoRecall).toBe(false);
  });

  test("project config overrides global config", async () => {
    const cwd = await tempDir();
    const home = await tempDir();
    await mkdir(join(home, ".pi", "agent"), { recursive: true });
    await mkdir(join(cwd, ".pi"), { recursive: true });
    await writeFile(
      join(home, ".pi", "agent", "byterover.json"),
      JSON.stringify({ autoRecall: false }),
    );
    await writeFile(
      join(cwd, ".pi", "byterover.json"),
      JSON.stringify({ autoRecall: true, brvPath: "/bin/brv" }),
    );

    const result = await loadConfig({ cwd, homeDir: home });

    expect(result.success).toBe(true);
    if (result.success) {
      expect(result.config.autoRecall).toBe(true);
      expect(result.config.brvPath).toBe("/bin/brv");
      expect(result.source).toBe(join(cwd, ".pi", "byterover.json"));
    }
  });

  test("returns validation error for invalid config", async () => {
    const cwd = await tempDir();
    const home = await tempDir();
    await mkdir(join(cwd, ".pi"), { recursive: true });
    await writeFile(
      join(cwd, ".pi", "byterover.json"),
      JSON.stringify({ recallTimeoutMs: "slow" }),
    );

    const result = await loadConfig({ cwd, homeDir: home });

    expect(result.success).toBe(false);
    if (!result.success) expect(result.error.message).toContain("Invalid Byterover configuration");
  });
});
```

- [ ] **Step 2: Run tests and verify they fail**

```bash
pnpm test src/config-loader.test.ts
```

Expected: FAIL because `src/config-loader.ts` does not exist.

- [ ] **Step 3: Implement config loader**

Create `src/config-loader.ts`:

```ts
import { readFile } from "node:fs/promises";
import { homedir } from "node:os";
import { join } from "node:path";
import { ConfigSchema } from "./config.js";

export type LoadConfigOptions = {
  cwd: string;
  homeDir?: string;
};

export type LoadConfigResult =
  | { success: true; config: ReturnType<typeof ConfigSchema.parse>; source?: string }
  | { success: false; error: Error; source?: string };

const readJsonIfExists = async (path: string) => {
  try {
    return await readFile(path, "utf8");
  } catch (error) {
    if (typeof error === "object" && error !== null && "code" in error && error.code === "ENOENT") {
      return undefined;
    }
    throw error;
  }
};

export const loadConfig = async ({
  cwd,
  homeDir = homedir(),
}: LoadConfigOptions): Promise<LoadConfigResult> => {
  const candidates = [
    join(cwd, ".pi", "byterover.json"),
    join(homeDir, ".pi", "agent", "byterover.json"),
  ];

  for (const source of candidates) {
    const raw = await readJsonIfExists(source);
    if (raw === undefined) continue;
    try {
      const parsed = JSON.parse(raw) as unknown;
      return { success: true, config: ConfigSchema.parse(parsed), source };
    } catch (error) {
      const message = error instanceof Error ? error.message : String(error);
      return {
        success: false,
        source,
        error: new Error(`Invalid Byterover configuration in ${source}: ${message}`),
      };
    }
  }

  return { success: true, config: ConfigSchema.parse(undefined) };
};
```

- [ ] **Step 4: Run config loader tests**

```bash
pnpm test src/config-loader.test.ts
```

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add src/config-loader.ts src/config-loader.test.ts
git commit -m "feat: load pi byterover config files"
```

---

### Task 3: Gitignore Bootstrap Module

**Files:**

- Create: `src/gitignore.ts`
- Modify: `src/index.test.ts` later in Task 6, but add focused tests here if extracting from existing tests is convenient.
- Modify: `src/index.ts` only after Task 5.

- [ ] **Step 1: Move gitignore helpers out of `src/index.ts`**

Create `src/gitignore.ts` with the existing helper logic from `src/index.ts`:

```ts
import { access, mkdir, readFile, writeFile } from "node:fs/promises";
import { join } from "node:path";
import {
  brvGitignore,
  brvGitignoreBeginMarker,
  brvGitignoreEndMarker,
  brvGitignoreRules,
} from "./config.js";

const hasCode = (error: unknown, code: string) => {
  return typeof error === "object" && error !== null && "code" in error && error.code === code;
};

const escapeRegExp = (value: string) => value.replace(/[.*+?^${}()|[\]\\]/g, "\\$&");

const managedGitignoreRules = new Set(
  brvGitignoreRules.split("\n").filter((line) => line.length > 0 && !line.startsWith("#")),
);

const managedGitignoreBlock = new RegExp(
  `(?:^|\\r?\\n)${escapeRegExp(brvGitignoreBeginMarker)}[\\s\\S]*?${escapeRegExp(brvGitignoreEndMarker)}\\r?\\n?`,
  "gu",
);

export const normalizeBrvGitignore = (existing: string) => {
  const output: string[] = [];
  let insertedManagedBlock = false;
  let skippingManagedBlock = false;

  const insertManagedBlock = () => {
    if (insertedManagedBlock) return;
    if (output.length > 0 && output[output.length - 1] !== "") output.push("");
    output.push(...brvGitignore.trimEnd().split("\n"));
    insertedManagedBlock = true;
  };

  for (const line of existing
    .replace(managedGitignoreBlock, `\n${brvGitignore}\n`)
    .split(/\r?\n/)) {
    if (line === brvGitignoreBeginMarker) {
      insertManagedBlock();
      skippingManagedBlock = true;
      continue;
    }
    if (skippingManagedBlock) {
      if (line === brvGitignoreEndMarker) skippingManagedBlock = false;
      continue;
    }
    if (line === "# ByteRover generated files" || managedGitignoreRules.has(line)) {
      insertManagedBlock();
      continue;
    }
    output.push(line);
  }

  while (output.length > 0 && output[output.length - 1] === "") output.pop();
  if (!insertedManagedBlock) insertManagedBlock();

  return `${output.join("\n")}\n`;
};

export const ensureBrvGitignore = async (cwd: string) => {
  await access(cwd);
  await mkdir(join(cwd, ".brv"), { recursive: true });

  const gitignorePath = join(cwd, ".brv", ".gitignore");

  try {
    const existing = await readFile(gitignorePath, "utf8");
    const normalized = normalizeBrvGitignore(existing);
    if (existing === normalized) return;
    await writeFile(gitignorePath, normalized, "utf8");
  } catch (error) {
    if (!hasCode(error, "ENOENT")) throw error;
    await writeFile(gitignorePath, brvGitignore, "utf8");
  }
};
```

- [ ] **Step 2: Verify compile errors are limited to old index usage**

```bash
pnpm typecheck
```

Expected: still FAIL due to old OpenCode `src/index.ts`, not due to `src/gitignore.ts`.

- [ ] **Step 3: Commit**

```bash
git add src/gitignore.ts
git commit -m "refactor: isolate byterover gitignore bootstrap"
```

---

### Task 4: Pi Session Message Helpers

**Files:**

- Replace: `src/messages.ts`
- Replace: `src/messages.test.ts`

- [ ] **Step 1: Replace message helper tests**

Replace `src/messages.test.ts` with:

```ts
import { describe, expect, test } from "vitest";
import {
  formatMessages,
  selectMessagesForRecall,
  selectMessagesInTurn,
  type PiSessionMessage,
  turnKey,
} from "./messages.js";

const entry = (id: string, role: "user" | "assistant", text: string): PiSessionMessage => ({
  id,
  role,
  text,
});

describe("Pi message helpers", () => {
  test("formats user and assistant text messages", () => {
    expect(
      formatMessages([entry("u1", "user", " question "), entry("a1", "assistant", " answer ")]),
    ).toBe("[user]: question\n\n[assistant]: answer");
  });

  test("skips empty text messages", () => {
    expect(
      formatMessages([entry("u1", "user", " question "), entry("a1", "assistant", "   ")]),
    ).toBe("[user]: question");
  });

  test("selects latest completed user request", () => {
    const selected = selectMessagesInTurn([
      entry("u1", "user", "old question"),
      entry("a1", "assistant", "old answer"),
      entry("u2", "user", "latest question"),
      entry("a2", "assistant", "latest answer"),
    ]);

    expect(selected.map((item) => item.id)).toEqual(["u2", "a2"]);
    expect(turnKey(selected)).toBe("u2:a2");
  });

  test("selects recent recall messages within turn and character limits", () => {
    const selected = selectMessagesForRecall(
      [
        entry("u1", "user", "old question"),
        entry("a1", "assistant", "old answer"),
        entry("u2", "user", "middle question"),
        entry("a2", "assistant", "middle answer"),
        entry("empty", "assistant", "   "),
        entry("u3", "user", "latest question"),
      ],
      { maxRecallTurns: 2, maxRecallChars: 4096 },
    );

    expect(selected.map((item) => item.id)).toEqual(["u2", "a2", "u3"]);
  });
});
```

- [ ] **Step 2: Run tests and verify failures**

```bash
pnpm test src/messages.test.ts
```

Expected: FAIL because `PiSessionMessage` and Pi helpers do not exist yet.

- [ ] **Step 3: Replace `src/messages.ts`**

Replace `src/messages.ts` with:

```ts
export type PiSessionMessage = {
  id: string;
  role: "user" | "assistant";
  text: string;
};

type TextBlock = { type: "text"; text: string };
type MessageEntry = {
  type: "message";
  id: string;
  message: {
    role: string;
    content?: string | Array<TextBlock | { type: string; [key: string]: unknown }>;
  };
};

const isTextBlock = (value: unknown): value is TextBlock => {
  return (
    typeof value === "object" &&
    value !== null &&
    "type" in value &&
    value.type === "text" &&
    "text" in value &&
    typeof value.text === "string"
  );
};

export const extractPiSessionMessages = (entries: Array<unknown>): PiSessionMessage[] => {
  const messages: PiSessionMessage[] = [];
  for (const entry of entries) {
    const candidate = entry as Partial<MessageEntry>;
    if (candidate.type !== "message" || typeof candidate.id !== "string") continue;
    const role = candidate.message?.role;
    if (role !== "user" && role !== "assistant") continue;

    const content = candidate.message.content;
    const text =
      typeof content === "string"
        ? content
        : Array.isArray(content)
          ? content
              .filter(isTextBlock)
              .map((part) => part.text.trim())
              .filter(Boolean)
              .join("\n")
          : "";

    messages.push({ id: candidate.id, role, text });
  }
  return messages;
};

export const formatMessage = (message: PiSessionMessage) => {
  const text = message.text.trim();
  if (!text) return "";
  return `[${message.role}]: ${text}`;
};

export const formatMessages = (messages: PiSessionMessage[]) =>
  messages.map(formatMessage).filter(Boolean).join("\n\n");

export const turnKey = (messages: PiSessionMessage[]) =>
  messages.map((message) => message.id).join(":");

export const selectMessagesInTurn = (messages: PiSessionMessage[]) => {
  const selected: PiSessionMessage[] = [];
  for (let i = messages.length - 1; i >= 0; i--) {
    const message = messages[i]!;
    selected.unshift(message);
    if (message.role === "user") break;
  }
  return selected;
};

export const selectMessagesForRecall = (
  messages: PiSessionMessage[],
  options: { maxRecallTurns: number; maxRecallChars: number },
) => {
  const selected: PiSessionMessage[] = [];
  let userTurns = 0;
  let charCount = 0;

  for (let i = messages.length - 1; i >= 0; i--) {
    const message = messages[i]!;
    const formatted = formatMessage(message);
    if (!formatted) continue;

    const separatorLength = selected.length === 0 ? 0 : 2;
    const nextCharCount = charCount + separatorLength + formatted.length;
    if (selected.length > 0 && nextCharCount > options.maxRecallChars) break;

    selected.unshift(message);
    charCount = nextCharCount;

    if (message.role === "user") {
      userTurns++;
      if (userTurns >= options.maxRecallTurns) break;
    }
  }

  return selected;
};
```

- [ ] **Step 4: Run tests**

```bash
pnpm test src/messages.test.ts
```

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add src/messages.ts src/messages.test.ts
git commit -m "feat: add pi session message helpers"
```

---

### Task 5: Pi Manual Tools

**Files:**

- Create: `src/tools.ts`
- Test through `src/index.test.ts` in Task 6.

- [ ] **Step 1: Create manual tool module**

Create `src/tools.ts`:

```ts
import { StringEnum } from "@mariozechner/pi-ai";
import type { ExtensionAPI } from "@mariozechner/pi-coding-agent";
import type { BrvBridge, SearchResultItem } from "@byterover/brv-bridge";
import { Type, type Static } from "typebox";
import type { ConfigSchema } from "./config.js";
import { stripEchoedRecallQuery } from "./recall.js";

type Config = ReturnType<typeof ConfigSchema.parse>;
type BridgeFactory = (override?: {
  cwd?: string;
  searchTimeoutMs?: number;
  recallTimeoutMs?: number;
  persistTimeoutMs?: number;
}) => BrvBridge;
type Notify = (variant: "success" | "info" | "warning" | "error", message: string) => void;
type Log = (level: "debug" | "info" | "warn" | "error", message: string) => void;

const timeoutSchema = Type.Optional(
  Type.Number({
    minimum: 1,
    description: "Optional timeout in milliseconds for this ByteRover operation.",
  }),
);

const recallSchema = Type.Object({
  query: Type.String({ minLength: 1, description: "Raw recall query." }),
  timeoutMs: timeoutSchema,
});

const searchSchema = Type.Object({
  query: Type.String({ minLength: 1, description: "Raw search query." }),
  limit: Type.Optional(
    Type.Number({
      minimum: 1,
      maximum: 50,
      description: "Maximum number of results to return, from 1 to 50.",
    }),
  ),
  scope: Type.Optional(
    Type.String({
      minLength: 1,
      description: "Optional ByteRover path prefix to scope search results.",
    }),
  ),
  timeoutMs: timeoutSchema,
});

const persistSchema = Type.Object({
  context: Type.String({ minLength: 1, description: "Raw memory text to persist." }),
  timeoutMs: timeoutSchema,
});

export type BrvRecallInput = Static<typeof recallSchema>;
export type BrvSearchInput = Static<typeof searchSchema>;
export type BrvPersistInput = Static<typeof persistSchema>;

export const formatSearchResults = (
  results: SearchResultItem[],
  totalFound: number,
  message: string,
) => {
  if (results.length === 0) return message || "No ByteRover search results found.";
  const header = `Found ${totalFound} ByteRover ${totalFound === 1 ? "result" : "results"}.`;
  const lines = results.flatMap((result, index) => {
    const details = [
      `score: ${result.score}`,
      result.symbolKind ? `kind: ${result.symbolKind}` : undefined,
      result.backlinkCount === undefined ? undefined : `backlinks: ${result.backlinkCount}`,
    ].filter(Boolean);
    const output = [
      `${index + 1}. ${result.title} (${result.path})`,
      details.length > 0 ? `   ${details.join(", ")}` : undefined,
      `   ${result.excerpt}`,
    ];
    if (result.relatedPaths && result.relatedPaths.length > 0)
      output.push(`   related: ${result.relatedPaths.join(", ")}`);
    return output.filter((line) => line !== undefined);
  });
  return [header, ...lines].join("\n");
};

export const registerManualTools = (input: {
  pi: ExtensionAPI;
  config: Config;
  bridge: BrvBridge;
  createBridge: BridgeFactory;
  log: Log;
  notify: Notify;
}) => {
  const { pi, config, bridge, createBridge, log } = input;

  const ensureBridgeReady = async () => {
    const ready = await bridge.ready();
    if (ready) return true;
    log("warn", "ByteRover bridge not ready for manual recall/search");
    return false;
  };

  pi.registerTool({
    name: "brv_recall",
    label: "ByteRover Recall",
    description: "Recall relevant context from ByteRover memory for a raw query.",
    promptSnippet: "Recall relevant context from ByteRover memory for a raw query.",
    parameters: recallSchema,
    async execute(_toolCallId, params, signal, _onUpdate, ctx) {
      if (!(await ensureBridgeReady()))
        return { content: [{ type: "text", text: "ByteRover bridge is not ready." }] };
      try {
        const recallBridge =
          params.timeoutMs === undefined
            ? bridge
            : createBridge({ cwd: ctx.cwd, recallTimeoutMs: params.timeoutMs });
        const result = await recallBridge.recall(params.query, { cwd: ctx.cwd, signal });
        const content =
          stripEchoedRecallQuery(result.content, params.query) ||
          "No relevant ByteRover context found.";
        return { content: [{ type: "text", text: content }], details: { status: "completed" } };
      } catch (error) {
        const message = error instanceof Error ? error.message : String(error);
        log("error", `Manual ByteRover recall failed: ${message}`);
        return {
          content: [{ type: "text", text: `ByteRover recall failed: ${message}` }],
          details: { status: "error", message },
        };
      }
    },
  });

  pi.registerTool({
    name: "brv_search",
    label: "ByteRover Search",
    description: "Search ByteRover memory for ranked file-level context results.",
    promptSnippet: "Search ByteRover memory for ranked file-level context results.",
    parameters: searchSchema,
    async execute(_toolCallId, params, _signal, _onUpdate, ctx) {
      if (!(await ensureBridgeReady()))
        return { content: [{ type: "text", text: "ByteRover bridge is not ready." }] };
      try {
        const searchBridge =
          params.timeoutMs === undefined
            ? bridge
            : createBridge({ cwd: ctx.cwd, searchTimeoutMs: params.timeoutMs });
        const result = await searchBridge.search(params.query, {
          cwd: ctx.cwd,
          ...(params.limit === undefined ? {} : { limit: params.limit }),
          ...(params.scope === undefined ? {} : { scope: params.scope }),
        });
        return {
          content: [
            {
              type: "text",
              text: formatSearchResults(result.results, result.totalFound, result.message),
            },
          ],
          details: result,
        };
      } catch (error) {
        const message = error instanceof Error ? error.message : String(error);
        log("error", `Manual ByteRover search failed: ${message}`);
        return {
          content: [{ type: "text", text: `ByteRover search failed: ${message}` }],
          details: { status: "error", message },
        };
      }
    },
  });

  pi.registerTool({
    name: "brv_persist",
    label: "ByteRover Persist",
    description: "Persist raw memory text into ByteRover without automatic curation wrapping.",
    promptSnippet: "Persist raw memory text into ByteRover without automatic curation wrapping.",
    parameters: persistSchema,
    async execute(_toolCallId, params, _signal, _onUpdate, ctx) {
      try {
        const persistBridge =
          params.timeoutMs === undefined
            ? bridge
            : createBridge({ cwd: ctx.cwd, persistTimeoutMs: params.timeoutMs });
        const result = await persistBridge.persist(params.context, { cwd: ctx.cwd, detach: true });
        const suffix = result.message ? `: ${result.message}` : "";
        return {
          content: [{ type: "text", text: `ByteRover persist ${result.status}${suffix}` }],
          details: result,
        };
      } catch (error) {
        const message = error instanceof Error ? error.message : String(error);
        log("error", `Manual ByteRover persist failed: ${message}`);
        return {
          content: [{ type: "text", text: `ByteRover persist failed: ${message}` }],
          details: { status: "error", message },
        };
      }
    },
  });
};
```

If `StringEnum` is unused after implementation, do not import it.

- [ ] **Step 2: Typecheck this module once index migration is done**

Do not run typecheck yet if old `src/index.ts` still imports OpenCode. Proceed to Task 6.

- [ ] **Step 3: Commit**

```bash
git add src/tools.ts
git commit -m "feat: add pi byterover manual tools"
```

---

### Task 6: Pi Extension Factory and Event Tests

**Files:**

- Replace: `src/index.ts`
- Replace: `src/index.test.ts`

- [ ] **Step 1: Replace index tests with Pi-focused tests**

Replace `src/index.test.ts` with tests that mock `@byterover/brv-bridge`, create a fake `ExtensionAPI`, call the default extension factory, and trigger registered handlers. Include these helpers at the top:

```ts
import type { ExtensionAPI } from "@mariozechner/pi-coding-agent";
import { mkdir, mkdtemp, readFile, rm, writeFile } from "node:fs/promises";
import { join } from "node:path";
import { tmpdir } from "node:os";
import { beforeEach, describe, expect, test, vi } from "vitest";
import byterover from "./index.js";

const bridgeInstances = vi.hoisted(
  () =>
    [] as Array<{
      config: Record<string, unknown>;
      ready: ReturnType<typeof vi.fn>;
      recall: ReturnType<typeof vi.fn>;
      search: ReturnType<typeof vi.fn>;
      persist: ReturnType<typeof vi.fn>;
    }>,
);

vi.mock("@byterover/brv-bridge", () => {
  class MockBrvBridge {
    config: Record<string, unknown>;
    ready = vi.fn(async () => true);
    recall = vi.fn(async () => ({ content: "remembered context" }));
    search = vi.fn(async () => ({ results: [], totalFound: 0, message: "No matches" }));
    persist = vi.fn(async () => ({ status: "completed", message: "ok" }));
    constructor(config: Record<string, unknown>) {
      this.config = config;
      bridgeInstances.push(this);
    }
  }
  return { BrvBridge: vi.fn(MockBrvBridge) };
});

const createPi = () => {
  const handlers = new Map<string, Function[]>();
  const tools = new Map<string, any>();
  const pi = {
    on: vi.fn((event: string, handler: Function) => {
      handlers.set(event, [...(handlers.get(event) ?? []), handler]);
    }),
    registerTool: vi.fn((tool: any) => {
      tools.set(tool.name, tool);
    }),
  } as unknown as ExtensionAPI;
  return { pi, handlers, tools };
};

const messageEntry = (id: string, role: "user" | "assistant", text: string) => ({
  type: "message",
  id,
  message: { role, content: [{ type: "text", text }] },
});

const createCtx = (cwd: string, branch = [messageEntry("u1", "user", "latest question")]) => ({
  cwd,
  hasUI: true,
  ui: { notify: vi.fn() },
  sessionManager: { getBranch: vi.fn(() => branch) },
});
```

Add tests equivalent to the existing OpenCode tests for: bridge config, disabled config, manual tools registered, gitignore bootstrapping, recall injection, autoRecall disabled, manual tool guidance, manual recall/search/persist, persist not blocked by `ready()`, agent_end curation, compact curation, dedupe.

- [ ] **Step 2: Run tests and verify failure**

```bash
pnpm test src/index.test.ts
```

Expected: FAIL because `src/index.ts` is still OpenCode-based.

- [ ] **Step 3: Replace `src/index.ts` with Pi extension**

Implement `src/index.ts` with this structure:

```ts
import type { ExtensionAPI, ExtensionContext } from "@mariozechner/pi-coding-agent";
import { BrvBridge } from "@byterover/brv-bridge";
import type { BrvLogger } from "@byterover/brv-bridge";
import { ConfigSchema, maxCuratedTurnCacheSize } from "./config.js";
import { loadConfig } from "./config-loader.js";
import { ensureBrvGitignore } from "./gitignore.js";
import { LruCache } from "./lru-cache.js";
import {
  extractPiSessionMessages,
  formatMessages,
  selectMessagesForRecall,
  selectMessagesInTurn,
  turnKey,
} from "./messages.js";
import { stripEchoedRecallQuery } from "./recall.js";
import { registerManualTools } from "./tools.js";

export const buildManualToolGuidance = (config: { autoRecall: boolean; autoPersist: boolean }) => {
  const guidance = [
    "ByteRover memory guidance:",
    `Automatic recall is ${config.autoRecall ? "enabled" : "disabled"}.`,
    `Automatic persist is ${config.autoPersist ? "enabled" : "disabled"}.`,
  ];
  if (config.autoRecall && config.autoPersist) {
    guidance.push(
      "Rely on automatic recall and automatic persist for routine memory behavior instead of consistently calling the manual tools.",
      "Use `brv_recall`, `brv_search`, or `brv_persist` when you need an extra targeted lookup, immediate durable save, or explicit user-requested memory operation.",
    );
  } else {
    guidance.push(
      "Use `brv_recall`, `brv_search`, and `brv_persist` when durable memory is useful because one or more automatic memory behaviors are disabled.",
    );
  }
  return guidance.join("\n");
};

export default function byterover(pi: ExtensionAPI) {
  const curatedTurns = new LruCache<string, string>(maxCuratedTurnCacheSize);
  const inFlightCurations = new Map<string, { key: string; promise: Promise<void> }>();

  pi.on("session_start", async (_event, ctx: ExtensionContext) => {
    const loaded = await loadConfig({ cwd: ctx.cwd });
    const log = (level: "debug" | "info" | "warn" | "error", message: string) => {
      const line = `[byterover] ${level}: ${message}`;
      if (level === "error") console.error(line);
      else console.warn(line);
    };
    if (!loaded.success) {
      if (ctx.hasUI) ctx.ui.notify(loaded.error.message, "error");
      log("error", loaded.error.message);
      return;
    }

    const config = loaded.config;
    if (!config.enabled) return;
    const notify = (variant: "success" | "info" | "warning" | "error", message: string) => {
      if (config.quiet || !ctx.hasUI) return;
      ctx.ui.notify(message, variant);
    };

    try {
      await ensureBrvGitignore(ctx.cwd);
    } catch (error) {
      const message = error instanceof Error ? error.message : String(error);
      notify("warning", "Failed to initialize ByteRover storage, some features may not work");
      log("warn", `Failed to bootstrap .brv/.gitignore: ${message}`);
    }

    const brvLogger: BrvLogger = {
      debug: (message) => log("debug", message),
      info: (message) => log("info", message),
      warn: (message) => log("warn", message),
      error: (message) => log("error", message),
    };
    const createBridge = (override?: {
      cwd?: string;
      searchTimeoutMs?: number;
      recallTimeoutMs?: number;
      persistTimeoutMs?: number;
    }) =>
      new BrvBridge({
        brvPath: config.brvPath ?? "brv",
        searchTimeoutMs: override?.searchTimeoutMs ?? config.searchTimeoutMs,
        recallTimeoutMs: override?.recallTimeoutMs ?? config.recallTimeoutMs,
        persistTimeoutMs: override?.persistTimeoutMs ?? config.persistTimeoutMs,
        cwd: override?.cwd ?? ctx.cwd,
        logger: brvLogger,
      });
    const bridge = createBridge();

    const branchMessages = (eventCtx: ExtensionContext) =>
      extractPiSessionMessages(eventCtx.sessionManager.getBranch());
    const curateTurn = async (eventCtx: ExtensionContext) => {
      if (!config.autoPersist) return;
      const messages = selectMessagesInTurn(branchMessages(eventCtx));
      if (messages.length === 0) return;
      const key = turnKey(messages);
      const sessionKey = eventCtx.sessionManager.getSessionFile?.() ?? eventCtx.cwd;
      if (curatedTurns.get(sessionKey) === key) return;
      const inFlight = inFlightCurations.get(sessionKey);
      if (inFlight?.key === key) {
        await inFlight.promise;
        return;
      }
      const formatted = formatMessages(messages);
      if (!formatted) return;
      const promise = bridge
        .persist(`${config.persistPrompt.trim()}\n\nConversation:\n\n---\n${formatted}`, {
          cwd: eventCtx.cwd,
        })
        .then((result) => {
          if (result.status === "error") {
            notify("error", "Failed to curate conversation turn, see logs for details");
            log("error", `ByteRover process failed: ${result.message}`);
            return;
          }
          curatedTurns.set(sessionKey, key);
        });
      inFlightCurations.set(sessionKey, { key, promise });
      try {
        await promise;
      } finally {
        if (inFlightCurations.get(sessionKey)?.promise === promise)
          inFlightCurations.delete(sessionKey);
      }
    };

    if (config.manualTools) registerManualTools({ pi, config, bridge, createBridge, log, notify });

    pi.on("before_agent_start", async (event, eventCtx: ExtensionContext) => {
      let systemPrompt = event.systemPrompt;
      if (config.manualTools) systemPrompt += `\n\n${buildManualToolGuidance(config)}`;
      if (!config.autoRecall) return { systemPrompt };
      if (!(await bridge.ready())) {
        notify("warning", "ByteRover bridge not ready, skipping recall");
        log("warn", "ByteRover bridge not ready, skipping recall");
        return { systemPrompt };
      }
      const messages = selectMessagesForRecall(branchMessages(eventCtx), config);
      const formatted = formatMessages(messages);
      if (!formatted) return { systemPrompt };
      try {
        const query = `${config.recallPrompt.trim()}\n\nRecent conversation:\n\n---\n${formatted}`;
        const result = await bridge.recall(query, { cwd: eventCtx.cwd });
        const content = stripEchoedRecallQuery(result.content, query);
        if (!content) return { systemPrompt };
        return {
          systemPrompt: `${systemPrompt}\n\n<${config.contextTagName}>\n${content}\n</${config.contextTagName}>`,
        };
      } catch (error) {
        const message = error instanceof Error ? error.message : String(error);
        notify("error", "Failed to recall context from ByteRover, see logs for details");
        log("error", `ByteRover recall failed: ${message}`);
        return { systemPrompt };
      }
    });

    pi.on("agent_end", async (_event, eventCtx: ExtensionContext) => {
      await curateTurn(eventCtx);
    });

    pi.on("session_before_compact", async (_event, eventCtx: ExtensionContext) => {
      await curateTurn(eventCtx);
    });
  });
}
```

Adjust type annotations to match the installed Pi types if TypeScript reports exact event type mismatches.

- [ ] **Step 4: Run targeted tests**

```bash
pnpm test src/index.test.ts src/messages.test.ts src/config-loader.test.ts src/recall.test.ts src/lru-cache.test.ts
```

Expected: PASS after fixing any Pi type/test helper mismatches.

- [ ] **Step 5: Commit**

```bash
git add src/index.ts src/index.test.ts
git commit -m "feat: port byterover extension to pi events"
```

---

### Task 7: README Rewrite

**Files:**

- Replace: `README.md`

- [ ] **Step 1: Rewrite README for Pi**

Replace OpenCode-specific README content with Pi docs covering:

````md
# pi-byterover

ByteRover memory integration for Pi.

## Overview

`pi-byterover` is a Pi package that connects Pi sessions to ByteRover through `@byterover/brv-bridge`. It can automatically recall relevant memory before each agent response and persist useful completed turns after each response.

## Prerequisites

- Pi installed.
- ByteRover CLI installed as `brv`, or configured with `brvPath`.
- A project where ByteRover can bootstrap or use `.brv` state.

## Installation

```bash
pi install npm:pi-byterover
```
````

For local development:

```bash
pnpm install
pnpm build
pi -e ./dist/index.js
```

## Configuration

Create `.pi/byterover.json` in a project, or `~/.pi/agent/byterover.json` globally.

```json
{
  "enabled": true,
  "brvPath": "brv",
  "searchTimeoutMs": 30000,
  "recallTimeoutMs": 30000,
  "persistTimeoutMs": 60000,
  "quiet": false,
  "autoRecall": true,
  "autoPersist": true,
  "manualTools": true,
  "contextTagName": "byterover-context",
  "maxRecallTurns": 3,
  "maxRecallChars": 4096
}
```

Explain all config fields from the existing README. Explicitly state that `persist` does not require `brv ready`; ByteRover bootstraps automatically when persist is called.

## Manual Tools

Document `brv_recall`, `brv_search`, `brv_persist` and when to use each.

## Development

```bash
pnpm install
pnpm format:check
pnpm lint
pnpm test
pnpm typecheck
pnpm build
```

````

- [ ] **Step 2: Commit**

```bash
git add README.md
git commit -m "docs: document pi byterover usage"
````

---

### Task 8: Full Verification and Cleanup

**Files:**

- Modify any files needed to satisfy checks.
- Add `.changeset/*.md` if this user-facing package change will be published.

- [ ] **Step 1: Run formatting check**

```bash
pnpm format:check
```

Expected: PASS. If it fails, run `pnpm format`, inspect changes, then continue.

- [ ] **Step 2: Run lint**

```bash
pnpm lint
```

Expected: PASS. Fix only real issues; do not silence useful errors.

- [ ] **Step 3: Run tests**

```bash
pnpm test
```

Expected: PASS.

- [ ] **Step 4: Run typecheck**

```bash
pnpm typecheck
```

Expected: PASS.

- [ ] **Step 5: Run build**

```bash
pnpm build
```

Expected: PASS and `dist/index.js` generated. Do not commit `dist/` because it is ignored.

- [ ] **Step 6: Add a Changeset for publishing**

Create `.changeset/pi-byterover-port.md`:

```md
---
"pi-byterover": minor
---

Port the ByteRover integration from an OpenCode plugin to a Pi extension package.
```

- [ ] **Step 7: Final commit**

```bash
git add .changeset/pi-byterover-port.md package.json README.md src docs/superpowers/plans/2026-05-01-pi-byterover-port.md
git commit -m "chore: verify pi byterover port"
```

---

## Self-Review

- Spec coverage: The plan covers Pi-only package metadata, `.pi/byterover.json` config loading, Pi events, manual tools, message formatting, persist readiness behavior, README, tests, and verification.
- Placeholder scan: No `TBD`/`TODO` placeholders are present. Each task has commands and expected outcomes.
- Type consistency: `PiSessionMessage`, `loadConfig`, `ensureBrvGitignore`, `registerManualTools`, and the default Pi extension factory are introduced before use in later tasks.
