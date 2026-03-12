# PHASE 0 — FULL INSPECTION
# Read everything. Report everything. Write nothing.

---

## YOUR ONLY JOB IN THIS PHASE

**Read. Analyze. Report.**
Do NOT write code. Do NOT create files. Do NOT modify anything.
Do NOT move to Phase 1 until you have shown the final verdict.

---

## ══════════════════════════════════════════
## TASK 1 — MAP THE PROJECT STRUCTURE
## ══════════════════════════════════════════

Before opening any file, print the complete directory tree of `aiShopzawy/`:

```
Run mentally or via terminal:
  find aiShopzawy/ -type f | sort
```

Report in this format:

```
PROJECT STRUCTURE
─────────────────────────────────────────────────────
aiShopzawy/
  data/
    [list every .json file with path + size in KB/MB]
  rag/
    [list every .py file]
  agent/
    [list every .py file]
  database/
    [list every .py and .db file]
  [any other folders]

.env file exists?  → YES / NO
  If YES → list variable names only (not values):
    [GROQ_API_KEY, DATABASE_URL, ...]
─────────────────────────────────────────────────────
```

---

## ══════════════════════════════════════════
## TASK 2 — INSPECT ALL JSON FILES
## ══════════════════════════════════════════

For EVERY `.json` file found inside `aiShopzawy/`:

### 2.1 — File Header

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📄 FILE     : [full path]
📦 SIZE     : [size in KB or MB]
📊 RECORDS  : [exact count]
🏗  STRUCTURE: [top-level array / object with key "products" / nested / other]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

### 2.2 — First Record (complete, zero truncation)

Paste the entire first record exactly as it appears in the file.
If it's longer than 100 fields, still paste it all.

```
FIRST RECORD — FULL:
────────────────────
[paste here]
```

### 2.3 — Critical Field Audit

After reading the first record, fill this table using REAL field names from the JSON.
Do not guess. Do not assume. Use only what you see.

```
┌─────────────────────────────────────────────────────────────────────┐
│  💰  PRICING                                                         │
├──────────────────────────────┬──────────────────┬───────────────────┤
│  Field                       │  Real Name Found  │  Sample Value     │
├──────────────────────────────┼──────────────────┼───────────────────┤
│  Base / original price       │                  │                   │
│  Sale / discounted price     │                  │                   │
│  Discount percent            │                  │                   │
│  Price data type             │  [int/float/str/null]               │
│  Price location              │  [top-level / nested path / variants only] │
└──────────────────────────────┴──────────────────┴───────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│  📦  STOCK                                                           │
├──────────────────────────────┬──────────────────┬───────────────────┤
│  Stock quantity field        │                  │                   │
│  In-stock boolean field      │                  │                   │
│  Stock location              │  [top-level / variants only / computed] │
└──────────────────────────────┴──────────────────┴───────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│  🆔  IDENTITY                                                        │
├──────────────────────────────┬──────────────────┬───────────────────┤
│  Primary ID                  │                  │                   │
│  Arabic name                 │                  │                   │
│  English name                │                  │                   │
│  Slug                        │                  │                   │
└──────────────────────────────┴──────────────────┴───────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│  🏷  CATEGORIZATION                                                  │
├──────────────────────────────┬──────────────────┬───────────────────┤
│  Category field name         │                  │                   │
│  Category value type         │  [flat string / nested object / ID only] │
│  Category full value         │                  │                   │
│  Brand field name            │                  │                   │
│  Brand value type            │  [flat string / nested object / ID only] │
│  Brand full value            │                  │                   │
└──────────────────────────────┴──────────────────┴───────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│  🖼  MEDIA                                                           │
├──────────────────────────────┬──────────────────┬───────────────────┤
│  Primary image field         │                  │                   │
│  Image value type            │  [direct URL / nested {url, path}]  │
│  Images array field          │                  │                   │
│  Images array count (first record) │           │                   │
└──────────────────────────────┴──────────────────┴───────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│  🔀  VARIANTS                                                        │
├──────────────────────────────┬──────────────────┬───────────────────┤
│  Variants field name         │                  │                   │
│  Variant count (first record)│                  │                   │
│  First variant — FULL PASTE: │                                      │
│  [paste complete first variant object]                              │
│  Fields inside variant:      │                                      │
│    price/sale_price field    │                  │                   │
│    stock/quantity field      │                  │                   │
│    color field               │                  │                   │
│    size field                │                  │                   │
│    SKU field                 │                  │                   │
└──────────────────────────────┴──────────────────┴───────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│  🕐  TIMESTAMPS                                                      │
├──────────────────────────────┬──────────────────┬───────────────────┤
│  Updated-at field            │                  │                   │
│  Created-at field            │                  │                   │
│  Timestamp format            │  [ISO 8601 / Unix / text / MISSING] │
└──────────────────────────────┴──────────────────┴───────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│  📝  EXTRA                                                           │
├──────────────────────────────┬──────────────────┬───────────────────┤
│  Description field           │                  │  first 100 chars  │
│  Tags field + format         │                  │                   │
│  is_published field          │                  │                   │
│  is_featured field           │                  │                   │
│  Any other useful fields     │                  │                   │
└──────────────────────────────┴──────────────────┴───────────────────┘
```

