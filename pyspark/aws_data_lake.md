
# 📘 AWS Data Lake with Glue + PySpark — Complete Documentation (Explanations First, Solutions Last)

This guide is a **self‑contained handbook** that explains everything you need to understand and implement Labs **1A, 1B, and 1C** on AWS.  
It teaches **concepts and functions first** (Spark, Glue, Deequ, Athena, S3) so you can build the labs **without copy‑pasting**.  
At the very end, you’ll find the **unchanged “Possible Solutions”** from the exercises for verification only.

---

## 1) Introduction — The Data Lake Ecosystem

### What is a data lake and why use it?
A **data lake** is a storage-centric architecture that keeps data in its **original format** (CSV/JSON/Parquet) until you need it.  
It decouples **storage (S3)**, **compute (Glue/Spark/Athena)**, and **metadata (Glue Data Catalog)**, allowing cheap storage and flexible compute.

### Layers (typical naming)
- **Raw (Bronze)** — landing area for unmodified files (e.g., `raw/landing/`).
- **Curated (Silver)** — cleaned/standardized datasets (often **Parquet**, partitioned).
- **Gold** — analytical marts/aggregations (ready for BI or ML).

### How S3, Glue, Athena, PySpark interact
- **S3** stores files (CSV/Parquet). It’s the “filesystem” of the lake.
- **Glue Crawler** reads files to **infer schema** and populates the **Glue Data Catalog**.
- **Glue Job (PySpark)** performs transforms and writes outputs back to S3.
- **Athena** runs **serverless SQL** on S3 files, using schemas from the Glue Data Catalog.

### Text Diagram (logical flow)
```
CSV → S3 (raw/landing) → Glue Crawler → Glue Data Catalog → Athena SQL
                                      ↘
                                       Glue Job (PySpark) → S3 (curated/silver, Parquet) → Crawler / saveAsTable → Athena
```

---

## 2) AWS Glue and PySpark Fundamentals

### 2.1 Apache Spark in one minute
- **Spark** is a distributed engine. A **Driver** coordinates **Executors** across nodes.
- The core abstraction is the **DataFrame** (distributed table).
- **Transformations** (e.g., `withColumn`, `filter`) are lazy; they build a **DAG** (plan).
- **Actions** (e.g., `show`, `count`, `write`) trigger execution of the DAG.

### 2.2 DataFrame vs DynamicFrame
- **PySpark DataFrame** (native Spark): rich SQL functions, wide ecosystem; best for ETL logic.
- **Glue DynamicFrame**: Glue’s wrapper with schema flexibility and built‑in transforms; easy Catalog integration.
- You can convert: **DynamicFrame ↔ DataFrame** via `toDF()` and `fromDF()` (Glue‑specific).

### 2.3 What Glue adds on top of Spark
- **Job orchestration** (retries, monitoring, logs, metrics).
- **IAM integration** (roles/policies control S3/Glue/Athena access).
- **Data Catalog** read/write helpers.
- **Serverless infrastructure** (you choose worker types/DPUs; Glue manages the cluster).

### 2.4 Execution flow of a Glue Job (S3 → PySpark → S3)
1. Glue creates the Spark environment.
2. Your script starts, parses **job parameters** (e.g., paths).
3. Spark **reads** source files from S3 into a DataFrame.
4. You apply **transformations** (columns, filters, joins).
5. Spark **writes** the result to S3 (e.g., **Parquet**, **partitioned** folders).
6. Optionally, you **register** a table in the **Data Catalog** (Crawler or `saveAsTable`).

### 2.5 GlueContext and SparkContext together
- `SparkContext` starts the low‑level Spark engine.
- `GlueContext(SparkContext)` extends it with Glue features (Catalog, DynamicFrames).
- `glueContext.spark_session` gives you the **SparkSession** used for DataFrames.

---

## 3) Function‑by‑Function Explanation — Your Toolbox

> For each function: what it does, syntax, example, type (PySpark/Glue/PyDeequ), what it returns, and pitfalls.

