# IT Helpdesk Assistant — Starter Repository

Starter repository untuk Assignment 4: End-to-End RAG & AI Agent Development.

## Struktur Project

```
app/
├── main.py       # Entry point FastAPI, berisi endpoint /ask
├── router.py     # Intent detection + routing ke RAG atau tool call
├── rag.py        # Modul RAG: embedding, FAISS, semantic search, LLM answer
└── tools.py      # Modul tool call: pengecekan status tiket

data/
├── it_helpdesk_knowledge_base.csv   # Knowledge base untuk RAG
└── mock_tickets.csv                 # Data mock untuk tool call
```

## TODO Peserta

Seluruh bagian yang perlu diimplementasikan ditandai dengan komentar `# TODO`
atau `# 🔧 TODO` di dalam masing-masing file. Ikuti urutan berikut:

1. `app/rag.py` — implementasikan `build_index()`, `semantic_search()`, `answer_from_knowledge_base()`
2. `app/tools.py` — implementasikan `get_ticket_status()`
3. `app/router.py` — implementasikan `detect_intent()` dan `extract_ticket_id()`
4. `Dockerfile` — lengkapi konfigurasi container

## Menjalankan Secara Lokal (tanpa Docker)

```bash
pip install -r requirements.txt
Copy-Item .env.example .env   # PowerShell; lalu isi GEMINI_API_KEY di dalam .env
uvicorn app.main:app --reload
```

## Menjalankan dengan Docker

```bash
docker build -t it-helpdesk-assistant .
docker run -p 8000:8000 --env-file .env it-helpdesk-assistant
```

## Menguji Endpoint

```bash
curl -X POST http://localhost:8000/ask \
  -H "Content-Type: application/json" \
  -d '{"query": "berapa jatah cuti tahunan saya?"}'
```

## Problem Statement dan Objective

Tim People & IT Operations menerima pertanyaan berulang tentang kebijakan dan
permintaan pengecekan tiket. Aplikasi ini membantu menjawab pertanyaan berdasarkan
knowledge base serta mengambil status tiket dari data mock dalam satu endpoint.

Dua jenis permintaan yang ditangani:

1. Informasi/kebijakan (`faq`), misalnya "Berapa jatah cuti tahunan karyawan tetap?"
   atau "Bagaimana cara reset password VPN perusahaan?".
2. Pengecekan tiket (`ticket_status`), misalnya "Apa status tiket TICKET-0001 saya?".

## Desain Workflow

```text
POST /ask -> intent detection
              |-> faq -> semantic search FAISS -> context -> Gemini -> response
              |-> ticket_status -> ekstrak ID -> cari CSV -> response
              |-> unknown -> pesan permintaan tidak dipahami
```

Router memakai baseline keyword. Query dengan ID `TICKET-XXXX`, atau kata
`tiket`/`ticket` bersama `status`, `cek`, `lacak`, `progres`, `progress`, atau
`perkembangan`, masuk ke tool. Query kosong masuk ke `unknown`. Query lainnya
masuk ke RAG, termasuk pertanyaan umum tentang cara membuat tiket. Pertanyaan
di luar cakupan diarahkan ke RAG dengan instruksi agar LLM menyatakan bahwa
informasinya tidak tersedia jika context tidak menjawab pertanyaan.

Setiap baris knowledge base menjadi satu dokumen berisi kategori, pertanyaan,
dan jawaban. Seluruh 31 dokumen dibuat embedding-nya saat permintaan FAQ pertama.
Index FAISS disimpan dalam variabel dan dipakai kembali pada permintaan berikutnya.
Vektor dinormalisasi, lalu `IndexFlatIP` mengambil 3 dokumen dengan cosine similarity
tertinggi. Prompt mengikuti format Persona, Task, Context, dan Format.

## Konfigurasi Environment

Jalankan perintah dari folder utama project dengan Python 3.11. Pada ile `.env`,
isi `GEMINI_API_KEY` dengan key milik sendiri.

Variabel konfigurasi:

- `GEMINI_API_KEY`: API key Gemini, wajib untuk RAG.
- `GEMINI_MODEL`: model jawaban, default `gemini-3.8-flash`.
- `GEMINI_EMBEDDING_MODEL`: model embedding, default `gemini-embedding-001`.


