# Template Kafka Consumer + Outbound Webhook



Gunakan template ini untuk service yang menerima business event dari Kafka, memproses event, lalu memanggil webhook HTTP ke sistem lain.

## General Information

| Item | Keterangan |
| --- | --- |
| **Service Name** | `<SERVICE_NAME>` |
| **Trigger** | Kafka Consumer |
| **Consumer Topic** | `<CONSUMER_TOPIC>` |
| **Outbound Integration** | HTTP Webhook |
| **Webhook Target** | `<WEBHOOK_TARGET>` |
| **Description** | `<Jelaskan fungsi service dalam 1 sampai 2 kalimat>` |
| **Category** | `<SERVICE_CATEGORY>` |
| **Kode Service** | `<SERVICE_CODE>` |

| **Tabel yang Digunakan** | `<TABLE_1>`, `<TABLE_2>` atau `N/A` |
| **Redis Key** | `<REDIS_KEY_PATTERN>` atau `N/A` |

### Version Control

| Version | Date | Author | Remarks |
| --- | --- | --- | --- |
| 1.0.0 | `<YYYY-MM-DD>` | System Analyst | Initial creation |

## Kafka Consumer Contract

Service menerima event dari `<PRODUCER_SERVICE>` melalui Kafka topic `<CONSUMER_TOPIC>`.

### Consumer Configuration

| Item | Value | Description |
| --- | --- | --- |
| **Topic** | `<CONSUMER_TOPIC>` | Topic sumber event. |
| **Producer Service** | `<PRODUCER_SERVICE>` | Service yang mem-publish event. |
| **Consumer Group** | `<CONSUMER_GROUP>` | Consumer group service. |
| **Message Key** | `<MESSAGE_KEY>` | Key event untuk routing atau ordering. |
| **Partition** | `<PARTITION_NUMBER>` atau `N/A` | Isi jika menggunakan fixed partition. |
| **Content Type** | `application/json` | Format payload event. |
| **Trigger Condition** | `<TRIGGER_CONDITION>` | Kondisi event yang diproses service. |

### Kafka Header

| No | Key | Type | M/O/C | Description |
| --- | --- | --- | --- | --- |
| 1 | `trace-id` | String | M | Trace ID untuk korelasi proses. |
| 2 | `<HEADER_KEY>` | String | O | `<Deskripsi header tambahan>` |

### Kafka Message Body

| No | Key | Type | M/O/C | Description |
| --- | --- | --- | --- | --- |
| 1 | `timestamp` | String | M | Waktu event dibuat. |
| 2 | `trace_id` | String | M | Trace ID event. |
| 3 | `service.name` | String | M | Nama producer service. |
| 4 | `deployment.environment` | String | M | Environment producer. |
| 5 | `payload` | Object | M | Payload business event. |
| 5.a | `payload.<fieldName>` | String | M | `<Deskripsi field>` |

### Contoh Kafka Message

```json
{
  "timestamp": "<TIMESTAMP>",
  "trace_id": "<TRACE_ID>",
  "service": {
    "name": "<PRODUCER_SERVICE>"
  },
  "deployment": {
    "environment": "<ENVIRONMENT>"
  },
  "payload": {
    "<fieldName>": "<value>"
  }
}
```

## Outbound Webhook Contract

Service memanggil webhook setelah `<WEBHOOK_TRIGGER>` terpenuhi.

### Webhook Configuration

| Item | Value | Description |
| --- | --- | --- |
| **Target** | `<WEBHOOK_TARGET>` | Sistem penerima webhook. |
| **URL Configuration** | `<WEBHOOK_URL_CONFIG_KEY>` | Konfigurasi URL webhook. |
| **Method** | `POST` | HTTP method outbound. |
| **Content Type** | `application/json` | Format request body. |
| **Authentication** | `<AUTH_METHOD>` atau `N/A` | Mekanisme autentikasi jika ada. |
| **Timeout** | `<TIMEOUT>` | Batas waktu pemanggilan webhook. |
| **Success Criteria** | `<HTTP_STATUS_OR_RULE>` | Kondisi webhook dianggap berhasil. |
| **Idempotency Key** | `<IDEMPOTENCY_KEY>` atau `N/A` | Key deduplikasi pada sistem penerima jika digunakan. |

### Webhook Request Header

| No | Key | Type | M/O/C | Description |
| --- | --- | --- | --- | --- |
| 1 | `Content-Type` | String | M | Wajib `application/json`. |
| 2 | `<TRACE_HEADER>` | String | M | Trace ID yang sama dengan event Kafka. |
| 3 | `<AUTH_HEADER>` | String | C | Header autentikasi jika digunakan. |
| 4 | `<IDEMPOTENCY_HEADER>` | String | C | Idempotency key jika digunakan. |

### Webhook Request Body

| No | Key | Type | M/O/C | Source | Description |
| --- | --- | --- | --- | --- | --- |
| 1 | `<fieldName>` | String | M | `<Kafka/DB/System>` | `<Deskripsi field>` |
| 2 | `<objectName>` | Object | O | `<Kafka/DB/System>` | `<Deskripsi object>` |
| 2.a | `<objectName.fieldName>` | String | M | `<Kafka/DB/System>` | `<Deskripsi field>` |

