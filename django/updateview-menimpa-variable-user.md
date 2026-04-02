## Django `UpdateView` Bisa Menimpa `user` di Template

Saat menggunakan `UpdateView` (atau `DetailView`, `DeleteView`) dengan model bernama `User`, Django secara otomatis menyisipkan objek tersebut ke dalam context template dengan key `"user"`. Hal ini dapat menimpa `user = request.user` yang berasal dari *auth context processor*.

### Kenapa Bisa Terjadi?

Di dalam `SingleObjectMixin.get_context_data()`, Django menambahkan objek ke context menggunakan nama model dalam huruf kecil:

```python
context[obj._meta.model_name] = self.object  # → context["user"] = <User A>
```

Sementara itu, `django.contrib.auth.context_processors.auth` juga menambahkan:

```python
context["user"] = request.user  # → context["user"] = <Admin>
```

Karena context dari view digabungkan setelah context processor, nilai `"user"` dari view (`User A`) akan menimpa `request.user` (`Admin`).

### Dampaknya

Tidak muncul error, tetapi perilaku aplikasi menjadi tidak sesuai:

* Sidebar atau bagian UI lain yang menampilkan pengguna login akan menunjukkan user yang sedang diedit, bukan user yang sedang login.
* Bug ini sering tidak disadari karena aplikasi tetap berjalan normal.

### Solusi

Tentukan `context_object_name` secara eksplisit agar tidak terjadi konflik nama:

```python
class UserEditView(UpdateView):
    model = User
    context_object_name = "edited_user"  # hindari penggunaan key "user"
```

### Catatan

Masalah ini juga bisa terjadi pada model lain yang namanya berbenturan dengan variabel bawaan Django, seperti:

* `messages`
* `request`
* `perms`

Semua key tersebut berpotensi tertimpa jika digunakan sebagai nama model atau `context_object_name`.
