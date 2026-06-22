# Data_Conversion
COBOL toolkit to parse copybook PIC definitions and extract filtered data from EBCDIC binary files. CLL31 builds a field schema table from any copybook; CLL41 applies criteria filters and writes pipe-delimited ASCII output with full EBCDIC-to-ASCII conversion.
# 📂 COBOL Copybook Parser & EBCDIC Data Extractor

> **Platform:** Standard COBOL (Windows / Mainframe-compatible) | **Language:** COBOL | **Author:** Nexapro Technologies

A two-program COBOL toolkit that automates the parsing of COBOL copybook definitions and extracts filtered data from raw EBCDIC binary data files. The suite handles 1D and 2D OCCURS table structures, all PIC clause variants, COMP/COMP-3 packed decimal types, and performs full EBCDIC-to-ASCII byte-level conversion on output.

---

## 📁 Module Overview

| File | Program ID | Purpose |
|------|------------|---------|
| `CLL31.CBL` | `CLL31` | Copybook Parser & Schema Builder — reads a COBOL copybook, parses all PIC clauses and data types, and builds an in-memory script table |
| `CLL41.CBL` | `CLL41` | Data Extractor & Converter — reads raw EBCDIC binary data using the script table, applies filter criteria, and writes delimited ASCII output |

---

## 🔍 CLL31 — Copybook Parser & Schema Builder

**File:** `CLL31.CBL`

The controller program. Reads a card/control file that specifies which copybook to parse, which data file to process, and what filter criteria to apply. Builds a complete field schema table from the copybook and passes it to `CLL41` via a `CALL`.

### Key Features

- **Multi-Job Card Processing:** Reads a control file (`DETAILCARD.TXT`) where each record defines a copybook path, output file path, and filter criteria — supports batch processing of multiple copybook-data file pairs in a single run.
- **PICTURE Normalization:** Automatically replaces full `PICTURE` keyword with `PIC` before parsing to handle both notations uniformly.
- **Multi-line Record Handling:** Detects copybook records that span multiple lines (no period on line end) and concatenates them before parsing — handles real-world copybooks with wrapped definitions.
- **REDEFINES & 88-level Bypass:** Silently skips `REDEFINES` groups and `88`-level condition names, which carry no physical data length.
- **OCCURS Table Detection (1D & 2D):** Identifies both single-dimension (`WS-1D-TABLE`) and two-dimension (`WS-2D-TABLE`) OCCURS structures and processes them through dedicated table-expansion routines.
- **PIC Clause Parser (`204-DETERMINE-CASE`):** Handles all major PIC clause patterns:
  - Simple PIC without brackets (`PIC X`, `PIC 9`)
  - PIC with explicit length brackets (`PIC X(10)`, `PIC 9(5)`)
  - PIC with VALUE clause (with and without decimal)
  - PIC with decimal `V` indicator (with and without brackets)
  - PIC with decimal and VALUE clause combined
- **COMP Storage Length Calculation (`900-CHECK-COMP-LENGTH`):** Converts logical digit count to physical COMP binary storage bytes (2/4/8 bytes depending on digit range).
- **COMP-3 Packed Decimal Length Calculation:** Computes actual packed byte length using `DIVIDE LEN BY 2` with remainder rounding.
- **Data Type Classification:** Tags each parsed field as `INT`, `DECIMAL`, `CHAR`, `VARCHAR`, `REAL` (COMP-1), or `FLOAT` (COMP-2) in the script table.
- **Filter Criteria Parsing:** Supports four filter modes passed to CLL41:
  - `WS-CRT-SPACE` — extract all records (no filter)
  - `WS-CRT-NOO-RANGE` — match exact value at field position
  - `WS-CRT-YES-RANGE` — match value within a hyphen-delimited range
  - `WS-CRT-LAST-ROW` — multi-value exclusion list (semicolon-delimited)
- **Copybook Record Length Output:** Writes a physical FD record length stub to `COPFD.cpy.txt` for use by CLL41.

### In-Memory Tables

