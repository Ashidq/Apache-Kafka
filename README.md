# Apache-Kafka

🎯 Latar Belakang Masalah
Sebuah perusahaan logistik mengelola beberapa gudang penyimpanan yang menyimpan barang sensitif seperti makanan, obat-obatan, dan elektronik. Untuk menjaga kualitas penyimpanan, gudang-gudang tersebut dilengkapi dengan dua jenis sensor:

- Sensor Suhu

- Sensor Kelembaban

Sensor akan mengirimkan data setiap detik. Perusahaan ingin memantau kondisi gudang secara real-time untuk mencegah kerusakan barang akibat suhu terlalu tinggi atau kelembaban berlebih.

📋 Tugas Mahasiswa
1. Buat Topik Kafka
Buat dua topik di Apache Kafka:

- sensor-suhu-gudang

- sensor-kelembaban-gudang

Topik ini akan digunakan untuk menerima data dari masing-masing sensor secara real-time.

2. Simulasikan Data Sensor (Producer Kafka)
Buat dua Kafka producer terpisah:

  a. Producer Suhu
Kirim data setiap detik

  Format:

{"gudang_id": "G1", "suhu": 82}
  b. Producer Kelembaban
Kirim data setiap detik

  Format:

{"gudang_id": "G1", "kelembaban": 75}
Gunakan minimal 3 gudang: G1, G2, G3.

3. Konsumsi dan Olah Data dengan PySpark
  a. Buat PySpark Consumer
Konsumsi data dari kedua topik Kafka.

  b. Lakukan Filtering:
Suhu > 80°C → tampilkan sebagai peringatan suhu tinggi

Kelembaban > 70% → tampilkan sebagai peringatan kelembaban tinggi

Contoh Output:
[Peringatan Suhu Tinggi] Gudang G2: Suhu 85°C [Peringatan Kelembaban Tinggi] Gudang G3: Kelembaban 74%
4. Gabungkan Stream dari Dua Sensor
Lakukan join antar dua stream berdasarkan gudang_id dan window waktu (misalnya 10 detik) untuk mendeteksi kondisi bahaya ganda.

  c. Buat Peringatan Gabungan:
Jika ditemukan suhu > 80°C dan kelembaban > 70% pada gudang yang sama, tampilkan peringatan kritis.

✅ Contoh Output Gabungan:
[PERINGATAN KRITIS] Gudang G1: - Suhu: 84°C - Kelembaban: 73% - Status: Bahaya tinggi! Barang berisiko rusak Gudang G2: - Suhu: 78°C - Kelembaban: 68% - Status: Aman Gudang G3: - Suhu: 85°C - Kelembaban: 65% - Status: Suhu tinggi, kelembaban normal Gudang G4: - Suhu: 79°C - Kelembaban: 75% - Status: Kelembaban tinggi, suhu aman

### Langkah Pengerjaannya

- Membuat Struktur direktori 
```
big-data/
├── docker-compose.yml
├── producer/
│   ├── suhu_producer.py
│   └── kelembaban_producer.py
├── pyspark/
│   └── consumer_stream.py
```

- Isi File `docker-compose.yml`, `suhu_producer.py`, `kelembaban_producer.py` dan `consumer_stream.py`

#### docker-compose.yml
```
version: '3.8'

services:
  zookeeper:
    image: confluentinc/cp-zookeeper:7.2.1
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181

  kafka:
    image: confluentinc/cp-kafka:7.2.1
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
    depends_on:
      - zookeeper

  spark:
    image: bitnami/spark:3
    depends_on:
      - kafka
    environment:
      - SPARK_MODE=master

```

#### suhu_producer.py
```
from kafka import KafkaProducer
import json, time, random

producer = KafkaProducer(bootstrap_servers='localhost:9092', value_serializer=lambda v: json.dumps(v).encode('utf-8'))
gudangs = ['G1', 'G2', 'G3']

while True:
    for g in gudangs:
        data = {"gudang_id": g, "suhu": random.randint(75, 90)}
        producer.send('sensor-suhu-gudang', data)
    time.sleep(1)

```

#### kelembaban_producer.py
```
from kafka import KafkaProducer
import json, time, random

producer = KafkaProducer(bootstrap_servers='localhost:9092', value_serializer=lambda v: json.dumps(v).encode('utf-8'))
gudangs = ['G1', 'G2', 'G3']

while True:
    for g in gudangs:
        data = {"gudang_id": g, "kelembaban": random.randint(65, 80)}
        producer.send('sensor-kelembaban-gudang', data)
    time.sleep(1)

```

#### consumer_stream.py
```
from pyspark.sql import SparkSession
from pyspark.sql.functions import from_json, col, udf
from pyspark.sql.types import StringType, IntegerType, StructType

# Inisialisasi Spark Session
spark = SparkSession.builder.appName("GudangMonitoring").getOrCreate()
spark.sparkContext.setLogLevel("WARN")

# Schema untuk suhu dan kelembaban
suhu_schema = StructType().add("gudang_id", StringType()).add("suhu", IntegerType())
kelembaban_schema = StructType().add("gudang_id", StringType()).add("kelembaban", IntegerType())

# Baca data suhu dari Kafka
suhu_df = spark.readStream.format("kafka") \
    .option("kafka.bootstrap.servers", "kafka:29092") \
    .option("subscribe", "sensor-suhu-gudang") \
    .load()

suhu_parsed = suhu_df.selectExpr("CAST(value AS STRING)") \
    .select(from_json(col("value"), suhu_schema).alias("data")).select("data.*")

# Baca data kelembaban dari Kafka
kelembaban_df = spark.readStream.format("kafka") \
    .option("kafka.bootstrap.servers", "kafka:29092") \
    .option("subscribe", "sensor-kelembaban-gudang") \
    .load()

kelembaban_parsed = kelembaban_df.selectExpr("CAST(value AS STRING)") \
    .select(from_json(col("value"), kelembaban_schema).alias("data")).select("data.*")

# Join berdasarkan gudang_id
gabungan_df = suhu_parsed.join(kelembaban_parsed, "gudang_id")

# Fungsi untuk menentukan status
def cek_status(suhu, kelembaban):
    if suhu > 80 and kelembaban > 70:
        return "[PERINGATAN KRITIS] Bahaya tinggi! Barang berisiko rusak"
    elif suhu > 80:
        return "Suhu tinggi, kelembaban normal"
    elif kelembaban > 70:
        return "Kelembaban tinggi, suhu aman"
    else:
        return "Aman"

status_udf = udf(cek_status, StringType())

# Tambahkan kolom status
gabungan_df = gabungan_df.withColumn("status", status_udf(col("suhu"), col("kelembaban")))

# Format laporan akhir
laporan_df = gabungan_df.selectExpr(
    """concat(
        gudang_id, ': - Suhu: ', suhu, '°C - Kelembaban: ', kelembaban, '% - Status: ', status
    ) as laporan"""
)

# Tampilkan hasil ke console
query = laporan_df.writeStream \
    .outputMode("append") \
    .format("console") \
    .option("truncate", False) \
    .start()

query.awaitTermination()

```