### 3.1 `getResolvedOptions(sys.argv, ["ARG1","ARG2", ...])`
- **Type:** Glue‑specific (`awsglue.utils`)
- **What it does:** reads job parameters you pass at run time (e.g., `--SRC_FILE s3://...`).
- **Syntax:**
  ```python
  from awsglue.utils import getResolvedOptions
  import sys
  args = getResolvedOptions(sys.argv, ["SRC_FILE","TGT_BUCKET"])
  src = args["SRC_FILE"]; tgt = args["TGT_BUCKET"]
  ```
- **Returns:** `dict` mapping names → strings.
- **Pitfalls:** Names in the list must **exactly** match the ones you pass in the console/CLI.

### 3.2 `SparkContext` / `GlueContext` / `spark_session`
- **Type:** `SparkContext` (PySpark), `GlueContext` (Glue), `spark_session` (PySpark)
- **What it does:** boots Spark and adds Glue capabilities.
- **Syntax:**
  ```python
  from pyspark.context import SparkContext
  from awsglue.context import GlueContext
  sc = SparkContext()                        # PySpark
  glueContext = GlueContext(sc)              # Glue wrapper
  spark = glueContext.spark_session          # PySpark SparkSession
  ```
- **Returns:** live contexts to read/write DataFrames and interact with Glue.

### 3.3 `spark.read.option(...).csv(path)`
- **Type:** PySpark
- **What it does:** reads CSVs (including from S3) into a **DataFrame**.
- **Useful options:** `"header"="true"`, `"inferSchema"="true"`.
- **Syntax:**
  ```python
  df = spark.read.option("header","true").csv("s3://bucket/raw/landing/file.csv")
  ```
- **Returns:** `DataFrame`
- **Pitfalls:** inferring schema may be slow/inaccurate for very large/dirty files.

### 3.4 Column helpers: `col`, `lit`, `expr`
- **Type:** PySpark (`pyspark.sql.functions`)
- **What:** build column expressions.
- **Syntax/Examples:**
  ```python
  from pyspark.sql.functions import col, lit, expr
  df = df.withColumn("x2", col("x") * 2)       # reference column
  df = df.withColumn("source", lit("csv"))     # constant
  df = df.withColumn("year", expr("year(date_col)"))
  ```

### 3.5 `withColumn(name, expr)`
- **Type:** PySpark
- **What:** add/replace a column; **returns a new DataFrame**.
- **Syntax:**
  ```python
  df = df.withColumn("order_year", year("order_date"))
  ```
- **Pitfalls:** Doesn’t mutate in place. Reassign the result.

### 3.6 Dates: `to_date`, `year`, `month`
- **Type:** PySpark (`pyspark.sql.functions`)
- **What:** parse and derive date parts.
- **Syntax:**
  ```python
  from pyspark.sql.functions import to_date, year, month
  df = df.withColumn("order_date", to_date("order_purchase_timestamp"))
  df = df.withColumn("order_year", year("order_date"))
  df = df.withColumn("order_month", month("order_date"))
  ```

### 3.7 Mapping text: `create_map`, `chain`, `lit`
- **Type:** PySpark
- **What:** convert a Python `dict` to a Spark map expression for lookups.
- **Syntax:**
  ```python
  from pyspark.sql.functions import create_map, lit
  from itertools import chain
  mapping_expr = create_map([lit(x) for x in chain(*mapping.items())])
  df = df.withColumn("category_std", mapping_expr[col("category_name")])
  ```
- **Returns:** DataFrame with a new mapped column.

### 3.8 Hashing PII: `sha2(col, 256)`
- **Type:** PySpark (`pyspark.sql.functions`)
- **What:** deterministic SHA‑256 hash of a column (anonymization).
- **Syntax:**
  ```python
  from pyspark.sql import functions as F
  df = df.withColumn("customer_id", F.sha2("customer_id", 256))
  ```

