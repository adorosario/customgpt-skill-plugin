# query-agent

Ask a plain-language question to the indexed project. Returns an AI answer with source citations.

## Critical Rules
- ALWAYS read `.customgpt-meta.json` to get the `agent_id`
- Each call starts a fresh conversation (no session reuse)
- Present the answer AND the sources clearly to the user
- Let Claude decide how to incorporate the answer into the current task — do not just paste it blindly

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

## Step 2 — Read Meta File

Walk up from the current directory to find `.customgpt-meta.json`:
```bash
dir="$PWD"
while [ "$dir" != "/" ]; do
  [ -f "$dir/.customgpt-meta.json" ] && cat "$dir/.customgpt-meta.json" && break
  dir=$(dirname "$dir")
done
```

Extract `agent_id`. If not found:
> "No agent found in this directory tree. Run `/create-agent` first."

---

## Step 3 — Check Freshness (Optional, Informational)

Check if any files have changed since the last index:
```bash
find "$INDEXED_FOLDER" -newer "$INDEXED_FOLDER/.customgpt-meta.json" -type f \
  -not -path "*/.git/*" -not -path "*/node_modules/*" -not -path "*/__pycache__/*" \
  -not -name ".DS_Store" -not -name "*.pyc" 2>/dev/null | head -20
```

If changed files are found, inform the user:
> "Note: {N} files have changed since the last index. Run `/refresh-agent` for up-to-date results."

Then continue with the query regardless.

---

## Step 4 — Get the Question

If the user provided a question with the skill invocation, use it directly.
Otherwise ask:
> "What would you like to know about the indexed project?"

---

## Step 5 — Create a Conversation Session

```bash
curl -s -X POST "https://app.customgpt.ai/api/v1/projects/$AGENT_ID/conversations" \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "name=Claude+Code+Query"
```

Extract `data.session_id` from the response.

---

## Step 6 — Send the Question

```bash
curl -s -X POST "https://app.customgpt.ai/api/v1/projects/$AGENT_ID/conversations/$SESSION_ID/messages" \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  --data-urlencode "prompt=$QUESTION" \
  -d "stream=false"
```

From the response extract:
- `data.openai_response` — the answer text
- `data.citations` — array of source citations (each has `title`, `url`, `document_name` or similar fields)

---

## Step 7 — Present Results

Show the answer and sources to the user:

**Answer:**
{openai_response}

**Sources:**
- {citation 1 title} — {citation 1 url or filename}
- {citation 2 title} — {citation 2 url or filename}
- ...

Then use the answer to help with whatever task the user is working on. If the answer is about code, read the referenced files to get full context before making changes.
