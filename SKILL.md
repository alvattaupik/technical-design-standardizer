---
name: technical-design-standardizer
description: Standarkan, review, dan perbaiki dokumen Technical Design backend/API berdasarkan pola integrasinya. Gunakan saat Codex diminta merapikan, menstandardisasi, mengaudit logic, atau memperbaiki alur Technical Design untuk REST API murni, API + Kafka Publisher, Kafka Consumer + Webhook, Kafka Consumer + Kafka Publisher, atau API + Kafka Consumer + Kafka Publisher. Pertahankan identifier teknis dan business rule yang diberikan, tandai konflik atau keputusan bisnis yang belum dapat dipastikan sebagai Open Point.
---

# Technical Design Standardizer

Gunakan workflow ini untuk mengubah Technical Design menjadi dokumen implementable tanpa mengarang business rule.

## Workflow

1. Identifikasi pola integrasi dokumen.
2. Baca hanya template reference yang sesuai.
3. Inventarisasi identifier yang harus dipertahankan: endpoint, method, topic, partition, key, header, table, stored procedure, status, action type, response code, Redis key, template code, dan field payload.
4. Review logic sebelum merapikan struktur.
5. Perbaiki logic yang jelas salah atau tidak lengkap jika perbaikannya bersifat teknis dan tidak mengubah keputusan bisnis.
6. Jika ada dua fakta sumber yang bertentangan, jangan memilih diam-diam. Pertahankan konteks dan masukkan ke `Open Points`.
7. Susun ulang dokumen mengikuti template terpilih. Hilangkan pengulangan tracker, retry, logging, atau dependency yang tidak menambah informasi.
8. Lakukan consistency pass pada seluruh kontrak, contoh payload, alur proses, status, dan failure handling.

## Pemilihan Template

| Pola | Reference |
| --- | --- |
| REST API tanpa Kafka/Webhook | `references/template-api.md` |
| REST API yang mem-publish event Kafka | `references/template-api-kafka-publisher.md` |
| Kafka Consumer yang memanggil outbound webhook | `references/template-kafka-consumer-webhook.md` |
| Kafka Consumer yang mem-publish event Kafka | `references/template-kafka-consumer-publisher.md` |
| REST API + Kafka Consumer sebagai entry point dan Kafka Publisher sebagai output | `references/template-api-kafka-consumer-publisher.md` |

Jangan memaksakan pola terdekat jika arsitektur aktual berbeda. Jika satu dokumen memuat integrasi lain, adaptasi struktur dengan prinsip kontrak masuk, business process, dan kontrak keluar yang sama.

## Review Logic Wajib

Periksa sebelum finalisasi:

1. Entry point dan trigger benar-benar sesuai dengan alur bisnis.
2. Request/event validation mencakup JSON, mandatory, format, enum, conditional field, dan konsistensi identifier yang relevan.
3. Authentication dan authorization berada sebelum perubahan data jika endpoint membutuhkannya.
4. Business validation terjadi sebelum mutation.
5. Database transaction memiliki batas commit dan rollback yang jelas.
6. Idempotency mencegah request/event duplikat menghasilkan mutation atau publish ganda.
7. Distributed lock dipakai hanya jika ada risiko concurrent mutation pada business key yang sama.
8. Retry hanya berlaku untuk error sementara. Validation error, corrupt payload, dan business rejection jangan di-retry tanpa alasan.
9. Untuk consumer, ACK/commit offset dilakukan setelah success criteria terpenuhi.
10. Untuk publisher, tangani kasus database commit sukses tetapi publish Kafka gagal. Rekomendasikan transactional outbox bila atomic DB + Kafka tidak tersedia.
11. Untuk webhook, tetapkan timeout, success criteria, idempotency key bila relevan, dan retry policy.
12. Status `SUCCESS`, `FAILED`, `PENDING`, atau status domain lain tidak boleh saling bertentangan antarstep.
13. Tracker ditempatkan pada aktivitas yang bernilai audit. Jangan menduplikasi tabel tracker identik pada setiap subbagian jika satu mapping table cukup.
14. Cleanup temporary file dan lock harus tetap terjadi pada jalur sukses maupun gagal jika resource tersebut digunakan.
15. Trace ID harus konsisten melintasi API, Kafka, webhook, log, dan tracker.
16. Data sensitif tidak boleh ditulis utuh ke application log.

## Aturan Perbaikan

- Pertahankan identifier teknis sumber kecuali ada konflik eksplisit.
- Pertahankan istilah domain milik user.
- Jangan mengarang topic, table, response code, partition, stored procedure, enum status, TTL, SLA, atau credential.
- Gunakan placeholder untuk informasi teknis yang memang belum tersedia.
- Untuk kekurangan teknis yang aman disimpulkan, tambahkan perbaikannya langsung ke flow.
- Untuk keputusan yang membutuhkan product owner, architect, security, DBA, atau pihak eksternal, masukkan ke `Open Points` dengan pertanyaan yang spesifik.
- Pisahkan fakta sumber, rekomendasi teknis, dan Open Point.

## Output Minimum

Dokumen akhir minimal memiliki General Information, kontrak input, kontrak output bila ada, database/Redis operations bila relevan, dependency, implementation flow, error handling, idempotency/concurrency sesuai kebutuhan, tracker bila digunakan, logging, checklist implementasi, dan Open Points bila masih ada konflik.

**PENTING**: Seluruh hasil dokumen Markdown (.md) yang di-generate atau distandarisasi harus langsung menimpa atau memperbarui file asli yang sedang dikerjakan (in-place modification). Jangan menyimpan hasilnya di folder terpisah kecuali diminta secara eksplisit.

Jika user hanya meminta review dan tidak meminta perubahan file, berikan temuan dan usulan tanpa membuat artefak baru.
