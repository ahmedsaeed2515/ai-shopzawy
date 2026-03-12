# PHASE 3 — DAILY SYNC SYSTEM
# Run ONLY after Phase 2 bot is working and verified

---

## PREREQUISITE — READ MIGRATIONS FIRST

Before writing a single line of code, find the PostgreSQL migration files
in the monorepo and fill in this table:

```
Search for: *.sql, migrations/, schema.prisma, knex migrations, sequelize migrations
Look in   : backend/, server/, api/, db/, prisma/, database/

POSTGRESQL SCHEMA FINDINGS
─────────────────────────────────────────────────────────────────────
  Products table name        : [e.g. "products" / "product" / "items"]
  Arabic name column         : [e.g. "name" / "name_ar" / "title"]
  English name column        : [e.g. "name_en" / MISSING]
  Base price column          : [e.g. "price" / "base_price"]
  Sale price column          : [e.g. "sale_price" / "discounted_price" / MISSING → compute]
  Stock column               : [e.g. "stock" / "quantity" / MISSING → sum from variants]
  Published flag column      : [e.g. "is_published" / "published" / "active"]
  Featured flag column       : [e.g. "is_featured" / MISSING → default false]
  Updated-at column          : [e.g. "updated_at" / "modified_at"]
  Category join              : [e.g. products.category_id → categories.id, name column = "name"]
  Brand join                 : [e.g. products.brand_id → brands.id, name column = "name"]
  Variants table             : [e.g. product_variants, join on product_id]
    variant price column     : [e.g. "price" / "sale_price"]
    variant stock column     : [e.g. "stock" / "quantity"]
    variant color column     : [e.g. "color" / MISSING]
    variant size column      : [e.g. "size" / MISSING]
    variant SKU column       : [e.g. "sku" / MISSING]
  Images table               : [e.g. product_images, join on product_id]
    primary image flag       : [e.g. "is_primary = true" / "sort_order = 0" / MISSING]
    image URL column         : [e.g. "url" / "path" / "image_url"]

From Phase 0 RAG inspection:
  Chroma DB path             : [exact path from Phase 0]
  Collection name            : [exact name from Phase 0]
  Embedding model            : [exact model name from Phase 0]
  Document text format       : [how was each product's text built for embedding?]
─────────────────────────────────────────────────────────────────────
```

Write this block out. **Every `# ← ADAPT` comment below must be replaced.**

---

## WHAT YOU ARE BUILDING

```
3 files:
  aiShopzawy/database/daily_sync.py    → core sync logic
  aiShopzawy/database/scheduler.py     → runs daily_sync at 04:00 AM
  aiShopzawy/database/ecosystem.config.js → PM2 config for VPS

What sync does each run:
  1. Connect to PostgreSQL (READ ONLY — cannot modify production data)
  2. Fetch only products changed in the last 25 hours
  3. Update SQLite (prices, stock, new products) inside a transaction
  4. Rebuild FTS5 index for changed products
  5. Re-embed changed products in Chroma RAG (upsert only)
  6. Log everything to sync_log table + sync.log file
  7. Run a health check and alert if something is wrong

Zero downtime — bot stays online during sync.
If nothing changed → completes in < 1 second.
```

---

## STEP 1 — CREATE `aiShopzawy/database/daily_sync.py`

