# ATSALARA — Blueprint Fitur Baru

> Ticketing · Shared Inbox · Knowledge Base · Reporting · Sistem Repair  
> Dokumen Teknis

---

## Ringkasan Eksekutif

| Fitur | Halaman User | Halaman Admin | API | Total |
|---|---|---|---|---|
| Ticketing | 2 | 2 | 1 | 5 |
| Shared Inbox | — | 1 | — | 1 |
| Knowledge Base | 2 | 3 | 1 | 6 |
| Reporting | — | 1 | — | 1 |
| Sistem Repair | 1 | 2 | 1 | 4 |
| **TOTAL** | **5** | **9** | **3** | **17** |

---

## 1. Fitur Ticketing (Support Tickets)

Artist dapat membuat tiket dukungan untuk melaporkan masalah, mengajukan pertanyaan, atau meminta bantuan. Admin dapat melihat, merespons, dan menutup tiket dari panel admin.

### 1.1 Halaman yang Dibutuhkan

#### Sisi User (Artist)

**`support.php`** — Halaman utama tiket milik user
- List semua tiket: nomor tiket, subjek, status badge (open/in_progress/closed), tanggal buat, terakhir dibalas
- Tombol "Buat Tiket Baru" → buka modal/form
- Form buat tiket: pilih kategori (Rilisan, Royalti, Akun, Teknis, Lainnya), subjek, deskripsi, upload lampiran (opsional)
- Filter tiket: Semua / Terbuka / Diproses / Selesai
- Empty state jika belum ada tiket

**`support-detail.php?id=xxx`** — Thread percakapan satu tiket
- Header: nomor tiket, subjek, status, kategori, tanggal buat
- Thread balasan berurutan (mirip email thread): pesan user dan admin dibedakan posisi/warna
- Form reply di bawah (hanya aktif jika status bukan "closed")
- Tombol "Tutup Tiket" oleh user jika masalah sudah selesai
- Badge "Tiket Ditutup" jika sudah closed

#### Sisi Admin

**`admin/tickets.php`** — List semua tiket
- Tabel: no. tiket, nama artist, kategori, subjek, status, prioritas, assign ke agen, tanggal buat
- Filter: status, kategori, prioritas, agen, rentang tanggal
- Search by nama artist atau subjek
- Bulk action: tutup massal, reassign massal
- Badge jumlah tiket open/unread di sidebar

**`admin/ticket-detail.php?id=xxx`** — Detail & manajemen tiket
- Header info: no. tiket, artist, kategori, prioritas, status
- Sidebar kanan: ubah status, ubah prioritas (low/medium/high/urgent), assign ke agen admin
- Thread percakapan dua arah
- Form reply admin + opsi "reply + tutup tiket" sekaligus
- Riwayat perubahan status (audit log ringkas)
- Tombol hapus tiket (admin only)

### 1.2 API — `api/tickets.php`

| Method | Endpoint | Fungsi |
|---|---|---|
| GET | `?action=list` | List tiket user / semua tiket (admin) |
| GET | `?id=xxx` | Detail satu tiket + semua reply-nya |
| POST | _(body)_ | Buat tiket baru: category, subject, message, attachment_path |
| POST | `?action=reply&id=xxx` | Tambah balasan |
| PATCH | `?id=xxx&action=status` | Ubah status (admin) |
| PATCH | `?id=xxx&action=assign` | Assign agen (admin) |
| DELETE | `?id=xxx` | Hapus tiket (admin) |

### 1.3 Database

**Tabel: `support_tickets`**

| Field | Tipe | Keterangan |
|---|---|---|
| `id` | VARCHAR(36) PK | UUID v4 |
| `ticket_number` | VARCHAR(20) UNIQUE | Format: TKT-YYYYMMDD-XXXX (auto-generate) |
| `user_id` | VARCHAR(36) FK → users | Artist yang membuat tiket |
| `category` | ENUM('release','royalty','account','technical','other') | Kategori masalah |
| `subject` | VARCHAR(255) | Judul/subjek tiket |
| `status` | ENUM('open','in_progress','waiting_user','closed') | Status tiket, default: open |
| `priority` | ENUM('low','medium','high','urgent') | Prioritas, default: medium |
| `assigned_to` | VARCHAR(36) FK → users NULL | Admin/agen yang menangani |
| `closed_at` | DATETIME NULL | Waktu tiket ditutup |
| `closed_by` | VARCHAR(36) FK → users NULL | Siapa yang menutup |
| `created_at` | DATETIME DEFAULT NOW() | Waktu dibuat |
| `updated_at` | DATETIME ON UPDATE NOW() | Waktu diperbarui terakhir |

