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
- Run the upload loop via Bash — do NOT upload files one by one yourself

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
curl -s -X POST "https://app.customgpt.ai/api/v1/projects" \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "project_name=$(basename $PROJECT_FOLDER)&is_chat_active=1"
```

Extract `data.id` as `agent_id`. Save `.customgpt-meta.json` to the project folder.

---

## Step 3 — Resolve What to Index

If the user specified files or directories, use those. Otherwise use `$PWD`.

For directories, collect files recursively. For individual files, include them directly if their extension is in the whitelist.

### Supported Extensions

**Code:** `.js` `.ts` `.tsx` `.jsx` `.mjs` `.cjs` `.py` `.go` `.rb` `.java` `.cs` `.cpp` `.c` `.h` `.hpp` `.rs` `.swift` `.kt` `.php` `.scala` `.sh` `.bash` `.zsh` `.fish` `.html` `.htm` `.css` `.scss` `.sass` `.less` `.sql` `.graphql` `.proto`

**Config/Data:** `.json` `.jsonc` `.yaml` `.yml` `.toml` `.xml` `.ini` `.cfg` `.conf` `.env.example`

**Docs:** `.md` `.mdx` `.rst` `.txt` `.csv` `.tsv` `.pdf` `.docx` `.xlsx` `.pptx`

Exclude: `.git/`, `node_modules/`, `__pycache__/`, `.next/`, `dist/`, `build/`, `.cache/`, `vendor/`, `coverage/`, `.venv/`, `venv/`

Tell the user the file count before uploading.

---

## Step 4 — Upload Files

```bash
UPLOADED=0; FAILED=0; TOTAL=$(wc -l < /tmp/customgpt-files.txt)
while IFS= read -r FILE; do
  [ -z "$FILE" ] && continue
  REL="${FILE#${BASE_DIR}/}"
  HTTP=$(curl -s -o /dev/null -w "%{http_code}" \
    -X POST "https://app.customgpt.ai/api/v1/projects/${AGENT_ID}/sources" \
    -H "Authorization: Bearer ${API_KEY}" \
    -F "file=@${FILE};filename=${REL}")
  if [ "$HTTP" = "200" ] || [ "$HTTP" = "201" ]; then
    UPLOADED=$((UPLOADED + 1))
    echo "OK [$UPLOADED/$TOTAL]: $REL"
  else
    FAILED=$((FAILED + 1))
    echo "FAILED [$HTTP]: $REL" >&2
  fi
done < /tmp/customgpt-files.txt
echo "Done — uploaded: $UPLOADED / $TOTAL, failed: $FAILED"
```

`$BASE_DIR` = `indexed_folder` from the meta file (used to compute relative paths).

---

## Step 5 — Update Freshness Timestamp

Touch the `.customgpt-meta.json` file to reset the freshness timestamp.

---

## Step 6 — Report

> "Indexed {N} file(s) into agent {agent_id}. Uploaded: {N}, Failed: {F}
> Use `/query-agent` to search or `/refresh-agent` for a full re-sync."
