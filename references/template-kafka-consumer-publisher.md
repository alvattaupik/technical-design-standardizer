# [Service Name]



## General Information

| Item | Keterangan |
| --- | --- |
| **Protocol** | Kafka Consumer & Publisher (Event-Driven) |
| **Consumer Topic** | `[consumer.topic]` |
| **Publisher Topic** | `[publisher.topic]` |
| **Description** | Service menerima event dari Kafka, menjalankan proses bisnis, lalu mempublikasikan event hasil ke topic tujuan. |
| **Category** | `[Service Category]` |
| **Kode Service** | `[service-code]` |

| **Tabel yang Digunakan** | `[table_1]`, `[table_2]`, `[table_n]` |

### Version Control

| Version | Date | Author | Remarks |
| --- | --- | --- | --- |
| 1.0.0 | YYYY-MM-DD | System Analyst | Initial creation |

---

## Event Messaging

### Consumer Contract

Service menerima event dari Kafka sebagai pemicu proses bisnis.

| Item | Value |
| --- | --- |
| **Topic** | `[consumer.topic]` |
| **Partition** | `[partition]` |
| **Key** | `[eventKey]` |
| **Header** | `trace-id` |

#### Consumer Payload

```json
{
  "timestamp": "2026-06-22T16:00:10.000Z",
  "trace_id": "ext-20260622-0001",
  "service": {
    "name": "[source-service]"
  },
  "deployment": {
    "environment": "production"
  },
  "payload": {
    "field1": "value1",
    "field2": "value2"
  }
}
```

#### Consumer Validation

| Validation | Rule |
| --- | --- |
| **JSON Format** | Payload harus berupa JSON valid. |
| **Mandatory Fields** | Field wajib harus tersedia dan tidak kosong. |
| **Field Format** | Validasi format setiap field sesuai kontrak event. |
| **Consistency** | Validasi hubungan antar-field jika kontrak mensyaratkannya. |

Jika event tidak valid, catat error dan proses sesuai kebijakan non-retryable agar event tidak diproses berulang tanpa hasil.

### Publisher Contract

Service mempublikasikan event setelah seluruh proses yang menjadi prasyarat publish selesai.

| Item | Value |
| --- | --- |
| **Topic** | `[publisher.topic]` |
| **Partition** | `[partition]` |
| **Key** | `[eventKey]` |
| **Header** | `trace-id`, `[additional-header]` |

#### Publisher Payload

```json
{
  "timestamp": "2026-06-22T16:10:00.000Z",
  "trace_id": "ext-20260622-0001",
  "service": {
    "name": "[publisher-service]"
  },
  "deployment": {
    "environment": "production"
  },
  "payload": {
    "field1": "value1",
    "field2": "value2"
  }
}
```

### Consumer to Publisher Mapping

Gunakan tabel ini hanya untuk field publisher yang bersumber dari event consumer atau hasil proses bisnis.

| Publisher Field | Source | Mapping Rule |
| --- | --- | --- |
| `trace_id` | Consumer header / payload | Pertahankan trace ID yang sama selama satu alur proses. |
| `[publisher.payload.field1]` | `[consumer.payload.field1]` | Direct mapping |
| `[publisher.payload.field2]` | `[database/process-result]` | Isi dari hasil proses bisnis. |

---

## Redis Operations

Gunakan bagian ini jika service memakai Redis.

Format Redis Key:
`[nama_aplikasi].[domain].[fitur].[fungsi].[jenis_identitas].{nilai_identitas}`

| Jenis Identitas | Redis Key Pattern | Data Type | TTL | Kegunaan |
| --- | --- | :---: | :---: | --- |
| **Distributed Lock** | `[app].[domain].[feature].lock.{eventKey}` | `STRING` (`PX`/`NX`) | `[TTL]` | Mencegah pemrosesan event yang sama secara bersamaan. |

---

## Dependency dan Komponen

| Komponen | Status | Fungsi |
| --- | :---: | --- |
| Kafka | Mandatory | Menerima event consumer dan mempublikasikan event hasil. |
| PostgreSQL | `[Mandatory/Optional]` | Menyimpan data transaksi dan status proses. |
| Redis | Optional | Menyimpan distributed lock atau state idempotensi. |
| `[SDK/Library]` | `[Mandatory/Optional]` | `[Fungsi]` |

Jangan simpan password, token, access key, secret key, atau credential lain di source code.

---

## Implementation Flow

### 1. Initialize Dependencies

Inisialisasi library, koneksi, dan konfigurasi yang diperlukan service.

### 2. Listen Consumer Event

Dengarkan event dari topic `[consumer.topic]`.

Validasi:

- Format JSON.
- Mandatory field.
- Format field.
- Konsistensi identifier.

Jika event tidak valid, catat error dan jalankan kebijakan non-retryable.

### 3. Check Idempotency and Concurrency

Periksa apakah event sudah selesai diproses.

Jika Redis digunakan, ambil distributed lock berdasarkan event key sebelum menjalankan proses yang mengubah data.

Jika proses sudah `SUCCESS`, jangan ulangi perubahan data atau publish event.

### 4. Execute Business Process

Jalankan proses bisnis sesuai urutan service.

1. `[Business step 1]`
2. `[Business step 2]`
3. `[Business step 3]`
4. `[Business step n]`

Catat status setiap tahap yang membutuhkan audit atau recovery.

