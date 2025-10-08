# Apache Arrow in PySpark/Spark and Beyond

---

## What is Apache Arrow?

Apache Arrow is an in-memory columnar data format that is language-agnostic, similar to how Apache Parquet provides a standardized format for on-disk data. The Apache Arrow specification can be implemented in any programming language, with official implementations available for many popular languages.

### Key Benefits

**Zero-Copy Data Sharing**

- Arrow's language-agnostic design enables zero-copy data sharing between languages such as Python and Java
- Zero-copy eliminates serialization overhead by allowing processes to share memory directly
- Traditional serialization (like Python's pickling) requires copying entire datasets, while Arrow avoids this overhead entirely
- Even when zero-copy isn't possible, Arrow enables vectorized operations using SIMD (Single Instruction, Multiple Data) instructions on modern CPUs, significantly improving performance

**Efficient Data Sharing Mechanisms**

Arrow provides multiple mechanisms for efficient data sharing:

*In-Process Memory Sharing:*

- **Arrow C Data Interface** - A stable C ABI (Application Binary Interface) that enables zero-copy data sharing between different Arrow implementations without serialization. It's simply a struct definition that any language can use
- **Arrow Stream Interface** - Similar to the C Data Interface but designed for streaming batches of data rather than single chunks. Streaming version allows passing a sequence of record batches but is still an in‑process API; it is not intended for inter‑process communication.

*Inter-Process/Network Communication:*

- **Arrow Flight** - Arrow Flight builds on the IPC format but adds RPC semantics and server/client infrastructure. It's built on gRPC for efficiently transferring Arrow data over networks between processes and machines
- **Arrow IPC (Inter-Process Communication)** - A standardized binary format for serializing Arrow data that supports:
  - Writing to disk as files
  - Transmission over network sockets
  - Sharing between processes via shared memory (mmap)

**Memory-Mapped File Sharing Example:**

When Arrow data is in IPC format on disk:

1. Process A writes an Arrow IPC file
2. Process B memory-maps the file
3. Process B accesses data directly from its address space - no copying or parsing required
4. Result: Multiple processes share the same physical memory pages, making data sharing extremely efficient

### Understanding Columnar Storage

Consider this sample table:

| Column1 | Column2 |
|---------|---------|
| Item1.1 | Item2.1 |
| Item1.2 | Item2.2 |
| Item1.3 | Item2.3 |

**Row-Based Storage** (Traditional databases)

```
|item1.1,item2.1|,|item1.2,item2.2|,|item1.3,item2.3|
```

Data is stored sequentially by row, with all columns for each row adjacent in memory.

**Column-Based Storage** (Arrow, Parquet)

```
|item1.1, item1.2, item1.3|, |item2.1, item2.2, item2.3|
```

Data is organized by column, making analytical operations more efficient.

**Why Columnar Format is Faster:**

- **Memory Locality**: When filtering or aggregating a single column, all relevant data is contiguous in memory, improving CPU cache utilization
- **Vectorization**: Modern CPUs can process multiple values simultaneously using SIMD instructions, which works best when data of the same type is stored together
- **Compression**: Similar values in a column compress better than mixed row data
- **Column Pruning**: Only needed columns are read, reducing I/O and memory usage

For detailed specifications on Arrow's memory layout, see the [Apache Arrow Columnar Format Documentation](https://arrow.apache.org/docs/format/Columnar.html#format-columnar).

---

## Arrow in PySpark/Spark

PySpark leverages Apache Arrow to accelerate data transfer between the JVM (where Spark runs) and Python processes. Arrow's columnar format and vectorized processing capabilities dramatically reduce serialization overhead.

### Configuration

#### Primary Settings

- `spark.sql.execution.arrow.pyspark.enabled` - Enables Apache Arrow for PySpark operations (default: `false`)
- `spark.sql.execution.arrow.pyspark.fallback.enabled` - Allows fallback to standard serialization if Arrow conversion fails (default: `true`)

#### Operations that Benefit from Arrow

##### Operations that require the arrow Configurations to be enabled

- `toPandas()` - Converting Spark DataFrames to Pandas
- `createDataFrame(pandas_df)` - Creating Spark DataFrames from Pandas

##### Operations that use Arrow either way and are not affected by the configurations

- `transformWithStateInPandas`
- `pandas_udf()` - Pandas UDFs for vectorized operations
- `applyInPandas()` / `mapInPandas()` - User-defined functions with Pandas DataFrames

### Prerequisites(Spark 4.0.x)

**For Spark 4.0.x**

- **PySpark**: Version 4.0.x or higher
- **PyArrow**: Install via `pip install pyarrow` (version 11.0.0 or higher recommended)
- **Pandas**: 2.0.0+ - Compatible version with your PyArrow installation

**For Spark 3.x**

- **PySpark**: Version 3.0.0 or higher
- **PyArrow**: Version 1.0.0 or higher (`pip install pyarrow>=1.0.0`)
- **Pandas**: Version 1.0.0 or higher

**Note:** Ensure your Pandas version is compatible with your PyArrow installation.

### When Arrow May NOT Help

- **Small datasets** (< 10,000 rows) - Overhead may exceed benefits
- **Complex nested types** - Some complex types have limited Arrow support
- **String-heavy datasets** - Variable-length strings can reduce vectorization benefits
- **Wide tables** - Extremely wide tables (hundreds of columns) may not see dramatic improvements

---

### Examples

- Lets write a simple program to which uses arrow for conversion from one format to another.

**First lets create a big csv**

```python
import pandas as pd

num_records = 1000000

data = {
    'name': ['Alice', 'Bob', 'Charlie', 'David', 'Emma'] * (num_records // 5),
    'age': [25, 30, 35, 40, 45] * (num_records // 5)
}

df = pd.DataFrame(data)
df.to_csv('large_dataset.csv', index=False)
```

#### Example 1: Convert from Spark DataFrame to Pandas DataFrame

```python
from pyspark.sql import SparkSession
import time

spark = SparkSession.builder.appName("arrow-bench").getOrCreate()

df = (spark.read
      .option("header", "true")
      .csv("large_dataset.csv"))
df = df.selectExpr("CAST(name AS STRING) name", "CAST(age AS INT) age")
df.cache().count()  # materialize once

def timed_to_pandas(use_arrow: bool):
    spark.conf.set("spark.sql.execution.arrow.pyspark.enabled", str(use_arrow).lower())
    t0 = time.time()
    pdf = df.toPandas()   # WARNING: collects to driver
    t1 = time.time()
    return t1 - t0, pdf

no_arrow_sec, _ = timed_to_pandas(False)
arrow_sec, _ = timed_to_pandas(True)

print(f"toPandas without Arrow: {no_arrow_sec:.2f}s")
print(f"toPandas with Arrow   : {arrow_sec:.2f}s")
```

#### Example 2: Convert Pandas DataFrame to Spark DataFrame

```python
import pandas as pd, time
from pyspark.sql import SparkSession

spark = SparkSession.builder.appName("arrow-bench").getOrCreate()
pdf = pd.read_csv("large_dataset.csv")  # beware of RAM

def timed_create_df(use_arrow: bool):
    spark.conf.set("spark.sql.execution.arrow.pyspark.enabled", str(use_arrow).lower())
    t0 = time.time()
    sdf = spark.createDataFrame(pdf)  # may fallback if unsupported types
    sdf.count()  # force materialization so we time the actual conversion path
    t1 = time.time()
    return t1 - t0

print("createDataFrame without Arrow:", timed_create_df(False))
print("createDataFrame with Arrow   :", timed_create_df(True))
```

## Resources

- [Apache Arrow in PySpark - Official Guide](https://spark.apache.org/docs/latest/api/python/tutorial/sql/arrow_pandas.html#apache-arrow-in-pyspark)
- [Apache Arrow Columnar Format Specification](https://arrow.apache.org/docs/format/index.html)
- [PyArrow Documentation](https://arrow.apache.org/docs/python/index.html)
- [Apache Arrow Official Website](https://arrow.apache.org/)
