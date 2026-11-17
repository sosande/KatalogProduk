# Katalog Produk (JavaFX + SQLite + Maven)

Aplikasi desktop sederhana untuk mengelola katalog produk. Data disimpan di file database SQLite (`data/katalog.db`) dan diakses dengan `java.sql` (JDBC).

README ini fokus pada dua hal:

1. [Cara menjalankan proyek di NetBeans (Maven)](#1-cara-menjalankan-di-netbeans-maven)
2. [Cara menambahkan fitur hapus produk ke database](#2-cara-menambahkan-fitur-hapus-produk)

---

## 1. Cara Menjalankan di NetBeans (Maven)

### Prasyarat

- JDK 17 atau lebih baru
- Apache NetBeans (versi yang mendukung Maven, misalnya 17 ke atas)
- Koneksi internet saat pertama kali build, karena Maven mengunduh JavaFX dan SQLite JDBC

### Langkah

**1. Buka proyek**

`File` → `Open Project...` → pilih folder `KatalogProduk` (folder yang berisi `pom.xml`). Ikon proyek akan berlogo Maven.

**2. Pastikan memakai JDK 17+**

Klik kanan proyek → `Properties` → `Build` → `Compile`, lalu pastikan Java Platform yang dipilih adalah JDK 17 atau lebih baru.

**3. Unduh dependency**

Klik kanan proyek → `Build with Dependencies`. Maven akan mengunduh library berikut (sudah tertulis di `pom.xml`):

| Dependency | Versi |
|------------|-------|
| `org.openjfx:javafx-controls` | 21 |
| `org.openjfx:javafx-fxml` | 21 |
| `org.xerial:sqlite-jdbc` | 3.49.1.0 |

**4. Atur cara menjalankan (sekali saja)**

Karena `MainApp` adalah aplikasi JavaFX, cara paling aman menjalankannya di NetBeans adalah lewat plugin `javafx-maven-plugin` yang sudah ada di `pom.xml`:

1. Klik kanan proyek → `Properties` → `Actions`.
2. Pilih `Run project`.
3. Pada kolom `Other Goals`, isi dengan:
   ```
   javafx:run
   ```
4. Klik `OK`.

**5. Jalankan**

Tekan **F5** (atau klik kanan proyek → `Run`).

Kalau tidak ingin mengubah pengaturan Actions, kamu bisa klik kanan proyek → `Run Maven` → `Goals...`, lalu isi `javafx:run`.

### Hasil saat pertama dijalankan

- Folder `data/` dan file `katalog.db` dibuat otomatis di dalam folder proyek.
- Tabel `produk` dibuat otomatis oleh `Database.prepareDatabase()`.
- Lokasi file database dicetak di jendela Output NetBeans dengan awalan `DATABASE AKTIF:`.

### Masalah umum

| Masalah | Solusi |
|---------|--------|
| `JavaFX runtime components are missing` | Jalankan dengan goal `javafx:run` (langkah 4), jangan lewat `Run File` biasa. |
| Dependency merah / tidak ditemukan | Klik kanan proyek → `Build with Dependencies`, pastikan internet aktif. |
| `data/katalog.db` muncul di folder lain | Database mengikuti `user.dir`. Jalankan dari NetBeans lewat `javafx:run` agar `user.dir` adalah folder proyek. |
| Gambar tidak muncul | Pastikan file ada di `src/main/resources/image/` lalu `Clean and Build`. |

---

## 2. Cara Menambahkan Fitur Hapus Produk

Proyek awal hanya punya fitur tambah dan tampil. Fitur hapus ditambahkan di tiga lapisan: **repository** (SQL), **controller** (logika UI), dan **CSS** (tampilan tombol). `Database.java`, `Produk.java`, `MainApp.java`, `katalog.fxml`, dan `pom.xml` tidak diubah.

### Alur kerja

```
Klik tombol "Hapus" di kartu produk
        ↓
Dialog konfirmasi (OK / Cancel)
        ↓ OK
ProdukRepository.delete(id)  →  DELETE FROM produk WHERE id = ?
        ↓
showData()  →  katalog dimuat ulang dari database
```

### Langkah 1: Tambah method `delete` di `data/ProdukRepository.java`

Method ini memakai `PreparedStatement` dengan parameter `?` agar aman. `executeUpdate()` mengembalikan jumlah baris yang terhapus, jadi hasilnya bisa dipakai untuk mengecek apakah ID memang ada.

```java
public boolean delete(long id) throws SQLException {
    String sql = "DELETE FROM produk WHERE id = ?";
    try (
        Connection koneksi = Database.bukaKoneksi();
        PreparedStatement perintah = koneksi.prepareStatement(sql)) {
        perintah.setLong(1, id);
        // executeUpdate mengembalikan jumlah baris yang terhapus
        return perintah.executeUpdate() > 0;
    }
}
```

### Langkah 2: Ubah `controller/KatalogController.java`

**a. Tambah import** di bawah `import java.util.ArrayList;`:

```java
import java.util.Optional;
import javafx.scene.control.Alert;
import javafx.scene.control.Button;
import javafx.scene.control.ButtonType;
```

**b. Tambah tombol Hapus** di method `createProductCard`, sebelum `return kartu;`:

```java
kartu.getChildren().add(idLabel);

Button hapusButton = new Button("Hapus");
hapusButton.getStyleClass().add("button-danger");
hapusButton.setOnAction(e -> deleteProduct(produk));
kartu.getChildren().add(hapusButton);
return kartu;
```

**c. Tambah method `deleteProduct`** tepat setelah `createProductCard`:

```java
private void deleteProduct(Produk produk) {
    Alert konfirmasi = new Alert(Alert.AlertType.CONFIRMATION);
    konfirmasi.setTitle("Hapus Produk");
    konfirmasi.setHeaderText("Hapus \"" + produk.getNama() + "\"?");
    konfirmasi.setContentText("Produk akan dihapus permanen dari database.");
    Optional<ButtonType> pilihan = konfirmasi.showAndWait();
    if (pilihan.isEmpty() || pilihan.get() != ButtonType.OK) {
        return;
    }
    try {
        if (repository.delete(produk.getId())) {
            pesanLabel.setText("Produk \"" + produk.getNama() + "\" berhasil dihapus.");
        } else {
            pesanLabel.setText("Produk tidak ditemukan di database.");
        }
    } catch (SQLException e) {
        pesanLabel.setText("Gagal menghapus. Lihat Output.");
        e.printStackTrace();
        return;
    }
    try {
        showData();
    } catch (SQLException e) {
        pesanLabel.setText("Data terhapus, tetapi katalog gagal dimuat ulang.");
        e.printStackTrace();
    }
}
```

### Langkah 3: Tambah gaya tombol di `resources/view/styles/app.css`

Tambahkan sebelum blok `.katalog { ... }`:

```css
.button-danger {
    -fx-background-color: #fee2e2;
    -fx-text-fill: #b91c1c;
    -fx-padding: 6 16;
    -fx-background-radius: 8;
}
.button-danger:hover {
    -fx-background-color: #fecaca;
}
```

### Langkah 4: Uji fitur

1. Jalankan aplikasi (`javafx:run`).
2. Tambahkan satu produk, lalu klik **Hapus** pada kartunya dan pilih **OK**.
3. Kartu hilang dan label jumlah produk berkurang.
4. Tutup aplikasi lalu jalankan lagi. Produk tetap tidak ada, artinya baris benar-benar terhapus dari `data/katalog.db`.

### Catatan tentang ID

Kolom `id` memakai `INTEGER PRIMARY KEY`, sehingga SQLite bisa memakai ulang ID yang sudah dihapus untuk produk baru. Kalau ID lama tidak boleh dipakai lagi, ubah di `Database.java` menjadi `id INTEGER PRIMARY KEY AUTOINCREMENT`. Perubahan ini hanya berlaku untuk tabel baru, jadi hapus `data/katalog.db` lama terlebih dahulu.

---

## Struktur Proyek

```
KatalogProduk/
├── pom.xml
├── data/                      # dibuat otomatis, berisi katalog.db
└── src/main/
    ├── java/
    │   ├── main/MainApp.java                  # entry point
    │   ├── controller/KatalogController.java  # logika UI (tambah, tampil, hapus)
    │   ├── data/ProdukRepository.java         # akses database (INSERT, SELECT, DELETE)
    │   ├── db/Database.java                   # koneksi & pembuatan tabel
    │   └── model/Produk.java                  # model data
    └── resources/
        ├── image/                             # roti.png, susu.png, kopi.png
        └── view/
            ├── katalog.fxml
            └── styles/app.css
```

## .gitignore yang disarankan

```
data/
target/
nbproject/private/
```
