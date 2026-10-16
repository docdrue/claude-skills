---
name: translate
description: Syncs i18n JSON translation files using en as the source of truth. On first run detects where translations live, asks user to confirm or override, builds app context, then translates missing or changed keys using AI with that context and a glossary. If a locale code is passed that does not exist yet, creates the folder and all JSON files before translating.
version: 1.0.0
user-invocable: true
argument-hint: "[locale] (optional ISO 639-1 code, e.g. pt — defaults to all non-en locales)"
---

# translate

Syncs `$LOCALES_ROOT/<locale>/*.json` files using `$LOCALES_ROOT/en/` as the source of truth. Uses cached translations, a developer-managed glossary, and app context to produce accurate, consistent translations with minimal AI calls.

**File ownership:**
- Claude-managed (generated): `.claude/translate/settings.json`, `.claude/translate/cache.json`, `.claude/translate/context.md`
- Developer-managed (hand-edited): `$LOCALES_ROOT/glossary.yaml`

---

## Step 0 — Detect translations root (first run only)

Check whether `.claude/translate/settings.json` exists and contains a `localesRoot` key.

**If it does NOT exist or has no `localesRoot`:**

1. Scan the entire project for translation files using this approach:
   - Find every directory named `en/` anywhere in the project tree (excluding `node_modules/`, `.git/`, `dist/`, `build/`, `.next/`, `coverage/`)
   - For each `en/` directory found, check whether it contains at least one `.json` file whose top-level values are strings or objects whose leaf values are strings (i.e. a standard i18n translation shape — not a config file, not a schema)
   - A valid translation JSON looks like `{ "key": "value" }` or `{ "section": { "key": "value" } }` — the leaf values must be strings, not numbers, booleans, or arrays
   - The parent of a qualifying `en/` directory is a candidate `localesRoot`

2. **If one match is found:** ask the user:
   "I found translation files at `<detected-path>`. Press Enter to use it, or type a different path:"
   - If the user presses Enter — use the detected path.
   - If the user types a path — use that instead.

3. **If multiple matches are found:** list them and ask:
   "I found translation files in multiple locations. Which should I use as the source of truth? (or type a custom path):"

4. **If no match is found:** ask:
   "I couldn't find translation files automatically. Please enter the path to your locales root (e.g. `public/locales`):"

5. Save the confirmed path as `localesRoot` in `.claude/translate/settings.json`. This value becomes `$LOCALES_ROOT` for all subsequent steps and future runs.

**If `settings.json` already has `localesRoot`** — load it as `$LOCALES_ROOT` and skip this step.

---

## Step 1 — Build app context (first run only)

Check whether `.claude/translate/context.md` exists.

**If it does NOT exist:**

1. Ask the user: "Do you want to provide any context about this app before I analyse the codebase? (e.g. what it does, target audience, tone of voice, domain-specific terms — or press Enter to skip):"
2. Analyse the codebase to infer what the application does:
   - Read `README.md`, `package.json`, any `.env.example`
   - Scan `src/` for domain terms, component names, API endpoints, key user flows
   - Read `$LOCALES_ROOT/en/translation.json` keys to understand the product vocabulary
3. Combine both sources into a single context:
   - Start with whatever the user provided (if anything)
   - Add inferences from the codebase that are not already covered by the user's input
   - The user's input always takes precedence — never contradict it with codebase inferences
4. Write the combined description (3–5 sentences) into `.claude/translate/context.md`
5. Print the final context and confirm: "Here's the app context I'll use for translations. Type a correction or press Enter to proceed:"
6. If the user provides a correction — rewrite `context.md` with their version. If they press Enter — keep the combined version.
7. Proceed to translation using whatever is now in `.claude/translate/context.md` as app context.

**If `context.md` already exists** — skip this step entirely and use its content as app context. Never overwrite it unless the user explicitly asks to regenerate it.

---

## New locale creation

If `$ARGUMENTS` is a locale code that does **not** yet exist under `$LOCALES_ROOT/`:

1. Create folder `$LOCALES_ROOT/<locale>/`
2. Copy the structure of every JSON file from `$LOCALES_ROOT/en/` into that folder with empty string values `""` for all keys — scaffold only, do not translate yet
3. Create `.claude/translate/cache.json` with an empty `chunks: {}` skeleton if it doesn't exist
4. Proceed with the full translation algorithm below as if all keys are missing

---

## Persistence files

### Claude-managed — `.claude/translate/`

Create `.claude/translate/` if it doesn't exist. These files are generated and updated by the skill — do not hand-edit them.

**`settings.json`** — written on first run, stores the confirmed locales root.
```json
{
  "localesRoot": "public/locales"
}
```

