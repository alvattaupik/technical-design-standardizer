---
name: technical-design-standardizer
description: Standarkan, tinjau, dan perbaiki dokumen Technical Design backend/API berdasarkan pola integrasinya. Gunakan alat ini untuk menata ulang, menstandardisasi, mengaudit logika, atau memperbaiki alur Technical Design untuk REST API murni, API + Kafka Publisher, Kafka Consumer + Webhook, Kafka Consumer + Kafka Publisher, atau API + Kafka Consumer + Kafka Publisher. Pertahankan identifikasi teknis dan aturan bisnis yang diberikan.
---

# Technical Design Standardizer

Gunakan alur kerja ini untuk mengubah Technical Design menjadi dokumen yang siap diimplementasikan tanpa membuat asumsi tidak berdasar pada aturan bisnis.

## Alur Kerja

1. Identifikasi pola integrasi dokumen.
2. Baca templat acuan (reference) yang sesuai.
3. Lakukan inventarisasi pengenal (identifier) yang harus dipertahankan: endpoint, method, topic, partition, key, header, table, stored procedure, status, action type, response code, Redis key, template code, dan field payload.
4. Tinjau logika sebelum menata ulang struktur.
5. Perbaiki logika yang tidak valid atau tidak lengkap, apabila perbaikan tersebut bersifat murni teknis dan tidak mengubah keputusan bisnis.
6. Apabila terdapat dua fakta sumber yang bertentangan, jangan mengambil keputusan tanpa konfirmasi dan pertahankan konteks asli.
7. Susun ulang dokumen mengikuti templat yang telah dipilih. Hilangkan pengulangan informasi terkait tracker, retry, logging, atau dependency yang tidak memberikan nilai tambah.
8. Lakukan pemeriksaan konsistensi pada seluruh kontrak, contoh payload, sequence diagram, alur proses, status, dan penanganan kegagalan (failure handling).

## Pemilihan Templat

| Pola | Acuan (Reference) |
| --- | --- |
| REST API tanpa Kafka/Webhook | `references/template-api.md` |
| REST API yang mem-publish event Kafka | `references/template-api-kafka-publisher.md` |
| Kafka Consumer yang memanggil outbound webhook | `references/template-kafka-consumer-webhook.md` |
| Kafka Consumer yang mem-publish event Kafka | `references/template-kafka-consumer-publisher.md` |
| REST API + Kafka Consumer sebagai entry point dan Kafka Publisher sebagai output | `references/template-api-kafka-consumer-publisher.md` |

Jangan memaksakan pola terdekat apabila arsitektur yang sebenarnya berbeda. Jika suatu dokumen mencakup integrasi lain, sesuaikan strukturnya dengan prinsip kontrak masuk, proses bisnis, dan kontrak keluar yang setara.

## Referensi Tambahan

Anda dapat mengacu pada dokumen berikut sebagai contoh tambahan mengenai format dan struktur dokumen yang diharapkan:
- `examples/reg-check-nik-nib.md`

## Tinjauan Logika Wajib

Lakukan pemeriksaan berikut sebelum melakukan finalisasi:

