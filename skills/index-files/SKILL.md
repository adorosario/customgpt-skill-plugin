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
  - "re-index"
  - "reindex"
  - "index and"
---

# index-files

Index specific files, a directory, or the current folder into a CustomGPT.ai agent. If no agent exists yet, automatically creates one named after the current folder before indexing. Adds files to the index without wiping existing pages.

## Critical Rules
- **THIS SKILL UPLOADS FILES TO THE CUSTOMGPT.AI REST API VIA `curl`.** It does NOT save to Claude's memory system, does NOT write local files, and does NOT use the Read/Write/Edit tools for indexing purposes.
- If `.customgpt-meta.json` is not found, **auto-create the agent** (Step 2b) — do NOT stop, do NOT tell the user to run another skill.
- NEVER upload files with unsupported extensions — use the whitelist in Step 4
- NEVER upload `.env`, `.env.*`, secrets, or binary files
- Run the upload script via Bash — do NOT upload files one by one in a loop yourself
- `touch` the meta file after upload to update the freshness timestamp

---

## Step 1 — Get API Key

Check in this order:

```bash
# 1. Already-exported env var
echo "${CUSTOMGPT_API_KEY:-}"

# 2. .env file in current or parent directories (walk up to Git root or /)
dir="$PWD"
while [ "$dir" != "/" ]; do
  if [ -f "$dir/.env" ]; then
    grep -E '^export\s+CUSTOMGPT_API_KEY=|^CUSTOMGPT_API_KEY=' "$dir/.env" \
      | sed 's/^export\s*//' | sed 's/CUSTOMGPT_API_KEY=//' | tr -d '"'"'" | head -1
    break
  fi
  dir=$(dirname "$dir")
done

# 3. Saved config file
cat ~/.claude/customgpt-config.json 2>/dev/null
```

Priority: env var → `.env` file → saved config. Use the first non-empty value found as `$API_KEY`.

If no key is found anywhere, ask the user:
> "Please provide your CustomGPT.ai API key. You can find it at https://app.customgpt.ai/profile#api-keys"
>
> "You can also set it permanently by adding this to your shell profile or a `.env` file in your project:
> `export CUSTOMGPT_API_KEY=your_key_here`"

Once you have the key, save it to the config file for future use:
```bash
mkdir -p ~/.claude && echo '{"apiKey":"KEY_HERE"}' > ~/.claude/customgpt-config.json
```

---

## Step 2 — Find or Create Agent

### Step 2a — Look for existing meta file

Walk up from the current directory to find `.customgpt-meta.json`:
```bash
dir="$PWD"
while [ "$dir" != "/" ]; do
  [ -f "$dir/.customgpt-meta.json" ] && cat "$dir/.customgpt-meta.json" && break
  dir=$(dirname "$dir")
done
```

If found, extract `agent_id` and `indexed_folder`, then skip to Step 3.

### Step 2b — No meta file found: auto-create the agent

Inform the user:
> "No agent found for this project. Creating one now..."

Determine the project folder: use the Git root if available, otherwise `$PWD`:
```bash
git rev-parse --show-toplevel 2>/dev/null || echo "$PWD"
```

The agent name is the folder's basename:
```bash
basename "$PROJECT_FOLDER"
```

Create the agent:
```bash
curl -s -X POST "https://app.customgpt.ai/api/v1/projects" \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "project_name=$(basename $PROJECT_FOLDER)&is_chat_active=1"
```

Extract `data.id` from the response — this is the `agent_id`. If the response has no `data.id`, show the error and stop.

Save `.customgpt-meta.json` to the project folder:
```bash
cat > "$PROJECT_FOLDER/.customgpt-meta.json" << EOF
{
  "agent_id": $AGENT_ID,
  "agent_name": "$(basename $PROJECT_FOLDER)",
  "indexed_folder": "$PROJECT_FOLDER",
  "created_at": "$(date -u +%Y-%m-%dT%H:%M:%SZ)"
}
EOF
```

Set `indexed_folder` = `$PROJECT_FOLDER` and continue to Step 3.

---

## Step 3 — Resolve What to Index

**If the user specified files or directories**, resolve each to an absolute path:
```bash
realpath "$TARGET" 2>/dev/null || echo "not found"
```

For each path:
- If it is a **file**: include it directly if its extension is in the whitelist below.
- If it is a **directory**: collect eligible files from it recursively (Step 4).

**If no paths were specified**, use the current working directory as the target directory.

### Supported Extensions (Whitelist)

Only index files with these extensions:

**Code:**
`.js` `.ts` `.tsx` `.jsx` `.mjs` `.cjs`
`.py` `.go` `.rb` `.java` `.cs` `.cpp` `.c` `.h` `.hpp`
`.rs` `.swift` `.kt` `.php` `.scala` `.r` `.m`
`.sh` `.bash` `.zsh` `.fish`
`.html` `.htm` `.css` `.scss` `.sass` `.less`
`.sql` `.graphql` `.proto`