**Tabel: `support_ticket_replies`**

| Field | Tipe | Keterangan |
|---|---|---|
| `id` | VARCHAR(36) PK | UUID v4 |
| `ticket_id` | VARCHAR(36) FK → support_tickets | Tiket induk |
| `sender_id` | VARCHAR(36) FK → users | User atau admin yang membalas |
| `sender_role` | ENUM('user','admin') | Untuk membedakan tampilan bubble |
| `message` | TEXT | Isi pesan |
| `attachment_path` | VARCHAR(500) NULL | Path file lampiran (jika ada) |
| `is_read` | TINYINT(1) DEFAULT 0 | Sudah dibaca oleh pihak seberang? |
| `created_at` | DATETIME DEFAULT NOW() | Waktu dikirim |

---

## 2. Fitur Shared Inbox

Tampilan terpusat di admin untuk melihat semua pesan masuk dari berbagai sumber (tiket baru, balasan tiket, pesan dari release) dalam satu antarmuka — mirip inbox email tim.

### 2.1 Halaman yang Dibutuhkan

#### Sisi Admin

**`admin/inbox.php`** — Inbox terpusat
- Panel kiri: list thread/percakapan diurutkan berdasarkan aktivitas terbaru
- Setiap item: avatar artist, nama, preview pesan terakhir, waktu, badge unread count
- Filter: Semua | Tiket Support | Pesan Rilisan | Unread
- Panel kanan: detail thread yang dipilih (percakapan full)
- Search by nama artist atau isi pesan
- Quick-reply langsung dari inbox tanpa pindah halaman
- Tombol "Tandai Dibaca Semua"

> **Catatan:** Shared Inbox tidak membutuhkan tabel database baru. Inbox mengagregasi data dari tabel yang sudah ada (`support_ticket_replies`, `release_messages`) dengan query UNION + ORDER BY waktu terbaru.

### 2.2 Logika Query Inbox

```sql
SELECT source, id, user_id, preview, created_at FROM (
  SELECT 'ticket' AS source, t.id, t.user_id, r.message AS preview, r.created_at
  FROM support_ticket_replies r
  JOIN support_tickets t ON r.ticket_id = t.id
  WHERE r.sender_role = 'user'

  UNION ALL

  SELECT 'release' AS source, rm.release_id AS id, rel.user_id, rm.message AS preview, rm.created_at
  FROM release_messages rm
  JOIN releases rel ON rm.release_id = rel.id
) AS combined
ORDER BY created_at DESC
-- kemudian GROUP BY user_id untuk thread per artist
```

---

## 3. Fitur Knowledge Base

Pusat dokumentasi/FAQ yang dapat diakses artist untuk mencari jawaban sebelum membuat tiket. Admin dapat membuat, mengedit, dan mengelola artikel serta kategori.

### 3.1 Halaman yang Dibutuhkan

#### Sisi User (Publik / Setelah Login)

**`help.php`** — Halaman utama Knowledge Base
- Search bar besar di atas (cari artikel berdasarkan keyword)
- Grid kartu kategori: ikon, nama kategori, jumlah artikel
- Artikel populer/terbaru (sortir by view_count atau published_at)
- Link ke halaman tiket jika tidak menemukan jawaban

**`help-article.php?slug=xxx`** — Detail artikel
- Judul, kategori breadcrumb, tanggal update terakhir
- Konten artikel dengan rich text (HTML yang di-sanitasi dari DB)
- Rating: "Apakah artikel ini membantu?" → Tombol Ya/Tidak
- Artikel terkait (dari kategori yang sama)
- Tombol "Hubungi Support" → link ke support.php
- Increment `view_count` setiap artikel dibuka

