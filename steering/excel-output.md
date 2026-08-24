# Excel Output — Formatting Rules

This document defines how the Excel workbook is structured and formatted by `scripts/generate_excel.py`.

---

## File Naming

```
aws-inventory-<accountId>-<YYYYMMDD-HHMM>.xlsx
```

Example: `aws-inventory-593845248500-20260824-1430.xlsx`

Output directory: `./inventory-reports/` (configurable via scope file).

---

## Workbook Structure

The workbook contains:
1. **Summary** sheet (always first)
2. One sheet per service category that has resources (skip empty categories)
3. **ScanNotes** sheet (always last) — errors, access-denied, skipped services

---

## Summary Sheet

The first sheet provides an overview of the entire inventory.

### Layout:

| Row | Content |
|-----|---------|
| 1 | **AWS Infrastructure Inventory Report** (merged, bold, 16pt) |
| 3 | Account ID: `<value>` |
| 4 | Caller Identity: `<arn>` |
| 5 | Scan Date: `<YYYY-MM-DD HH:MM UTC>` |
| 6 | Regions Scanned: `<comma-separated>` |
| 7 | Total Resources: `<count>` |
| 9 | **Resource Summary by Category** (bold, 14pt) |
| 10+ | Table: Category | Resource Count | Regions |

### Summary table columns:
| Category | Resource Count | Regions Found |
|----------|---------------|---------------|

Sort by resource count descending.

---

## Service Sheets

Each service category gets its own sheet with:

### Header Row (Row 1):
- Bold text
- Background color: `#1F4E79` (dark blue)
- Font color: White
- Auto-filter enabled on all columns
- Freeze panes (freeze row 1 so headers stay visible when scrolling)

### Data Rows (Row 2+):
- Standard font (Calibri 11pt)
- Alternate row shading: white / `#F2F7FC` (light blue) for readability
- Text wrapping enabled for long content (Tags, ARNs)
- Column widths auto-fitted to content (min 12, max 50 characters)

### Column formatting rules:
| Data Type | Format |
|-----------|--------|
| Dates | `YYYY-MM-DD HH:MM` |
| Numbers | Comma-separated with no decimals (e.g., `1,234`) |
| Booleans | "Yes" / "No" (not True/False) |
| ARNs | Full ARN, no truncation |
| Tags | `Key=Value; Key2=Value2` (semicolon-separated) |
| IPs | Plain text (no number formatting) |
| Sizes (bytes) | Convert to human-readable: KB/MB/GB/TB |
| Costs | USD with 2 decimals: `$1,234.56` |
| Lists | Comma-separated |
| Null/Empty | Leave cell blank (not "None" or "null") |

### Tags column:
- Format: `Environment=Production; Team=Backend; CostCenter=12345`
- Mask any tag whose key contains (case-insensitive): `password`, `secret`, `key`, `token`, `credential`
  - Show: `Key=****MASKED****`
- If no tags: leave blank

---

## ScanNotes Sheet

The last sheet captures operational notes from the scan.

### Columns:
| Timestamp | Region | Service | Issue Type | Details |

### Issue Types:
- `ACCESS_DENIED` — IAM permission missing
- `THROTTLED` — API rate limit hit (include retry count)
- `NOT_AVAILABLE` — Service not available in region
- `NO_RESOURCES` — No resources found (informational)
- `ERROR` — Unexpected error (include error message)

---

## Styling Constants

```python
HEADER_FILL = "1F4E79"       # Dark blue header background
HEADER_FONT_COLOR = "FFFFFF" # White header text
ALT_ROW_FILL = "F2F7FC"     # Light blue alternate rows
TITLE_FONT_SIZE = 16         # Summary title
SUBTITLE_FONT_SIZE = 14      # Section headers
BODY_FONT = "Calibri"
BODY_FONT_SIZE = 11
MIN_COL_WIDTH = 12
MAX_COL_WIDTH = 50
```

---

## Sheet Naming Rules

- Sheet names max 31 characters (Excel limit)
- No special characters: `[ ] : * ? / \`
- Use the short names from the inventory-workflow.md (e.g., `EC2-Instances`, `S3`, `Lambda`)
- If a sheet name would be duplicated, append region suffix: `WAF-Global`, `WAF-Regional`

---

## Performance Considerations

- For sheets with > 10,000 rows: disable auto-fit column width (use fixed widths instead)
- Write data in streaming mode (`openpyxl.worksheet.write_only`) if > 50,000 rows
- Maximum rows per sheet: 1,048,576 (Excel limit) — if exceeded, split into multiple sheets with suffix `_1`, `_2`

---

## Data Input Format

The Python script reads a JSON file (`inventory-data.json`) with this structure:

```json
{
  "metadata": {
    "accountId": "593845248500",
    "callerArn": "arn:aws:iam::593845248500:user/admin",
    "scanDate": "2026-08-24T14:30:00Z",
    "regions": ["us-east-1", "eu-west-1"],
    "totalResources": 1234
  },
  "categories": {
    "EC2-Instances": {
      "displayName": "EC2 Instances",
      "columns": ["Region", "Instance ID", "Name", "Type", "State", "VPC ID", "Private IP", "Public IP", "Launch Time", "Tags"],
      "data": [
        ["us-east-1", "i-0abc123", "web-server-1", "t3.medium", "running", "vpc-123", "10.0.1.5", "3.14.15.92", "2026-01-15 08:30", "Environment=Production; Team=Web"],
        ...
      ]
    },
    ...
  },
  "scanNotes": [
    {
      "timestamp": "2026-08-24T14:31:00Z",
      "region": "us-east-1",
      "service": "Redshift",
      "issueType": "NO_RESOURCES",
      "details": "No Redshift clusters found"
    },
    ...
  ]
}
```

---

## Validation Before Writing

Before generating the Excel file:
1. Verify all category data arrays have consistent row lengths matching column count.
2. Verify no cell value exceeds 32,767 characters (Excel cell limit) — truncate with `...` if needed.
3. Verify sheet names are unique and within 31 characters.
4. Verify the output directory exists (create if missing).
