# Patch PRD — SATRIA-RSPAD

> **Status:** Baseline final (P0 struktur organisasi + Prompt 2 mapping HIS/SIMRS)
> **Sifat dokumen:** Patch siap tempel ke PRD utama. Penomoran section bersifat indikatif — sesuaikan dengan PRD utama.
> **Tidak mengandung:** kode Laravel, migration, atau seeder final (sesuai batasan fase).

---

## §0. Identitas Aplikasi (Branding Resmi)

| Aspek | Nilai |
|---|---|
| Nama aplikasi | **SATRIA-RSPAD** |
| Kepanjangan | Sistem Administrasi Terpadu Riwayat Insan Aparatur RSPAD |
| Tagline | *Satu Data Personel, Siap Dukung Komando.* |

**Aturan penamaan:** Branding resmi pada PRD, dokumentasi, label utama UI, dan narasi produk adalah **SATRIA-RSPAD**. Istilah generik seperti *Personalia RSPAD* hanya boleh dipakai sebagai deskripsi modul/fungsi, bukan nama produk.

---

## §7.1 (REVISI) Struktur Organisasi Resmi — Master `units`

### 7.1.1 Source of Truth

Struktur organisasi resmi SATRIA-RSPAD mengikuti **Peraturan Kepala Staf Angkatan Darat (Perkasad) No. 44 Tahun 2025, Lampiran I** tentang Organisasi dan Tugas RSPAD Gatot Soebroto. Tabel `units` adalah **satu-satunya master** struktur organisasi. Tidak ada sumber lain (termasuk HIS/SIMRS) yang boleh mengubah `units`.

### 7.1.2 Model Penamaan 4-Lapis

| Lapis | Kolom | Definisi | Contoh |
|---|---|---|---|
| Telusur sumber | `source_label` | Teks **mentah/verbatim** dari sumber/lampiran (apa adanya, termasuk typo) | `DIRBIN YANKES` |
| Designation resmi | `official_name` | Nama resmi/display aplikasi **setelah koreksi**, gaya bagan (ringkas) | `DIRBIN YANKES` |
| Display organ | `nama_unit` | Nama organ yang **diperluas** untuk UI | `Direktorat Pembinaan Pelayanan Kesehatan` |
| Hint jabatan | `head_position_name` | Gelar pemangku jabatan; menjadi **hint** untuk tabel `positions` (sumber kanonik jabatan tetap `positions`) | `Direktur Pembinaan Pelayanan Kesehatan` |

Untuk unit direktorat: `nama_unit` = **organ** ("Direktorat…"), sedangkan `head_position_name` = **kursi/jabatan** ("Direktur…").

### 7.1.3 Skema Konseptual `units` (Final)

| Field | Tipe | Null | Default | Catatan |
|---|---|---:|---|---|
| `id` | BIGINT | No | auto | PK |
| `parent_id` | BIGINT | Yes | NULL | self-ref; **command hierarchy** |
| `kode_unit` | STRING(50) | No | — | UNIQUE; kode kanonik (`KADEP_ANESTESI`) |
| `source_label` | STRING(200) | Yes | NULL | verbatim sumber/lampiran |
| `official_name` | STRING(200) | No | — | designation resmi terkoreksi |
| `nama_unit` | STRING(150) | No | — | display organ (perluasan) |
| `unsur` | STRING(30) | No | — | `pimpinan` / `pembantu_pimpinan` / `pelayanan` / `pelaksana` / `jabfung` |
| `kelompok` | STRING(20) | Yes | NULL | `departemen` / `instalasi` / `unit` (hanya `unsur=pelaksana`) |
| `unit_type` | STRING(20) | No | `structural_unit` | `structural_unit` / `functional_group` |
| `parent_status` | STRING(15) | No | `unknown` | `confirmed` / `provisional` / `unknown` |
| `needs_verification` | BOOLEAN | No | false | status **penamaan/ekspansi/klasifikasi** |
| `head_position_code` | STRING(50) | Yes | NULL | override hint untuk seeder `positions` |
| `head_position_name` | STRING(200) | Yes | NULL | override hint untuk seeder `positions` |
| `notes` | TEXT | Yes | NULL | catatan koreksi/mapping |
| `sort_order` | INTEGER | Yes | 0 | basis urutan bagan (urut baca: atas→bawah, kiri→kanan) |
| `level_unit` | INTEGER | No | — | **informasional saja** (kedalaman command tree) |
| `is_active` | BOOLEAN | No | true | — |
| `created_at` / `updated_at` / `deleted_at` | DATETIME | Yes | — | soft delete |