### 3.9 Partition strategies: `repartition` vs `coalesce` vs `partitionBy`
- **Type:** PySpark
- **What:** control **in‑memory** shuffle and **on‑disk** layout.
- **Syntax:**
  ```python
  df = df.repartition("order_year","order_month")    # in-memory shuffle
  (df.write
     .mode("overwrite")
     .partitionBy("order_year","order_month")       # folder structure
     .parquet("s3://bucket/curated/retail_orders/"))
  ```
- **Tip:** Use `repartition` on the same keys you plan to `partitionBy` to reduce small files.

### 3.10 Writing: `.write.mode(...).format(...).option(...).parquet(...)`
- **Type:** PySpark
- **What:** persist DataFrames to S3.
- **Common patterns:**
  ```python
  df.write.mode("overwrite").parquet(tgt_path)         # Parquet (columnar, compressed)
  df.write.mode("append").format("parquet").save(tgt)   # equivalent
  df.write.option("path", tgt).saveAsTable("db.table")  # also registers a table
  ```
- **Modes:** `overwrite`, `append`, `ignore`, `errorifexists`.

### 3.11 Catalog‑aware writes: `saveAsTable("db.table")`
- **Type:** Glue‑integrated (via Spark SQL + Glue Catalog)
- **What:** writes files **and** registers a **table** in the **Glue Data Catalog**.
- **Syntax:**
  ```python
  spark.sql("CREATE DATABASE IF NOT EXISTS retail_silver_03320")
  (df.write
     .mode("overwrite")
     .format("parquet")
     .option("path", "s3://bucket/silver/retail_orders_clean/")
     .partitionBy("order_year","order_month")
     .saveAsTable("retail_silver_03320.retail_orders_clean"))
  ```

### 3.12 Glue Studio Notebook magics
- **Type:** Glue‑specific (Notebook only)
- **What:** configure session and dependencies.
- **Common magics:**
  ```python
  %extra_py_files s3://.../pydeequ.zip       # add Python zip deps to executors
  %extra_jars     s3://.../deequ-*.jar       # add Spark JARs
  %idle_timeout   2880                       # minutes
  %glue_version   5.0                        # Glue runtime
  %worker_type    G.1X                       # worker SKU
  %number_of_workers 5
  ```

### 3.13 Data quality with **PyDeequ**: `Check`, `VerificationSuite`
- **Type:** PyDeequ (library on top of Spark)
- **What:** declare rules and verify against a DataFrame.
- **Syntax:**
  ```python
  from pydeequ.checks import Check, CheckLevel
  from pydeequ.verification import VerificationSuite

  check = (Check(spark, CheckLevel.Error, "retail-quality")
           .isComplete("order_id")
           .isUnique("order_id")
           .isContainedIn("order_status", ["delivered","shipped","processing","canceled"])
           # .isNonNegative("payment_value")     # typical numeric rule
           )

  result = VerificationSuite(spark).onData(df).addCheck(check).run()
  status = "PASS" if result.status == "Success" else "FAIL"
  ```
- **Returns:** a result object with `.status` and detailed metrics.

### 3.14 Athena + Glue Catalog—how queries work
- **Athena** reads files directly from S3; schemas come from the **Glue Data Catalog**.
- For **partitioned** datasets, Athena uses the folder structure (e.g., `year=2024/month=1/…`).
- Query cost depends on **bytes scanned** — Parquet + partition pruning = cheaper/faster.

---

## 4) LAB 1A — Build a Basic Data Lake (S3 + Glue + Athena)

**Goal:** land CSV in S3 → register with Crawler → query in Athena.

1. **Create the lake bucket**: S3 Console → Create bucket (e.g., `my-etl-lake-<ctwnumber>`).  
   Create folders: `raw/landing/`.
2. **Upload the CSV**: put `retail_orders_2024q1.csv` into `raw/landing/`.
3. **Create a Glue database**: name it `bronze_raw_<ctwnumber>`.
4. **Create & run a Crawler**: point to `raw/landing/`, choose the database above, and run.
   - **What happens:** Crawler infers schema (columns/types) and creates a table in the Catalog.