1. Entry point dan pemicu (trigger) benar-benar sesuai dengan alur bisnis.
2. Validasi request/event mencakup struktur JSON, kewajiban parameter (mandatory), format, enumerasi, field kondisional, dan konsistensi pengenal yang relevan.
3. Autentikasi dan otorisasi diletakkan sebelum proses perubahan data (mutasi), apabila endpoint mensyaratkannya.
4. Validasi bisnis dijalankan sebelum mutasi data.
5. Transaksi basis data memiliki batasan komit (commit) dan pembatalan (rollback) yang tegas.
6. Prinsip idempotensi mencegah proses duplikat (request/event ganda) yang dapat menghasilkan mutasi atau publikasi berulang.
7. Mekanisme distributed lock digunakan hanya apabila terdapat risiko mutasi konkuren pada kunci bisnis yang sama.
8. Upaya ulang (retry) hanya diterapkan untuk kendala sementara. Kesalahan validasi, kerusakan payload, atau penolakan logika bisnis tidak boleh dilakukan upaya ulang tanpa alasan yang tepat.
9. Pada consumer, komitmen offset (ACK) dijalankan setelah kriteria keberhasilan terpenuhi.
10. Pada publisher, antisipasi skenario di mana komit basis data berhasil, tetapi publikasi Kafka gagal. Berikan rekomendasi penggunaan transactional outbox apabila komit atomik (basis data + Kafka) tidak tersedia.
11. Pada webhook, tentukan batasan waktu (timeout), kriteria keberhasilan, kunci idempotensi (jika relevan), dan kebijakan upaya ulang.
12. Status `SUCCESS`, `FAILED`, `PENDING`, atau status domain lainnya tidak boleh saling bertentangan antartahapan.
13. Tracker ditempatkan pada aktivitas yang bernilai audit. Hindari duplikasi tabel pencatatan pada setiap sub-bagian; gunakan satu tabel pemetaan jika dirasa memadai.
14. Pembersihan berkas sementara dan pelepasan kunci (lock) harus selalu dijalankan, baik pada alur proses berhasil maupun gagal.
15. Trace ID harus konsisten di seluruh integrasi API, Kafka, webhook, catatan log, dan tracker.
16. Data sensitif tidak diizinkan untuk dicatat secara utuh ke dalam log aplikasi.
17. **Standardisasi OTP**: Semua proses *request/generate* OTP tidak boleh dilakukan melalui pemanggilan API REST sinkron ke OTP Service. Wajib distandarkan menggunakan mekanisme **Kafka Publisher** ke topik `hibank.qris.otp.generate.requested` (mengacu pada contoh format envelope di `reg-check-nik-nib.md`).
18. **Standardisasi Application Logs**: Setiap dokumen wajib mencantumkan klausul bahwa seluruh log aplikasi dipublikasikan secara asinkron ke Kafka Topic `hibank.qris.application.logs` sesuai standar REQ-NFR-008.
19. **Standardisasi Request Header**: Tabel Request Header untuk seluruh REST API wajib berisi tepat 3 atribut header standar (urutan, penamaan bold + backtick, format markdown `|----|-----|------|-------|-------------|`, dan deskripsi) mengacu pada `reg-check-email.md`:
    ```markdown
    #### Request Header

    | No | Key | Type | M/O/C | Description |
    |----|-----|------|-------|-------------|
    | 1 | **`Content-Type`** | String | M | Wajib: `application/json` |
    | 2 | **`X-TIMESTAMP`** | String | M | Waktu request sesuai standar ISO8601. Format: `YYYY-MM-DDThh:mm:ssTZD` |
    | 3 | **`traceId`** | String | M | Kode unik tracing request. |
    ```