```python
"""
Shopzawy Daily Sync
===================
Pulls only changed products from PostgreSQL → updates SQLite + Chroma RAG.

Safety guarantees:
  - PostgreSQL connection is READ ONLY (cannot modify production data)
  - SQLite updates are wrapped in a single transaction (atomic)
  - FTS5 index is rebuilt for changed products only
  - Chroma RAG is upserted (never wiped)
  - Any failure is logged and does not crash the scheduler

Run manually:
  python aiShopzawy/database/daily_sync.py

Run via scheduler:
  pm2 start aiShopzawy/database/ecosystem.config.js
"""

import sqlite3
import json
import os
import sys
import time
import logging
import traceback
from datetime import datetime, timedelta
from pathlib import Path
from dotenv import load_dotenv

load_dotenv()


# ── Configuration ────────────────────────────────────────────────────────────
_HERE    = Path(__file__).parent
DB_PATH  = str(_HERE / "shopzawy.db")
LOG_PATH = str(_HERE / "sync.log")

SYNC_WINDOW_HOURS  = 25    # fetch products updated in last N hours (buffer = 1hr)
PG_CONNECT_TIMEOUT = 15    # seconds
PG_MAX_RETRIES     = 3     # retry PG connection this many times on failure
CHROMA_BATCH_SIZE  = 50    # embed this many products at once


# ── Logging ──────────────────────────────────────────────────────────────────
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s  %(levelname)s  %(message)s",
    handlers=[
        logging.FileHandler(LOG_PATH, encoding="utf-8"),
        logging.StreamHandler(sys.stdout),
    ],
)
log = logging.getLogger("shopzawy_sync")


# ══════════════════════════════════════════════════════════════════════════════
# POSTGRESQL — READ ONLY
# ══════════════════════════════════════════════════════════════════════════════

def _get_pg(retries: int = PG_MAX_RETRIES):
    """
    Open a read-only PostgreSQL connection with retry logic.
    options="-c default_transaction_read_only=on" makes it physically
    impossible to write to production, even if code accidentally tries.
    """
    import psycopg2
    from psycopg2.extras import RealDictCursor

    dsn = dict(
        host=os.getenv("DB_HOST", "localhost"),      # ← ADAPT: env var name
        port=int(os.getenv("DB_PORT", 5432)),         # ← ADAPT: env var name
        dbname=os.getenv("DB_NAME"),                  # ← ADAPT: env var name
        user=os.getenv("DB_USER"),                    # ← ADAPT: env var name
        password=os.getenv("DB_PASSWORD"),            # ← ADAPT: env var name
        options="-c default_transaction_read_only=on",
        connect_timeout=PG_CONNECT_TIMEOUT,
        cursor_factory=RealDictCursor,
    )

    last_err = None
    for attempt in range(1, retries + 1):
        try:
            conn = psycopg2.connect(**dsn)
            log.info(f"PostgreSQL connected (attempt {attempt})")
            return conn
        except Exception as e:
            last_err = e
            log.warning(f"PG connect attempt {attempt}/{retries} failed: {e}")
            if attempt < retries:
                time.sleep(3 * attempt)

    raise ConnectionError(f"PostgreSQL unreachable after {retries} attempts: {last_err}")


def fetch_changed_products(since: datetime) -> list[dict]:
    """
    Fetch all published products updated since `since`.

    !! ADAPT: replace every column/table name marked with # ← ADAPT !!
    Use the schema findings from your migration files above.

    Returns a list of dicts — one per product, with all fields needed
    to update both SQLite and Chroma RAG.
    """
    SQL = """
        SELECT
            p.id,

            -- Identity
            p.name                                          AS name_ar,    -- ← ADAPT
            p.name_en                                       AS name_en,    -- ← ADAPT (or '' if missing)
            p.slug                                          AS slug,       -- ← ADAPT
            p.description                                   AS description,-- ← ADAPT
            p.tags                                          AS tags,       -- ← ADAPT (or NULL if missing)

            -- Category & Brand (adapt join columns)
            c.name                                          AS category_name, -- ← ADAPT
            b.name                                          AS brand_name,    -- ← ADAPT

            -- Pricing: compute from variants, fall back to product-level
            COALESCE(MIN(pv.price), 0)                     AS base_price,

            COALESCE(
                NULLIF(MIN(pv.sale_price), 0),
                MIN(pv.price),
                0
            )                                              AS sale_price,

            CASE
                WHEN COALESCE(MIN(pv.price), 0) > 0
                 AND COALESCE(
                       NULLIF(MIN(pv.sale_price), 0),
                       MIN(pv.price), 0
                     ) < MIN(pv.price)
                THEN ROUND(
                    (MIN(pv.price) - COALESCE(NULLIF(MIN(pv.sale_price),0), MIN(pv.price)))
                    / MIN(pv.price) * 100,
                1)
                ELSE 0
            END                                            AS discount_percent,

            -- Stock: sum across variants
            COALESCE(SUM(pv.stock), 0)                     AS stock_quantity, -- ← ADAPT column name
            CASE WHEN COALESCE(SUM(pv.stock), 0) > 0
                 THEN 1 ELSE 0 END                         AS in_stock,

            -- Primary image
            (
                SELECT pi.url                               -- ← ADAPT column name
                FROM product_images pi                      -- ← ADAPT table name
                WHERE pi.product_id = p.id
                  AND pi.is_primary = true                  -- ← ADAPT (or ORDER BY sort_order LIMIT 1)
                LIMIT 1
            )                                              AS image_url,

            -- All images as JSON array
            (
                SELECT json_agg(pi2.url ORDER BY pi2.sort_order) -- ← ADAPT
                FROM product_images pi2
                WHERE pi2.product_id = p.id
            )                                              AS images_json,

            -- Variants as normalized JSON array
            json_agg(
                json_build_object(
                    'sku',   pv.sku,           -- ← ADAPT or remove if missing
                    'color', pv.color,         -- ← ADAPT or remove if missing
                    'size',  pv.size,          -- ← ADAPT or remove if missing
                    'price', COALESCE(NULLIF(pv.sale_price, 0), pv.price, 0),  -- ← ADAPT
                    'stock', COALESCE(pv.stock, 0)  -- ← ADAPT
                )
                ORDER BY pv.id
            )                                              AS variants,

            -- Flags
            p.is_featured,                                  -- ← ADAPT (or false if missing)
            p.is_published,                                 -- ← ADAPT
            p.updated_at                                    -- ← ADAPT

        FROM products p                                     -- ← ADAPT table name
        LEFT JOIN categories c  ON c.id  = p.category_id   -- ← ADAPT join
        LEFT JOIN brands b      ON b.id  = p.brand_id       -- ← ADAPT join
        LEFT JOIN product_variants pv ON pv.product_id = p.id -- ← ADAPT table/join

        WHERE p.updated_at >= %(since)s                     -- ← ADAPT column name
          AND p.is_published = true                         -- ← ADAPT column name

        GROUP BY
            p.id, p.name, p.name_en, p.slug, p.description,
            p.tags, c.name, b.name,
            p.is_featured, p.is_published, p.updated_at     -- ← ADAPT to match SELECT

        ORDER BY p.updated_at DESC
    """

    pg  = _get_pg()
    cur = pg.cursor()
    cur.execute(SQL, {"since": since})
    rows = [dict(r) for r in cur.fetchall()]
    pg.close()
    log.info(f"PostgreSQL returned {len(rows)} changed product(s)")
    return rows


def _clean_variants(raw) -> list[dict]:
    """Normalize the json_agg variants output — removes nulls from LEFT JOINs."""
    if not raw:
        return []
    if isinstance(raw, str):
        try:
            raw = json.loads(raw)
        except Exception:
            return []
    if not isinstance(raw, list):
        return []
    return [
        v for v in raw
        if isinstance(v, dict) and any(
            v.get(k) is not None for k in ("sku", "price", "stock", "color", "size")
        )
    ]


def _clean_images(raw) -> list[str]:
    """Normalize the json_agg images output."""
    if not raw:
        return []
    if isinstance(raw, str):
        try:
            raw = json.loads(raw)
        except Exception:
            return []
    return [str(u) for u in raw if u]


# ══════════════════════════════════════════════════════════════════════════════
# SQLITE — ATOMIC UPDATE
# ══════════════════════════════════════════════════════════════════════════════

def update_sqlite(products: list[dict]) -> dict:
    """
    Update SQLite inside a single transaction.
    If anything fails, the entire batch is rolled back — no partial writes.
    Also rebuilds FTS5 entries for changed products.

    Returns stats dict: added, updated, skipped, errors
    """
    conn = sqlite3.connect(DB_PATH)
    conn.execute("PRAGMA journal_mode=WAL")
    conn.execute("PRAGMA synchronous=NORMAL")
    stats = {"added": 0, "updated": 0, "skipped": 0, "errors": 0}

    try:
        with conn:    # single atomic transaction — auto-rollback on exception
            for p in products:
                ext_id = str(p.get("id", "")).strip()
                if not ext_id or ext_id in ("None", "null", ""):
                    stats["skipped"] += 1
                    continue

                variants  = _clean_variants(p.get("variants"))
                images    = _clean_images(p.get("images_json"))
                tags_raw  = p.get("tags") or []
                tags      = (
                    ",".join(str(t) for t in tags_raw if t)
                    if isinstance(tags_raw, list)
                    else str(tags_raw)
                )

                exists = conn.execute(
                    "SELECT 1 FROM products WHERE external_id = ?", (ext_id,)
                ).fetchone()

                conn.execute("""
                    INSERT INTO products (
                        external_id, name_ar, name_en, slug, description,
                        category_name, brand_name,
                        base_price, sale_price, discount_percent,
                        in_stock, stock_quantity,
                        image_url, images, variants, tags,
                        is_featured, is_published,
                        updated_at, synced_at
                    ) VALUES (?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,datetime('now'))
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
                    ext_id,
                    str(p.get("name_ar") or ""),
                    str(p.get("name_en") or ""),
                    str(p.get("slug")    or ""),
                    str(p.get("description") or ""),
                    str(p.get("category_name") or ""),
                    str(p.get("brand_name")    or ""),
                    float(p.get("base_price")       or 0),
                    float(p.get("sale_price")       or 0),
                    float(p.get("discount_percent") or 0),
                    int(bool(p.get("in_stock"))),
                    int(p.get("stock_quantity")  or 0),
                    str(p.get("image_url") or ""),
                    json.dumps(images,    ensure_ascii=False),
                    json.dumps(variants,  ensure_ascii=False),
                    tags,
                    int(bool(p.get("is_featured",  False))),
                    int(bool(p.get("is_published", True))),
                    str(p.get("updated_at") or ""),
                ))

                if exists:
                    stats["updated"] += 1
                else:
                    stats["added"] += 1

        # Rebuild FTS5 for changed products (outside the main transaction is fine)
        _rebuild_fts(conn, [str(p.get("id", "")) for p in products])

        # Refresh lookup tables
        _rebuild_lookups(conn)

        # Log this run
        conn.execute("""
            INSERT INTO sync_log
              (sync_type, added, updated, skipped, errors, status, notes)
            VALUES ('daily_sync', ?, ?, ?, ?, 'success', 'Incremental sync')
        """, (stats["added"], stats["updated"], stats["skipped"], stats["errors"]))
        conn.commit()

    except Exception as e:
        stats["errors"] += len(products)
        log.error(f"SQLite transaction rolled back: {e}")
        try:
            conn.execute("""
                INSERT INTO sync_log (sync_type, status, notes)
                VALUES ('daily_sync', 'failed', ?)
            """, (str(e),))
            conn.commit()
        except Exception:
            pass
        raise
    finally:
        conn.close()

    return stats


def _rebuild_fts(conn: sqlite3.Connection, ext_ids: list[str]) -> None:
    """
    Rebuild FTS5 entries for the given external_ids.
    The triggers in Phase 1 handle INSERT/UPDATE, but we force-rebuild
    here to guarantee consistency after bulk upserts.
    """
    if not ext_ids:
        return
    try:
        placeholders = ",".join("?" * len(ext_ids))
        rows = conn.execute(f"""
            SELECT id, external_id, name_ar, category_name, brand_name, description
            FROM products
            WHERE external_id IN ({placeholders})
        """, ext_ids).fetchall()

        for row in rows:
            # Delete old FTS entry
            conn.execute(
                "DELETE FROM products_fts WHERE rowid = ?", (row[0],)
            )
            # Insert fresh FTS entry
            conn.execute("""
                INSERT INTO products_fts
                    (rowid, external_id, name_ar, category_name, brand_name, description)
                VALUES (?, ?, ?, ?, ?, ?)
            """, (row[0], row[1], row[2], row[3], row[4], row[5]))

        conn.commit()
        log.info(f"FTS5 rebuilt for {len(rows)} product(s)")
    except Exception as e:
        log.warning(f"FTS5 rebuild failed (non-critical): {e}")


def _rebuild_lookups(conn: sqlite3.Connection) -> None:
    """Keep categories and brands lookup tables in sync."""
    try:
        conn.execute("""
            INSERT OR IGNORE INTO categories (name)
            SELECT DISTINCT category_name FROM products
            WHERE category_name != '' AND category_name IS NOT NULL
        """)
        conn.execute("""
            INSERT OR IGNORE INTO brands (name)
            SELECT DISTINCT brand_name FROM products
            WHERE brand_name != '' AND brand_name IS NOT NULL
        """)
        conn.commit()
    except Exception as e:
        log.warning(f"Lookup table refresh failed (non-critical): {e}")


# ══════════════════════════════════════════════════════════════════════════════
# CHROMA RAG — INCREMENTAL UPSERT
# ══════════════════════════════════════════════════════════════════════════════

def _build_document_text(p: dict, variants: list[dict]) -> str:
    """
    Build the searchable text string for a product.

    !! ADAPT: this MUST match the exact format used when the RAG was
    originally built in Phase 0. If the format differs, the new embeddings
    will be inconsistent with existing ones.

    Check Phase 0 RAG inspection report for the original text format.
    """
    colors = sorted(set(v.get("color", "") for v in variants if v.get("color")))
    sizes  = sorted(set(v.get("size",  "") for v in variants if v.get("size")))
    disc   = p.get("discount_percent", 0) or 0

    disc_text  = f" (خصم {disc}%)" if disc > 0 else ""
    stock_text = (
        f"متوفر ({p.get('stock_quantity', 0)} قطعة)"
        if p.get("in_stock")
        else "غير متوفر"
    )

    # ← ADAPT: change this format to match Phase 0 RAG build format exactly
    return (
        f"منتج: {p.get('name_ar', '')}\n"
        f"الفئة: {p.get('category_name', '')} | "
        f"البراند: {p.get('brand_name', '')}\n"
        f"السعر: {p.get('sale_price', 0)} جنيه{disc_text} | {stock_text}\n"
        f"الوصف: {(p.get('description') or '')[:400]}\n"
        f"الألوان: {', '.join(colors) or 'غير محدد'} | "
        f"المقاسات: {', '.join(sizes) or 'غير محدد'}"
    ).strip()


def update_chroma(products: list[dict]) -> dict:
    """
    Upsert changed products into Chroma RAG in batches.
    Never wipes the collection — only adds or replaces changed documents.

    Returns stats dict: updated, failed
    """
    stats = {"updated": 0, "failed": 0}

    try:
        import chromadb
        from sentence_transformers import SentenceTransformer

        # ← ADAPT: these must match Phase 0 RAG inspection exactly
        chroma_path     = os.getenv("CHROMA_PATH",      "aiShopzawy/chroma_db")
        collection_name = os.getenv("CHROMA_COLLECTION", "shopzawy_products")
        model_name      = os.getenv("EMBEDDING_MODEL",   "intfloat/multilingual-e5-large")

        chroma = chromadb.PersistentClient(path=chroma_path)
        col    = chroma.get_collection(name=collection_name)
        model  = SentenceTransformer(model_name)
        log.info(f"Chroma connected | collection: {collection_name} | model: {model_name}")

    except ImportError as e:
        log.warning(f"Chroma/SentenceTransformers not installed: {e} — skipping RAG update")
        return stats
    except Exception as e:
        log.error(f"Chroma init failed: {e} — SQLite was updated; RAG will sync next run")
        return stats

    # Process in batches to avoid OOM on large syncs
    batches = [
        products[i : i + CHROMA_BATCH_SIZE]
        for i in range(0, len(products), CHROMA_BATCH_SIZE)
    ]

    for batch_num, batch in enumerate(batches, 1):
        log.info(f"Chroma batch {batch_num}/{len(batches)} ({len(batch)} products)")

        ids, docs, embeddings, metadatas = [], [], [], []

        for p in batch:
            try:
                variants = _clean_variants(p.get("variants"))
                text     = _build_document_text(p, variants)
                doc_id   = f"product_{p['id']}"

                # Prefix format depends on embedding model
                # intfloat/multilingual-e5: use "passage: " prefix
                # all-MiniLM: no prefix needed
                # ← ADAPT: prefix if needed
                embedding = model.encode(f"passage: {text}").tolist()

                metadata = {
                    "product_id":     int(p["id"]),
                    "name":           str(p.get("name_ar", "")),
                    "category":       str(p.get("category_name", "")),
                    "brand":          str(p.get("brand_name", "")),
                    "sale_price":     float(p.get("sale_price", 0)),
                    "in_stock":       bool(p.get("in_stock")),
                    "stock_quantity": int(p.get("stock_quantity", 0)),
                    "discount":       float(p.get("discount_percent", 0)),
                    "image_url":      str(p.get("image_url", "")),
                }

                ids.append(doc_id)
                docs.append(text)
                embeddings.append(embedding)
                metadatas.append(metadata)

            except Exception as e:
                stats["failed"] += 1
                log.error(f"Chroma encode error (product {p.get('id')}): {e}")

        if ids:
            try:
                col.upsert(
                    ids=ids,
                    embeddings=embeddings,
                    documents=docs,
                    metadatas=metadatas,
                )
                stats["updated"] += len(ids)
            except Exception as e:
                stats["failed"] += len(ids)
                log.error(f"Chroma upsert batch {batch_num} failed: {e}")

    log.info(f"Chroma done — updated: {stats['updated']}, failed: {stats['failed']}")
    return stats


# ══════════════════════════════════════════════════════════════════════════════
# HEALTH CHECK
# ══════════════════════════════════════════════════════════════════════════════

def run_health_check() -> dict:
    """
    Quick post-sync sanity check on SQLite.
    Returns a dict of health metrics and any warnings.
    """
    issues  = []
    metrics = {}

    try:
        conn = sqlite3.connect(DB_PATH)

        total      = conn.execute("SELECT COUNT(*) FROM products").fetchone()[0]
        in_stock   = conn.execute("SELECT COUNT(*) FROM products WHERE in_stock=1").fetchone()[0]
        zero_price = conn.execute("SELECT COUNT(*) FROM products WHERE sale_price=0").fetchone()[0]
        last_sync  = conn.execute(
            "SELECT run_at FROM sync_log ORDER BY id DESC LIMIT 1"
        ).fetchone()

        metrics = {
            "total_products": total,
            "in_stock":        in_stock,
            "zero_price":      zero_price,
            "last_sync_at":    last_sync[0] if last_sync else "never",
        }

        zero_pct = zero_price / max(total, 1) * 100
        if zero_pct > 10:
            issues.append(f"HIGH zero-price rate: {zero_pct:.1f}% ({zero_price} products)")
        if in_stock == 0:
            issues.append("WARNING: zero in-stock products — check stock sync")
        if total == 0:
            issues.append("CRITICAL: products table is empty")

        conn.close()

    except Exception as e:
        issues.append(f"Health check query failed: {e}")

    metrics["issues"] = issues
    return metrics


# ══════════════════════════════════════════════════════════════════════════════
# MAIN
# ══════════════════════════════════════════════════════════════════════════════

def run_sync(window_hours: int = SYNC_WINDOW_HOURS) -> dict:
    """
    Main sync entry point.
    Returns a summary dict for the scheduler to inspect.
    """
    t0    = time.time()
    since = datetime.now() - timedelta(hours=window_hours)

    log.info("=" * 60)
    log.info("  SHOPZAWY DAILY SYNC STARTED")
    log.info(f"  Window : last {window_hours} hours (since {since:%Y-%m-%d %H:%M})")
    log.info("=" * 60)

    summary = {
        "status":     "success",
        "fetched":    0,
        "sqlite":     {},
        "chroma":     {},
        "duration_s": 0,
        "issues":     [],
    }

    try:
        # ── Step 1: Fetch from PostgreSQL ─────────────────────────────────────
        log.info("Step 1/4 — Fetching from PostgreSQL...")
        products         = fetch_changed_products(since)
        summary["fetched"] = len(products)

        if not products:
            log.info("Nothing changed since last sync — done.")
            summary["duration_s"] = round(time.time() - t0, 1)
            return summary

        # ── Step 2: Update SQLite ─────────────────────────────────────────────
        log.info(f"Step 2/4 — Updating SQLite ({len(products)} products)...")
        summary["sqlite"] = update_sqlite(products)
        log.info(
            f"  SQLite: added={summary['sqlite']['added']}  "
            f"updated={summary['sqlite']['updated']}  "
            f"errors={summary['sqlite']['errors']}"
        )

        # ── Step 3: Update Chroma RAG ─────────────────────────────────────────
        log.info(f"Step 3/4 — Updating Chroma RAG ({len(products)} products)...")
        summary["chroma"] = update_chroma(products)
        log.info(
            f"  Chroma: updated={summary['chroma']['updated']}  "
            f"failed={summary['chroma']['failed']}"
        )

        # ── Step 4: Health check ──────────────────────────────────────────────
        log.info("Step 4/4 — Running health check...")
        health           = run_health_check()
        summary["issues"] = health.get("issues", [])
        log.info(
            f"  DB total={health['total_products']}  "
            f"in_stock={health['in_stock']}  "
            f"zero_price={health['zero_price']}"
        )
        if summary["issues"]:
            for issue in summary["issues"]:
                log.warning(f"  ⚠️  {issue}")

    except Exception as e:
        summary["status"] = "failed"
        summary["issues"].append(str(e))
        log.error(f"SYNC FAILED: {e}")
        log.debug(traceback.format_exc())

        try:
            conn = sqlite3.connect(DB_PATH)
            conn.execute("""
                INSERT INTO sync_log (sync_type, status, notes)
                VALUES ('daily_sync', 'failed', ?)
            """, (str(e),))
            conn.commit()
            conn.close()
        except Exception:
            pass

    finally:
        summary["duration_s"] = round(time.time() - t0, 1)
        log.info("=" * 60)
        log.info(
            f"  SYNC {'COMPLETE' if summary['status'] == 'success' else 'FAILED'} "
            f"— {summary['duration_s']}s"
        )
        log.info("=" * 60)

    return summary


if __name__ == "__main__":
    result = run_sync()
    sys.exit(0 if result["status"] == "success" else 1)
```

