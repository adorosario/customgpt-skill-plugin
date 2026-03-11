# delete-agent

Permanently delete a CustomGPT.ai agent and remove its local meta file.

## Critical Rules
- ALWAYS confirm with the user before deleting — this is irreversible
- Remove `.customgpt-meta.json` after successful deletion
- If the API call fails, do NOT remove the meta file

---

## Step 1 — Get API Key

```bash
cat ~/.claude/customgpt-config.json 2>/dev/null
```

Use field `apiKey`, or env var `CUSTOMGPT_API_KEY`. If missing, ask the user for it.

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

Extract `agent_id`, `agent_name`, and `indexed_folder`. If not found:
> "No agent found in this directory tree. Nothing to delete."

---

## Step 3 — Confirm with User

Ask:
> "Are you sure you want to permanently delete agent '{agent_name}' (ID: {agent_id})? This will remove the agent and all its indexed data from CustomGPT.ai. This cannot be undone. Type 'yes' to confirm."

If the user does not confirm, stop:
> "Cancelled. Agent was not deleted."

---

## Step 4 — Delete the Agent

```bash
curl -s -X DELETE "https://app.customgpt.ai/api/v1/projects/$AGENT_ID" \
  -H "Authorization: Bearer $API_KEY"
```

Check the response. On success the API returns `{"status":"success"}` or HTTP 200.

If the call fails, show the error and stop — do NOT remove the meta file.

---

## Step 5 — Remove Meta File

```bash
rm -f "$META_FILE_PATH"
```

Where `$META_FILE_PATH` is the full path to the `.customgpt-meta.json` file found in Step 2.

---

## Step 6 — Report

> "Agent '{agent_name}' (ID: {agent_id}) has been deleted.
> Local meta file removed from {indexed_folder}."