#### Sisi Admin

**`admin/kb-categories.php`** — Kelola kategori
- List kategori: nama, ikon (FontAwesome class), jumlah artikel, sort order, status aktif
- Form tambah/edit kategori (inline atau modal)
- Input urutan manual (atau drag-and-drop opsional)
- Toggle aktif/nonaktif kategori

**`admin/kb-articles.php`** — List semua artikel
- Tabel: judul, kategori, status (draft/published), penulis, view count, tanggal publish
- Filter by kategori dan status
- Search by judul
- Tombol Buat Artikel Baru → arahkan ke editor
- Toggle publish/unpublish langsung dari list

**`admin/kb-article-editor.php?id=xxx`** — Editor artikel
- Form: judul, slug (auto-generate dari judul, bisa diedit), kategori, status, meta description
- WYSIWYG editor (Quill.js atau TinyMCE via CDN)
- Preview artikel sebelum publish
- Simpan sebagai draft atau langsung publish
- Mode edit jika ada `?id=xxx` di URL

### 3.2 API — `api/kb.php`

| Method | Endpoint | Fungsi |
|---|---|---|
| GET | `?action=categories` | List semua kategori aktif |
| GET | `?action=articles&category_id=xxx` | List artikel per kategori |
| GET | `?slug=xxx` | Detail satu artikel (+ increment view_count) |
| GET | `?action=search&q=keyword` | Full-text search artikel |
| POST | _(admin)_ | Buat artikel baru |
| PUT | `?id=xxx` _(admin)_ | Update artikel |
| DELETE | `?id=xxx` _(admin)_ | Hapus artikel |
| POST | `?action=rate&id=xxx` _(user)_ | Kirim rating helpful/not |

### 3.3 Database

**Tabel: `kb_categories`**

| Field | Tipe | Keterangan |
|---|---|---|
| `id` | VARCHAR(36) PK | UUID v4 |
| `name` | VARCHAR(100) | Nama kategori (misal: "Cara Upload Rilisan") |
| `slug` | VARCHAR(120) UNIQUE | URL-friendly (misal: cara-upload-rilisan) |
| `icon` | VARCHAR(80) | FontAwesome class (misal: fa-solid fa-music) |
| `description` | TEXT NULL | Deskripsi singkat kategori |
| `sort_order` | INT DEFAULT 0 | Urutan tampil di halaman help |
| `is_active` | TINYINT(1) DEFAULT 1 | 1 = tampil, 0 = sembunyikan |
| `created_at` | DATETIME DEFAULT NOW() | Waktu dibuat |

**Tabel: `kb_articles`**

| Field | Tipe | Keterangan |
|---|---|---|
| `id` | VARCHAR(36) PK | UUID v4 |
| `category_id` | VARCHAR(36) FK → kb_categories | Kategori artikel |
| `author_id` | VARCHAR(36) FK → users | Admin yang membuat |
| `title` | VARCHAR(255) | Judul artikel |
| `slug` | VARCHAR(280) UNIQUE | URL slug (auto dari judul) |
| `content` | LONGTEXT | Konten HTML dari WYSIWYG editor |
| `meta_description` | VARCHAR(300) NULL | SEO meta description |
| `status` | ENUM('draft','published') | Status artikel, default: draft |
| `view_count` | INT DEFAULT 0 | Jumlah kali artikel dibuka |
| `helpful_yes` | INT DEFAULT 0 | Rating positif dari user |
| `helpful_no` | INT DEFAULT 0 | Rating negatif dari user |
| `published_at` | DATETIME NULL | Waktu pertama kali dipublish |
| `created_at` | DATETIME DEFAULT NOW() | Waktu dibuat |
| `updated_at` | DATETIME ON UPDATE NOW() | Waktu diupdate |

---

## 4. Fitur Reporting

Dashboard laporan untuk gambaran kinerja tim support dan aktivitas platform. Semua data diambil via query agregasi dari tabel yang sudah ada.

### 4.1 Halaman yang Dibutuhkan

#### Sisi Admin