| Table | Size | Purpose |
|-------|------|---------|
| `WS-HOLD-TABLE` | 200 entries | Staging area for OCCURS group members during table expansion |
| `WS-SCRIPT-TABLE` | 500 entries | Final field schema: name, data type, length, decimal flag/position, COMP flag |
| `WS-CRITERIA-TABLE` | 50 entries | Filter criteria values |

### Processing Flow

```
Read DETAILCARD.TXT
    │
    ▼ (per card record)
052-EXTRACT-NAMES → parse copybook path, output file, criteria
    │
    ▼
055-PROCESS-CARD-FILE
    ├── 100-OPEN-INPUT      → open copybook file
    ├── 140-INI-TABLE       → zero/clear both tables
    ├── LOOP: 102-START-CONVERSION-PARA
    │       ├── Skip: comments, 88-levels, REDEFINES groups
    │       ├── 151-OCCUR-RECORDS  → handle OCCURS (1D/2D table expansion)
    │       └── 200-INSIDE-CON-PARA → parse PIC clause → fill WS-SCRIPT-TABLE
    ├── UPDATE-COPYBOOK-REC-LENGTH → write FD stub to COPFD.cpy.txt
    └── CALL 'CLL41' → pass script table + file names + criteria
```

---

## ⚙️ CLL41 — Data Extractor & EBCDIC Converter

**File:** `CLL41.CBL`

Called by CLL31. Receives the field schema table and filter parameters via Linkage Section. Opens the raw binary data file, applies the filter criteria record by record, and writes pipe-delimited (`|^`) ASCII output for each matching record.

### Key Features

- **EBCDIC-to-ASCII Conversion:** Full 256-byte EBCDIC and ASCII translation tables are embedded as literal hex data and redefined as indexed arrays (`ASCII-BYTE`, `EBCDIC-BYTE`). Field-level conversion is applied via `INSPECT ... CONVERTING E-INFO TO A-INFO`.
- **Four Filter Modes (matching CLL31 criteria):**
  - `430-WS-CRT-SPACE` — write every record as-is
  - `430-WS-CRT-NOO-RANGE` — write only records where the criteria field exactly matches the specified value
  - `430-WS-CRT-YES-RANGE` — write records where the criteria field falls within a start–end range
  - `430-WS-CRT-LAST-ROW` — write records except those matching any value in a 20-entry exclusion table (validated in `435-VALIDATE-REC`)
- **Character Field Conversion (`480-CHAR-CONVERSION`):** Byte-by-byte EBCDIC-to-ASCII mapping for `VARCHAR` and `CHAR` fields, covering the full printable EBCDIC character set including alphabetics, numerics, punctuation, and special characters.
- **Numeric/COMP-3 Field Conversion (`490-INT-CONVERSION` + `497-CONVERSION-COMP-VALUE`):** Converts packed decimal and binary numeric fields to printable decimal strings, including sign detection (`C` = positive, `D` = negative) and decimal point insertion at the correct position.
- **Pipe-Delimited Output Format:** Each output record is prefixed and field-delimited by `|^` separator, making it directly loadable into databases or spreadsheet tools.
- **COMP-3 Zero Suppression:** Uses an embedded `ALL-20-TABLE` (hex `40` patterns of increasing length) to detect and replace all-zero packed fields with `'0'`.
- **Read/Write Counters:** Tracks and displays `READ-COUNT` and `WRITE-COUNT` per output file for audit visibility.

### Linkage Section Parameters (received from CLL31)

| Parameter | Type | Description |
|-----------|------|-------------|
| `LS-SCRIPT-TABLE` | Table (500 entries) | Field schema from copybook parser |
| `LS-FILE-NAMES` | Group | Data file path + output file path |
| `LS-CRT-STRT-POS` | PIC X(3) | Filter field start position |
| `LS-CRT-LEN` | PIC X(3) | Filter field length |
| `LS-FINAL-CRITERIA` | PIC X(100) | Filter value(s) |
| `LS-CRITERIA-DEFINE` | PIC X(1) | Filter mode flag (S/N/Y/L) |

