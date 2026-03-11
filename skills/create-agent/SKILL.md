# create-agent

Create a CustomGPT.ai agent for a local folder and index its files into it.

## Critical Rules
- NEVER skip the API key check
- ALWAYS save `.customgpt-meta.json` before reporting success
- NEVER index `.git/`, `node_modules/`, `__pycache__/`, binary files, or `.env` files
- Run the upload script via Bash — do NOT upload files one by one in a loop yourself

---

## Step 1 — Get API Key

Check in this order:
1. Environment variable `CUSTOMGPT_API_KEY`
2. File `~/.claude/customgpt-config.json` → field `apiKey`

Read the config file with:
```bash
cat ~/.claude/customgpt-config.json 2>/dev/null
```

If no key is found, ask the user:
> "Please provide your CustomGPT.ai API key. You can find it at https://app.customgpt.ai/profile#api-keys"

Once you have the key, save it for future use:
```bash
mkdir -p ~/.claude
echo '{"apiKey":"KEY_HERE"}' > ~/.claude/customgpt-config.json
```

---

## Step 2 — Get Folder to Index

Ask the user:
> "Which folder should I index? Enter an absolute path, or press Enter to use the current working directory."

If no path given, use the current working directory (`$PWD`).

Confirm the folder exists:
```bash
test -d "$FOLDER" && echo "exists" || echo "not found"
```

---

## Step 3 — Create the Agent

```bash
curl -s -X POST "https://app.customgpt.ai/api/v1/projects" \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "project_name=$(basename $FOLDER)&is_chat_active=1"
```

Extract `data.id` from the JSON response — this is the `agent_id`.
If the response has no `data.id`, show the user the error and stop.

---

## Step 4 — Find and Upload Files

**4a. Collect files** (save to `/tmp/customgpt-files.txt`):
```bash
find "$FOLDER" -type f \
  -not -path "*/.git/*" \
  -not -path "*/node_modules/*" \
  -not -path "*/__pycache__/*" \
  -not -path "*/.next/*" \
  -not -path "*/dist/*" \
  -not -path "*/build/*" \
  -not -path "*/.cache/*" \
  -not -path "*/vendor/*" \
  -not -path "*/coverage/*" \
  -not -name "*.pyc" \
  -not -name "*.pyo" \
  -not -name "*.class" \
  -not -name "*.exe" \
  -not -name "*.bin" \
  -not -name "*.zip" \
  -not -name "*.tar" \
  -not -name "*.gz" \
  -not -name "*.png" \
  -not -name "*.jpg" \
  -not -name "*.jpeg" \
  -not -name "*.gif" \
  -not -name "*.ico" \
  -not -name "*.webp" \
  -not -name "*.svg" \
  -not -name "*.mp3" \
  -not -name "*.mp4" \
  -not -name "*.mov" \
  -not -name "*.woff" \
  -not -name "*.woff2" \
  -not -name "*.ttf" \
  -not -name "*.lock" \
  -not -name ".DS_Store" \
  -not -name ".env" \
  -not -name ".env.*" \
  > /tmp/customgpt-files.txt

wc -l < /tmp/customgpt-files.txt
```

Tell the user how many files were found before uploading.

**4b. Write and run upload script**:
```bash
cat > /tmp/customgpt-upload.sh << 'UPLOAD_EOF'
#!/bin/bash
API_KEY="__API_KEY__"
AGENT_ID="__AGENT_ID__"
FOLDER="__FOLDER__"
UPLOADED=0
FAILED=0
TOTAL=$(wc -l < /tmp/customgpt-files.txt)

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

echo "Done — uploaded: $UPLOADED / $TOTAL, failed: $FAILED"
UPLOAD_EOF

# Substitute real values
sed -i "s|__API_KEY__|$API_KEY|g" /tmp/customgpt-upload.sh
sed -i "s|__AGENT_ID__|$AGENT_ID|g" /tmp/customgpt-upload.sh
sed -i "s|__FOLDER__|$FOLDER|g" /tmp/customgpt-upload.sh

chmod +x /tmp/customgpt-upload.sh
bash /tmp/customgpt-upload.sh
```

---

## Step 5 — Save Meta File

Write `.customgpt-meta.json` to the indexed folder:
```bash
cat > "$FOLDER/.customgpt-meta.json" << EOF
{
  "agent_id": $AGENT_ID,
  "agent_name": "$(basename $FOLDER)",
  "indexed_folder": "$FOLDER",
  "created_at": "$(date -u +%Y-%m-%dT%H:%M:%SZ)"
}
EOF
touch "$FOLDER/.customgpt-meta.json"
```

---

## Step 6 — Report

Tell the user:
> "Agent created successfully.
> - Agent ID: {agent_id}
> - Folder: {folder}
> - Files uploaded: {N}
>
> Use `/query-agent` to search it, or `/refresh-agent` to re-sync after changes."