5. **Query in Athena**: select the Catalog/database, run a `GROUP BY` query on the new table.
   - **Why CSV now:** get baseline correctness; next step converts to Parquet to save cost.

---

## 5) LAB 1B — Convert CSV to Partitioned Parquet with Glue (PySpark)

**Goal:** read CSV → add `order_date/year/month` → write **Parquet** partitioned by `(order_year, order_month)`.

### Design (map to Toolbox)
- **Runtime params**: `getResolvedOptions` (Glue) → `SRC_FILE`, `TGT_BUCKET`.
- **Contexts**: `SparkContext` + `GlueContext` → `spark_session`.
- **Read**: `spark.read.option("header","true").csv(src)`.
- **Transform**: `withColumn` + `to_date` + `year` + `month`.
- **Write**: `repartition` (memory) + `partitionBy` (folders) + `.write.mode("overwrite").parquet(tgt)`.
- **Inspect**: S3 folder layout: `.../order_year=YYYY/order_month=M/…`

### Why Parquet + partitioning
- Parquet is **columnar**: Athena reads only needed columns.
- Partition pruning: Athena reads only needed folders (e.g., a specific month/year).
- Dramatically reduces **bytes scanned** and cost.

### Under the hood (Spark)
- Transformations build a logical plan (DAG).  
- The `write` is an **action** that triggers the job; data is shuffled according to `repartition` keys and written to S3 as compressed Parquet files (Snappy by default).

---

## 6) LAB 1C — Clean & Validate with Glue Notebook + Deequ

**Goal:** standardize categories, hash PII, validate with Deequ, and write **Silver** (clean) vs **quarantine** (bad).

### Notebook setup (map to Toolbox)
- **Magics** to attach dependencies: `%extra_py_files`, `%extra_jars` (for PyDeequ).  
- **Contexts**: `SparkContext.getOrCreate()`, `GlueContext`, `spark_session`.  
- **Read**: `spark.read.format("parquet").load(source)` recursively loads Parquet under a prefix.  
- **Standardize**: mapping with `create_map` + `lit` + `withColumn` (plus optional `trim`, `lower`).  
- **Hash PII**: `F.sha2("customer_id", 256)`.  
- **Data quality**: PyDeequ `Check`, `VerificationSuite`.  
- **Conditional write**: if PASS → Silver (partitioned, `saveAsTable`) else → Quarantine.  
- **Athena**: query Silver database/table.

### Silver layer in practice
- Silver is a **trusted** version of your data with standardized columns and guaranteed constraints.
- It’s stored as **Parquet** partitions and **registered** in the Glue Catalog for SQL access.

---

## 7) Final Section — Solutions Reference (Unchanged)

> These are the **original “Possible Solutions”** from the labs. Use them only to verify your work.  
> The **main learning** is in the explanations above.

### Lab 1A — Athena query
```sql
SELECT order_status, COUNT(*) AS cnt
FROM "bronze_raw_<ctwnumber>"."retail_orders_<ctwnumber>_landing"
GROUP BY order_status;
```

### Lab 1B — Glue Job (PySpark) — Possible solution (as provided)
```python
import sys
from awsglue.utils import getResolvedOptions
from awsglue.context import GlueContext
from pyspark.context import SparkContext
from pyspark.sql.functions import to_date, year, month

args = getResolvedOptions(sys.argv, ["JOB_NAME","SRC_BUCKET","TGT_BUCKET"])
src = args['SRC_FILE']
tgt = args['TGT_BUCKET']

sc  = SparkContext()
glueContext = GlueContext(sc)
spark = glueContext.spark_session

df = (spark.read.option("header", "true").csv(src))

df = (df
      .withColumn("order_date", to_date("order_purchase_timestamp"))
      .withColumn("order_year", year("order_date"))
      .withColumn("order_month", month("order_date")))

(df.repartition("order_year","order_month")
 .write
 .mode("overwrite")
 .partitionBy("order_year","order_month")
 .parquet(tgt))

print("=== Transformation complete ===")
```

