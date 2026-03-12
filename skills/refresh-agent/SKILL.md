---
description: Delete all pages from the CustomGPT.ai agent and re-sync the entire indexed folder from scratch.
triggers:
  - "reindex"
  - "re-index"
  - "reindex skills"
  - "refresh agent"
  - "refresh the agent"
  - "re-sync"
  - "resync"
  - "full reindex"
  - "full re-index"
---

# refresh-agent

Delete all files from the agent and re-sync from the same indexed folder. Use after making changes to the codebase.

## Critical Rules
- ALWAYS read `.customgpt-meta.json` to get `agent_id` and `indexed_folder`
- Delete ALL pages before re-uploading — do not skip deletion
- Paginate the delete loop — there may be more than 100 pages
- Touch the meta file after upload to reset the freshness timestamp

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

## Step 2 — Read Meta File

Walk up from `$PWD` to find `.customgpt-meta.json`. Extract `agent_id` and `indexed_folder`.

If not found:
> "No agent found in this directory tree. Run `/create-agent` first."

Show the user what will be refreshed and confirm:
> "I will delete all pages from agent {agent_id} and re-index {indexed_folder}. Continue? (yes/no)"

---

## Step 3 — Check for Changed Files (Informational)

List files in `indexed_folder` newer than `.customgpt-meta.json` — informational only. Refresh always re-syncs everything.

---

## Step 4 — Delete ALL Existing Pages

```bash
DELETED=0; PAGE=1
while true; do
  RESP=$(curl -s \
    "https://app.customgpt.ai/api/v1/projects/${AGENT_ID}/pages?page=${PAGE}&per_page=100" \
    -H "Authorization: Bearer ${API_KEY}")
  IDS=$(echo "$RESP" | grep -o '"id":[0-9]*' | grep -o '[0-9]*')
  [ -z "$IDS" ] && break
  for ID in $IDS; do
    HTTP=$(curl -s -o /dev/null -w "%{http_code}" \
      -X DELETE "https://app.customgpt.ai/api/v1/projects/${AGENT_ID}/pages/${ID}" \
      -H "Authorization: Bearer ${API_KEY}")
    [ "$HTTP" = "200" ] && DELETED=$((DELETED + 1))
  done
  COUNT=$(echo "$IDS" | wc -l)
  [ "$COUNT" -lt 100 ] && break
  PAGE=$((PAGE + 1))
done
echo "Deleted $DELETED pages"
```

---

## Step 5 — Re-Upload Files

Collect eligible files from `indexed_folder` (same exclusion rules as `index-files`). Write paths to `/tmp/customgpt-files.txt`.

```bash
UPLOADED=0; FAILED=0
while IFS= read -r FILE; do
  REL="${FILE#${INDEXED_FOLDER}/}"
  HTTP=$(curl -s -o /dev/null -w "%{http_code}" \
    -X POST "https://app.customgpt.ai/api/v1/projects/${AGENT_ID}/sources" \
    -H "Authorization: Bearer ${API_KEY}" \
    -F "file=@${FILE};filename=${REL}")
  if [ "$HTTP" = "200" ] || [ "$HTTP" = "201" ]; then
    UPLOADED=$((UPLOADED + 1))
  else
    FAILED=$((FAILED + 1))
    echo "FAILED [$HTTP]: $REL" >&2
  fi
done < /tmp/customgpt-files.txt
echo "Done — uploaded: $UPLOADED, failed: $FAILED"
```

---

## Step 6 — Reset Freshness Timestamp

Touch `indexed_folder/.customgpt-meta.json`.

---

## Step 7 — Report

> "Agent {agent_id} refreshed. Uploaded: {N} files.
> Use `/query-agent` to search the updated index."