### 7.1.4 Keputusan Desain Kunci

1. **`parent_status` dipisah dari `needs_verification`.**
   - `parent_status` → kebenaran **garis komando** (edge `parent_id`): `confirmed` / `provisional` / `unknown`.
   - `needs_verification` → kebenaran **penamaan/ekspansi singkatan/klasifikasi** (BOOLEAN).
   - Satu unit bisa `parent_status=confirmed` namun `needs_verification=true`, atau sebaliknya.
2. **`unit_type` dipisah dari `unsur`.**
   - `unsur` → untuk **tampilan bagan** (banding visual, 5 nilai).
   - `unit_type` → untuk **perilaku sistem, validasi, dan seeder**.
3. **`JABFUNG` = `functional_group`.**
   - Tidak meng-auto-generate kepala struktural untuk JABFUNG.
   - Jabatan fungsional di bawah JABFUNG dibahas/di-seed terpisah setelah rincian stakeholder tersedia.
4. **Tampilan bagan organisasi** menggunakan `unsur` + `kelompok` + `sort_order`, **bukan** `level_unit`.
5. **`parent_id` + `parent_status`** untuk command hierarchy. Bila garis komando belum pasti → `parent_status = provisional` atau `unknown`. UI Filament wajib menampilkan badge bila `parent_status ∈ {provisional, unknown}`.
6. `head_position_code` / `head_position_name` hanya **override hint** untuk hybrid PositionSeeder; **sumber kanonik jabatan tetap tabel `positions`.**

### 7.1.5 Aturan Tampilan Bagan

| Kebutuhan | Sumber data |
|---|---|
| Layout & urutan render bagan | group `unsur` → `kelompok`, `ORDER BY sort_order` |
| Header/banding visual | `unsur`, `kelompok` |
| Garis komando (siapa lapor ke siapa) | `parent_id` + `parent_status` |
| Kedalaman (informasional) | `level_unit` |

---

## §7.1.6 Koreksi Final Penamaan (Contoh Kanonik: KADEP ANESTESI)

Pada lampiran sumber terbaca `KADEP ANETSESI` — ini **typo sumber**. Penanganan final:

| Kolom | Nilai |
|---|---|
| `kode_unit` | `KADEP_ANESTESI` |
| `official_name` | `KADEP ANESTESI` |
| `source_label` | `KADEP ANETSESI` |
| `nama_unit` | `Departemen Anestesi` |
| `unit_type` | `structural_unit` |
| `kelompok` | `departemen` |
| `needs_verification` | `false` *(koreksi typo sudah dipastikan)* |
| `head_position_name` | `Kepala Departemen Anestesi` |
| `notes` | `Teks sumber terbaca ANETSESI, dikoreksi menjadi ANESTESI karena typo.` |

**Larangan:** Jangan membuat unit `KADEP_ANETSESI`. Jangan menjadikan `ANETSESI` sebagai nama resmi aplikasi. Teks `ANETSESI` hanya hidup di `source_label`.

---

## §7.3 (ADDENDUM) Positions — Hybrid Derivation

Tambahan minimal pada `positions`: kolom `is_head` (BOOLEAN, default `false`) untuk menandai jabatan kepala echelon yang diturunkan dari unit. Enum `tipe_jabatan` tidak berubah.

