---
description: "Fetch a SharePoint or OneDrive file and save Markdown output in the spec directory"
---

# M365 File Fetch to Markdown

You are fetching a file from Microsoft 365 storage and saving it as Markdown.
This command performs ingestion only. Do not generate a specification in this command.

Follow each step sequentially. Do not skip steps.

---

## Step 0 — Pre-flight Checks

Run these checks before proceeding. Stop and report if any check fails.

### 0a. Verify M365 CLI is installed

```bash
m365 version
```

### 0b. Verify authentication

```bash
m365 status -o json
```

If not authenticated, stop and instruct the user to run `m365 login`.

---

## Step 1 — Resolve File Target and Output Path

Parse user input for:

| Parameter | Description |
|-----------|-------------|
| `source` | `sharepoint` or `onedrive` (default: `sharepoint`) |
| `fileWebUrl` | SharePoint file URL |
| `fileId` | OneDrive file ID |
| `output` | Optional output Markdown path |

### 1a. Resolve file reference

If `source=sharepoint`, require `fileWebUrl`.
If `source=onedrive`, require `fileId`.

### 1b. Resolve output path

If `output` is provided, use it.

If `output` is not provided:
- derive a readable base name from file URL or ID
- default to `.specify/specs/m365-file-<base-name>.md`

Sanitize filename characters to `a-z`, `A-Z`, `0-9`, `-`, `_`, and `.`.

### 1c. Enforce workspace-safe output

Resolve absolute output path and verify it is inside the workspace root.
If outside workspace, stop and request a workspace-local path.

### 1d. Confirm with the user

Show resolved source, file reference, and output path.
Ask for confirmation before fetch.

---

## Step 2 — Download File

### 2a. Download to a temporary local file

For SharePoint:

```bash
m365 spo file get --webUrl <FILE_WEB_URL> --asFile --path <TEMP_FILE>
```

For OneDrive:

```bash
m365 onedrive file get --id <FILE_ID> --asFile --path <TEMP_FILE>
```

If download fails, report the exact failure reason and stop.

### 2b. Detect file type

Use file extension to classify the file:
- **Text formats** (`.md`, `.txt`, `.csv`, `.json`, `.xml`, `.html`): proceed to Step 3.
- **Binary formats** (`.pdf`, `.docx`, `.pptx`, `.xlsx`, images, audio): stop and tell the user:
  > This file is a binary format that cannot be converted to Markdown directly. For richer document conversion, compose this extension with `spec-kit-markitdown` via a Spec Kit workflow.

Security check:
- Never write auth tokens, cookies, headers, or CLI credential data into output Markdown.

---

## Step 3 — Convert and Write Markdown

### 3a. Convert content to Markdown

- Normalize to UTF-8
- If HTML, strip tags while preserving readable text structure
- Preserve headings/tables where possible

### 3b. Add source metadata header

Ensure output Markdown begins with:

```markdown
---
Source:
  type: "m365-file"
  sourceSystem: "sharepoint|onedrive"
  fileReference: "<fileWebUrl|fileId>"
  fetchedAt: "<ISO timestamp>"
---
```

If conversion produced Markdown without frontmatter, prepend this block.

### 3c. Cleanup temporary file

Delete temporary downloaded file after output is written successfully.

### 3d. Verify output

Confirm output file exists and is non-empty.

---

## Step 4 — Output Summary

Display:

```
✅ File fetched and converted to Markdown

📄 Output: <output-path>
📊 Summary:
   • Source: <sharepoint|onedrive>
   • File reference: <fileWebUrl|fileId>
   • Conversion: direct text

💡 Next Step:
   Once saved, use this file as input to `speckit specify` to generate a specification with your presets applied.
```

---

## Notes

- Least-privilege delegated permissions: `Files.Read` (OneDrive) and `Sites.Read.All` when reading SharePoint-hosted files.
- Keep all writes inside the workspace.
- This command is ingestion-only and must not generate a spec.