---

## STEP 2 — CREATE `aiShopzawy/database/scheduler.py`

```python
"""
Shopzawy Sync Scheduler
========================
Runs daily_sync at 04:00 AM every day.
Detects missed runs (e.g. server was down) and catches up automatically.
Runs once immediately on start to verify PG connection.

Start:
  python scheduler.py                   (foreground)
  pm2 start ecosystem.config.js         (background, auto-restart)
"""

import schedule
import time
import logging
import sys
import os
from datetime import datetime, date
from pathlib import Path

sys.path.insert(0, str(Path(__file__).parent))
from daily_sync import run_sync, DB_PATH, LOG_PATH

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s  %(levelname)s  %(message)s",
    handlers=[
        logging.FileHandler(LOG_PATH, encoding="utf-8"),
        logging.StreamHandler(sys.stdout),
    ],
)
log = logging.getLogger("shopzawy_scheduler")

SYNC_TIME      = "04:00"    # daily sync time (24h format)
MISSED_WINDOW  = 2          # re-run with 2× window if last sync was > 26hr ago


def _last_sync_was_recent() -> bool:
    """Check if a successful sync ran in the last 26 hours."""
    try:
        import sqlite3
        conn = sqlite3.connect(DB_PATH)
        row  = conn.execute("""
            SELECT run_at FROM sync_log
            WHERE status = 'success'
            ORDER BY id DESC LIMIT 1
        """).fetchone()
        conn.close()
        if not row:
            return False
        last = datetime.fromisoformat(str(row[0]))
        return (datetime.now() - last).total_seconds() < 26 * 3600
    except Exception:
        return False


def job():
    """Scheduled sync job — called by schedule library."""
    log.info(f"Scheduler triggered sync at {datetime.now():%Y-%m-%d %H:%M:%S}")
    try:
        result = run_sync()
        if result["status"] == "failed":
            log.error(f"Sync completed with failure: {result['issues']}")
        else:
            log.info(
                f"Sync OK — fetched={result['fetched']}  "
                f"duration={result['duration_s']}s"
            )
    except Exception as e:
        log.error(f"Scheduler job exception: {e}")


# ── Boot sequence ─────────────────────────────────────────────────────────────
log.info("=" * 55)
log.info("  Shopzawy Sync Scheduler")
log.info(f"  Daily sync at : {SYNC_TIME}")
log.info(f"  DB path       : {DB_PATH}")
log.info(f"  Log           : {LOG_PATH}")
log.info("=" * 55)

# Run immediately on start to verify PostgreSQL connection is alive
# and catch up if last sync was missed (e.g. server restart)
log.info("Boot check: verifying PostgreSQL connection and running catch-up if needed...")
if not _last_sync_was_recent():
    log.info("No recent successful sync found — running catch-up sync now...")
    try:
        run_sync(window_hours=26 * MISSED_WINDOW)
        log.info("Catch-up sync complete ✅")
    except Exception as e:
        log.error(f"Catch-up sync failed: {e}")
        log.warning("Will retry at scheduled time. Bot is still operational.")
else:
    log.info("Recent sync found — skipping catch-up. Waiting for scheduled time.")

# Register daily schedule
schedule.every().day.at(SYNC_TIME).do(job)
log.info(f"Scheduler running. Next sync at {SYNC_TIME}. Press Ctrl+C to stop.\n")

# Main loop
while True:
    try:
        schedule.run_pending()
        time.sleep(30)
    except KeyboardInterrupt:
        log.info("Scheduler stopped by user.")
        sys.exit(0)
    except Exception as e:
        log.error(f"Scheduler loop error: {e}")
        time.sleep(60)
```

