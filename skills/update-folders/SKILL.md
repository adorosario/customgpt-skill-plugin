# update-folders

Change which folder is indexed for an existing agent. Deletes all current pages from the agent and re-indexes the new folder selection.

## Critical Rules
- ALWAYS read `.customgpt-meta.json` before doing anything
- ALWAYS delete ALL existing pages before uploading the new folder
- Paginate the pages list — there may be more than 100 pages
- Update `.customgpt-meta.json` with the new folder after success

---

## Step 1 — Get API Key

```bash
cat ~/.claude/customgpt-config.json 2>/dev/null
```

Use field `apiKey`, or env var `CUSTOMGPT_API_KEY`. If missing, ask the user for it and save:
```bash
mkdir -p ~/.claude && echo '{"apiKey":"KEY_HERE"}' > ~/.claude/customgpt-config.json
```

---

## Step 2 — Read Existing Meta

Find `.customgpt-meta.json` by walking up from the current directory:
```bash
dir="$PWD"
while [ "$dir" != "/" ]; do
  [ -f "$dir/.customgpt-meta.json" ] && cat "$dir/.customgpt-meta.json" && break
  dir=$(dirname "$dir")
done
```

Extract `agent_id` and `indexed_folder`. If no meta file found, tell the user:
> "No agent found. Run `/create-agent` first."

---

## Step 3 — Ask for New Folder

Ask the user:
> "Which folder should I index now? Enter an absolute path, or press Enter to keep the current folder: {current_folder}"

If no input, keep the existing folder.

Confirm the folder exists:
```bash
test -d "$NEW_FOLDER" && echo "exists" || echo "not found"
```

---

## Step 4 — Delete ALL Existing Pages

Paginate through all pages and delete each one:

```bash
cat > /tmp/customgpt-delete-all.sh << 'DELETE_EOF'
#!/bin/bash
API_KEY="__API_KEY__"
AGENT_ID="__AGENT_ID__"
DELETED=0
PAGE=1

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
DELETE_EOF

sed -i "s|__API_KEY__|$API_KEY|g" /tmp/customgpt-delete-all.sh
sed -i "s|__AGENT_ID__|$AGENT_ID|g" /tmp/customgpt-delete-all.sh
chmod +x /tmp/customgpt-delete-all.sh
bash /tmp/customgpt-delete-all.sh
```

---

## Step 5 — Upload New Folder

**Collect files:**
```bash
find "$NEW_FOLDER" -type f \
  -not -path "*/.git/*" \
  -not -path "*/node_modules/*" \
  -not -path "*/__pycache__/*" \
  -not -path "*/.next/*" \
  -not -path "*/dist/*" \
  -not -path "*/build/*" \
  -not -path "*/.cache/*" \
  -not -path "*/vendor/*" \
  -not -path "*/coverage/*" \
  -not -name "*.pyc" -not -name "*.pyo" -not -name "*.class" \
  -not -name "*.exe" -not -name "*.bin" -not -name "*.zip" \
  -not -name "*.tar" -not -name "*.gz" \
  -not -name "*.png" -not -name "*.jpg" -not -name "*.jpeg" \
  -not -name "*.gif" -not -name "*.ico" -not -name "*.webp" \
  -not -name "*.svg" -not -name "*.mp3" -not -name "*.mp4" \
  -not -name "*.mov" -not -name "*.woff" -not -name "*.woff2" \
  -not -name "*.ttf" -not -name "*.lock" \
  -not -name ".DS_Store" -not -name ".env" -not -name ".env.*" \
  > /tmp/customgpt-files.txt

wc -l < /tmp/customgpt-files.txt
```

**Upload:**
```bash
cat > /tmp/customgpt-upload.sh << 'UPLOAD_EOF'
#!/bin/bash
API_KEY="__API_KEY__"
AGENT_ID="__AGENT_ID__"
FOLDER="__FOLDER__"
UPLOADED=0
FAILED=0

while IFS= read -r FILE; do
  REL="${FILE#${FOLDER}/}"
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
UPLOAD_EOF

sed -i "s|__API_KEY__|$API_KEY|g" /tmp/customgpt-upload.sh
sed -i "s|__AGENT_ID__|$AGENT_ID|g" /tmp/customgpt-upload.sh
sed -i "s|__FOLDER__|$NEW_FOLDER|g" /tmp/customgpt-upload.sh
chmod +x /tmp/customgpt-upload.sh
bash /tmp/customgpt-upload.sh
```

---

## Step 6 — Update Meta File

```bash
cat > "$NEW_FOLDER/.customgpt-meta.json" << EOF
{
  "agent_id": $AGENT_ID,
  "agent_name": "$(basename $NEW_FOLDER)",
  "indexed_folder": "$NEW_FOLDER",
  "created_at": "$(date -u +%Y-%m-%dT%H:%M:%SZ)"
}
EOF
touch "$NEW_FOLDER/.customgpt-meta.json"
```

If the folder changed, also remove the old meta file:
```bash
rm -f "$OLD_FOLDER/.customgpt-meta.json"
```

---

## Step 7 — Report

> "Done. Agent {agent_id} is now indexed from {new_folder}.
> Uploaded: {N} files."
