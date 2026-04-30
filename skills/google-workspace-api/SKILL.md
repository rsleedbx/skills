---
name: google-workspace-api
description: >-
  Read, write, comment, and update Google Docs, Sheets, and Slides via the
  Google REST APIs using gcloud Application Default Credentials (ADC).
  Auth requires gcloud CLI; no OAuth client secret needed. Use when reading
  or writing Google Docs/Sheets/Slides content, adding comments, or updating
  cells/slides programmatically from a local shell or agent.
---

# Google Workspace API — Read / Write / Comment / Update

## Validation legend

| Mark | Meaning |
|------|---------|
| ✓ tested | Live-tested in this environment |
| ⚠ not tested | Same pattern, not run live |

## Auth: gcloud Application Default Credentials

All operations use a short-lived OAuth access token from ADC. No OAuth client secret file needed — just the gcloud CLI.

### Step 1 — Authenticate (one-time per scope set)

Choose the minimum scope set for the task:

```bash
# Read-only (Docs + Drive metadata/comments)
gcloud auth application-default login \
  --scopes=https://www.googleapis.com/auth/cloud-platform,\
https://www.googleapis.com/auth/documents.readonly,\
https://www.googleapis.com/auth/drive.readonly

# Read + Write + Comment (Docs, Sheets, Slides, Drive comments)
gcloud auth application-default login \
  --scopes=https://www.googleapis.com/auth/cloud-platform,\
https://www.googleapis.com/auth/documents,\
https://www.googleapis.com/auth/drive
```

✓ tested: scopes must be a **single comma-separated string** with no spaces or line breaks.

Re-run with the write scope when comments or edits are needed — existing credentials are overwritten.

### Step 2 — Get token

```bash
TOKEN=$(gcloud auth application-default print-access-token)
GCP_PROJECT=$(gcloud config get-value project 2>/dev/null)
```

Tokens expire in ~1 hour. Re-run if you get HTTP 401.

### Step 3 — Add quota project header to every request

Local ADC credentials require a billing quota project for Google Workspace APIs. Without it, requests return HTTP 403 `SERVICE_DISABLED`:

```
"reason": "SERVICE_DISABLED",
"message": "The sheets.googleapis.com API requires a quota project..."
```

**Fix**: add `-H "x-goog-user-project: $GCP_PROJECT"` to every `curl` call. This is required even for read-only requests. Setting the quota project permanently via `gcloud auth application-default set-quota-project` requires write access to `~/.config/gcloud/application_default_credentials.json` — use the header approach when that file is not writable.

```bash
# Correct pattern — always include x-goog-user-project
curl -s -H "Authorization: Bearer $TOKEN" \
  -H "x-goog-user-project: $GCP_PROJECT" \
  "https://sheets.googleapis.com/v4/spreadsheets/${SS_ID}/values/Sheet1!A1:Z50"
```

✓ tested — the `x-goog-user-project` header unblocks all Sheets, Docs, and Slides API calls under local ADC.

---

## Google Docs

**Document ID** is the long string in the URL:
`https://docs.google.com/document/d/<DOCUMENT_ID>/edit`

### Read document content

```bash
DOC_ID="<document_id>"
curl -s -H "Authorization: Bearer $TOKEN" \
  "https://docs.googleapis.com/v1/documents/${DOC_ID}" \
  | jq '.body.content[].paragraph.elements[]?.textRun.content // empty' \
  | tr -d '"' | tr -d '\n'
```

✓ tested — returns plain text extracted from all paragraph text runs.

### Append text

```bash
DOC_ID="<document_id>"
TEXT="hello"

# Get the current end-of-document index
END_INDEX=$(curl -s -H "Authorization: Bearer $TOKEN" \
  "https://docs.googleapis.com/v1/documents/${DOC_ID}" \
  | jq '.body.content[-1].endIndex')

curl -s -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "https://docs.googleapis.com/v1/documents/${DOC_ID}:batchUpdate" \
  -d "{\"requests\":[{\"insertText\":{\"location\":{\"index\":$((END_INDEX - 1))},\"text\":\"${TEXT}\"}}]}"
```

✓ tested — inserts text at the end of the document body.

### Add a comment on selected text

Comments are Drive API objects, not Docs API. The `anchor` field pins the comment to a text range using the `r` (rich text anchor) format.