---

## STEP 3 — CREATE `aiShopzawy/database/ecosystem.config.js`

```javascript
/**
 * PM2 process config for Shopzawy sync scheduler.
 *
 * Commands:
 *   pm2 start ecosystem.config.js    → start in background
 *   pm2 status                        → check it's running
 *   pm2 logs shopzawy-sync            → watch live logs
 *   pm2 restart shopzawy-sync         → restart after code changes
 *   pm2 stop shopzawy-sync            → stop without removing
 *   pm2 startup                       → auto-start on server reboot
 *   pm2 save                          → save process list for auto-start
 */
module.exports = {
  apps: [
    {
      name:          "shopzawy-sync",
      script:        "scheduler.py",
      interpreter:   "python3",
      cwd:           "./aiShopzawy/database",
      watch:         false,
      autorestart:   true,
      max_restarts:  10,
      restart_delay: 10000,         // 10s between restarts
      min_uptime:    "30s",         // must stay up 30s to count as stable

      // Log files
      log_file:  "./aiShopzawy/database/pm2_combined.log",
      error_file: "./aiShopzawy/database/pm2_error.log",
      out_file:  "./aiShopzawy/database/pm2_out.log",
      log_date_format: "YYYY-MM-DD HH:mm:ss",
      merge_logs: true,

      env: {
        PYTHONUNBUFFERED: "1",
        NODE_ENV:         "production",
      },
    },
  ],
};
```

