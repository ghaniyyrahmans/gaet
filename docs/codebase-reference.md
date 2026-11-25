# Referensi Codebase gaet

Dokumen ini menjelaskan struktur dan perilaku aktif proyek `gaet` berdasarkan source code pada branch `main`, commit `ab046b8944bb35b96b4af3ca08da065c9fa0d5f1`. Dokumen ini ditujukan untuk kontributor, maintainer, dan pengembang yang ingin memakai CLI, dashboard, atau SDK tanpa harus mulai dari seluruh codebase.

> **Catatan sumber:** struktur simbol, hubungan pemanggilan, route, dan entry point diverifikasi menggunakan Codebase Memory MCP pada 21 November 2026. Source code tetap menjadi otoritas terakhir apabila dokumen ini berbeda dari implementasi.

## 1. Ringkasan produk

`gaet` adalah CLI Go tanpa dependensi runtime eksternal untuk mencadangkan database PostgreSQL lokal, menyinkronkannya ke database remote, mengambil state remote ke lokal, serta memulihkan database dari snapshot lokal.

Selain CLI, proyek menyediakan:

- dashboard web lokal melalui `gaet serve`;
- REST API untuk integrasi sidecar dengan aplikasi web;
- SDK TypeScript/JavaScript `@ghaniyyrahmans/gaet`;
- scheduler native untuk Linux, macOS, dan Windows;
- shell completion untuk Bash, Zsh, Fish, dan PowerShell.

## 2. Peta codebase

```text
cmd/gaet/main.go       entry point CLI, flag global, dan dispatcher
pkg/core/              fondasi: env, path, proses, output, error, lock, PostgreSQL
pkg/backup/            push, fetch, restore, retention, dan cron backup
pkg/config/            baca/tulis konfigurasi ~/.gaet/.env
pkg/detect/            penemuan instance PostgreSQL lokal
pkg/init/              wizard setup dan profile database
pkg/log/               pembacaan log aktivitas
pkg/remote/            pengelolaan URL database remote
pkg/scheduler/         systemd, launchd, dan Task Scheduler
pkg/serve/             dashboard embedded, HTTP API, dan job async
pkg/snapshots/         daftar dan pengelolaan snapshot
pkg/status/            check, status, diff, dan doctor
pkg/completion/        generator shell completion
sdk/                   client TypeScript/JavaScript dan type definitions
tests/                 unit/integration test Go
tests/e2e/              test end-to-end dengan binary nyata
docs/                  panduan, ADR, dan referensi
```

Entry point graph adalah `cmd/gaet/main.go:main`. Fungsi `dispatch` memilih package pelaksana berdasarkan subcommand. Package subcommand bergantung ke `pkg/core`; `pkg/serve` juga mengadaptasi package backup, detect, dan scheduler ke HTTP API.

## 3. Cara kerja operasi utama

### 3.1 `gaet push`

`push` memindahkan database lokal ke remote dan membuat snapshot lokal.

```text
dispatch
  -> backup.RunPush
     -> AcquireLock
     -> FindDBTools / GetLocalDB
     -> pg_dump --format=custom --compress=9
     -> pg_restore --list                 validasi snapshot
     -> resetTargetObjects                bersihkan target remote
     -> pg_restore                         restore ke remote
     -> applyRetention
     -> WriteJSONLog
     -> Release lock
```

Snapshot disimpan di `~/.gaet/backups/gaet_YYYYMMDD_HHmmss.dump`. Operasi menggunakan timeout dinamis: timeout dasar `GAET_TIMEOUT` ditambah anggaran per GB dari `GAET_TIMEOUT_PER_GB`. Nilai default yang didokumentasikan saat ini adalah 900 detik dan 300 detik per GB.

### 3.2 `gaet fetch`

`fetch` mengambil dump dari remote lalu menimpa database lokal. Karena bersifat destruktif, operasi ini memerlukan konfirmasi pada terminal interaktif atau flag `--yes` pada lingkungan non-TTY.

```text
RunFetch
  -> guard konfirmasi
  -> AcquireLock
  -> pg_dump dari remote ke cloud_TIMESTAMP.dump
  -> resetTargetObjects pada database lokal
  -> pg_restore ke database lokal
  -> hapus dump sementara
  -> WriteJSONLog
  -> Release lock
```

### 3.3 `gaet restore [snapshot]`

`restore` memulihkan database lokal dari snapshot yang ditentukan. Jika nama file tidak diberikan, snapshot terbaru dipilih berdasarkan waktu modifikasi. Snapshot divalidasi dengan `pg_restore --list` sebelum target lokal dibersihkan.