Setelah aplikasi berjalan, buka `http://localhost:8000/docs` untuk mencoba
`POST /ask`. `GET /health` mengembalikan `{"status": "ok"}`.
Docker Desktop harus berjalan dengan Linux containers sebelum menjalankan
perintah build dan run di atas. API memakai `0.0.0.0:8000` di container agar
dapat diakses melalui port yang dipetakan.

Contoh pengujian PowerShell:

```powershell
$body = @{ query = "Apa status tiket TICKET-0001 saya?" } | ConvertTo-Json
Invoke-RestMethod -Uri http://localhost:8000/ask -Method Post -ContentType "application/json" -Body $body
```

Contoh response dari data tiket:

```json
{
  "status": "success",
  "intent": "ticket_status",
  "data": "Tiket TICKET-0001 - Laptop tidak bisa menyala. Status: Selesai. Update terakhir: 2026-08-20."
}
```

## Basic Testing dan Observasi

Pengujian lokal menggunakan FastAPI TestClient, CSV asli, dan FAISS asli.
Embedding dan jawaban Gemini disimulasikan karena API key belum tersedia.
Hasil ini memverifikasi alur aplikasi; kualitas jawaban dan retrieval Gemini
yang sebenarnya masih perlu diuji dengan API key aktif.

1. "Berapa jatah cuti tahunan karyawan tetap?"
   - Hasil: `success`, intent `faq`, jalur RAG dipanggil.
   - Observasi: context memuat jawaban cuti; jawaban simulasi berisi 12 hari.
2. "Bagaimana cara reset password VPN perusahaan?"
   - Hasil: `success`, intent `faq`, jalur RAG dipanggil.
   - Observasi: context memuat reset VPN; jawaban simulasi menyebut portal self-service IT.
3. "Apa status tiket TICKET-0001 saya?"
   - Hasil: `success`, intent `ticket_status`, status `Selesai` dari CSV.
   - Observasi: router memanggil tool tanpa memanggil Gemini.
4. "Cek status tiket TICKET-9999"
   - Hasil: `success`, intent `ticket_status`, pesan tiket tidak ditemukan.
   - Observasi: aplikasi tetap memberikan response tanpa crash. `success` berarti
     proses pencarian berhasil dilakukan, bukan berarti tiket ditemukan.
5. "Siapa pemenang pertandingan sepak bola kemarin?"
   - Hasil: `success`, intent `faq`, jalur RAG dipanggil.
   - Observasi: jawaban simulasi menyatakan informasi tidak tersedia. Penolakan
     oleh Gemini sebenarnya belum diverifikasi.

Pemeriksaan tambahan mencakup index berisi seluruh 31 dokumen, cache index,
batas `top_k`, query kosong, ID tiket tidak dicantumkan, ID huruf kecil,
FAQ tentang membuat tiket, validasi request, LLM gagal, dan jawaban LLM kosong.
Error pemrosesan mengembalikan `status: error`, `intent: unknown`, dan pesan
informatif dalam format response yang sama. Request tanpa field `query`
ditolak oleh validasi FastAPI dengan HTTP 422.

Build dan run Docker belum terverifikasi: saat pemeriksaan, Docker engine lokal
belum aktif. Setelah engine aktif dan API key diisi, jalankan ulang kelima query
melalui `/docs` pada aplikasi lokal dan container. Jangan menganggap hasil
simulasi sebagai hasil pengujian end-to-end Gemini atau container.

## Batasan dan Asumsi

- Data hanya berasal dari dua CSV bawaan; data tidak ditambah atau diubah.
- Satu permintaan diasumsikan memiliki satu intent dan satu ID tiket.
- ID tiket menggunakan format `TICKET-XXXX` dengan empat digit.
- Router keyword belum memahami semua variasi kalimat atau permintaan campuran.
- RAG memerlukan internet, API key aktif, akses model, dan kuota Gemini.
- Pembuatan index pertama melakukan embedding satu per satu sehingga lebih lambat;
  restart aplikasi akan membangun ulang index pada permintaan FAQ berikutnya.
- Embedding masih sinkron sehingga prototype ini cocok untuk pengujian sederhana,
  belum untuk banyak request bersamaan.
- Penolakan pertanyaan di luar cakupan bergantung pada kepatuhan LLM terhadap prompt.