---

## STEP 4 — RUN AND VERIFY

### 4.1 — Test daily_sync.py manually first

```bash
cd [monorepo root]
python aiShopzawy/database/daily_sync.py
```

Expected output:
```
SHOPZAWY DAILY SYNC STARTED
  Window : last 25 hours (since YYYY-MM-DD HH:MM)
Step 1/4 — Fetching from PostgreSQL...
  PostgreSQL returned X changed product(s)
Step 2/4 — Updating SQLite (X products)...
  SQLite: added=X  updated=X  errors=0
Step 3/4 — Updating Chroma RAG (X products)...
  Chroma: updated=X  failed=0
Step 4/4 — Running health check...
  DB total=X  in_stock=X  zero_price=X
SYNC COMPLETE — Xs
```

**If PostgreSQL connection fails:**
```
Fix: Check .env for DB_HOST, DB_PORT, DB_NAME, DB_USER, DB_PASSWORD
     Run: psql -h $DB_HOST -U $DB_USER -d $DB_NAME -c "SELECT 1;"
     If refused: verify the DB server allows connections from this IP
```

**If SQL query fails:**
```
Fix: Read your migration files again — adapt the SQL in fetch_changed_products()
     Common issues: wrong table name, wrong column name in GROUP BY
     Test the SQL directly in psql or pgAdmin with a fixed date
```