### 3.4 Backup terjadwal

`gaet auto N` atau `gaet push --auto=N` mendaftarkan scheduler native:

| Platform | Artefak |
|---|---|
| Linux | `~/.config/systemd/user/gaet-backup.service` dan `.timer` |
| macOS | `~/Library/LaunchAgents/com.gaet.backup.plist` |
| Windows | Windows Task Scheduler melalui `schtasks` |

Scheduler memanggil `gaet push --cron`. Mode cron menulis ke `cron.log` dan tidak mengandalkan output terminal.

## 4. State, file, dan konfigurasi

| Lokasi | Isi |
|---|---|
| `~/.gaet/.env` | konfigurasi aktif; harus memiliki permission `0600` |
| `~/.gaet/backups/` | dump snapshot dan lockfile |
| `~/.gaet/backups/.gaet.lock` | lock proses backup/restore |
| `~/.gaet/backups/gaet.log` | log aktivitas terstruktur |
| `~/.gaet/backups/cron.log` | log operasi scheduler |
| `~/.local/share/gaet` | asset/bundle aplikasi sesuai resolusi path platform |

Konfigurasi utama:

| Variabel | Peran |
|---|---|
| `GAET_LOCAL_URL` | URL database PostgreSQL lokal |
| `GAET_LOCAL_DB_HOST`, `PORT`, `USER`, `NAME`, `PASS` | parameter lokal terpisah |
| `GAET_REMOTE_URL` | URL database target remote |
| `GAET_REMOTE_SSLMODE` | mode SSL remote; default `require` |
| `GAET_RETENTION_DAYS` | umur snapshot sebelum dibersihkan; default `7` |
| `GAET_TIMEOUT` | timeout dasar operasi database; default `900` |
| `GAET_TIMEOUT_PER_GB` | tambahan timeout per GB; default `300` |
| `GAET_TABLES` | daftar tabel dipisahkan koma |
| `GAET_PG_DUMP`, `GAET_PG_RESTORE`, `GAET_PSQL` | override lokasi binary PostgreSQL |
| `GAET_DASHBOARD_HOST` | alamat bind dashboard; default localhost |
| `GAET_DASHBOARD_PORT` | port dashboard; default `6161` |

> Jangan menaruh secret nyata di repository. Gunakan `gaet init` atau `gaet set`. URL yang ditampilkan oleh aplikasi harus dimasking; password tidak boleh masuk ke argumen proses atau log.

## 5. Model keamanan dan konkurensi

- Operasi PostgreSQL dijalankan sebagai subprocess `pg_dump`, `pg_restore`, dan `psql`, bukan melalui library database yang ditautkan ke binary.
- Lock dibuat atomik dengan `O_CREATE|O_EXCL` di `~/.gaet/backups/.gaet.lock`. Lock stale dapat dipulihkan berdasarkan pemeriksaan PID atau umur lock.
- Password disediakan ke proses database melalui environment, sedangkan tool PostgreSQL dipanggil dengan mode tanpa prompt password (`-w`).
- File `.env` dan lockfile memakai mode `0600`.
- `fetch`, `restore`, dan operasi destruktif lain menolak eksekusi non-TTY tanpa persetujuan eksplisit.
- Dashboard default bind ke `127.0.0.1`. CORS saat ini terbuka (`*`) untuk mendukung integrasi sidecar; jangan mengekspos dashboard ke jaringan publik tanpa lapisan autentikasi dan kontrol jaringan tambahan.

## 6. CLI dan kode exit

Sintaks umum:

```bash
gaet [global-options] <command> [args]
```

Global option penting adalah `--json`, `--plain`, `--quiet`, dan `--dry-run`. Kode exit didefinisikan di `pkg/core/errors.go`:

| Kode | Makna |
|---:|---|
| `0` | sukses |
| `1` | kegagalan umum atau restore/integrity check gagal |
| `2` | penggunaan salah atau command tidak dikenal |
| `80` | konfigurasi tidak ada/tidak valid |
| `81` | database lokal tidak dapat dijangkau |
| `82` | database cloud tidak dapat dijangkau |
| `83` | proses lain sedang memegang lock |
| `84` | tool PostgreSQL tidak ditemukan |

Untuk otomasi, gunakan output `--json` dan periksa exit code. Jangan parsing output dekoratif terminal jika format mesin tersedia.

## 7. Dashboard dan REST API

`gaet serve` menjalankan `net/http` server standar Go dan menyajikan asset dashboard yang di-embed dari `pkg/serve/static`. Default-nya tersedia di `http://127.0.0.1:6161`.

