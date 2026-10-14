---
name: translate
description: Syncs public/locales translation JSON files using en as the source of truth. On first run analyses the project to build app context, asks the user to confirm or extend it, then translates missing or changed keys using AI with that context and a glossary. If a locale code is passed that does not exist yet, creates the folder and all JSON files before translating.
version: 1.0.0
user-invocable: true
argument-hint: "[locale] (optional ISO 639-1 code, e.g. pt — defaults to all non-en locales)"
---

# translate

Syncs `public/locales/<locale>/*.json` files using `public/locales/en/` as the source of truth. Uses cached translations, developer-managed overrides, a glossary, and app context to produce accurate, consistent translations with minimal AI calls.

---

## Step 0 — Build app context (first run only)

Check whether `src/translations/.translate/context.md` exists.

**If it does NOT exist:**

1. Ask the user: "Do you want to provide any context about this app before I analyse the codebase? (e.g. what it does, target audience, tone of voice, domain-specific terms — or press Enter to skip):"
2. Analyse the codebase to infer what the application does:
   - Read `README.md`, `package.json`, any `.env.example`
   - Scan `src/` for domain terms, component names, API endpoints, key user flows
   - Read `public/locales/en/translation.json` keys to understand the product vocabulary
3. Combine both sources into a single context:
   - Start with whatever the user provided (if anything)
   - Add inferences from the codebase that are not already covered by the user's input
   - The user's input always takes precedence — never contradict it with codebase inferences
4. Write the combined description (3–5 sentences) into `src/translations/.translate/context.md`
5. Print the final context and confirm: "Here's the app context I'll use for translations. Type a correction or press Enter to proceed:"
6. If the user provides a correction — rewrite `context.md` with their version. If they press Enter — keep the combined version.
7. Proceed to translation using whatever is now in `context.md` as app context.

**If `context.md` already exists** — skip this step entirely and use its content as app context. Never overwrite it unless the user explicitly asks to regenerate it.

---

## New locale creation

If `$ARGUMENTS` is a locale code that does **not** yet exist under `public/locales/`:

1. Create folder `public/locales/<locale>/`
2. Copy the structure of every JSON file from `public/locales/en/` into that folder with empty string values `""` for all keys — scaffold only, do not translate yet
3. Initialise a cache entry for this locale as `{}` in `src/translations/.translate/cache.json` if not already present
4. Proceed with the full translation algorithm below as if all keys are missing

---

## Persistence files

All persistence files live at `src/translations/.translate/`. Create the directory if it doesn't exist.

### `cache.json`
Written by this skill after every run. Stores the English source value and its translation for every key translated via AI. On the next run, if a key's English value hasn't changed, the cached translation is reused — no AI call needed. Create as `{}` if it doesn't exist.

Structure: `locale → namespace → flat dot-path key → { "source": "<en value>", "translation": "<translated>" }`

```json
{
  "pt": {
    "translation": {
      "login.title": { "source": "Sign in", "translation": "Entrar" }
    }
  }
}
```

### `config.json`
Managed by the developer — **never written by this skill**. Contains hardcoded overrides that always win over cache and AI (brand names, pinned legal terms). Create as `{}` if it doesn't exist, but do not populate it.

Structure: `locale → namespace → flat dot-path key → hardcoded translation string`

### `glossary.yaml`
Managed by the developer — **never written by this skill**. Contains preferred translations for specific terms per locale. Create as an empty file if it doesn't exist, but do not populate it.

Structure:
```yaml
# term: preferred translation per locale
wallet:
  pt: carteira
  es: billetera
  uk: гаманець
deposit:
  pt: depósito
  es: depósito
  uk: поповнення
```

When translating any string, check `glossary.yaml` first. If any word or phrase in the English value matches a glossary term, use the preferred translation for that term in the target locale rather than letting the AI choose freely. Glossary terms are case-insensitive.

### `context.md`
Auto-generated on first run, then developer-managed. Contains the app description used as translation context. Never overwrite after the first run unless the user explicitly asks to regenerate it.

---

## Terminology

- **Namespace** = filename without `.json`
- **Flat dot-path key** = nested structure flattened: `{ "a": { "b": "val" } }` → key `"a.b"`

---

## Source of truth

`public/locales/en/` is the source of truth. Every other folder under `public/locales/` must have the same key structure as `en` with values translated into that locale's language.

**Target locale(s):** `$ARGUMENTS` — if empty, process all locales found under `public/locales/` other than `en`.

---

## Algorithm

For each target locale and each JSON file in `public/locales/en/`:

1. Read `public/locales/en/<file>` and `public/locales/<locale>/<file>` (treat a missing target file as `{}`).
2. Read `cache.json`, `config.json`, `glossary.yaml`, and `context.md`.
3. Walk the `en` object recursively. Collect every leaf as a flat dot-path key and its English string value.
4. For each key, determine the result in priority order:
   - **Config override** — `config[locale][namespace][key]` exists: use that string. No cache write.
   - **Cache hit** — `cache[locale][namespace][key]` exists AND `.source === en_value`: use `.translation`. No AI call.
   - **Cache miss or source changed** — translate `en_value` using the app context from `context.md` and any matching glossary terms. Keep `{{var}}` tokens, `<tag>` markup, and HTML entities untouched — translate surrounding text only. Write `{ source: en_value, translation: result }` to cache.
5. Keys present in the target file but absent from `en`: delete them.
6. Reconstruct the target JSON with the same nested structure and key order as `en`.
7. Write the target file with 2-space indentation and a trailing newline.
8. Merge cache updates into `cache.json` — add/overwrite AI-translated keys, remove keys no longer in `en`, leave other locales untouched.

**Never touch `public/locales/en/*`** — it is the source, never the target.  
**Never modify `config.json` or `glossary.yaml`** — they are developer-managed.

---

## Summary

After processing, print per locale/file:

- Keys added
- Keys removed
- Keys served from cache (no AI call)
- Keys served from config override
- Keys translated by AI (with old → new English value if source changed)
- Keys where glossary terms were applied
- Potentially stale existing values (flag only, do not change)
- If a new locale was created — confirm folder and files scaffolded

Do not run `yarn build` or `yarn lint`.
