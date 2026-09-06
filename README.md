# 🏗️ Data Lakehouse — Delta + Iceberg + MinIO

Medallion arxitekturası (Bronze → Silver → Gold) əsasında **Delta Lake** və **Apache Iceberg** formatlarından istifadə edərək **MinIO** üzərində tam data lakehouse qurulması.

---

## 🛠️ Texnologiyalar

| Texnologiya | Versiya | Məqsəd |
|---|---|---|
| Apache Spark (PySpark) | 3.5 | Data emalı |
| Delta Lake | 3.2.0 | ACID transactions, Time Travel |
| Apache Iceberg | 1.11.0 | Open table format, Schema Evolution |
| MinIO | latest | S3-compatible object store |
| JupyterLab | latest | İnteraktiv development mühiti |

---

## 🚀 Başlatmaq

```bash
# Servisləri başlat
docker-compose up -d

# Jupyter token-i öyrən
docker exec -it jupyter jupyter server list
```

Brauzerdə aç: `http://localhost:8888`

---

## 📁 Layihə Strukturu

```
Build_Lakehouse/
├── notebooks/
│   ├── Data_process.ipynb      # Əsas notebook
│   └── transactions_raw.csv    # Dataset (52,100 sətir)
├── docker-compose.yml          # Servis konfiqurasiyası
├── jupyter_server_config.py    # Jupyter konfiqurasiyası
├── .gitignore
└── README.md
```

---

## 🏛️ Medallion Arxitekturası

```
Raw CSV
   │
   ▼
┌─────────────────────────────────────────┐
│  BRONZE LAYER                           │
│  • Raw data dəyişdirilmədən saxlanılır  │
│  • 52,100 sətir                         │
│  • Delta + Iceberg formatında           │
└─────────────────────────────────────────┘
   │
   ▼  NULL, Duplicate silindi
┌─────────────────────────────────────────┐
│  SILVER LAYER                           │
│  • Təmizlənmiş data                     │
│  • 50,600 sətir                         │
│  • Boş dəyərlər NULL-a çevrildi         │
│  • Delta + Iceberg formatında           │
└─────────────────────────────────────────┘
   │
   ▼  Gündəlik / Currency üzrə qruplaşdırma
┌─────────────────────────────────────────┐
│  GOLD LAYER                             │
│  • Biznes analizi üçün summary          │
│  • dt + currency üzrə aggregasiya      │
│  • Delta + Iceberg formatında           │
└─────────────────────────────────────────┘
```

---

## 📊 Data Quality Nəticələri

| Problem | Say |
|---|---|
| NULL amount | 500 |
| NULL customer_id | 200 |
| Mənfi amount | 600 |
| Duplicate id | 800 |
| **Cəmi silindi** | **1,500** |

---

## ⚡ CRUD Əməliyyatları

Delta və Iceberg üzərində aşağıdakı əməliyyatlar test edilib:

| Əməliyyat | Delta | Iceberg |
|---|---|---|
| INSERT | `df.write.mode("append")` | `df.writeTo().append()` |
| UPDATE | `DeltaTable.update()` | `SQL UPDATE` |
| DELETE | `DeltaTable.delete()` | `SQL DELETE` |
| MERGE | `DeltaTable.merge()` | `SQL MERGE INTO` |

---

## ⏱️ Time Travel

**Delta:**
```python
# Versiya ilə
spark.read.format("delta").option("versionAsOf", 0).load(path)

# Timestamp ilə
spark.read.format("delta").option("timestampAsOf", "2026-09-06").load(path)
```

**Iceberg:**
```sql
SELECT * FROM iceberg.silver.transactions VERSION AS OF <snapshot_id>
```

---

## 🔄 Schema Evolution

Hər iki formatda `loyalty_score` sütunu əlavə edilərək schema evolution test edilib:

| Loyalty | Şərt |
|---|---|
| GOLD | tx_count >= 58 |
| SILVER | tx_count >= 48 |
| BRONZE | tx_count < 48 |

**Delta** — `ALTER TABLE` + `overwriteSchema=true`  
**Iceberg** — `ALTER TABLE ADD COLUMN` + `MERGE INTO`

---

## ⚖️ Delta vs Iceberg Müqayisəsi

| Xüsusiyyət | Delta | Iceberg |
|---|---|---|
| Metadata | `_delta_log/` JSON faylları | `metadata/` JSON + Avro |
| Time Travel | `versionAsOf` | `VERSION AS OF snapshot_id` |
| History | `delta_table.history()` | `.snapshots` metadata cədvəli |
| MERGE | Python API + SQL | Yalnız SQL |
| Catalog | Path-based | Catalog-based |
| Schema Evolution | `overwriteSchema=true` lazımdır | `ALTER TABLE` ilə sadədir |

---

## ❗ Qarşılaşılan Problemlər

| Problem | Səbəb | Həll |
|---|---|---|
| `PATH_NOT_FOUND` | CSV `/home/jovyan/work/` deyil, `/home/jovyan/`-da idi | Path düzəldildi |
| `DELTA_FAILED_TO_MERGE_FIELDS` | `id` sütunu LongType/IntegerType uyğunsuzluğu | `.cast(IntegerType())` əlavə edildi |
| Iceberg Time Travel xətası | `format("iceberg").load()` path-based işləmir | SQL `VERSION AS OF` ilə həll edildi |