### 5. Update Processing Status

Perbarui status transaksi sesuai hasil proses.

```text
update [processing_table]:
  processing_status         = SUCCESS/FAILED
  processing_status_message = [message]
```

### 6. Build Publisher Event

Bentuk payload publisher dari event consumer, data transaksi, dan hasil proses bisnis.

Pastikan:

- `trace-id` tetap konsisten.
- Event key sesuai kontrak publisher.
- Data sensitif tidak dikirim jika tidak diperlukan oleh kontrak.
- Payload sesuai schema publisher.

### 7. Publish Event

Publikasikan event ke topic `[publisher.topic]`.

Jika publish berhasil:

```text
update [processing_table]:
  notification_status = SUCCESS
  sync_status         = SUCCESS
```

Jika publish gagal, jalankan retry atau mekanisme fallback yang ditetapkan.

### 8. Commit Consumer Offset

Commit offset setelah seluruh kriteria sukses selesai, termasuk publish event jika publish merupakan bagian dari transaksi utama.

### 9. Cleanup

Hapus file lokal, temporary state, atau lock yang tidak lagi diperlukan.

---

## Tracker

Gunakan tabel `[tracker_table]` jika service membutuhkan audit tracker.

| Column | Contoh Value | Aturan |
| --- | --- | --- |
| `transaction_id` | `[transaction_id]` | Identifier transaksi utama. |
| `created_by` | `{"id":"service","name":"[service-name]"}` | Identitas service pemrakarsa. |
| `action_type` | `[ACTION_TYPE]` | Kode aktivitas. |
| `action_result` | `SUCCESS` / `FAILED` | Hasil aktivitas. |
| `action_notes` | `-` / `[ERROR_CODE]` | Detail singkat atau kode error. |
| `show_to` | `MERCHANT_AND_BANK` / `BANK_ONLY` | Visibilitas tracker. |

### Action Type Mapping

| Step | Action Type | Success Condition | Failure Result |
| ---: | --- | --- | --- |
| 1 | `[ACTION_STEP_1]` | Tahap selesai | `FAILED` |
| 2 | `[ACTION_STEP_2]` | Tahap selesai | `FAILED` |
| 3 | `[ACTION_STEP_3]` | Tahap selesai | `FAILED` |
| n | `[PUBLISH_EVENT]` | Event berhasil dipublikasikan | `FAILED` |

---

## Retry dan Dead-Letter Topic

### Retryable Error

- Kafka broker timeout.
- Database timeout.
- Network timeout.
- Gangguan dependency sementara.

Contoh konfigurasi:

```text
maxAttempts       : 5
initialBackoff    : 1 detik
backoffMultiplier : 2
maxBackoff        : 30 detik
```

### Non-Retryable Error

- Payload event tidak valid.
- Struktur file atau data input tidak sesuai kontrak.
- Mandatory field tidak tersedia.
- Data tidak dapat diproses karena error permanen.

### Retry dan DLT Topic

```text
[consumer.topic].retry
[consumer.topic].dlt
```

Kirim event ke DLT jika:

- Retry mencapai batas maksimum.
- Event mengalami non-retryable error.
- Data input tidak dapat diproses secara permanen.

---

## Kafka Acknowledgment

Commit offset setelah seluruh proses yang diwajibkan berhasil.

```text
Receive Consumer Event
  |
Validate Event
  |
Check Idempotency / Lock
  |
Execute Business Process
  |
Update Processing Status
  |
Build Publisher Event
  |
Publish Event
  |
Commit Consumer Offset
```

Jangan commit offset sebelum tahap yang menjadi syarat sukses selesai.

---

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

---

## Checklist Implementasi

### Normal Flow

| No | Aktivitas | Status | Catatan |
| ---: | --- | :---: | --- |
| 1 | Consumer menerima event dari `[consumer.topic]` | | |
| 2 | Event lolos validasi | | |
| 3 | Pemeriksaan idempotensi dan concurrency selesai | | |
| 4 | Proses bisnis selesai | | |
| 5 | Status transaksi diperbarui | | |
| 6 | Payload publisher berhasil dibentuk | | |
| 7 | Event berhasil dipublikasikan ke `[publisher.topic]` | | |
| 8 | Tracker sukses tersimpan | | |
| 9 | Consumer offset berhasil di-commit | | |
| 10 | Temporary resource dibersihkan | | |

### Failure Flow

| No | Skenario | Expected Handling |
| ---: | --- | --- |
| 1 | Consumer payload tidak valid | Tandai sebagai non-retryable dan proses sesuai kebijakan DLT. |
| 2 | Dependency mengalami transient error | Jalankan retry dengan backoff. |
| 3 | Proses bisnis gagal permanen | Simpan status `FAILED` dan kirim event ke DLT sesuai kebijakan. |
| 4 | Publisher gagal mengirim event | Jalankan retry atau fallback yang ditetapkan. |

### Idempotency dan Concurrency

| No | Skenario | Expected Result |
| ---: | --- | --- |
| 1 | Event yang sama diterima dua kali | Jangan duplikasi perubahan data atau publisher event. |
| 2 | Dua worker memproses event key yang sama | Hanya satu worker menjalankan critical process pada satu waktu. |
| 3 | Event selesai lalu terkirim ulang | Kenali status akhir dan hentikan reprocessing. |