**Config / Data:**
`.json` `.jsonc` `.yaml` `.yml` `.toml` `.xml` `.ini` `.cfg` `.conf` `.env.example`

**Docs / Text:**
`.md` `.mdx` `.rst` `.txt` `.csv` `.tsv`

**Documents:**
`.pdf` `.docx` `.xlsx` `.pptx`

---

## Step 4 — Collect Eligible Files

For each **directory** target, collect files with the supported extensions:

```bash
find "$TARGET_DIR" -type f \
  -not -path "*/.git/*" \
  -not -path "*/node_modules/*" \
  -not -path "*/__pycache__/*" \
  -not -path "*/.next/*" \
  -not -path "*/dist/*" \
  -not -path "*/build/*" \
  -not -path "*/.cache/*" \
  -not -path "*/vendor/*" \
  -not -path "*/coverage/*" \
  -not -path "*/.venv/*" \
  -not -path "*/venv/*" \
  \( \
    -name "*.js"   -o -name "*.ts"   -o -name "*.tsx"  -o -name "*.jsx"  \
    -o -name "*.mjs"  -o -name "*.cjs" \
    -o -name "*.py"   -o -name "*.go"   -o -name "*.rb"   -o -name "*.java" \
    -o -name "*.cs"   -o -name "*.cpp"  -o -name "*.c"    -o -name "*.h"   \
    -o -name "*.hpp"  -o -name "*.rs"   -o -name "*.swift" -o -name "*.kt" \
    -o -name "*.php"  -o -name "*.scala" -o -name "*.r"   -o -name "*.m"   \
    -o -name "*.sh"   -o -name "*.bash" -o -name "*.zsh"  -o -name "*.fish" \
    -o -name "*.html" -o -name "*.htm"  -o -name "*.css"  -o -name "*.scss" \
    -o -name "*.sass" -o -name "*.less" -o -name "*.sql"  -o -name "*.graphql" \
    -o -name "*.proto" \
    -o -name "*.json" -o -name "*.jsonc" -o -name "*.yaml" -o -name "*.yml" \
    -o -name "*.toml" -o -name "*.xml"  -o -name "*.ini"  -o -name "*.cfg" \
    -o -name "*.conf" -o -name "*.env.example" \
    -o -name "*.md"   -o -name "*.mdx"  -o -name "*.rst"  -o -name "*.txt" \
    -o -name "*.csv"  -o -name "*.tsv" \
    -o -name "*.pdf"  -o -name "*.docx" -o -name "*.xlsx" -o -name "*.pptx" \
  \) \
  > /tmp/customgpt-files.txt

wc -l < /tmp/customgpt-files.txt
```

For **individual files** passed directly, write their paths into `/tmp/customgpt-files.txt` instead (after verifying extension is in whitelist).

Tell the user how many files will be uploaded before proceeding.

---

## Step 5 — Upload Files

```bash
cat > /tmp/customgpt-upload.sh << 'UPLOAD_EOF'
#!/bin/bash
API_KEY="__API_KEY__"
AGENT_ID="__AGENT_ID__"
BASE_DIR="__BASE_DIR__"
UPLOADED=0
FAILED=0
TOTAL=$(wc -l < /tmp/customgpt-files.txt)

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

echo ""
echo "Done — uploaded: $UPLOADED / $TOTAL, failed: $FAILED"
UPLOAD_EOF

sed -i "s|__API_KEY__|$API_KEY|g"     /tmp/customgpt-upload.sh
sed -i "s|__AGENT_ID__|$AGENT_ID|g"   /tmp/customgpt-upload.sh
sed -i "s|__BASE_DIR__|$BASE_DIR|g"   /tmp/customgpt-upload.sh

chmod +x /tmp/customgpt-upload.sh
bash /tmp/customgpt-upload.sh
```

`$BASE_DIR` should be set to `indexed_folder` from the meta file (used to compute relative paths). If the files being indexed live outside that folder, use their common parent directory instead.

---

## Step 6 — Update Freshness Timestamp

```bash
touch "$META_FILE_PATH"
```

Where `$META_FILE_PATH` is the full path to the `.customgpt-meta.json` file that was found in Step 2.

---

## Step 7 — Report

> "Indexed N file(s) into agent {agent_id}.
>
> - Uploaded: {N}
> - Failed: {F}
>
> Use `/query-agent` to search the updated index, or `/refresh-agent` to do a full re-sync."

If `FAILED > 0`, suggest:
> "Some files failed. Check that your API key is valid (`cat ~/.claude/customgpt-config.json`) and that your plan has capacity."