```bash
DOC_ID="<document_id>"
WORD="hello"   # text the comment should be anchored to

# Get the character offset of the word
OFFSET=$(curl -s -H "Authorization: Bearer $TOKEN" \
  "https://docs.googleapis.com/v1/documents/${DOC_ID}" \
  | python3 -c "
import json, sys
doc = json.load(sys.stdin)
text = ''
for block in doc['body']['content']:
    for el in block.get('paragraph', {}).get('elements', []):
        text += el.get('textRun', {}).get('content', '')
idx = text.find('${WORD}')
print(idx)
")

# Post comment anchored to the word
curl -s -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "https://www.googleapis.com/drive/v3/files/${DOC_ID}/comments?fields=id,content,anchor" \
  -d "{
    \"content\": \"Your comment text here\",
    \"anchor\": \"{\\\"r\\\":\\\"head\\\",\\\"a\\\":[{\\\"n\\\":\\\"s\\\",\\\"v\\\":\\\"${OFFSET}\\\"},{\\\"n\\\":\\\"e\\\",\\\"v\\\":\\\"$((OFFSET + ${#WORD}))\\\"}]}\"
  }"
```

✓ tested — comment appears anchored to the word in Google Docs UI.

---

## Google Sheets

**Spreadsheet ID** is in the URL:
`https://docs.google.com/spreadsheets/d/<SPREADSHEET_ID>/edit`

### Read cell range

```bash
SS_ID="<spreadsheet_id>"
RANGE="Sheet1!A1:D10"   # A1 notation

curl -s -H "Authorization: Bearer $TOKEN" \
  "https://sheets.googleapis.com/v4/spreadsheets/${SS_ID}/values/${RANGE}" \
  | jq '.values'
```

⚠ not tested — same ADC auth pattern as Docs.

### Write cell range

```bash
SS_ID="<spreadsheet_id>"
RANGE="Sheet1!A1"

curl -s -X PUT \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "https://sheets.googleapis.com/v4/spreadsheets/${SS_ID}/values/${RANGE}?valueInputOption=RAW" \
  -d '{"values": [["value1", "value2"], ["value3", "value4"]]}'
```

⚠ not tested.

### Append rows

```bash
curl -s -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "https://sheets.googleapis.com/v4/spreadsheets/${SS_ID}/values/${RANGE}:append?valueInputOption=RAW&insertDataOption=INSERT_ROWS" \
  -d '{"values": [["new_row_col1", "new_row_col2"]]}'
```

⚠ not tested.

---

## Google Slides

**Presentation ID** is in the URL:
`https://docs.google.com/presentation/d/<PRESENTATION_ID>/edit`

### Read presentation structure

```bash
PRES_ID="<presentation_id>"

curl -s -H "Authorization: Bearer $TOKEN" \
  "https://slides.googleapis.com/v1/presentations/${PRES_ID}" \
  | jq '[.slides[] | {slideId: .objectId, elements: [.pageElements[]? | {id: .objectId, text: .shape.text.textElements[]?.textRun.content}]}]'
```

⚠ not tested.

### Update text in a shape

```bash
PRES_ID="<presentation_id>"
SHAPE_ID="<objectId from above>"
NEW_TEXT="Updated slide text"

curl -s -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "https://slides.googleapis.com/v1/presentations/${PRES_ID}:batchUpdate" \
  -d "{\"requests\":[
    {\"deleteText\":{\"objectId\":\"${SHAPE_ID}\",\"textRange\":{\"type\":\"ALL\"}}},
    {\"insertText\":{\"objectId\":\"${SHAPE_ID}\",\"insertionIndex\":0,\"text\":\"${NEW_TEXT}\"}}
  ]}"
```

⚠ not tested.

---

## Scope reference

| Operation | Minimum scope |
|-----------|--------------|
| Read Doc / Sheet / Slide content | `documents.readonly` + `drive.readonly` |
| Write / edit Doc / Sheet / Slide | `documents` + `drive` |
| Add / read Drive comments | `drive` (write) or `drive.readonly` (read) |
| List files in Drive | `drive.readonly` or `drive.metadata.readonly` |

Always include `cloud-platform` in the `--scopes` list — required for gcloud ADC to issue a token.

## Troubleshooting

| Error | Cause | Fix |
|-------|-------|-----|
| HTTP 401 | Token expired | `TOKEN=$(gcloud auth application-default print-access-token)` |
| HTTP 403 `insufficientPermissions` | Missing scope | Re-run `gcloud auth application-default login` with write scopes |
| HTTP 403 `forbidden` | Not shared with your Google account | Share the doc with the Google account used for gcloud auth |
| `anchor` comment not pinned visually | Anchor JSON format issue | Verify `n:s` / `n:e` offsets — must be character positions within the document plain text |