### Data Flow

```
Open DATCPY (raw EBCDIC binary)
Open OUTCPYONE (ASCII delimited output)
    │
    ▼ (per record)
Apply criteria filter (NOO-RANGE / YES-RANGE / LAST-ROW / SPACE)
    │
    ▼ (matching records only)
470-DATA-BRK: iterate WS-SCRIPT-TABLE fields
    ├── CHAR/VARCHAR fields → 480-CHAR-CONVERSION (byte-level EBCDIC→ASCII)
    └── INT/DECIMAL/COMP fields → 490-INT-CONVERSION → 497-CONVERSION-COMP-VALUE
    │
    ▼
Write pipe-delimited record to OUTCPYONE
```

---

## 🗂️ File Dependencies

| File | Direction | Description |
|------|-----------|-------------|
| `DETAILCARD.TXT` | Input | Control file: one record per job (copybook path; output path; criteria) |
| `<copybook>.cpy` | Input | COBOL copybook file to parse (path from DETAILCARD) |
| `COPFD.cpy.txt` | Output/Input | Generated FD stub with physical record length; included by CLL41 via COPY |
| `<data file>` | Input | Raw EBCDIC binary data file (path from DETAILCARD) |
| `<output file>` | Output | Pipe-delimited ASCII extracted records (path from DETAILCARD) |

---

## 📋 Control File Format (`DETAILCARD.TXT`)

Each record in the card file is semicolon-delimited:

```
<copybook_path>;<output_file_path>;<criteria>
```

**Criteria formats:**

| Criteria | Mode | Example |
|----------|------|---------|
| *(blank)* | Extract all records | `;` |
| `SSS LLL VALUE` | Exact match (no range) | `001 003 ABC` |
| `SSS LLL VAL1-VAL2` | Range match | `001 003 AAA-ZZZ` |
| `SSS LLL ;N VAL1;Y VAL2-VAL3;...` | Multi-value exclusion list | `001 003 ;NXXX;YAAA-BBB` |

Where `SSS` = start position (3 digits), `LLL` = length (3 digits).

---

## 🛠️ Technical Environment

| Item | Value |
|------|-------|
| Language | Standard COBOL (MicroFocus / GnuCOBOL compatible) |
| Platform | Windows (file paths use `C:\BACKUP2\`) or Mainframe with path override |
| Data Encoding | EBCDIC input → ASCII output |
| File Organisation | LINE SEQUENTIAL (copybook, control), SEQUENTIAL (data files) |
| Output Format | Pipe-delimited (`\|^`) flat file |
| Numeric Types Supported | CHAR, VARCHAR, INT, DECIMAL, COMP (binary), COMP-1 (REAL), COMP-2 (FLOAT), COMP-3 (packed decimal) |

---

## ⚠️ Known Limitations & Notes

| Item | Description |
|------|-------------|
| **Hardcoded paths** | `DETAILCARD.TXT` and `COPFD.cpy.txt` paths are hardcoded to `C:\BACKUP2\`. Update for your environment. |
| **Table size limits** | `WS-HOLD-TABLE` is capped at 200 entries; `WS-SCRIPT-TABLE` at 500. Large copybooks may need these increased. |
| **No REDEFINES expansion** | REDEFINES groups with PIC clauses are skipped entirely — only the base field is counted. |
| **COMP-3 sign detection** | Sign nibble detection is hardcoded to scan the last 18 hex positions in `497-CONVERSION-COMP-VALUE`. Very long packed fields may need adjustment. |
| **CLL41 exclusion table** | The `WS-CRT-LAST-ROW` exclusion list supports a maximum of 20 values per run. |
| **No nested OCCURS** | OCCURS within OCCURS is partially handled (2D tables) but deeper nesting is not supported. |

---

## 🏢 About

Developed by **Nexapro Technologies** — IBM i, Mainframe & Legacy Modernization specialists.

📧 nexaprotechnology@gmail.com | 🌐 nexaprotechnologies.com | 📞 +91 63502 82519