**`admin/reports.php`** — Dashboard Laporan

**Seksi 1 — Overview (Kartu Ringkasan)**
- Total tiket bulan ini vs bulan lalu
- Rata-rata waktu respons pertama (first response time)
- Rata-rata waktu penyelesaian tiket
- Persentase tiket diselesaikan dalam 24 jam
- Total artikel KB dibaca bulan ini
- Total request repair bulan ini

**Seksi 2 — Grafik Tiket (Chart.js — sudah ada di project)**
- Line chart: jumlah tiket masuk per hari/minggu (rentang 30 hari)
- Bar chart: tiket per kategori (release, royalty, account, technical, other)
- Donut chart: distribusi status tiket (open, in_progress, closed)

**Seksi 3 — Performa Agen**
- Tabel: nama agen, jumlah tiket ditangani, rata-rata waktu respons, jumlah tiket closed
- Hanya tampil jika ada lebih dari 1 admin

**Seksi 4 — Repair & Knowledge Base**
- Jumlah request repair per status (pending, in_progress, done, rejected)
- Top 5 artikel KB paling sering dibaca
- Top 5 artikel KB dengan rating helpful tertinggi

**Filter Global:** Rentang tanggal (date range picker)

> **Catatan:** Reporting tidak membutuhkan tabel database baru. Semua data diambil via `COUNT`, `AVG`, `GROUP BY` dari tabel-tabel yang ada.

---

## 5. Fitur Sistem Repair

Artist dapat mengajukan permintaan perbaikan teknis (audio bermasalah, cover art salah, metadata keliru di platform, dll). Admin dapat melihat, menangani, dan memperbarui status request.

### 5.1 Halaman yang Dibutuhkan

#### Sisi User (Artist)

**`repair.php`** — Form & list request perbaikan
- List semua request repair milik user: nomor, jenis masalah, rilisan terkait, status, tanggal
- Tombol "Ajukan Request Perbaikan"
- Form: pilih rilisan (dropdown dari releases user), jenis masalah, deskripsi detail, upload lampiran/screenshot
- Status badge: Menunggu / Diproses / Selesai / Ditolak
- Catatan dari admin (jika rejected, ditampilkan alasan penolakan)

#### Sisi Admin

**`admin/repair.php`** — List semua request repair
- Tabel: nomor request, nama artist, rilisan terkait, jenis masalah, status, tanggal ajukan, tanggal update
- Filter: status, jenis masalah, rentang tanggal
- Search by nama artist atau nama rilisan
- Badge jumlah request pending di sidebar

**`admin/repair-detail.php?id=xxx`** — Detail request & update status
- Info lengkap: artist, rilisan, jenis masalah, deskripsi, lampiran bukti
- Timeline perubahan status
- Form update status: pilih status baru + catatan admin (catatan wajib jika rejected)
- Notifikasi otomatis ke artist saat status berubah (pakai `insert_notification()` yang sudah ada)
- Link ke rilisan terkait di halaman admin releases

### 5.2 API — `api/repair.php`

| Method | Endpoint | Fungsi |
|---|---|---|
| GET | _(user)_ | List semua request repair milik user |
| GET | `?id=xxx` | Detail satu request |
| POST | _(user)_ | Ajukan request: release_id, problem_type, description, attachment_path |
| PATCH | `?id=xxx` _(admin)_ | Update status + admin_note |
| DELETE | `?id=xxx` _(admin)_ | Hapus request |

### 5.3 Database

**Tabel: `repair_requests`**

| Field | Tipe | Keterangan |
|---|---|---|
| `id` | VARCHAR(36) PK | UUID v4 |
| `request_number` | VARCHAR(20) UNIQUE | Format: RPR-YYYYMMDD-XXXX (auto-generate) |
| `user_id` | VARCHAR(36) FK → users | Artist yang mengajukan |
| `release_id` | VARCHAR(36) FK → releases NULL | Rilisan yang bermasalah (nullable) |
| `problem_type` | ENUM('audio','cover_art','metadata','platform_link','isrc','other') | Jenis masalah |
| `description` | TEXT | Deskripsi detail masalah dari artist |
| `attachment_path` | VARCHAR(500) NULL | Path file bukti/screenshot |
| `status` | ENUM('pending','in_progress','done','rejected') | Status penanganan, default: pending |
| `admin_note` | TEXT NULL | Catatan admin (wajib jika rejected) |
| `handled_by` | VARCHAR(36) FK → users NULL | Admin yang menangani |
| `resolved_at` | DATETIME NULL | Waktu status berubah ke done |
| `created_at` | DATETIME DEFAULT NOW() | Waktu pengajuan |
| `updated_at` | DATETIME ON UPDATE NOW() | Waktu update terakhir |

