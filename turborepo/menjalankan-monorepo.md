## Menjalankan Monorepo (Astro + FastAPI) dengan Turborepo & pnpm

Saat menggunakan monorepo dengan Turborepo, kita bisa menjalankan beberapa aplikasi sekaligus (misalnya Astro untuk frontend dan FastAPI untuk backend) dalam satu repo menggunakan `pnpm`.

### Kenapa Bisa Jalan Bareng?

Turborepo membaca script yang sama (misalnya `dev`) di setiap app dalam folder `apps/`, lalu menjalankannya secara paralel.

Struktur contoh:

```bash
apps/
  web/      # Astro
  api/      # FastAPI
packages/
turbo.json
```

Di masing-masing app:

```json
{
  "scripts": {
    "dev": "..."
  }
}
```

Saat menjalankan:

```bash
pnpm dev
```

Turborepo akan mengeksekusi semua `dev` script di `apps/*` secara bersamaan.

### Dampaknya

* Frontend (Astro) dan backend (FastAPI) bisa jalan dalam satu command
* Tidak perlu buka banyak terminal
* Workflow development jadi lebih rapi dan konsisten

### Solusi / Best Practice

Pastikan setiap app punya script `dev` yang jelas, misalnya:

```json
"dev": "astro dev"
```

```json
"dev": "uvicorn main:app --reload"
```

### Catatan

* Gunakan filter jika ingin menjalankan satu app saja:

```bash
pnpm --filter @app_name/web dev
```

* Penamaan folder di `apps/` akan mempengaruhi filter dan pipeline Turborepo
