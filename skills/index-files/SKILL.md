---
description: Index specific files or a directory into an existing CustomGPT.ai agent for RAG search. Supports explicit file lists, directory paths, or the current folder. Respects file extension whitelist and exclusion rules.
triggers:
  - "index"
  - "index the"
  - "index this"
  - "index these"
  - "index just"
  - "index only"
  - "index overview"
  - "index troubleshooting"
  - "index the overview"
  - "index the troubleshooting"
  - "index these files"
  - "index this file"
  - "index this folder"
  - "index this directory"
  - "index this project"
  - "index this repo"
  - "index the files"
  - "index the folder"
  - "index the directory"
  - "index the project"
  - "index the repo"
  - "add to the index"
  - "add these files to"
  - "add this file to"
  - "add to index"
  - "upload to the agent"
  - "upload these files"
  - "upload to customgpt"
  - "index and"
---

# index-files

Index specific files, a directory, or the current folder into a CustomGPT.ai agent. Auto-creates an agent if none exists. Adds files without wiping existing pages.

## Critical Rules
- THIS SKILL UPLOADS FILES TO THE CUSTOMGPT.AI REST API VIA `curl` — it does NOT write to Claude's memory or use Read/Write/Edit for indexing
- If `.customgpt-meta.json` is not found, auto-create the agent — do NOT stop or ask the user to run another skill
- NEVER upload `.env`, `.env.*`, secrets, or binary files
- Use `find` to collect eligible files, then upload each one with curl — Claude handles the iteration

---

## Step 1 — Get API Key

Check in order:
1. Env var `$CUSTOMGPT_API_KEY`
2. `.env` file — walk up from `$PWD` to `/` looking for a file containing `CUSTOMGPT_API_KEY=`
3. Read `~/.claude/customgpt-config.json` and extract the `apiKey` field

Use the first non-empty value. If none found, ask the user then save:
```bash
echo '{"apiKey":"KEY_HERE"}' > ~/.claude/customgpt-config.json
```

---

## Step 2 — Find or Create Agent

Walk up from `$PWD` to find `.customgpt-meta.json`. If found, extract `agent_id` and `indexed_folder` and skip to Step 3.

If not found, inform the user and auto-create:
> "No agent found. Creating one now..."

Use the Git root (or `$PWD`) as the project folder. Agent name = folder basename.

```bash
curl -s --request POST \
  --url "https://app.customgpt.ai/api/v1/projects" \
  --header "Authorization: Bearer ${API_KEY}" \
  --header "Content-Type: application/x-www-form-urlencoded" \
  --data "project_name=$(basename $PROJECT_FOLDER)&is_chat_active=1"
```

Read the response and extract `data.id` as `agent_id`. Save `.customgpt-meta.json` to the project folder.

---

## Step 3 — Resolve What to Index

If the user specified files or directories, use those. Otherwise use `$PWD`.

**Single file by name:** If the user gave just a filename (not a full path), search for it under `indexed_folder`:

```bash
find "$indexed_folder" -name "${FILENAME}" -type f
```

Use the first match. If not found, stop and tell the user.

**Single file by path:** Resolve to absolute path and verify it exists. Check its extension against the whitelist — if not supported, stop and tell the user.

**Directory:** Collect files recursively with `find` using the exclusions and whitelist below.

### Supported Extensions

**Code:** `.js` `.ts` `.tsx` `.jsx` `.mjs` `.cjs` `.py` `.go` `.rb` `.java` `.cs` `.cpp` `.c` `.h` `.hpp` `.rs` `.swift` `.kt` `.php` `.scala` `.sh` `.bash` `.zsh` `.fish` `.html` `.htm` `.css` `.scss` `.sass` `.less` `.sql` `.graphql` `.proto`

**Config/Data:** `.json` `.jsonc` `.yaml` `.yml` `.toml` `.xml` `.ini` `.cfg` `.conf` `.env.example`

**Docs:** `.md` `.mdx` `.rst` `.txt` `.csv` `.tsv` `.pdf` `.docx` `.xlsx` `.pptx`, 'doc', `odt`, `txt`

**Images** `jpg`, `jpeg`, `png`, `webp`

Exclude: `.git/`, `node_modules/`, `__pycache__/`, `.next/`, `dist/`, `build/`, `.cache/`, `vendor/`, `coverage/`, `.venv/`, `venv/`

Tell the user the file count before uploading.

---

## Step 4 — Vision Check (Images Only)

If any files to be uploaded are images (`.jpg`, `.jpeg`, `.png`, `.webp`), ask the user before uploading:

> "Image files detected. Do you want to use AI Vision for image processing? (yes/no)"

If **yes**, also ask:
> "Compress images before vision processing? (yes/no)"

Set the following form fields for image uploads based on the answers:
- `is_vision_enabled=true`
- `ocr_mode=2` (AI Vision)
- `is_vision_compress_image=true` or `false`

If **no**, omit all three fields (defaults to no vision processing).

Non-image files always omit these fields.

---

## Step 5 — Upload Each File

For each eligible file, compute `REL` as its path relative to `indexed_folder` (strip the `indexed_folder/` prefix). Then run:

**For non-image files:**
```bash
curl -s --request POST \
  --url "https://app.customgpt.ai/api/v1/projects/${AGENT_ID}/sources" \
  --header "Authorization: Bearer ${API_KEY}" \
  --form "file=@${ABSOLUTE_PATH};filename=${REL}"
```

**For image files with vision enabled:**
```bash
curl -s --request POST \
  --url "https://app.customgpt.ai/api/v1/projects/${AGENT_ID}/sources" \
  --header "Authorization: Bearer ${API_KEY}" \
  --form "file=@${ABSOLUTE_PATH};filename=${REL}" \
  --form "is_vision_enabled=true" \
  --form "ocr_mode=2" \
  --form "is_vision_compress_image=${COMPRESS}"
```

Read the HTTP status from the response. HTTP 200 or 201 = success. Report each result: `OK: {REL}` or `FAILED [{status}]: {REL}`.

---

## Step 6 — Update Freshness Timestamp

```bash
touch "${META_FILE_PATH}"
```

---

## Step 7 — Report

> "Indexed {N} file(s) into agent {agent_id}. Uploaded: {N}, Failed: {F}
> Use `/query-agent` to search or `/refresh-agent` for a full re-sync."
