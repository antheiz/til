# Fix `XAmzContentSHA256Mismatch` saat Upload File ke Cloudflare R2

Saat mengupload file dari Django Admin ke Cloudflare R2, muncul error berikut:

```
An error occurred (XAmzContentSHA256Mismatch) when calling the PutObject operation: None
```

**Stack:** Django 5.1 · django-storages 1.14.6 · boto3 1.42.88 · botocore 1.42.88

### Kenapa Bisa Terjadi?

boto3 secara default tidak menghitung hash SHA256 dari body request. Ia mengirim nilai literal `UNSIGNED-PAYLOAD` di header `x-amz-content-sha256`:

```
x-amz-content-sha256: UNSIGNED-PAYLOAD
```

S3 Provider menolak `UNSIGNED-PAYLOAD` dan mengharuskan nilai SHA256 yang benar-benar dihitung dari isi file.

### Yang Tidak Berhasil

Beberapa pendekatan yang tampak masuk akal ternyata tidak bekerja di versi ini:

- `"payload_signing_enabled": True` di `OPTIONS` → bukan nama valid di django-storages 1.14.6
- `Config(payload_signing_enabled=True)` → parameter tidak dikenal di botocore 1.42.88
- `AWS_S3_PAYLOAD_SIGNING_ENABLED = True` di global settings → juga tidak dikenal di versi ini

### Solusi

Tambahkan dua baris `os.environ` di `settings.py` sebelum blok `STORAGES`:

```python
import os

if config("USE_S3", default=False, cast=bool):
    # Paksa botocore hitung SHA256 aktual dari setiap payload
    os.environ.setdefault("AWS_REQUEST_CHECKSUM_CALCULATION", "when_required")
    os.environ.setdefault("AWS_RESPONSE_CHECKSUM_VALIDATION", "when_required")
    
    STORAGES = {
        "default": {
            "BACKEND": "storages.backends.s3boto3.S3Boto3Storage",
            "OPTIONS": {
                "access_key": config("R2_ACCESS_KEY_ID"),
                "secret_key": config("R2_SECRET_ACCESS_KEY"),
                "bucket_name": config("R2_BUCKET_NAME"),
                "endpoint_url": config("R2_ENDPOINT_URL"),
                "signature_version": "s3v4",
            },
        },
        ...
    }
```

Header yang dikirim berubah dari:

```
x-amz-content-sha256: UNSIGNED-PAYLOAD   ✗
```

menjadi:

```
x-amz-content-sha256: e3b0c44298fc1c149afb...   ✓
```

### Mengapa `os.environ`, Bukan `.env`?

`AWS_REQUEST_CHECKSUM_CALCULATION` dibaca langsung oleh botocore di level paling rendah dari `os.environ` — bukan dari variabel Python biasa. Meletakkannya di `.env` hanya akan bekerja jika environment variable tersebut benar-benar diinjeksi ke process environment (misalnya via systemd atau Docker), bukan sekadar dibaca oleh python-decouple.

Selain itu, nilai `when_required` adalah konstanta teknis yang tidak berubah antar environment, sehingga tidak perlu masuk ke `.env` — cukup hardcode langsung di `settings.py`.

### Catatan Tambahan

- Hapus `"default_acl": "public-read"` jika bucket sudah dikonfigurasi public di dashboard provider.
- Pastikan `endpoint_url` menggunakan format sesuai provider S3
