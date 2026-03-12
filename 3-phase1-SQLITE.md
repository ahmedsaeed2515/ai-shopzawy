# PHASE 1 — BUILD SQLITE DATABASE
# Run ONLY after Phase 0 verdict is PERFECT or FIXABLE

---

## PREREQUISITE — READ PHASE 0 OUTPUT FIRST

Before writing a single line of code, read your Phase 0 report and confirm:

```
From Phase 0, I will use these EXACT field names:
─────────────────────────────────────────────────────────────
  ID field              : [e.g. "id"]
  Arabic name field     : [e.g. "name" or "name_ar"]
  English name field    : [e.g. "name_en" or MISSING]
  Base price field      : [e.g. "price" or "base_price"]
  Sale price field      : [e.g. "sale_price" or MISSING → from variants]
  Stock field           : [e.g. "stock" or MISSING → sum from variants]
  In-stock boolean      : [e.g. "in_stock" or MISSING → compute from stock]
  Category location     : [e.g. "category_name" / "category.name" / "category" nested]
  Brand location        : [e.g. "brand_name" / "brand.name" / "brand" nested]
  Image field           : [e.g. "image_url" / "image" / first of "images" array]
  Variants field        : [e.g. "variants" / "product_variants" / MISSING]
  Updated-at field      : [e.g. "updated_at" / MISSING]
  Published field       : [e.g. "is_published" / MISSING → default true]
  Featured field        : [e.g. "is_featured" / MISSING → default false]

  JSON top-level wrapper: [array / object with key "products" / object with key "data"]
─────────────────────────────────────────────────────────────
```

Write this block out and verify it against Phase 0 before proceeding.
**Every placeholder comment marked `# ← ADAPT` below must be replaced with the real value.**

---

## WHAT YOU ARE BUILDING

```
Input  : aiShopzawy/data/**/*.json     (all JSON files, recursive)
Output : aiShopzawy/database/shopzawy.db

Tables created:
  products          → main product table (one row per product)
  products_fts      → full-text search virtual table (auto-synced via triggers)
  categories        → unique category names (auto-populated from products)
  brands            → unique brand names (auto-populated from products)
  sync_log          → build/sync history
```

---

## STEP 1 — CREATE `aiShopzawy/database/build_sqlite.py`

Write the complete file below. **Replace every `# ← ADAPT` comment with the real value from Phase 0.**