**If Chroma update fails:**
```
Fix: Verify CHROMA_PATH and CHROMA_COLLECTION match Phase 0 exactly
     Run: python -c "import chromadb; c = chromadb.PersistentClient('aiShopzawy/chroma_db'); print(c.list_collections())"
```

### 4.2 — Verify sync_log was written

```python
import sqlite3
conn = sqlite3.connect("aiShopzawy/database/shopzawy.db")
for row in conn.execute(
    "SELECT run_at, sync_type, added, updated, errors, status, notes "
    "FROM sync_log ORDER BY id DESC LIMIT 5"
):
    print(dict(zip(
        ["run_at","type","added","updated","errors","status","notes"], row
    )))
conn.close()
```

### 4.3 — Start PM2 on VPS

```bash
# Install PM2 if not installed
npm install -g pm2

# Start scheduler
pm2 start aiShopzawy/database/ecosystem.config.js

# Verify running
pm2 status

# Watch live logs
pm2 logs shopzawy-sync --lines 50

# Set up auto-start on server reboot
pm2 startup
pm2 save
```

---

## STEP 5 — CONFIRM AND STOP

After both manual test and PM2 are verified, write exactly:

```
Phase 3 complete ✅
  manual sync test  : PASSED
  sync_log written  : YES
  PostgreSQL fetch  : X products
  SQLite updated    : added=X  updated=X  errors=0
  Chroma updated    : X products  (or SKIPPED — not installed)
  health check      : X issues
  PM2 running       : YES / NO (if VPS not set up yet — that is OK)
  scheduler file    : aiShopzawy/database/scheduler.py
  sync file         : aiShopzawy/database/daily_sync.py

Ready to proceed to PHASE 4.
```

**Do not start Phase 4 until this confirmation block is shown.**