---

## 6. Integrasi dengan Sistem yang Ada

### 6.1 Notifikasi

Semua fitur baru menggunakan `insert_notification()` yang sudah ada di `helpers.php`. Tidak perlu membuat sistem notifikasi baru.

| Event | Penerima | Type |
|---|---|---|
| Artist buat tiket baru | Semua admin | `info` |
| Admin reply tiket | Artist | `warning` |
| Status tiket berubah | Artist | `info` / `success` |
| Artist ajukan repair | Semua admin | `info` |
| Status repair → done | Artist | `success` |
| Status repair → rejected | Artist | `warning` |
| Status repair → in_progress | Artist | `info` |

### 6.2 Sidebar Navigation

**Tambahan di `admin/partials/sidebar.php`:**
```php
<a href="/admin/tickets.php" class="sidebar-link ...">
  <i class="fa-solid fa-ticket"></i> Tiket Support
  <span class="badge" id="unread-tickets">0</span>
</a>
<a href="/admin/inbox.php" class="sidebar-link ...">
  <i class="fa-solid fa-inbox"></i> Inbox
</a>
<a href="/admin/kb-articles.php" class="sidebar-link ...">
  <i class="fa-solid fa-book-open"></i> Knowledge Base
</a>
<a href="/admin/repair.php" class="sidebar-link ...">
  <i class="fa-solid fa-wrench"></i> Repair
  <span class="badge" id="pending-repairs">0</span>
</a>
<a href="/admin/reports.php" class="sidebar-link ...">
  <i class="fa-solid fa-chart-bar"></i> Laporan
</a>
```

**Tambahan di `partials/desktop-sidebar.php` dan `partials/bottom-nav.php`:**
- Support / Bantuan — `fa-solid fa-headset` → `support.php`
- Repair — `fa-solid fa-wrench` → `repair.php`
- Help / KB — `fa-regular fa-circle-question` → `help.php` _(menggantikan link yang saat ini disabled)_

### 6.3 Format Nomor Auto-Generate

```php
// Tiket Support
$ticket_number = 'TKT-' . date('Ymd') . '-' . strtoupper(substr(uuid4(), 0, 4));
// Contoh: TKT-20250612-A3F7

// Request Repair
$request_number = 'RPR-' . date('Ymd') . '-' . strtoupper(substr(uuid4(), 0, 4));
// Contoh: RPR-20250612-B1D9
```

### 6.4 SQL — Buat Semua Tabel Baru Sekaligus