Presedensi seeder (hybrid, bukan 100% auto-derive):

| Prioritas | Kondisi unit | Aksi |
|---|---|---|
| 1 | `unit_type = functional_group` (mis. JABFUNG) | **skip** kepala struktural; jabatan fungsional di-seed terpisah |
| 2 | `head_position_code/name` terisi | buat jabatan **verbatim** dari override (`is_head=true`, `struktural`) |
| 3 | pelaksana berpola (`KADEP`/`KAINSTAL`/`KANIT`) | **auto-derive** dari prefix (`is_head=true`, `struktural`) |

---

## §13. Integrasi & Mapping HIS/SIMRS (Operasional)

### 13.1 Prinsip

- `units` adalah **satu-satunya source of truth** struktur organisasi (Perkasad No. 44/2025 Lampiran I). HIS/SIMRS **tidak pernah** mengubah `units`.
- Tabel departemen HIS adalah **referensi operasional** (titik layanan/pendaftaran/penunjang), **bukan** struktur komando.
- Relasi HIS↔resmi bersifat **many-to-one dari sisi unit**: banyak `DepartemenID` boleh menunjuk satu `unit`; satu `DepartemenID` menunjuk **maksimal satu** unit owner untuk baseline awal.
- `DepartemenID` (kode operasional SIMRS) dan `kode_unit` (kode kanonik resmi) **tidak boleh dicampur**.

### 13.2 Model Data Referensi (Terpisah dari `units`)

| Tabel | Peran | Kunci | Prioritas |
|---|---|---|---|
| `his_departments` | Salinan **mentah** baris SIMRS apa adanya (+`raw_payload` JSON) | `departemen_id` UNIQUE | Wajib (fondasi) |
| `unit_his_department_mappings` | Jembatan HIS→`units`; **wajib** FK `his_department_id` | `his_department_id` UNIQUE | Inti |
| `unit_aliases` | Alias bebas (legacy org / manual / HFIS kurasi) **tanpa** baris HIS | `(alias_code, alias_source)` UNIQUE | **Deferred** |

### 13.3 Field Konseptual

**`his_departments`** (read-only, sumber kebenaran = re-import CSV):
`id`, `departemen_id` (UNIQUE), `nama`, `kode_departemen`, `blok_departemen`, `parent_poli_id` (**STRING mentah**), `is_ri`, `is_rj`, `is_pd`, `is_mcu`, `is_pendaftaran`, `kategori`, `is_inactive` (dari `NA='Y'`), `status_online`, `kode_poli_hfis`, `kode_sub_poli_hfis`, `raw_payload` (JSON), timestamps.

**`unit_his_department_mappings`:**
`id`, `unit_id` (FK), `his_department_id` (FK, UNIQUE), `mapping_type`, `confidence`, `is_primary`, `needs_verification`, `notes`, timestamps.

**`unit_aliases`** *(deferred):*
`id`, `unit_id` (FK), `alias_code`, `alias_name`, `alias_source` (`simrs_admin` / `legacy_org` / `bpjs_hfis` / `manual`), `confidence`, `needs_verification`, `notes`, timestamps.

### 13.4 Atribut Mapping

- `mapping_type`: `direct` | `child` | `service` | `legacy` | `administrative`
- `confidence`: `high` | `medium` | `low`
- `is_primary` (BOOLEAN) — satu owner utama per unit.
- `needs_verification` (BOOLEAN) — **terpisah** dari `mapping_type` dan `is_primary`.
- `notes` — wajib untuk kasus ambigu (sebutkan alternatif unit).

**Catatan HFIS:** Kode HFIS tetap disimpan sebagai kolom di `his_departments`. Bila suatu saat unit memerlukan alias HFIS langsung, masukkan ke `unit_aliases` dengan `alias_source = bpjs_hfis` — **bukan** sebagai `mapping_type`.

### 13.5 Aturan Kasus Ambigu