---

## ══════════════════════════════════════════
## TASK 3 — FULL DATA QUALITY SCAN
## ══════════════════════════════════════════

Scan EVERY record in EVERY JSON file (not just the first one).
Count each problem with precision.

```
FULL DATA QUALITY REPORT
══════════════════════════════════════════════════════════════
Total files scanned      : [N]
Total records scanned    : [N]
══════════════════════════════════════════════════════════════

💰  PRICING PROBLEMS
────────────────────────────────────────────────
  base_price = 0 or null           : [N] records  ([X]%)
  sale_price missing or null       : [N] records  ([X]%)
  sale_price = 0                   : [N] records  ([X]%)
  price is string not number       : [N] records  ([X]%)
  price exists only inside variants: [N] records  ([X]%)
  base_price == sale_price         : [N] records  (no discount — normal ✓)

📦  STOCK PROBLEMS
────────────────────────────────────────────────
  stock field completely missing   : [N] records  ([X]%)
  stock = 0 (out of stock)         : [N] records  ([X]%)  ← expected, OK
  stock exists only inside variants: [N] records  ([X]%)

🆔  IDENTITY PROBLEMS
────────────────────────────────────────────────
  id field missing                 : [N] records  ([X]%)
  duplicate IDs across all records : [N] duplicates
  name / name_ar missing           : [N] records  ([X]%)

🏷  CATEGORIZATION PROBLEMS
────────────────────────────────────────────────
  category missing entirely        : [N] records  ([X]%)
  category is ID-only (no name)    : [N] records  ([X]%)
  brand missing entirely           : [N] records  ([X]%)
  brand is ID-only (no name)       : [N] records  ([X]%)

🖼  MEDIA PROBLEMS
────────────────────────────────────────────────
  no image at all                  : [N] records  ([X]%)

📋  PUBLISH / ACTIVE STATE
────────────────────────────────────────────────
  is_published = false             : [N] records  ([X]%)
  is_published field missing       : [N] records  ([X]%)

══════════════════════════════════════════════════════════════
CRITICAL ERROR RATE:
  (price=0 OR price_missing) AND (stock_missing) → [N] records  ([X]%)
══════════════════════════════════════════════════════════════
```

---

## ══════════════════════════════════════════
## TASK 4 — INSPECT THE RAG (CHROMA) CODE
## ══════════════════════════════════════════

Search recursively inside `aiShopzawy/` for every Python file.
Find the one that initializes Chroma and defines a search function.
Read it completely. Report:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
RAG INSPECTION REPORT
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Main RAG file path      : [full path]
Chroma DB path          : [full path as it appears in code]
Collection name         : [exact name string]
Embedding model         : [exact model name string]

Search function signature:
  Name       : [function name]
  Parameters : [list all params with types]
  Returns    : [describe return type and structure]

  Example expected:
    def search(query: str, top_k: int = 5) -> dict
    Returns: {"documents": [str, ...], "metadatas": [dict, ...], "ids": [...]}

Metadata fields stored per product in Chroma:
  [list every key stored in metadata]
  Example: id, name_ar, price, category, image_url, ...

Does RAG return image URLs?    YES / NO
Does RAG support top_k param?  YES / NO
Is Chroma persistent (on disk)? YES / NO  — path: [...]

ACTUAL SEARCH TEST:
  Call search("موبايل") or search("mobile") and paste the complete raw output:
  ─────────────────────────────────────────────────────
  [paste full output here — do not truncate]
  ─────────────────────────────────────────────────────

Current RAG ↔ LLM flow (describe step by step):
  Step 1: [...]
  Step 2: [...]
  ...
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## ══════════════════════════════════════════
## TASK 5 — INSPECT THE GROQ CONNECTION
## ══════════════════════════════════════════

Find the file where Groq is initialized. Read it completely. Report:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
GROQ INSPECTION REPORT
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Groq file path          : [full path]
Model name              : [exact string — e.g. "llama-3.3-70b-versatile"]
API key env variable    : [e.g. GROQ_API_KEY]

System prompt (paste in full):
  ─────────────────────────────────────────────────────
  [paste complete system prompt — no truncation]
  ─────────────────────────────────────────────────────

Message building flow:
  [describe exactly how messages array is built per turn]

Uses Function Calling?     YES / NO
  If YES → list tool names defined:
    - [tool name 1]
    - [tool name 2]
    ...

Maintains conversation history?  YES / NO
  If YES → how many turns kept?  [N]

Temperature setting   : [value]
Max tokens setting    : [value]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## ══════════════════════════════════════════
## TASK 6 — CHECK POSTGRESQL MIGRATIONS
## ══════════════════════════════════════════

This is needed in case the JSON verdict is REBUILD.
Search the monorepo for migration files (`.sql`, `migrations/`, `schema.prisma`, etc.)