```python
"""
Shopzawy SQLite Builder
=======================
Converts JSON product files → SQLite database.

Phase 0 field mappings are applied in:
  - extract_prices()    → base_price, sale_price, discount_percent
  - extract_stock()     → in_stock, stock_quantity
  - extract_category()  → category_name
  - extract_brand()     → brand_name
  - extract_image()     → image_url
  - upsert_product()    → name_ar, name_en, slug, description, etc.

Run:
  python aiShopzawy/database/build_sqlite.py
"""

import sqlite3
import json
import os
import sys
import time
import logging
from pathlib import Path
from datetime import datetime

# ── Configuration ───────────────────────────────────────────────────────────
DB_PATH   = "aiShopzawy/database/shopzawy.db"
DATA_PATH = "aiShopzawy/data/"
LOG_PATH  = "aiShopzawy/database/build.log"
# ────────────────────────────────────────────────────────────────────────────

os.makedirs(os.path.dirname(DB_PATH), exist_ok=True)

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s  %(levelname)s  %(message)s",
    handlers=[
        logging.StreamHandler(sys.stdout),
        logging.FileHandler(LOG_PATH, encoding="utf-8"),
    ]
)
log = logging.getLogger("shopzawy_build")


# ── Database Connection ──────────────────────────────────────────────────────
conn = sqlite3.connect(DB_PATH)
conn.execute("PRAGMA journal_mode=WAL")
conn.execute("PRAGMA synchronous=NORMAL")
conn.execute("PRAGMA foreign_keys=ON")
conn.execute("PRAGMA cache_size=-32000")    # 32 MB cache


# ── Schema ───────────────────────────────────────────────────────────────────
conn.executescript("""

-- ── Core products table ─────────────────────────────────────────────────────
CREATE TABLE IF NOT EXISTS products (
    id               INTEGER PRIMARY KEY AUTOINCREMENT,
    external_id      TEXT    UNIQUE NOT NULL,
    name_ar          TEXT    NOT NULL DEFAULT '',
    name_en          TEXT    DEFAULT '',
    slug             TEXT    DEFAULT '',
    description      TEXT    DEFAULT '',
    category_name    TEXT    DEFAULT '',
    brand_name       TEXT    DEFAULT '',
    base_price       REAL    DEFAULT 0,
    sale_price       REAL    DEFAULT 0,
    discount_percent REAL    DEFAULT 0,
    in_stock         INTEGER DEFAULT 0,     -- 0 or 1
    stock_quantity   INTEGER DEFAULT 0,     -- actual count
    image_url        TEXT    DEFAULT '',
    images           TEXT    DEFAULT '[]',  -- JSON array of URLs
    variants         TEXT    DEFAULT '[]',  -- JSON array of variant objects
    tags             TEXT    DEFAULT '',    -- comma-separated
    is_featured      INTEGER DEFAULT 0,
    is_published     INTEGER DEFAULT 1,
    updated_at       TEXT    DEFAULT '',    -- ISO string from source
    synced_at        TEXT    DEFAULT (datetime('now'))
);

-- ── Full-text search (FTS5) ─────────────────────────────────────────────────
-- Enables fast Arabic keyword search across name + category + brand + description
CREATE VIRTUAL TABLE IF NOT EXISTS products_fts USING fts5(
    external_id UNINDEXED,
    name_ar,
    category_name,
    brand_name,
    description,
    content='products',
    content_rowid='id',
    tokenize='unicode61 remove_diacritics 2'
);

-- Auto-sync FTS when products table changes
CREATE TRIGGER IF NOT EXISTS products_ai AFTER INSERT ON products BEGIN
    INSERT INTO products_fts(rowid, external_id, name_ar, category_name, brand_name, description)
    VALUES (new.id, new.external_id, new.name_ar, new.category_name, new.brand_name, new.description);
END;

CREATE TRIGGER IF NOT EXISTS products_ad AFTER DELETE ON products BEGIN
    INSERT INTO products_fts(products_fts, rowid, external_id, name_ar, category_name, brand_name, description)
    VALUES ('delete', old.id, old.external_id, old.name_ar, old.category_name, old.brand_name, old.description);
END;

CREATE TRIGGER IF NOT EXISTS products_au AFTER UPDATE ON products BEGIN
    INSERT INTO products_fts(products_fts, rowid, external_id, name_ar, category_name, brand_name, description)
    VALUES ('delete', old.id, old.external_id, old.name_ar, old.category_name, old.brand_name, old.description);
    INSERT INTO products_fts(rowid, external_id, name_ar, category_name, brand_name, description)
    VALUES (new.id, new.external_id, new.name_ar, new.category_name, new.brand_name, new.description);
END;

-- ── Lookup tables ───────────────────────────────────────────────────────────
CREATE TABLE IF NOT EXISTS categories (
    id   INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT UNIQUE NOT NULL
);

CREATE TABLE IF NOT EXISTS brands (
    id   INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT UNIQUE NOT NULL
);

-- ── Sync history ────────────────────────────────────────────────────────────
CREATE TABLE IF NOT EXISTS sync_log (
    id           INTEGER PRIMARY KEY AUTOINCREMENT,
    run_at       TEXT    DEFAULT (datetime('now')),
    sync_type    TEXT,
    added        INTEGER DEFAULT 0,
    updated      INTEGER DEFAULT 0,
    skipped      INTEGER DEFAULT 0,
    errors       INTEGER DEFAULT 0,
    duration_sec REAL    DEFAULT 0,
    status       TEXT,
    notes        TEXT
);

-- ── Indexes ─────────────────────────────────────────────────────────────────
CREATE INDEX IF NOT EXISTS idx_sale_price    ON products(sale_price);
CREATE INDEX IF NOT EXISTS idx_base_price    ON products(base_price);
CREATE INDEX IF NOT EXISTS idx_stock         ON products(in_stock, stock_quantity);
CREATE INDEX IF NOT EXISTS idx_brand         ON products(brand_name);
CREATE INDEX IF NOT EXISTS idx_category      ON products(category_name);
CREATE INDEX IF NOT EXISTS idx_discount      ON products(discount_percent);
CREATE INDEX IF NOT EXISTS idx_featured      ON products(is_featured);
CREATE INDEX IF NOT EXISTS idx_published     ON products(is_published);
CREATE INDEX IF NOT EXISTS idx_external_id   ON products(external_id);
CREATE INDEX IF NOT EXISTS idx_updated_at    ON products(updated_at);

""")
conn.commit()
log.info("Schema created / verified.")


# ── Helper: Safe Type Coercion ───────────────────────────────────────────────

def safe_float(val, default: float = 0.0) -> float:
    if val is None:
        return default
    try:
        return float(val)
    except (TypeError, ValueError):
        return default


def safe_int(val, default: int = 0) -> int:
    if val is None:
        return default
    try:
        return int(float(val))
    except (TypeError, ValueError):
        return default


# ── Helper: Multi-key Getter ─────────────────────────────────────────────────

def get(obj: dict, *keys, default=None):
    """
    Try multiple field name variants in order.
    Supports dot notation for nested objects: 'category.name'

    Examples:
      get(r, 'price', 'base_price')          → tries 'price' then 'base_price'
      get(r, 'category.name', 'category')    → tries nested path then flat key
    """
    for key in keys:
        if '.' in key:
            val = obj
            for part in key.split('.'):
                if isinstance(val, dict):
                    val = val.get(part)
                else:
                    val = None
                    break
            if val is not None:
                return val
        else:
            val = obj.get(key)
            if val is not None:
                return val
    return default


def coerce_name(val) -> str:
    """Handle both flat string and nested dict for category/brand fields."""
    if isinstance(val, dict):
        return (
            val.get('name') or
            val.get('name_ar') or
            val.get('title') or
            str(next(iter(val.values()), ''))
        ).strip()
    return str(val).strip() if val else ''


# ── Extractors — ADAPT ALL FIELD NAMES FROM PHASE 0 ────────────────────────

def extract_prices(record: dict) -> tuple[float, float, float]:
    """
    Returns (base_price, sale_price, discount_percent).

    Strategy:
      1. Try top-level price fields
      2. Fallback: extract from variants array
      3. Compute discount_percent if not present
    """
    # ── 1. Top-level price fields ──────────────────────────────────────────
    base = safe_float(get(record,
        'price',            # ← ADAPT: replace with Phase 0 field name
        'base_price',
        'original_price',
        'regular_price',
        'unit_price',
    ))

    sale = safe_float(get(record,
        'sale_price',       # ← ADAPT: replace with Phase 0 field name
        'discounted_price',
        'offer_price',
        'special_price',
        'promo_price',
    ))

    discount = safe_float(get(record,
        'discount_percent', # ← ADAPT: replace with Phase 0 field name
        'discount',
        'discount_rate',
    ))

    # ── 2. Variant fallback ────────────────────────────────────────────────
    if base == 0 or sale == 0:
        variants = get(record, 'variants', 'product_variants', default=[]) or []  # ← ADAPT
        if isinstance(variants, list) and variants:
            sale_prices = [
                safe_float(get(v, 'sale_price', 'price', 'offer_price'))
                for v in variants if isinstance(v, dict)
            ]
            sale_prices = [p for p in sale_prices if p > 0]

            orig_prices = [
                safe_float(get(v, 'price', 'original_price', 'base_price'))
                for v in variants if isinstance(v, dict)
            ]
            orig_prices = [p for p in orig_prices if p > 0]

            if sale_prices and sale == 0:
                sale = min(sale_prices)
            if orig_prices and base == 0:
                base = min(orig_prices)
            if base == 0 and sale > 0:
                base = sale

    # ── 3. Normalize ───────────────────────────────────────────────────────
    sale = sale or base
    base = base or sale

    if discount == 0 and base > 0 and sale > 0 and sale < base:
        discount = round((base - sale) / base * 100, 1)

    return base, sale, discount


def extract_stock(record: dict) -> tuple[int, int]:
    """
    Returns (in_stock: 0|1, stock_quantity: int).

    Strategy:
      1. Try top-level quantity field
      2. Fallback: sum stock across all variants
      3. Derive in_stock from quantity if boolean not present
    """
    # ── 1. Top-level stock ─────────────────────────────────────────────────
    qty = get(record,
        'stock',            # ← ADAPT: replace with Phase 0 field name
        'stock_quantity',
        'quantity',
        'inventory',
        'qty',
        'available_qty',
    )

    # ── 2. Variant fallback ────────────────────────────────────────────────
    if qty is None:
        variants = get(record, 'variants', 'product_variants', default=[]) or []  # ← ADAPT
        if isinstance(variants, list):
            qty = sum(
                safe_int(get(v, 'stock', 'quantity', 'qty', 'stock_quantity'))
                for v in variants if isinstance(v, dict)
            )
        else:
            qty = 0

    qty = safe_int(qty)

    # ── 3. In-stock boolean ────────────────────────────────────────────────
    flag = get(record,
        'in_stock',         # ← ADAPT: replace with Phase 0 field name
        'available',
        'is_available',
        'is_in_stock',
    )

    if flag is not None:
        in_stock = 1 if bool(flag) else 0
    else:
        in_stock = 1 if qty > 0 else 0

    return in_stock, qty


def extract_category(record: dict) -> str:
    """
    Returns category as a clean string.
    Handles flat string, nested dict, and ID-only (returns empty if ID-only).
    """
    val = get(record,
        'category_name',    # ← ADAPT: replace with Phase 0 field name
        'category.name',    # nested object path
        'category',
        'cat_name',
        'category_ar',
    )
    result = coerce_name(val)
    # If result is purely numeric (ID only), return empty
    return result if not result.isdigit() else ''


def extract_brand(record: dict) -> str:
    """
    Returns brand as a clean string.
    """
    val = get(record,
        'brand_name',       # ← ADAPT: replace with Phase 0 field name
        'brand.name',       # nested object path
        'brand',
        'manufacturer',
        'brand_ar',
    )
    result = coerce_name(val)
    return result if not result.isdigit() else ''


def extract_image(record: dict) -> str:
    """Returns the primary image URL as a string."""
    # ── Direct image field ────────────────────────────────────────────────
    img = get(record,
        'image_url',        # ← ADAPT: replace with Phase 0 field name
        'image',
        'thumbnail',
        'main_image',
        'cover_image',
        'primary_image',
        'photo',
    )
    if img and isinstance(img, str) and img.startswith('http'):
        return img

    # ── First item of images array ────────────────────────────────────────
    imgs = get(record,
        'images',           # ← ADAPT: replace with Phase 0 field name
        'product_images',
        'gallery',
        'photos',
        default=[]
    ) or []

    if isinstance(imgs, list) and imgs:
        first = imgs[0]
        if isinstance(first, str):
            return first
        if isinstance(first, dict):
            return (
                first.get('url') or
                first.get('image_url') or
                first.get('src') or
                first.get('path') or ''
            )
    return ''


def extract_images_list(record: dict) -> str:
    """Returns all image URLs as a JSON array string."""
    imgs = get(record,
        'images',           # ← ADAPT: replace with Phase 0 field name
        'product_images',
        'gallery',
        'photos',
        default=[]
    ) or []

    if not isinstance(imgs, list):
        return '[]'

    urls = []
    for img in imgs:
        if isinstance(img, str) and img:
            urls.append(img)
        elif isinstance(img, dict):
            url = (img.get('url') or img.get('image_url') or
                   img.get('src') or img.get('path') or '')
            if url:
                urls.append(url)

    return json.dumps(urls, ensure_ascii=False)


def extract_variants(record: dict) -> str:
    """
    Normalizes variants into a standard JSON array.
    Each variant has: sku, color, size, price, stock.
    """
    variants = get(record,
        'variants',         # ← ADAPT: replace with Phase 0 field name
        'product_variants',
        'options',
        default=[]
    ) or []

    if not isinstance(variants, list):
        return '[]'

    normalized = []
    for v in variants:
        if not isinstance(v, dict):
            continue
        normalized.append({
            'sku':   str(get(v, 'sku', 'SKU', 'barcode', default='') or ''),
            'color': str(get(v, 'color', 'colour', 'Color', 'اللون', default='') or ''),
            'size':  str(get(v, 'size', 'Size', 'المقاس', default='') or ''),
            'price': safe_float(get(v, 'sale_price', 'price', 'offer_price')),
            'stock': safe_int(get(v, 'stock', 'quantity', 'qty', 'stock_quantity')),
        })

    return json.dumps(normalized, ensure_ascii=False)


def extract_tags(record: dict) -> str:
    """Returns tags as a comma-separated string."""
    tags = (
        record.get('tags') or
        record.get('keywords') or
        record.get('labels') or
        []
    )
    if isinstance(tags, list):
        return ','.join(str(t) for t in tags if t)
    if isinstance(tags, str):
        return tags
    return ''


# ── Upsert: One Product ──────────────────────────────────────────────────────

def upsert_product(record: dict, stats: dict) -> None:
    """Insert or replace one product. Updates stats in place."""

    # ── Required: external ID ──────────────────────────────────────────────
    ext_id = str(get(record,
        'id',               # ← ADAPT: replace with Phase 0 field name
        'product_id',
        'ID',
        'uuid',
        default=''
    ) or '')

    if not ext_id or ext_id in ('None', 'null', ''):
        stats['skipped'] += 1
        return

    # ── Extract all fields ─────────────────────────────────────────────────
    base_price, sale_price, discount = extract_prices(record)
    in_stock, stock_qty              = extract_stock(record)
    category                         = extract_category(record)
    brand                            = extract_brand(record)
    image                            = extract_image(record)
    images                           = extract_images_list(record)
    variants                         = extract_variants(record)
    tags                             = extract_tags(record)

    name_ar = str(get(record,
        'name',             # ← ADAPT: replace with Phase 0 field name
        'name_ar',
        'title',
        'product_name',
        'اسم المنتج',
        default=''
    ) or '').strip()

    name_en = str(get(record,
        'name_en',          # ← ADAPT: replace with Phase 0 field name
        'name_english',
        'title_en',
        default=''
    ) or '').strip()

    slug = str(get(record,
        'slug',             # ← ADAPT: replace with Phase 0 field name
        'url_key',
        'handle',
        'permalink',
        default=''
    ) or '').strip()

    description = str(get(record,
        'description',      # ← ADAPT: replace with Phase 0 field name
        'body',
        'details',
        'desc',
        'body_html',
        default=''
    ) or '').strip()

    is_published = int(bool(get(record,
        'is_published',     # ← ADAPT: replace with Phase 0 field name
        'published',
        'active',
        'status',
        default=True
    )))

    is_featured = int(bool(get(record,
        'is_featured',      # ← ADAPT: replace with Phase 0 field name
        'featured',
        'highlight',
        'is_highlight',
        default=False
    )))

    updated_at = str(get(record,
        'updated_at',       # ← ADAPT: replace with Phase 0 field name (or '' if missing)
        'updatedAt',
        'modified_at',
        'last_modified',
        default=''
    ) or '')

    # ── Track add vs update ────────────────────────────────────────────────
    exists = conn.execute(
        "SELECT 1 FROM products WHERE external_id = ?", (ext_id,)
    ).fetchone()

    # ── Upsert ────────────────────────────────────────────────────────────
    conn.execute("""
        INSERT INTO products (
            external_id, name_ar, name_en, slug, description,
            category_name, brand_name,
            base_price, sale_price, discount_percent,
            in_stock, stock_quantity,
            image_url, images, variants, tags,
            is_featured, is_published, updated_at
        ) VALUES (?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?)
        ON CONFLICT(external_id) DO UPDATE SET
            name_ar          = excluded.name_ar,
            name_en          = excluded.name_en,
            slug             = excluded.slug,
            description      = excluded.description,
            category_name    = excluded.category_name,
            brand_name       = excluded.brand_name,
            base_price       = excluded.base_price,
            sale_price       = excluded.sale_price,
            discount_percent = excluded.discount_percent,
            in_stock         = excluded.in_stock,
            stock_quantity   = excluded.stock_quantity,
            image_url        = excluded.image_url,
            images           = excluded.images,
            variants         = excluded.variants,
            tags             = excluded.tags,
            is_featured      = excluded.is_featured,
            is_published     = excluded.is_published,
            updated_at       = excluded.updated_at,
            synced_at        = datetime('now')
    """, (
        ext_id, name_ar, name_en, slug, description,
        category, brand,
        base_price, sale_price, discount,
        in_stock, stock_qty,
        image, images, variants, tags,
        is_featured, is_published, updated_at,
    ))

    if exists:
        stats['updated'] += 1
    else:
        stats['added'] += 1


# ── Populate Lookup Tables ───────────────────────────────────────────────────

def rebuild_lookup_tables() -> None:
    """Rebuild categories and brands tables from products table."""
    conn.execute("DELETE FROM categories")
    conn.execute("DELETE FROM brands")

    conn.execute("""
        INSERT OR IGNORE INTO categories (name)
        SELECT DISTINCT category_name FROM products
        WHERE category_name != '' AND category_name IS NOT NULL
        ORDER BY category_name
    """)

    conn.execute("""
        INSERT OR IGNORE INTO brands (name)
        SELECT DISTINCT brand_name FROM products
        WHERE brand_name != '' AND brand_name IS NOT NULL
        ORDER BY brand_name
    """)

    cat_count   = conn.execute("SELECT COUNT(*) FROM categories").fetchone()[0]
    brand_count = conn.execute("SELECT COUNT(*) FROM brands").fetchone()[0]
    log.info(f"Lookup tables rebuilt → {cat_count} categories, {brand_count} brands")


# ── Main: Process All JSON Files ─────────────────────────────────────────────

start_time = time.time()
stats = {'added': 0, 'updated': 0, 'skipped': 0, 'errors': 0}

json_files = sorted(Path(DATA_PATH).glob("**/*.json"))
log.info(f"Found {len(json_files)} JSON file(s) to process")

if not json_files:
    log.error(f"No JSON files found in {DATA_PATH} — check the path.")
    sys.exit(1)

for json_file in json_files:
    log.info(f"Processing: {json_file}")

    try:
        with open(json_file, 'r', encoding='utf-8') as f:
            data = json.load(f)
    except json.JSONDecodeError as e:
        log.error(f"  JSON parse error in {json_file.name}: {e}")
        continue
    except Exception as e:
        log.error(f"  File read error: {e}")
        continue

    # ── Unwrap common top-level wrappers ───────────────────────────────────
    if isinstance(data, dict):
        for wrapper_key in ['products', 'data', 'items', 'results', 'records', 'payload']:
            if wrapper_key in data and isinstance(data[wrapper_key], list):
                log.info(f"  Unwrapped top-level key: '{wrapper_key}'")
                data = data[wrapper_key]
                break
        else:
            data = [data]   # Single product object
    elif not isinstance(data, list):
        log.warning(f"  Unexpected top-level type: {type(data).__name__} — skipping")
        continue

    file_stats = {'added': 0, 'updated': 0, 'skipped': 0, 'errors': 0}

    for record in data:
        if not isinstance(record, dict):
            file_stats['skipped'] += 1
            continue
        try:
            upsert_product(record, file_stats)
        except Exception as e:
            file_stats['errors'] += 1
            if file_stats['errors'] <= 3:
                log.error(f"  Record error: {e} | record id={record.get('id', '?')}")

    # Merge file stats into global stats
    for k in stats:
        stats[k] += file_stats[k]

    log.info(
        f"  → added={file_stats['added']} updated={file_stats['updated']} "
        f"skipped={file_stats['skipped']} errors={file_stats['errors']}"
    )

# ── Commit + Rebuild Lookups ─────────────────────────────────────────────────
conn.commit()
rebuild_lookup_tables()
conn.commit()

elapsed = round(time.time() - start_time, 2)


# ── Log This Build Run ───────────────────────────────────────────────────────
conn.execute("""
    INSERT INTO sync_log (sync_type, added, updated, skipped, errors, duration_sec, status, notes)
    VALUES ('initial_build', ?, ?, ?, ?, ?, 'success', 'Built from JSON files')
""", (stats['added'], stats['updated'], stats['skipped'], stats['errors'], elapsed))
conn.commit()


# ── Verification Report ───────────────────────────────────────────────────────
total        = conn.execute("SELECT COUNT(*) FROM products").fetchone()[0]
in_stock_cnt = conn.execute("SELECT COUNT(*) FROM products WHERE in_stock=1").fetchone()[0]
with_price   = conn.execute("SELECT COUNT(*) FROM products WHERE sale_price > 0").fetchone()[0]
with_disc    = conn.execute("SELECT COUNT(*) FROM products WHERE discount_percent > 0").fetchone()[0]
zero_price   = conn.execute("SELECT COUNT(*) FROM products WHERE sale_price = 0").fetchone()[0]
no_image     = conn.execute("SELECT COUNT(*) FROM products WHERE image_url = ''").fetchone()[0]
cat_count    = conn.execute("SELECT COUNT(*) FROM categories").fetchone()[0]
brand_count  = conn.execute("SELECT COUNT(*) FROM brands").fetchone()[0]

error_rate = stats['errors'] / max(total, 1) * 100
zero_price_rate = zero_price / max(total, 1) * 100

print(f"\n{'═'*65}")
print(f"  SHOPZAWY SQLITE BUILD — COMPLETE")
print(f"{'═'*65}")
print(f"  Duration       : {elapsed}s")
print(f"  ✅ Added       : {stats['added']:,}")
print(f"  🔄 Updated     : {stats['updated']:,}")
print(f"  ⏭  Skipped     : {stats['skipped']:,}")
print(f"  ❌ Errors      : {stats['errors']:,}  ({error_rate:.1f}%)")
print(f"{'─'*65}")
print(f"  Total products : {total:,}")
print(f"  In stock       : {in_stock_cnt:,}  ({in_stock_cnt/max(total,1)*100:.1f}%)")
print(f"  Out of stock   : {total - in_stock_cnt:,}")
print(f"  With price     : {with_price:,}  ({with_price/max(total,1)*100:.1f}%)")
print(f"  Price = 0      : {zero_price:,}  ({zero_price_rate:.1f}%)")
print(f"  With discount  : {with_disc:,}")
print(f"  No image       : {no_image:,}")
print(f"  Categories     : {cat_count:,}")
print(f"  Brands         : {brand_count:,}")

# ── Health check ──────────────────────────────────────────────────────────────
issues = []
if error_rate > 5:
    issues.append(f"HIGH ERROR RATE: {error_rate:.1f}% — investigate before Phase 2")
if zero_price_rate > 10:
    issues.append(f"HIGH ZERO PRICE RATE: {zero_price_rate:.1f}% — check extract_prices()")
if cat_count == 0:
    issues.append("NO CATEGORIES — extract_category() returned empty for all records")
if brand_count == 0:
    issues.append("NO BRANDS — extract_brand() returned empty for all records")

if issues:
    print(f"\n{'─'*65}")
    print(f"  ⚠️  ISSUES FOUND — FIX BEFORE PHASE 2:")
    for issue in issues:
        print(f"     ❌ {issue}")
    print(f"{'─'*65}")
else:
    print(f"\n  ✅ All health checks passed.")

# ── Price stats ───────────────────────────────────────────────────────────────
price_row = conn.execute("""
    SELECT MIN(sale_price), MAX(sale_price), ROUND(AVG(sale_price), 0)
    FROM products WHERE in_stock=1 AND sale_price > 0
""").fetchone()
if price_row[0]:
    print(f"\n  Price range (in-stock only):")
    print(f"  → Min: {price_row[0]}  |  Max: {price_row[1]}  |  Avg: {price_row[2]}")

# ── All categories ────────────────────────────────────────────────────────────
print(f"\n  ALL CATEGORIES ({cat_count}):")
for row in conn.execute("""
    SELECT category_name, COUNT(*) c, SUM(in_stock) s
    FROM products WHERE category_name != ''
    GROUP BY category_name ORDER BY c DESC
"""):
    print(f"  → {row[0]:<35} {row[1]:>5} products  ({row[2]} in stock)")

# ── All brands ────────────────────────────────────────────────────────────────
print(f"\n  ALL BRANDS ({brand_count}):")
for row in conn.execute("""
    SELECT brand_name, COUNT(*) c, SUM(in_stock) s
    FROM products WHERE brand_name != ''
    GROUP BY brand_name ORDER BY c DESC LIMIT 30
"""):
    print(f"  → {row[0]:<35} {row[1]:>5} products  ({row[2]} in stock)")

# ── Sample: cheapest in-stock ─────────────────────────────────────────────────
print(f"\n  CHEAPEST 5 (in stock):")
for row in conn.execute("""
    SELECT name_ar, brand_name, category_name, sale_price, stock_quantity
    FROM products WHERE in_stock=1 AND sale_price > 0
    ORDER BY sale_price ASC LIMIT 5
"""):
    print(f"  → {row[0][:40]:<40} | {row[1]:<15} | {row[3]} | stock:{row[4]}")

# ── Sample: most discounted ───────────────────────────────────────────────────
print(f"\n  MOST DISCOUNTED 5 (in stock):")
for row in conn.execute("""
    SELECT name_ar, base_price, sale_price, discount_percent
    FROM products WHERE in_stock=1 AND discount_percent > 0
    ORDER BY discount_percent DESC LIMIT 5
"""):
    print(f"  → {row[0][:40]:<40} | {row[1]} → {row[2]} ({row[3]}% off)")

conn.close()

print(f"\n  💾 Database saved to : {DB_PATH}")
print(f"  📋 Build log saved to: {LOG_PATH}")
print(f"{'═'*65}")

# ── Phase 2 Handoff Block ─────────────────────────────────────────────────────
print(f"""
┌─────────────────────────────────────────────────────────────┐
│  COPY THIS INTO Phase 2 (bot.py)                            │
├─────────────────────────────────────────────────────────────┤
│  SQLITE_PATH = "{DB_PATH}"           │
│                                                             │
│  Run these in Phase 2 to get real names for bot:           │
│    SELECT DISTINCT category_name FROM products ORDER BY     │
│      category_name;                                         │
│    SELECT DISTINCT brand_name FROM products ORDER BY        │
│      brand_name;                                            │
└─────────────────────────────────────────────────────────────┘
""")
```