### Lab 1C — Glue Studio Notebook — Possible solution (as provided)

**Notebook cell 1**
```python
%extra_py_files s3://dependencies-03320/dependencies/pydeequ.zip
%extra_jars s3://dependencies-03320/dependencies/deequ-2.0.10-spark-3.5.jar
import os
os.environ["SPARK_VERSION"] = '3.5'
```

**Notebook cell 2**
```python
%idle_timeout 2880
%glue_version 5.0
%worker_type G.1X
%number_of_workers 5

import sys
from awsglue.transforms import *
from awsglue.utils import getResolvedOptions
from pyspark.context import SparkContext
from awsglue.context import GlueContext
from awsglue.job import Job
from pyspark.sql.functions import col, create_map, lit
from itertools import chain

sc = SparkContext.getOrCreate()
glueContext = GlueContext(sc)
spark = glueContext.spark_session
job = Job(glueContext)
```

**Load Parquet**
```python
from pyspark.sql import functions as F

lake = "s3://my-etl-lake-<ctwnumber>"
source = f"{lake}/curated/retail_orders/"

orders_raw = spark.read.format("parquet").load(source)

print(orders_raw.count(), "rows")
print("----------------------------------------------------------------------------------------------------")

orders_raw.printSchema()
print("----------------------------------------------------------------------------------------------------")

orders_raw.show(10)
```

**Standardize category**
```python
mapping = {
"electronics": "electronics",
"toys":        "toys",
"furniture":   "furniture",
"garden":      "garden",
"sports":      "sports",
"books":       "books",
"electronics & gadgets": "electronics",
"kids toys": "toys",
"home furniture": "furniture",
"outdoor garden": "garden",
"sport gear": "sports",
"book-store": "books"
}

mapping_expr = create_map([lit(x) for x in chain(*mapping.items())])
orders_raw = orders_raw.withColumn("category_std",
              mapping_expr[col("category_name")])
orders_raw.show(10)
```

**Hash PII**
```python
orders_raw = orders_raw.withColumn('customer_id', F.sha2('customer_id', 256))
orders_raw.show(truncate=False)
```

**Deequ checks**
```python
from pydeequ.checks import *
from pydeequ.verification import *
from pydeequ.repository import *

check = (Check(spark, CheckLevel.Error, "retail-quality")
         .isComplete("order_id")
         .isUnique("order_id")
         .isContainedIn("order_status", ["delivered","shipped","processing","canceled"])
         .isNonNegative("category_std")
         .hasMin("payment_value", lambda v: v >= 0))

result = VerificationSuite(spark).onData(orders_raw).addCheck(check).run()

if result.status == "Success":
    status = "PASS"
else:
    status = "FAIL"

print("Deequ status =", status)
# print(result.checkResults)
```

**Write clean vs quarantine**
```python
target_ok   = f"{lake}/silver/retail_orders_clean/"
target_bad  = f"{lake}/quarantine/retail_orders_bad/"

db = "retail_silver_<ctw-number>"
table_name_ok = "retail_orders_clean"

spark.sql(f"CREATE DATABASE IF NOT EXISTS {db}")

# status="PASS"

if status == "PASS":
    (orders_raw
     .repartition("order_year", "order_month")
     .write
     .mode("overwrite")
     .format("parquet")
     .option("path", target_ok)
     .partitionBy("order_year", "order_month")
     .saveAsTable(f"{db}.{table_name_ok}"))    
else:
    orders_raw.write.mode("overwrite").parquet(target_bad)
```

**Athena validation**
```sql
SELECT category_std, COUNT(*)
FROM "retail_silver_<ctwnumber>"."retail_orders_clean"
GROUP BY category_std;
```

---

## What to do next
- Add a **Gold** layer: daily/weekly aggregates (order counts, revenue by category).
- Schedule **crawlers** and **jobs** (on demand or via workflows).
- Add **more Deequ checks** (value ranges, patterns, correlation, etc.).

---

You now have the knowledge and the tools to build the labs **from scratch**, and adapt them to any dataset.