Untuk baseline awal, satu `DepartemenID` HIS = maksimal satu unit owner. Kasus ambigu (mis. `TINDAK_SALIN`, `ODC`, `RADTERAPI`, `DSA`, `NYERI`) ditetapkan **satu kandidat terbaik** dengan `needs_verification = true`, `confidence = medium/low`, dan alternatif dijelaskan di `notes`. **Tidak** membuat multi-mapping.

### 13.6 Governance & Batasan Teknis

- `parent_poli_id` disimpan **mentah (string)**; normalisasi cluster/self-FK **ditunda** (ParentPoliID = ID cluster internal HIS, bukan `DepartemenID`).
- Aturan satu `is_primary = true` per unit **ditegakkan lewat logika seeder/import + validasi Filament**, **bukan** partial unique index DB (kompatibilitas MySQL/MariaDB). DB-level constraint dipertimbangkan kemudian (generated column / pendekatan kompatibel).
- Unit resmi tanpa padanan HIS diberi status logis `not_found_in_his_departemen` dan **tidak dihapus** dari `units`.
- Departemen HIS tanpa unit owner tetap tersimpan di `his_departments` tanpa baris mapping.
- Import bersifat **idempotent** dan **tidak pernah** menulis ke `units`.

### 13.7 Mapping High Confidence (Diterima)

`(P)` = `is_primary`. Tipe: `d`=direct, `c`=child, `s`=service, `a`=administrative.

| Unit resmi | DepartemenID HIS | tipe |
|---|---|---|
| KADEP_PDALAM | **INT (P)** · GASTRO · KGH · HEMATO_ONCO · IMUNO · GERIATRI · TINFEKSI · REUMA · KEMD · HEPATOLOGI · VCT · PKEMO | d/c |
| KADEP_BEDAH | **BED (P)** · BDA · BDIG · BORT · BDP · BSY · BTRX · BTMR · BURO · BVAS | d/c |
| KADEP_GILUT | **GIGI (P)** · GIGI_PNJ · BDM · GND · GOR · GPS · PNM · GPR · PTD · KSV_GIGI | d/c |
| KADEP_OBSGIN | **OBG (P)** · OBG_KB · OBG_FETO · ONKO_GINEKO · ONKO | d/c |
| KADEP_JANTUNG | **JAN (P)** | d |
| KADEP_IKA | **ANA (P)** | d |
| KADEP_KESWA | **JIWA (P)** | d |
| KADEP_MATA / THT / PARU / SARAF / PKULKEL / ANESTESI | **MATA / THT / PARU / SARAF / KLT / ANESTESI** (P) | d |
| KAINSTAL_REHABMED | **IRM (P)** · FISIO · OKUPASI · PROTESA · TWICARA | d/c |
| KAINSTAL_KEDNUKLIR | **NUKLIR (P)** · KED_NUKLIR · NUKLIR_PNJ | d/c |
| KAINSTAL_PATKLIN / PATANAT / RADIOLOGI / FARMASI | **LABPK / LABPA / RAD / FAR** (P) | d |
| KAINSTAL_GIZI | **INSTALGIZI (P)** | d |
| KAINSTAL_KMROPS | **IKO (P)** | d |
| KAINSTAL_GADAR | **IGD (P)** · RIGD | d/s |
| KAINSTAL_RIKKES_MCU | **MCU (P)** | d |
| KAINSTAL_FORENSIK | **FORENSIK (P)** | d |
| KANIT_HD | **HDL (P)** | d |
| KEPALA / WAKIL_KEPALA | KARUMKIT / WAKARUMKIT | a |

### 13.8 Daftar `needs_verification` (Mapping Tentatif)