**`cache.json`** — one file for all locales, written after every run. Tracks the English source only — chunk keys are flat dot-path keys from `en` (prefixed with namespace when multiple files exist, e.g. `translation.login.title`). The hash is SHA-256 of the English value. If the hash matches on the next run, the key hasn't changed in `en` and all existing locale translations for it are still valid — no AI call needed. Create as the skeleton below if it doesn't exist.

```json
{
  "version": 1,
  "sourceFile": "public/locales/en/translation.json",
  "sourceLang": "en",
  "chunks": {
    "login.title": {
      "hash": "2cb537852dcb546f1c2d1523d708d50c334d704d90187d58d4a0155e690f4392",
      "translatedAt": "2026-06-05T10:46:06.289Z"
    },
    "login.description": {
      "hash": "75ecdda22a855a5717e435b8181f52a820fe6e2bd2997c57f5fb41e19ce7dc52",
      "translatedAt": "2026-06-05T10:46:10.881Z"
    }
  }
}
```

`translatedAt` records when the AI last translated this key across all locales. When a key's hash changes, it is re-translated into every target locale and `translatedAt` is updated.

**`context.md`** — auto-generated on first run. Contains the app description used as translation context. Never overwrite after the first run unless the user explicitly asks to regenerate it.

---

### Developer-managed — `$LOCALES_ROOT/glossary.yaml`

Managed by the developer — **never written or modified by this skill**. Create as an empty file if it doesn't exist, but do not populate it.

Contains preferred and pinned translations per locale — covers both preferred term choices (e.g. which word to use for "wallet") and hardcoded overrides that must never vary (brand names, legal terms). Glossary entries always win over cache and AI.

Structure:
```yaml
# term: preferred/pinned translation per locale
wallet:
  pt: carteira
  es: billetera
  uk: гаманець
deposit:
  pt: depósito
  es: depósito
  uk: поповнення
AppName:
  pt: AppName
  es: AppName
  uk: AppName
```

When translating any string, check `glossary.yaml` first. If any word or phrase in the English value matches a glossary term (case-insensitive), use the preferred translation for that term in the target locale rather than letting the AI choose freely.

---

## Terminology

- **Namespace** = filename without `.json`
- **Flat dot-path key** = nested structure flattened: `{ "a": { "b": "val" } }` → key `"a.b"`

---

## Source of truth

`$LOCALES_ROOT/en/` is the source of truth. Every other folder under `$LOCALES_ROOT/` must have the same key structure as `en` with values translated into that locale's language.

**Target locale(s):** `$ARGUMENTS` — if empty, process all locales found under `$LOCALES_ROOT/` other than `en`.

---

## Algorithm

**Phase 1 — Diff against cache (run once, not per locale)**

1. Read `.claude/translate/cache.json`, `$LOCALES_ROOT/glossary.yaml`, and `.claude/translate/context.md`.
2. For each JSON file in `$LOCALES_ROOT/en/`, walk the object recursively and collect every leaf as a flat chunk key (`<namespace>.<dot-path>`) and its English string value.
3. For each chunk key, classify it:
   - **Unchanged** — `chunks[key]` exists in `cache.json` AND `.hash === sha256(en_value)`. No AI call needed for any locale.
   - **Changed or new** — hash missing or mismatched. Must be re-translated into all target locales.
   - **Removed** — present in `cache.json` but absent from current `en` files. Remove from cache and from all locale files.

**Phase 2 — Translate changed keys (AI calls)**

4. For all *changed or new* keys, translate each `en_value` into every target locale in a single batch where possible. Apply rules:
   - Check `glossary.yaml` first — use pinned terms for that locale, translate the rest with AI.
   - Keep `{{var}}` tokens, `<tag>` markup, and HTML entities untouched — translate surrounding text only.
   - Keys classified as *unchanged* receive no AI call.

**Phase 3 — Write outputs**

5. For each target locale and each JSON file:
   - Merge results: unchanged keys → keep existing value from the locale file; changed/new keys → use the new translation from Phase 2; removed keys → delete.
   - Reconstruct the JSON with the same nested structure and key order as `en`.
   - Write the file with 2-space indentation and a trailing newline.
6. Write `.claude/translate/cache.json` — update chunks for all changed/new keys with `{ hash: sha256(en_value), translatedAt: <current ISO 8601 timestamp> }`, remove chunks for deleted keys, leave unchanged chunks untouched.

**Never touch `$LOCALES_ROOT/en/*`** — it is the source, never the target.  
**Never modify `glossary.yaml`** — it is developer-managed.

---

## Summary

After processing, print per locale/file:

- Keys added
- Keys removed
- Keys served from cache (no AI call)
- Keys translated by AI (with old → new English value if source changed)
- Keys where glossary terms were applied
- Potentially stale existing values (flag only, do not change)
- If a new locale was created — confirm folder and files scaffolded

Do not run `yarn build` or `yarn lint`.