Report what you find:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
POSTGRESQL SCHEMA INSPECTION
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Migration files found:
  [list full paths]

Products table:
  Table name           : [exact name]
  Arabic name column   : [exact column name]
  Price column         : [exact column name]
  Sale price column    : [exact column name]
  Stock column         : [exact column name]
  Updated-at column    : [exact column name]
  Published flag       : [exact column name]

Related tables:
  Categories table     : [name + join column]
  Brands table         : [name + join column]
  Variants table       : [name + join column]
  Images table         : [name + join column]

DATABASE_URL in .env:
  Format detected      : [postgresql://... — show only the pattern, not credentials]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

If no migration files exist → write:
`No migration files found — cannot generate rebuild SQL without schema confirmation.`

---

## ══════════════════════════════════════════
## TASK 7 — FINAL VERDICT
## ══════════════════════════════════════════

After completing Tasks 1–6, give one clear verdict.

### Scoring Rules

| Score | Verdict | Condition |
|-------|---------|-----------|
| 90–100% | ✅ PERFECT | price, sale_price, stock, name, id, category, brand all present and clean |
| 60–89% | ⚠️ FIXABLE | 1–2 secondary fields missing or in wrong format — fixable during conversion |
| < 60% | 🔴 REBUILD | price OR stock OR id missing in >20% of records — must pull fresh from PostgreSQL |

---

```
╔══════════════════════════════════════════════════════════════╗
║                    INSPECTION VERDICT                        ║
╚══════════════════════════════════════════════════════════════╝

Total products inspected : [N]
Critical error rate      : [X]%
Data quality score       : [X]%

VERDICT: [ PERFECT ✅ / FIXABLE ⚠️ / REBUILD 🔴 ]

══════════════════════════════════════════════════════════════

IF FIXABLE — list problems and their conversion-time fixes:

  Problem 1: [describe]
  Fix:       [how to handle it in build_sqlite.py]

  Problem 2: [describe]
  Fix:       [how to handle it in build_sqlite.py]

══════════════════════════════════════════════════════════════

IF REBUILD — write the complete PostgreSQL query below.
  The query must:
    ✓ JOIN products + categories + brands + product_variants + product_images
    ✓ Compute sale_price correctly (with or without discount table)
    ✓ Compute discount_percent = ROUND((base - sale) / base * 100, 1)
    ✓ SUM stock from variants
    ✓ Get primary image URL
    ✓ Build variants as a JSON array
    ✓ Filter: is_published = true only
    ✓ Use EXACT column names found in Task 6 migrations

  Save output to: aiShopzawy/data/raw/products_fresh.json
  Then STOP. Wait for developer to run the query and provide new JSON.

══════════════════════════════════════════════════════════════

IF PERFECT or FIXABLE — write this block exactly:

"Inspection complete ✅ — ready to proceed to PHASE 1"

FIELD MAPPING FOR PHASE 1 (copy this into build_sqlite.py):
┌─────────────────────────────────────────────────────────────┐
│  JSON field → SQLite column mapping                         │
├────────────────────────────┬────────────────────────────────┤
│  id                        │  [exact JSON field name]       │
│  name_ar                   │  [exact JSON field name]       │
│  name_en                   │  [exact JSON field name]       │
│  slug                      │  [exact JSON field name]       │
│  description               │  [exact JSON field name]       │
│  base_price                │  [exact JSON field name]       │
│  sale_price                │  [exact JSON field name]       │
│  discount_percent          │  [computed / exact field name] │
│  stock_quantity            │  [exact JSON field name / sum of variants] │
│  in_stock                  │  [exact JSON field name / computed] │
│  category_name             │  [exact path: e.g. category.name] │
│  brand_name                │  [exact path: e.g. brand.name] │
│  image_url                 │  [exact JSON field name]       │
│  images                    │  [exact JSON field name]       │
│  variants                  │  [exact JSON field name]       │
│  updated_at                │  [exact JSON field name / MISSING] │
│  is_published              │  [exact JSON field name / default true] │
│  is_featured               │  [exact JSON field name / default false] │
├────────────────────────────┴────────────────────────────────┤
│  RAG import:  from [path] import [function_name]            │
│  RAG returns: [describe return structure]                   │
│  SQLite path: aiShopzawy/database/shopzawy.db               │
└─────────────────────────────────────────────────────────────┘
```

---

## ══════════════════════════════════════════
## HARD RULES — DO NOT VIOLATE
## ══════════════════════════════════════════

```
✅  Read EVERY record in EVERY JSON file — not just the first
✅  Report EXACT field names as they appear in the JSON
✅  Run the actual RAG search and paste the real output
✅  Calculate percentages for every problem count
✅  Fill the field mapping table completely before stopping
❌  Do NOT write any Python code
❌  Do NOT create any files
❌  Do NOT modify any existing file
❌  Do NOT move to PHASE 1 without showing the final verdict block
❌  Do NOT skip TASK 6 — even if you think the JSON is fine
```

---

**Start with TASK 1 now.**
```