| DepartemenID | Kandidat unit | conf | Catatan / alternatif |
|---|---|---|---|
| GIZI | KADEP_PDALAM (child) | M | Gizi **klinik**, BUKAN `KAINSTAL_GIZI` (≠ INSTALGIZI) |
| RKEMO | KADEP_PDALAM (service) | M | kemoterapi onkologi PD |
| KESLING | KAINSTAL_KESLING | M | nama HIS: "Kedokteran Okupasi (Kesling)" |
| ICU · CICU_KARTIKA | KAINSTAL_WATSIF | M | CICU mungkin hibrida paviliun-intensif |
| CVC | KAINSTAL_CVC | M | konfirmasi cakupan |
| INSTALPAV + PAV_* · AMINO · DOKPRES | KAINSTAL_PAVILUN | M | klasifikasi VIP/VVIP per paviliun belum jelas |
| VIPACVC · VIPBCVC | KANIT_VVIP | L | alt: KAINSTAL_CVC / PAVILUN |
| DIRYANKES · DIRUM · KAKOMED · KA_SPI · DIRBANGDANRISET | DIRBIN_YANKES · DIRBIN_UM · KE_KOMMED · KA_SPI · DIRBIN_BANG_RISET | M | mapping administratif |
| TINDAK_SALIN | KANIT_PONEK | M | alt: KADEP_OBSGIN |
| TIND | KAINSTAL_CVC | L | alt: KADEP_JANTUNG (tindakan) |
| RADTERAPI | KAINSTAL_RADIOLOGI | L | alt: KAINSTAL_TERAPIPROTON |
| DSA · DSA_poli | KAINSTAL_RADIOLOGI | L | alt: KAINSTAL_CVC (intervensi) |
| AKU | KAINSTAL_REHABMED | L | alt: medik komplementer |
| ODC | KAINSTAL_GADAR | L | alt: KAINSTAL_WATNAP |
| NYERI | KADEP_ANESTESI | L | alt: layanan multidisiplin |
| SCREENING | KAINSTAL_RIKKES_MCU | L | alt: administratif |
| vaksin | KAINSTAL_PAVILUN | L | klinik vaksin paviliun; alt: imunisasi |
| IAPP | KAINSTAL_APP | M | konfirmasi "IAPP" = Instalasi APP |

**Tanpa unit owner** (tetap di `his_departments`, tanpa baris mapping; perlu keputusan stakeholder): `CELLCURE`, `IMMUNNUS`, `KARTIKA_ESTETIKA`, `MUTU`, `KCDI`, `MINPAS`, `RUMKITLAP`. Menambah unit baru untuk ini = **amandemen P0** (jalur governance terpisah, bukan via HIS).

### 13.9 Deferred Pasca-Demo

| Item | Alasan tunda |
|---|---|
| Seluruh modul HIS mapping | Bukan target demo (lihat §13.10) |
| Verifikasi daftar §13.8 + mapping `administrative` | Harus dikonfirmasi tim RSPAD sebelum dikunci |
| `unit_aliases` (legacy org / HFIS kurasi) | Non-kritis; perlu saat lookup alias dibutuhkan |
| Rekonsiliasi BPJS/HFIS | Data sudah aman di `his_departments`; pemanfaatan menyusul |
| Normalisasi `parent_poli_id` → cluster table / self-FK | Tunggu pola data dipahami |
| Derivasi WATLAN/WATNAP via flag `RJ=1` / `RI=1` | Konseptual; bukan mapping langsung |
| Keputusan unit baru (CELLCURE/IMMUNNUS/dll.) | Butuh amandemen P0 |

### 13.10 Penegasan Scope Demo

Modul Integrasi & Mapping HIS/SIMRS (§13) **bukan target demo Selasa**. Demo tetap fokus pada **Scope 1 core**:

- Employees
- Units resmi P0
- Ranks
- Positions
- Assignments
- Mutation Orders
- Physical Attributes
- Medical Checkups
- RBAC
- Audit dasar

---

## Catatan Fase Implementasi

Dokumen ini adalah **patch desain/PRD**. Belum ada kode Laravel, migration, maupun seeder final. Langkah berikutnya (mis. `HisDepartmentImport`, migration `his_departments` / `unit_his_department_mappings`) hanya dimulai setelah ada aba-aba eksplisit.