```sql
-- 1. Tiket Support
CREATE TABLE support_tickets (
  id              VARCHAR(36)  NOT NULL PRIMARY KEY,
  ticket_number   VARCHAR(20)  NOT NULL UNIQUE,
  user_id         VARCHAR(36)  NOT NULL,
  category        ENUM('release','royalty','account','technical','other') NOT NULL,
  subject         VARCHAR(255) NOT NULL,
  status          ENUM('open','in_progress','waiting_user','closed') NOT NULL DEFAULT 'open',
  priority        ENUM('low','medium','high','urgent') NOT NULL DEFAULT 'medium',
  assigned_to     VARCHAR(36)  NULL,
  closed_at       DATETIME     NULL,
  closed_by       VARCHAR(36)  NULL,
  created_at      DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at      DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  FOREIGN KEY (user_id)    REFERENCES users(id) ON DELETE CASCADE,
  FOREIGN KEY (assigned_to) REFERENCES users(id) ON DELETE SET NULL,
  FOREIGN KEY (closed_by)   REFERENCES users(id) ON DELETE SET NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 2. Reply Tiket
CREATE TABLE support_ticket_replies (
  id              VARCHAR(36)  NOT NULL PRIMARY KEY,
  ticket_id       VARCHAR(36)  NOT NULL,
  sender_id       VARCHAR(36)  NOT NULL,
  sender_role     ENUM('user','admin') NOT NULL,
  message         TEXT         NOT NULL,
  attachment_path VARCHAR(500) NULL,
  is_read         TINYINT(1)   NOT NULL DEFAULT 0,
  created_at      DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (ticket_id) REFERENCES support_tickets(id) ON DELETE CASCADE,
  FOREIGN KEY (sender_id) REFERENCES users(id) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 3. Kategori KB
CREATE TABLE kb_categories (
  id          VARCHAR(36)  NOT NULL PRIMARY KEY,
  name        VARCHAR(100) NOT NULL,
  slug        VARCHAR(120) NOT NULL UNIQUE,
  icon        VARCHAR(80)  NOT NULL DEFAULT 'fa-solid fa-circle-question',
  description TEXT         NULL,
  sort_order  INT          NOT NULL DEFAULT 0,
  is_active   TINYINT(1)   NOT NULL DEFAULT 1,
  created_at  DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 4. Artikel KB
CREATE TABLE kb_articles (
  id               VARCHAR(36)  NOT NULL PRIMARY KEY,
  category_id      VARCHAR(36)  NOT NULL,
  author_id        VARCHAR(36)  NOT NULL,
  title            VARCHAR(255) NOT NULL,
  slug             VARCHAR(280) NOT NULL UNIQUE,
  content          LONGTEXT     NOT NULL,
  meta_description VARCHAR(300) NULL,
  status           ENUM('draft','published') NOT NULL DEFAULT 'draft',
  view_count       INT          NOT NULL DEFAULT 0,
  helpful_yes      INT          NOT NULL DEFAULT 0,
  helpful_no       INT          NOT NULL DEFAULT 0,
  published_at     DATETIME     NULL,
  created_at       DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at       DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  FOREIGN KEY (category_id) REFERENCES kb_categories(id) ON DELETE RESTRICT,
  FOREIGN KEY (author_id)   REFERENCES users(id) ON DELETE RESTRICT,
  FULLTEXT KEY ft_articles (title, content)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 5. Request Repair
CREATE TABLE repair_requests (
  id              VARCHAR(36)  NOT NULL PRIMARY KEY,
  request_number  VARCHAR(20)  NOT NULL UNIQUE,
  user_id         VARCHAR(36)  NOT NULL,
  release_id      VARCHAR(36)  NULL,
  problem_type    ENUM('audio','cover_art','metadata','platform_link','isrc','other') NOT NULL,
  description     TEXT         NOT NULL,
  attachment_path VARCHAR(500) NULL,
  status          ENUM('pending','in_progress','done','rejected') NOT NULL DEFAULT 'pending',
  admin_note      TEXT         NULL,
  handled_by      VARCHAR(36)  NULL,
  resolved_at     DATETIME     NULL,
  created_at      DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at      DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  FOREIGN KEY (user_id)    REFERENCES users(id) ON DELETE CASCADE,
  FOREIGN KEY (release_id) REFERENCES releases(id) ON DELETE SET NULL,
  FOREIGN KEY (handled_by) REFERENCES users(id) ON DELETE SET NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

### 6.5 Urutan Implementasi yang Disarankan

1. **Database** — Buat semua 5 tabel sekaligus via SQL di atas
2. **Ticketing** — API + halaman user + halaman admin _(prioritas tertinggi: menggantikan komunikasi manual via WhatsApp)_
3. **Sistem Repair** — API + halaman user + halaman admin
4. **Knowledge Base** — Admin editor dulu, lalu halaman publik
5. **Shared Inbox** — Query agregasi saja, tidak ada tabel baru
6. **Reporting** — Terakhir, bergantung data dari semua fitur sebelumnya

---

**Total: 17 halaman + 3 file API + 5 tabel database baru**

_Dokumen ini dibuat untuk project Atsalara Distribution Platform._