### Contoh Webhook Request

```http
POST <WEBHOOK_PATH> HTTP/1.1
Host: <WEBHOOK_HOST>
Content-Type: application/json
<TRACE_HEADER>: <TRACE_ID>

{
  "<fieldName>": "<value>"
}
```

### Webhook Response

| No | Item | Type | M/O/C | Description |
| --- | --- | --- | --- | --- |
| 1 | HTTP Status | Integer | M | HTTP status dari sistem penerima. |
| 2 | `<responseField>` | String | O | `<Deskripsi response jika digunakan>` |

### Contoh Webhook Response

```json
{
  "<responseField>": "<value>"
}
```

### Mapping Kafka ke Webhook

| Kafka / Data Source | Webhook Field | Transformation | M/O/C | Description |
| --- | --- | --- | --- | --- |
| `trace_id` | `<TRACE_HEADER>` | Direct mapping | M | Gunakan trace ID yang sama. |
| `payload.<field>` | `<field>` | Direct mapping | M | `<Deskripsi>` |
| `<db.field>` | `<field>` | `<TRANSFORMATION>` | C | `<Deskripsi>` |

## Redis Operations

Gunakan bagian ini hanya jika service mengakses Redis.

Format key:

`[nama_aplikasi].[domain].[fitur].[fungsi].[jenis_identitas].{nilai_identitas}`

| Jenis Identitas | Redis Key Pattern | Data Type | TTL | Description |
| --- | --- | --- | --- | --- |
| `<IDENTITY>` | `<REDIS_KEY_PATTERN>` | `<STRING/JSON/HASH>` | `<TTL>` | `<Fungsi key>` |

## Database Operations

| No | Operation | Table | Condition | Description |
| --- | --- | --- | --- | --- |
| 1 | `<SELECT/INSERT/UPDATE>` | `<TABLE_NAME>` | `<CONDITION>` | `<Tujuan operasi>` |

## Dependency dan Komponen

| Komponen | Status | Fungsi |
| --- | --- | --- |
| `<Kafka Client SDK>` | Mandatory | Menerima event Kafka. |
| `<HTTP Client>` | Mandatory | Memanggil outbound webhook. |
| `AWS Secrets Manager SDK` | `<Mandatory/Optional>` | Mengambil secret aplikasi jika digunakan. |
| `<Database Driver>` | `<Mandatory/Optional>` | Mengakses database jika digunakan. |
| `Redis` | `<Mandatory/Optional>` | Mengakses cache atau konfigurasi jika digunakan. |
| `<SDK lain>` | Optional | `<Fungsi>` |

Jangan simpan credential, token, password, atau secret secara langsung di source code.

## Implementation Flow

### 1. Initialize Dependencies

1. Inisialisasi Kafka consumer.
2. Inisialisasi HTTP client untuk webhook.
3. Inisialisasi dependency lain sesuai kebutuhan service.

### 2. Consume Kafka Event

1. Terima event dari `<CONSUMER_TOPIC>`.
2. Ambil `trace_id` untuk korelasi proses.
3. Validasi struktur dan field wajib event.
4. Proses hanya event yang memenuhi `<TRIGGER_CONDITION>`.

### 3. Check Idempotency

1. Periksa `<BUSINESS_ID>` pada `<IDEMPOTENCY_STORE>`.
2. Gunakan kembali data proses yang sudah tersimpan saat event yang sama diterima ulang.
3. Jangan ulangi tahap yang sudah berstatus `SUCCESS`.
4. Lanjutkan dari tahap terakhir yang belum selesai jika desain service mendukung resume.

### 4. Load and Validate Business Data

1. Ambil data berdasarkan `<BUSINESS_ID>` dari `<DATA_SOURCE>`.
2. Validasi field yang dibutuhkan proses.
3. Hentikan proses jika error bersifat non-retryable.

### 5. Execute Business Process

1. `<BUSINESS_STEP_1>`
2. `<BUSINESS_STEP_2>`
3. `<BUSINESS_STEP_3>`
4. Simpan status setiap tahap yang diperlukan untuk retry atau resume.

### 6. Call Outbound Webhook

1. Bentuk request sesuai Outbound Webhook Contract.
2. Gunakan trace ID yang sama dengan event Kafka.
3. Panggil `<WEBHOOK_TARGET>` setelah `<WEBHOOK_TRIGGER>` terpenuhi.
4. Simpan hasil pemanggilan webhook jika service menyimpan status integrasi.
5. Jalankan retry untuk error webhook yang dikategorikan retryable.

### 7. Acknowledge Kafka

Commit offset atau acknowledge event setelah kriteria sukses `<ACK_SUCCESS_CRITERIA>` terpenuhi.

Jangan acknowledge event sebelum tahap yang wajib selesai berhasil diproses.

## Retry dan Dead-Letter Topic