| Method | Endpoint | Fungsi |
|---|---|---|
| GET | `/api/status` | status database dan sinkronisasi |
| GET | `/api/check` | cek tool dan konektivitas |
| GET | `/api/doctor` | diagnosis lengkap |
| GET | `/api/diff` | perbandingan row count |
| POST | `/api/push` | backup lokal ke remote |
| POST | `/api/fetch` | restore remote ke lokal |
| POST | `/api/restore` | restore snapshot lokal |
| GET | `/api/snapshots` | daftar snapshot |
| GET | `/api/snapshots/download` | unduh snapshot dengan validasi path |
| POST | `/api/snapshots/delete` | hapus snapshot |
| GET | `/api/logs` | baca log |
| GET/POST | `/api/config` | baca atau ubah konfigurasi |
| GET/POST/DELETE | `/api/schedule` | inspeksi atau ubah scheduler |
| GET | `/api/jobs` | daftar job asynchronous |

Handler HTTP memanggil `backup.RunPush`, `RunFetch`, dan `RunRestore` yang sama dengan CLI. Dashboard adalah adapter HTTP di atas business logic yang sama.

## 8. SDK TypeScript

Paket `@ghaniyyrahmans/gaet` memakai `fetch` untuk mengakses REST API lokal.

```ts
import { gaet } from '@ghaniyyrahmans/gaet';

const status = await gaet.status();
const result = await gaet.push();

if (!result.ok) {
  console.error(result.msg);
}
```

Untuk aplikasi Node.js, `startServer()` dapat menjalankan `gaet serve`; untuk deployment yang sudah memiliki daemon, gunakan `GaetClient` dengan `baseUrl` dan `timeout` yang sesuai. SDK menyediakan wrapper untuk status, push, fetch, restore, snapshots, logs, check, doctor, diff, detect, konfigurasi, dan export.

## 9. Pengembangan dan validasi

Prasyarat pengembangan adalah Go yang sesuai dengan `go.mod`, tool PostgreSQL untuk test yang memerlukan database, dan Node.js bila mengembangkan SDK.

```bash
go vet ./...
go test -v -count=1 ./tests/...
```

Test di `tests/` umumnya membangun binary aktual dan menjalankan subcommand dengan `GAET_DIR` sementara. Test E2E di `tests/e2e/` mencakup lifecycle, backup/fetch/restore, job asynchronous, endpoint dashboard, snapshot download, dan skenario kegagalan.

Sebelum mengubah dokumentasi, jalankan:

```bash
bash sandbox/verify-docs.sh
```

Script tersebut memeriksa konsistensi klaim dokumentasi terhadap repository dan juga dijalankan oleh job CI `docs-consistency`.

## 10. Panduan perubahan berdasarkan area

| Jika mengubah... | Periksa juga... |
|---|---|
| command atau flag | `cmd/gaet/main.go`, README command reference, test CLI |
| pipeline backup | `pkg/backup`, `docs/architecture.md`, test backup/E2E |
| konfigurasi | `.env.example`, `pkg/core/env.go`, `pkg/config`, SDK types |
| route dashboard | `pkg/serve/server.go`, REST guide, server/E2E tests, SDK |
| scheduler | `pkg/scheduler`, platform-specific tests, install/support docs |
| exit code/error | `pkg/core/errors.go`, troubleshooting, CI integration guide |
| format snapshot/log | snapshot/log tests dan dokumentasi operasional |

Perubahan yang menyentuh `pkg/core` berisiko tinggi karena graph menunjukkan fan-in terbesar berada di package tersebut. Setelah perubahan, prioritaskan test untuk `core`, backup, CLI dispatch, dan endpoint dashboard yang memakai jalur yang sama.

## 11. Batasan dokumentasi ini

Indeks Codebase Memory mencatat beberapa file benchmark SQL, installer PowerShell, asset HTML, dan completion Zsh dengan parsing parsial atau tidak terindeks. Area tersebut tidak dipakai sebagai dasar klaim detail dalam dokumen ini. Dokumentasi ini juga mendeskripsikan implementasi pada commit yang disebutkan di atas; perubahan source harus diikuti review dan pembaruan dokumen.

## Referensi repository

- [README utama](../README.md)
- [Indeks dokumentasi](README.md)
- [Quickstart](quickstart.md)
- [Architecture & Design](architecture.md)
- [REST API & Sidecar Integration](rest-api-integration.md)
- [Troubleshooting](troubleshooting.md)
- [CI/CD Integration](ci-cd-integration.md)
- [`.env.example`](../.env.example)
- [ADR index](adr/README.md)
