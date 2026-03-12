# 🔍 Cursor Agent — JSON Structure Inspector & Validator

---

## MISSION

Inspect ALL JSON files in the `aiShopzawy/` folder and tell me:
1. Is the current JSON good enough to build SQLite from?
2. What's missing or wrong?
3. What exact SQL query should I run to get better data?

**Do NOT build anything yet. Only inspect and report.**

---

## STEP 1 — FIND ALL JSON FILES

Search recursively inside `aiShopzawy/` for every `.json` file.

For each file found, report:
```
📄 File: [full path]
📦 Size: [size in KB or MB]
📊 Records: [count of items]
🏗️  Top-level structure: [array / object / nested]
```

---

## STEP 2 — DEEP INSPECT EACH FILE

Open each JSON file and extract:

### 2.1 — Print the FIRST record in full
Show the complete first record exactly as it is — no truncation.

### 2.2 — Check these critical fields

For each file, check if these fields exist and show their exact value from the first record:

```
PRICING:
  base_price / price / original_price     → [value or ❌ NOT FOUND]
  sale_price / discounted_price           → [value or ❌ NOT FOUND]
  discount_percent / discount             → [value or ❌ NOT FOUND]

STOCK:
  stock / stock_quantity / quantity       → [value or ❌ NOT FOUND]
  in_stock / available / is_available     → [value or ❌ NOT FOUND]

IDENTITY:
  id                                      → [value or ❌ NOT FOUND]
  name / name_ar / title                  → [value or ❌ NOT FOUND]
  name_en / name_english                  → [value or ❌ NOT FOUND]
  slug                                    → [value or ❌ NOT FOUND]

CATEGORIZATION:
  category / category_name / category_id → [value or ❌ NOT FOUND]
  brand / brand_name / brand_id          → [value or ❌ NOT FOUND]

MEDIA:
  image_url / image / thumbnail          → [value or ❌ NOT FOUND]
  images / product_images                → [value or ❌ NOT FOUND]

VARIANTS:
  variants / product_variants            → [exists? count? show first variant]

EXTRA:
  description                            → [exists? first 50 chars]
  tags                                   → [value or ❌ NOT FOUND]
  is_published / published               → [value or ❌ NOT FOUND]
  is_featured / featured                 → [value or ❌ NOT FOUND]
```

---

## STEP 3 — SCORE EACH FILE

After inspecting, give each file a score:

```
✅ PERFECT (90-100%)
   - Has: price, sale_price, stock, name_ar, category, brand, image, variants
   - Ready to use as-is

⚠️  GOOD ENOUGH (60-89%)
   - Missing 1-2 minor fields
   - Can be used with small fixes

🔴 NEEDS REBUILD (below 60%)
   - Missing critical fields like price or stock
   - Must pull fresh data from PostgreSQL
```

---

## STEP 4 — CHECK FOR THESE SPECIFIC PROBLEMS

Go through all records (not just the first one) and check:

```python
problems = {
    "price_is_zero": 0,        # records where price = 0 or null
    "price_in_text": 0,        # records where price is inside description text
    "price_nested": 0,         # records where price is inside variants only
    "no_sale_price": 0,        # records with base_price but no sale_price
    "no_stock_field": 0,       # records with no stock info at all
    "stock_in_variants_only": 0, # stock only inside variants array
    "no_category": 0,          # records with no category
    "no_brand": 0,             # records with no brand
    "no_image": 0,             # records with no image at all
    "no_variants": 0,          # records with no variants
    "category_is_id_only": 0,  # category is just a number, not a name
    "brand_is_id_only": 0,     # brand is just a number, not a name
    "out_of_stock": 0,         # records where stock = 0 (ok, just count them)
    "total_records": 0,
}
```

Print the final count for each problem.

---

## STEP 5 — FINAL VERDICT

After all inspection, give me ONE of these verdicts:

### VERDICT A — JSON is Perfect ✅
```
الـ JSON الموجود ممتاز ومناسب تماماً.
ممكن نبني SQLite منه مباشرة بدون أي تعديل.
الخطوة الجاية: تحويل JSON → SQLite
```

### VERDICT B — JSON needs small fixes ⚠️
```
الـ JSON كويس بس فيه مشاكل بسيطة:
- [list specific issues found]

الحل: نعدل على الـ JSON أثناء التحويل لـ SQLite
الخطوة الجاية: تحويل مع إصلاح المشاكل دي
```

### VERDICT C — JSON needs rebuild 🔴
```
الـ JSON الموجود ناقص حاجات مهمة:
- [list what's missing]

لازم نسحب داتا جديدة من PostgreSQL.
الـ SQL المطلوب:

[paste the exact SQL query needed]

بعد السحب، احفظ في: aiShopzawy/data/raw/products_fresh.json
```

---

## STEP 6 — IF VERDICT IS C, PROVIDE THE SQL

If the JSON needs a rebuild, generate the complete SQL query
based on what you found in the JSON structure.

The SQL must:
- JOIN products with categories, brands, product_variants, product_images
- Calculate sale_price correctly
- Calculate discount_percent
- Sum total stock from variants
- Get primary image URL
- Build variants as JSON array
- Filter: is_published = true only
- Include ALL fields needed for both RAG and SQL tools

Format the SQL so it can be copy-pasted directly into psql or pgAdmin.

---

## OUTPUT FORMAT

Write your complete report like this:

```
╔══════════════════════════════════════════╗
║     JSON INSPECTION REPORT               ║
║     Shopzawy AI Bot                      ║
╚══════════════════════════════════════════╝

📁 FILES FOUND
─────────────────────────────────────────
[list all files]

📄 FILE 1: [filename]
─────────────────────────────────────────
Size: X MB | Records: X

First Record:
[paste full first record]

Field Check:
[show all fields with values]

Problems Found:
[show counts]

Score: XX% — [PERFECT / GOOD ENOUGH / NEEDS REBUILD]

[repeat for each file]

══════════════════════════════════════════
FINAL VERDICT: [A / B / C]
══════════════════════════════════════════
[explain what to do next]

[if C: paste the complete SQL query]
```

---

## RULES

- Read EVERY JSON file, not just the first one
- Check ALL records for problems, not just the first record
- Be precise with field names — show exact names as they appear in JSON
- If a field is nested (e.g., `product.price.amount`), show the full path
- Do NOT modify any files
- Do NOT build SQLite yet
- Do NOT write any code yet

Just inspect, analyze, and report.

**Start now.**
