# Praktikum Big Data — Apache Spark

**Nama:** Intan Virginia Aulia Putri
**Modul:** 02 - Pemrosesan Data Besar  
**Kasus Studi:** Word Count (Menghitung Frekuensi Kata)  
**Tools:** Hadoop Streaming, Spark RDD, Spark DataFrame  

---

## Persiapan Lingkungan
- **Docker** dengan image `bde2020/spark-master:3.3.0-hadoop3.2`
- Folder `/data` untuk menyimpan file input dan skrip.
- File contoh:
  ```bash
  echo "Halo dunia halo spark dunia besar halo" > input.txt
  ````

---

## Sesi 1 — MapReduce (Hadoop Streaming)

### File yang Digunakan

**mapper.py**

```python
#!/usr/bin/env python3
import sys
for line in sys.stdin:
    words = line.strip().split()
    for word in words:
        print(f"{word.lower()}\t1")
```

**reducer.py**

```python
#!/usr/bin/env python3
import sys
from itertools import groupby

def read_input(file):
    for line in file:
        yield line.strip().split('\t', 1)

data = read_input(sys.stdin)
for key, group in groupby(sorted(data), key=lambda x: x[0]):
    total = sum(int(count) for _, count in group)
    print(f"{key}\t{total}")
```

### Eksekusi Lokal

```bash
cat input.txt | ./mapper.py | sort | ./reducer.py
```

**Hasil:**

```
besar	1
dunia	2
halo	3
spark	1
```

### Analisis

1️⃣ **Waktu eksekusi:** <1 detik (lokal), beberapa detik jika dijalankan di Hadoop karena operasi I/O besar.
2️⃣ **Dua skrip Python:** diperlukan karena tahap *Map* dan *Reduce* dijalankan terpisah oleh framework Hadoop di node yang berbeda.

---

## Sesi 2 — Spark RDD (Arsitektur Generasi Kedua)

### Kode Word Count (RDD)

```python
lines_rdd = spark.sparkContext.textFile("/data/input.txt")
words_rdd = lines_rdd.flatMap(lambda line: line.lower().split(" "))
pairs_rdd = words_rdd.map(lambda word: (word, 1))
counts_rdd = pairs_rdd.reduceByKey(lambda a, b: a + b)
final_counts = counts_rdd.collect()

for word, count in final_counts:
    print(f"{word}: {count}")
```

**Hasil:**

```
halo: 3
dunia: 2
spark: 1
besar: 1
```

### Analisis

1️⃣ Sintaks Spark RDD jauh lebih ringkas dibanding MapReduce karena semua tahap bisa ditulis dalam satu blok kode.
2️⃣ Jika `collect()` dihapus, tidak ada eksekusi karena Spark menggunakan **Lazy Evaluation** — eksekusi hanya terjadi saat ada *action*.

---

## Sesi 3 — Spark DataFrame (Arsitektur Generasi Ketiga)

### Kode Word Count (DataFrame)

```python
from pyspark.sql.functions import explode, split, col

df = spark.read.text("/data/input.txt")
words_df = df.select(
    explode(split(col("value"), " ")).alias("word")
).filter(col("word") != "")
counts_df = words_df.groupBy("word").count()
counts_df.orderBy(col("count").desc()).show()
```

**Hasil:**

```
+-----+-----+
| word|count|
+-----+-----+
| halo|    3|
|dunia|    2|
|spark|    1|
|besar|    1|
+-----+-----+
```

### Analisis

1️⃣ DataFrame lebih mudah dan intuitif karena berbasis struktur tabel (kolom & baris) seperti SQL, cocok untuk analis data.
2️⃣ **Catalyst Optimizer** menganalisis, menyusun ulang, dan mengoptimalkan langkah-langkah seperti `explode` dan `groupBy` agar dieksekusi seefisien mungkin dalam satu pipeline.

---

## Sesi 4 — Perbandingan Kinerja & Kesimpulan

| Teknologi             | Estimasi Waktu Eksekusi | Catatan                                     |
| --------------------- | ----------------------- | ------------------------------------------- |
| MapReduce (Streaming) | 5–15 detik              | Lambat (I/O tinggi)                         |
| Spark RDD             | 1–2 detik               | Lebih cepat (in-memory)                     |
| Spark DataFrame       | <1 detik                | Paling cepat (optimasi Catalyst & Tungsten) |

### Diskusi

* **Abstraksi:**
  MapReduce = key-value, RDD = objek, DataFrame = skema (kolom-baris).
* **Kinerja:**
  DataFrame tercepat karena menggunakan *query optimizer* dan *code generation*.
* **Kasus penggunaan:**
  RDD tetap berguna untuk data tidak terstruktur atau algoritma kustom yang tidak bisa direpresentasikan dengan SQL.

---

## Kesimpulan Akhir

* MapReduce menunjukkan dasar pemrosesan terdistribusi berbasis disk.
* Spark RDD meningkatkan kecepatan dengan in-memory computation dan API fungsional.
* Spark DataFrame menjadi solusi paling efisien berkat Catalyst Optimizer yang melakukan optimasi otomatis.
* Secara umum, DataFrame lebih direkomendasikan untuk analisis data, sedangkan RDD digunakan untuk tugas-tugas yang memerlukan kontrol penuh atas transformasi data.