### Retryable Error

| Sumber | Contoh |
| --- | --- |
| Database | Timeout atau koneksi sementara gagal. |
| Redis | Timeout atau koneksi sementara gagal. |
| External Storage | Timeout atau gangguan jaringan. |
| Webhook | Timeout, gangguan jaringan, atau response yang dikategorikan retryable. |
| Kafka | Gangguan koneksi consumer. |

### Retry Configuration

| Configuration | Value |
| --- | --- |
| `maxAttempts` | `<MAX_ATTEMPTS>` |
| `initialBackoff` | `<INITIAL_BACKOFF>` |
| `backoffMultiplier` | `<BACKOFF_MULTIPLIER>` |
| `maxBackoff` | `<MAX_BACKOFF>` |

### Non-Retryable Error

- Event tidak valid.
- Field wajib event tidak tersedia.
- Data bisnis tidak ditemukan.
- Data melanggar aturan validasi bisnis.
- `<NON_RETRYABLE_ERROR_LAIN>`

### Retry Topic dan DLT

```text
<RETRY_TOPIC>
<DEAD_LETTER_TOPIC>
```

Kirim event ke DLT jika `<DLT_CONDITION>` terpenuhi.

## Failure Handling

| Kondisi | Status Proses | Webhook | Kafka Ack | Tindakan |
| --- | --- | --- | --- | --- |
| Event tidak valid | Failed | Tidak dipanggil | `<ACK/DLT>` | `<Tindakan>` |
| Proses bisnis gagal, retryable | Pending retry | Tidak dipanggil | Tidak | Retry. |
| Proses bisnis gagal, non-retryable | Failed | Tidak dipanggil | `<ACK/DLT>` | `<Tindakan>` |
| Proses bisnis sukses, webhook sukses | Success | Success | Ya | Selesaikan proses. |
| Proses bisnis sukses, webhook gagal retryable | Pending retry | Failed | Tidak | Retry webhook sesuai konfigurasi. |
| Webhook gagal setelah batas retry | Failed | Failed | `<ACK/DLT>` | `<Tindakan sesuai desain service>` |

## Tracker

Gunakan bagian ini jika service mencatat tracker atau audit trail.

| Column | Contoh Value | Description |
| --- | --- | --- |
| `<business_id>` | `<ID>` | ID transaksi utama. |
| `created_by` | `{"id":"<ID>","name":"<SERVICE_NAME>"}` | Identitas service pemrakarsa. |
| `action_type` | `<ACTION_TYPE>` | Jenis aktivitas. |
| `action_result` | `SUCCESS` / `FAILED` | Hasil aktivitas. |
| `action_notes` | `<NOTES>` | Detail hasil atau kode error. |
| `show_to` | `<VISIBILITY>` | Visibilitas tracker. |

### Tracker Mapping

| Tahap | `action_type` | Success | Failed |
| --- | --- | --- | --- |
| Idempotency check | `<DUPLICATION_CHECK>` | `SUCCESS` | `FAILED` |
| Load business data | `<GET_DATA>` | `SUCCESS` | `FAILED` |
| Business process | `<PROCESS_DATA>` | `SUCCESS` | `FAILED` |
| Outbound webhook | `<WEBHOOK_STATUS>` | `SUCCESS` | `FAILED` |

## Logging Standards (Kafka Application Logs)

Semua log aplikasi dipublikasikan secara asinkron ke Kafka Topic `hibank.qris.application.logs` sesuai dengan standar REQ-NFR-008 (referensi: `system-design/docs/technical-docs/kafka-logging-standards.md`).

| Atribut | Nilai |
| --- | --- |
| Topic | `hibank.qris.application.logs` |
| Partition Key | OTel `traceId` |
| Format | JSON Envelope |
| Consumer | Data Prepper (`hibank-qris-data-prepper`) |

### Contoh Envelope Log Kafka

```json
{
  "timestamp": "2026-06-22T16:00:00.000+07:00",
  "trace_id": "ext-20260622-0001",
  "service": {
    "name": "four-eyes-management"
  },
  "deployment": {
    "environment": "production"
  },
  "payload": {
    "level": "INFO",
    "message": "Create approval request processed",
    "context": {
      "action": "CREATE_APPROVAL_REQUEST",
      "status": "SUCCESS"
    }
  }
}
```

## Implementation Checklist

| No | Aktivitas | Status | Catatan |
| ---: | --- | :---: | --- |
| 1 | Consumer menerima event Kafka | | |
| 2 | Event tervalidasi | | |
| 3 | Idempotency check berhasil | | |
| 4 | Data bisnis berhasil dimuat dan divalidasi | | |
| 5 | Proses bisnis berhasil | | |
| 6 | Request webhook terbentuk sesuai kontrak | | |
| 7 | Webhook berhasil dipanggil | | |
| 8 | Status proses dan tracker berhasil diperbarui | | |
| 9 | Kafka offset berhasil di-commit atau di-acknowledge | | |
| 10 | Retry dan DLT tervalidasi untuk skenario gagal | | |