20. **Standardisasi API List Back Office**: Setiap API penyajian daftar data (List API) pada Web Back Office wajib menggunakan struktur request body (`search`, `filter`, `pagination` (`page` & `limit` mandatory), `sorting`) dan response body envelope (`result`: `totalData`, `totalPages`, `currentPage`, `data`) sesuai dengan standar pada `design-list-role.md`.
21. **Standardisasi Sequence Diagram**: Setiap dokumen wajib mencantumkan bagian `## Sequence Diagram` menggunakan diagram Mermaid (mengacu pada contoh di `reg-check-email.md`), yang diletakkan setelah `Version Control` dan sebelum `HTTP REST Contract` / `JSON Messaging`.
22. **Standardisasi Response Body Envelope & Response Code Mapping**: Seluruh dokumen Technical Design REST API wajib menggunakan struktur *envelope* `Response Body` 6 atribut standar (`responseCode`, `responseMessage`, `traceId`, `timestamp`, `result`, `additionalInfo`) dan tabel `### Response Code Mapping` dengan 4 kolom (`| No | HTTP Status | Response Code | Description |`) serta contoh respons sukses & gagal lengkap mengacu pada `reg-check-email.md`.
23. **Standardisasi Log Activity via Kafka Publisher**: Apabila proses API atau transaksi membutuhkan publikasi log/event secara asinkron via Kafka, gunakan protokol `HTTP REST + Kafka Publisher` dan wajib mencantumkan seksi `## Event Messaging (Kafka Publisher)` (memuat tabel atribut `Topic`, `Partition`, `Publisher`, `Consumer`, `Message format`, `Key`, `Headers`, serta contoh JSON envelope) dan seksi `## Dependency dan Komponen` mengacu pada contoh di `reg-check-email.md`.
24. **Standardisasi Penentuan Kode Service dari Nomor FEAT**: Kode Service wajib diambil langsung dari 2-digit angka nomor FEAT tempat fitur tersebut berada (misal: FEAT-001 = Kode Service `01`, FEAT-002 = Kode Service `02`, FEAT-007 = Kode Service `07`, FEAT-015 = Kode Service `15`). Seluruh penulisan `responseCode` 7-digit standar SNAP BI wajib mengikuti format `{HTTP_STATUS}{KODE_SERVICE}{INDEX}` (contoh untuk FEAT-007: `2000700`, `4000700`, `4040700`, `5000700`).
25. **Standardisasi API Gateway pada Sequence Diagram**: Seluruh alur internal Web Back Office, Mobile App, atau antar-service internal wajib menggunakan **API Gateway Internal** (`Internal API Gateway`). **Kong API Gateway** (`Kong API Gateway`) KHUSUS digunakan untuk jalur integrasi/callback dari/ke pihak eksternal Jaringan Switching (Artajasa, Jalin, PTEN). DILARANG menggunakan Kong untuk alur internal Web Back Office.
26. **Penghapusan Atribut Kode Template**: DILARANG mencantumkan baris **Kode Template** (misal: `| **Kode Template** | ... |`) pada tabel `## General Information`. Seluruh dokumen Technical Design tidak boleh memuat baris Kode Template.
27. **Standardisasi Fokus Checklist Implementasi**: Tabel `## Checklist Implementasi` wajib berfokus KHUSUS pada alur teknis implementasi (misal: validasi header/payload, verifikasi record di DB, penanganan duplikasi, eksekusi query INSERT/UPDATE/SELECT DB, penanganan Redis cache, dan pembentukan response envelope). DILARANG memasukkan poin meta/dokumentasi seperti kepatuhan aturan skill, ketiadaan baris Kode Template, atau kelengkapan dokumen.
28. **Standardisasi Format User Object CreatedBy / UpdatedBy**: Seluruh atribut `createdBy` dan `updatedBy` pada Request/Response JSON contract dan dokumentasi wajib menggunakan format JSON Object dengan contoh nama Jane / John (contoh: `{"id": "1", "name": "Jane"}` atau `{"id": "1", "name": "John"}`).

## Aturan Perbaikan

- Pertahankan pengenal teknis sumber kecuali jika terdapat konflik yang tertulis secara eksplisit.
- Pertahankan peristilahan domain yang dimiliki oleh pengguna.
- Dilarang menciptakan topic, table, response code, partition, stored procedure, enum status, TTL, SLA, atau kredensial secara sepihak.
- Gunakan teks substitusi (placeholder) untuk informasi teknis yang belum tersedia.
- Apabila terdapat kekurangan teknis yang dapat disimpulkan dengan pasti secara teknis, integrasikan perbaikan tersebut langsung ke dalam alur proses.
- Pisahkan antara fakta sumber dan rekomendasi teknis.
- Pada bagian `### Version Control`, pertahankan riwayat aslinya tanpa menambahkan entri revisi baru (misal: atas nama AI Assistant). Gunakan format tabel rapat (`|---|---|---|---|`) dan berikan garis pembatas horizontal (`---`) di bawah tabel, sesuai contoh pada `reg-check-nik-nib.md`.

## Output Minimum

Dokumen final wajib memuat: Informasi Umum (General Information), Version Control, Sequence Diagram (`## Sequence Diagram` dalam format Mermaid), kontrak input, kontrak output (Response Body & Mapping Response Code), Event Messaging (apabila menggunakan Kafka Publisher), operasi basis data/Redis (apabila relevan), dependensi dan komponen, alur implementasi, penanganan kendala (error handling), mekanisme idempotensi/konkurensi sesuai spesifikasi, pencatatan tracker (apabila digunakan), standar logging, dan daftar periksa implementasi (checklist).

**PENTING**: Seluruh hasil dokumen Markdown (.md) yang distandarisasi atau disusun harus langsung menimpa atau memperbarui berkas asli yang sedang diproses (in-place modification). Dilarang menyimpan hasilnya di dalam direktori terpisah kecuali ada instruksi eksplisit untuk melakukannya.

Apabila pengguna hanya meminta peninjauan (review) tanpa perubahan berkas, cukup berikan hasil temuan dan usulan tanpa menciptakan artefak dokumen yang baru.