---

## STEP 2 — RUN IT

```bash
cd [monorepo root]
python aiShopzawy/database/build_sqlite.py
```

---

## STEP 3 — PASTE THE COMPLETE OUTPUT

Do not proceed to Phase 2 until you show the full terminal output including:

- Duration
- Added / Updated / Skipped / Errors counts
- Error rate %
- Zero-price rate %
- All categories with product counts
- All brands with product counts
- Cheapest 5 and most discounted 5

**If any health check fails → fix the relevant extractor function and re-run. Do not continue with broken data.**

---

## STEP 4 — POST-BUILD VERIFICATION QUERIES

After the script finishes, run these four queries and show all results:

```python
import sqlite3, json
conn = sqlite3.connect("aiShopzawy/database/shopzawy.db")

# ── Q1: Confirm prices are real numbers, not zero ──────────────────
print("=== Q1: Price sanity ===")
print("  sale_price = 0 :", conn.execute("SELECT COUNT(*) FROM products WHERE sale_price = 0").fetchone()[0])
print("  sale_price > 0 :", conn.execute("SELECT COUNT(*) FROM products WHERE sale_price > 0").fetchone()[0])

# ── Q2: Confirm categories are names, not IDs ─────────────────────
print("\n=== Q2: Category names (should be text, not numbers) ===")
for row in conn.execute("SELECT DISTINCT category_name FROM products WHERE category_name != '' LIMIT 10"):
    print(" ", row[0])

# ── Q3: Confirm variants are valid JSON ───────────────────────────
print("\n=== Q3: Variants sample (should be JSON array) ===")
for row in conn.execute("SELECT external_id, variants FROM products WHERE variants != '[]' LIMIT 3"):
    variants = json.loads(row[1])
    print(f"  Product {row[0]} → {len(variants)} variants → first: {variants[0] if variants else 'empty'}")

# ── Q4: Confirm FTS works ─────────────────────────────────────────
print("\n=== Q4: FTS search test ===")
for row in conn.execute("""
    SELECT p.name_ar, p.sale_price
    FROM products_fts fts
    JOIN products p ON p.id = fts.rowid
    WHERE products_fts MATCH 'موبايل'
    LIMIT 5
"""):
    print(f"  {row[0]} — {row[1]}")

conn.close()
```

Show all output. All four queries must return meaningful results.

---

## STEP 5 — STOP AND CONFIRM

After all outputs look correct, write exactly:

```
Phase 1 complete ✅
  Total products : [N]
  In stock       : [N]
  Zero price     : [N]  ([X]%)
  Categories     : [N]
  Brands         : [N]
  FTS works      : YES / NO
  DB path        : aiShopzawy/database/shopzawy.db

Ready to proceed to PHASE 2.
```

**Do not start Phase 2 until this confirmation block is shown.**
