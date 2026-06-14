## 🎫 Ticketing (Support Tickets)

**Sisi User (`/`):**
- `support.php` — List semua tiket milik user + tombol buat tiket baru
- `support-detail.php?id=xxx` — Detail percakapan satu tiket (thread reply)

**Sisi Admin (`/admin/`):**
- `admin/tickets.php` — List semua tiket dari semua user, bisa filter by status/prioritas
- `admin/ticket-detail.php?id=xxx` — Balas tiket, ubah status, assign ke agen

**API:**
- `api/tickets.php` — CRUD tiket + reply

---

## 📬 Shared Inbox

**Sisi Admin saja:**
- `admin/inbox.php` — Semua pesan masuk (dari semua channel: tiket, email, dll) dalam satu tampilan
- *(Tidak perlu halaman terpisah di sisi user, cukup terintegrasi dengan support.php)*

---

## 📚 Knowledge Base

**Sisi User (publik):**
- `help.php` — Halaman utama KB: list kategori + search
- `help-article.php?slug=xxx` — Detail artikel

**Sisi Admin:**
- `admin/kb-categories.php` — Kelola kategori KB
- `admin/kb-articles.php` — List semua artikel
- `admin/kb-article-editor.php?id=xxx` — Editor artikel (buat/edit, bisa WYSIWYG)

**API:**
- `api/kb.php` — CRUD artikel & kategori

---

## 📊 Reporting

**Sisi Admin saja:**
- `admin/reports.php` — Dashboard laporan: tiket per periode, rata-rata waktu respons, tiket per agen, volume per kategori

*(Bisa jadi satu halaman dengan tab/filter, tidak perlu dipecah)*

---

## 🔧 Sistem Repair

Ini tergantung konteksnya — apakah "repair" maksudnya **laporan kerusakan/bug dari user**, atau **maintenance internal**? Bisa dua arah:

**Sisi User:**
- `repair.php` — Form laporan kerusakan/kendala teknis (submit request perbaikan)

**Sisi Admin:**
- `admin/repair.php` — List semua request repair, status penanganan, assign teknisi
- `admin/repair-detail.php?id=xxx` — Detail & update status repair

---

## Ringkasan Jumlah Halaman

| Fitur | User | Admin | API |
|---|---|---|---|
| Ticketing | 2 | 2 | 1 |
| Shared Inbox | — | 1 | — |
| Knowledge Base | 2 | 3 | 1 |
| Reporting | — | 1 | — |
| Repair | 1 | 2 | 1 |
| **Total** | **5** | **9** | **3** |
