### 🔓Broken Access Control (BAC)

"WAF mungkin bisa nahan SQLi & XSS, tapi WAF gak bakal paham logika bisnis aplikasi. nah di situlah Broken Access Control ambil peran."

Dokumentasi pribadi hasil *hunting*, nemu *logic flaw* di berbagai aplikasi, kayak aplikasi dinas/pemerintah, sistem internal kampus sampai startup SaaS besar. btw BAC saat ini masih kokoh di peringkat #1 OWASP Top 10 karena emang selalu lolos dari *automated scanner*.

## 🧠Definisi Singkat
Broken Access Control (BAC) tuh terjadi ketika aplikasi gagal membatasi hak akses *user* sesuai dengan *role* atau otorisasi yang seharusnya. Lu bisa ngelihat, ngubah, atau ngehapus data milik orang lain, atau bahkan naik kasta jadi Admin (*Privilege Escalation*).

---

## 🔍 Skenario Kerentanan & Payload Eksploitasi

### 1. IDOR Klasik via URL Parameter (B2A / Horizontal PrivEsc)
Skenario paling dasar tapi masih sering kena. Aplikasi mengandalkan input dari *client-side* (URL/Parameter) buat nentuin data siapa yang mau ditarik, tapi ga nge-cek apakah *user* yang *login* punya hak atas ID tersebut.

```
* **URL Normal Lu:** `https://target.com/dashboard/upload/ktpuser?id=20045`
* **Manipulasi (IDOR):** Ubah `id` ke barisan angka lain.
  `https://target.com/profile/upload/ktpuser?id=20044`
  `https://target.com/profile/upload/ktpuser?id=20046`
```
**Varian IDOR di API Endpoint (2026):**
Banyak aplikasi modern pake REST API atau GraphQL. Cek *request* di Burp Suite:
```http
GET /api/v2/users/me HTTP/1.1  --> Balikin data lu sendiri
GET /api/v2/users/1029 HTTP/1.1 --> Ganti 'me' jadi ID target buat nyomot data orang lain
```

### 🚀 2. IDOR Privilege Escalation via Fitur Ganti Password
Ini high-risk banget. Fitur ganti password atau reset akun yang cacat logika, di mana parameter identitas user dilempar di dalam request body (JSON/Form) dan bisa diganti secara paksa (Parameter Tampering).
```
Request Asli (Akun Attacker, ID: 551):

HTTP
POST /api/account/update-password HTTP/1.1
Host: target.com
Content-Type: application/json
Authorization: Bearer <token_attacker_lu>

{
  "user_id": 551,
  "new_password": "Hacked2026",
  "password": "Hacked2026!"
}
Eksploitasi (Mengubah Password Akun Korban/Admin, ID: 1):
```
Cukup ganti "user_id" jadi milik target. Kalo backend cuma nge-validasi token JWT lu aktif (tanpa mencocokkan isi token dengan "user_id" di body), password target bakal langsung berubah!
```
JSON
{
  "user_id": 1,
  "current_password": "password_attacker", 
  "new_password": "Hacked2026!"
}
```
Note: Kadang parameter current_password malah gak divalidasi sama backend kalo parameter user_id diubah, atau bisa langsung dihapus barisnya dari JSON. btw ga cuma bagian password, kadang pas update biodata atau semua request post kalau ada param id layak dicoba tuh Idor kek gini, siapa tau hoki ya.

### 🏰 3. Path Authorization Bypass (/admin/dashboard)
Kasus di mana lu nyoba forced browsing ke halaman admin tanpa login, atau login make akun role rendah (User biasa). Di lapangan, gue sering nemu dua situasi unik ini apalagi app hasil vibe coding yang make ai murahan:
```
Situasi A: Cuma Bisa Lihat Doang (Read-Only Access)
Lu langsung coba akses https://target.com/admin/dashboard biasanya langsung kebuka. dari situ bisa liat grafik penjualan, total user, bahkan log sistem, tapi pas lu klik tombol "Delete User" atau "Add Admin", aksinya gagal (Error 403 / 401).
```
Kenapa Bisa? Developer masang middleware / filter otorisasi cuma di fungsi-fungsi pemroses data (POST/PUT/DELETE), tapi lupa masang middleware di fungsi routing utama yang nampilin halaman (GET).

Dampak: Information Disclosure (Kebocoran data sensitif lewat dasbor).
```
Situasi B: Akses Full (Full CRUD Access / Takeover)
Lu tembak /admin/dashboard atau endpoint API adminnya, halaman kebuka dan semua fitur modifikasi data berfungsi normal tanpa validasi token sama sekali.
```
Kenapa Bisa? Sering terjadi karena developer pake routing grup, tapi folder /admin/ lupa dimasukin ke dalam security middleware array di source code-nya (misal di Laravel atau CodeIgniter lupa pasang ->middleware(['auth', 'admin'])).

Trick Bypass Tambahan via Header / Path Manipulation:
Kalo halaman /admin/dashboard nge-block lu (403), coba bypass pake teknik path traversal atau kustom header:
```
HTTP
GET /admin/dashboard HTTP/1.1  --> (403 Forbidden)

-- Coba Teknik Ini:
GET /./admin/dashboard HTTP/1.1
GET /admin/dashboard/..;/ HTTP/1.1
X-Original-URL: /admin/dashboard
X-Rewrite-URL: /admin/dashboard
```
### 🔐4. API tidak memberlakukan validasi server-side terhadap immutable fields
Jujur gw sebenernya ga yakin ini namanya kerentanan apa, tapi gw masukin ke list BAC karna kasusnya lumayan simpel tapi berhubungan sama cacat logika aplikasi
```
Contohnya:
ada menu biodata, isinya pasti ya form nama, ttl, atau mungkin biodata, nah terkadang di instansi seperti pemerintah atau kampus, kolom/form seperti nama/nim atau TTL tuh gabisa kita edit, tapiii pas kita intercept request nya make burpsuite, kolom/form yang sifatnya immutable/terkunci/gabisa diedit tadi tuh bisa diedit dan pas kita send ternyata kolom tadi isinya bisa berubah, bug kek gini lumayan, untuk reward sekitar 700k dengan severity *medium*.
```
catatan: ini terjadi karna validasi hanya diterapkan disisi front end, bukan disisi backend, jadi ketika diwebsite form pasti gabisa diedit karna ada validasi dari front end, tapi ketika tembak langsung ke backend, eh diterima aja tanpa validasi, btw biasanya kalau kek gini bisa diisi xss stored lohh, 

--- 
### 🛡️ Checklist Nyari Celah BAC & IDOR (Buat Pemula)
[ ] Bikin 2 Akun Berbeda Level: Akun A (Admin/High Priv) dan Akun B (User biasa/Low Priv).

[ ] Ganti HTTP Method: Kalo GET /api/user/1 gak boleh, coba PUT /api/user/1 atau DELETE /api/user/1.

[ ] Fuzzing Numerik ID: Ubah ID dari urutan numerik (id=101 ke id=102) atau acak UUID kalo tebakannya lemah.

[ ] Uji Parameter di Body: Cek setiap parameter user_id, owner_id, account_number, atau email yang dikirim lewat JSON POST request.

[ ] Akses Tanpa Header Auth: Drop header Authorization: Bearer atau hapus Cookie di Burp Suite, lalu jalankan request ke panel sensitif untuk cek forced browsing.

---

### 🔐 Cara Aman (Buat Developer)
Jangan pernah percaya data kontrol akses yang dikirim dari client (URL, Cookies, Body).

Object-Level Access Control:
Setiap ada request masuk ke database, validasi apakah user yang sedang aktif session-nya (auth()->user()->id) bener-bener pemilik dari data yang diminta.

PHP
// Contoh di Backend (Laravel)
$invoice = Invoice::find($request->id);
if ($invoice->user_id !== auth()->user()->id) {
    return response()->json(['error' => 'Unauthorized'], 403);
}
Global Middleware Otorisasi:
Terapkan deny-by-default. Semua route harus terkunci di awal, baru dibuka secara spesifik untuk role tertentu.

Pake Non-Predictable ID (UUID):
Ganti auto-increment ID (1, 2, 3) pake UUID (de305d54-75b4-431b-adb2-eb6b9e546013). Ini menutup celah mass-fuzzing IDOR, walaupun logic-nya tetep harus diperbaiki.
---

###✍️ Catatan Akhir
"Menemukan IDOR/BAC itu masalah ketelitian membaca alur aplikasi (Business Logic). Jangan cuma fokus sama apa yang tampil di layar, tapi lihat apa yang dikirim di belakang layar (Burp History)."
