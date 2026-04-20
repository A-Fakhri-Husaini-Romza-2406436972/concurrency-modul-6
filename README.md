Commit 1 reflection notes

### Fungsi main.rs saat ini
Kode `src/main.rs` saat ini mengimplementasikan sebuah server TCP sederhana di Rust. Berikut adalah hal-hal yang dilakukannya:
1. **Mendengarkan Koneksi (Listening)**: Mengikat (bind) alamat IP lokal `127.0.0.1` pada port `7878` menggunakan `TcpListener`.
2. **Menerima Koneksi**: Melakukan iterasi dan menerima setiap koneksi masuk (`stream`) dari klien yang terhubung ke server tersebut.
3. **Membaca Request (handle_connection)**: Setiap `TcpStream` yang masuk akan diproses dengan membacanya menggunakan `BufReader`.
4. **Mencetak Output**: Seluruh baris dari HTTP request akan dibaca hingga terdapat baris kosong, lalu dikumpulkan dalam sebuah *Vector*, dan akhirnya dicetak (print) bentuk format debug dari HTTP request tersebut ke konsol.

Commit 2 Reflection notes

### Returning HTML
Pada commit kedua ini, fungsi `handle_connection` di `src/main.rs` telah diperbarui untuk mengembalikan file HTML ("hello.html") sebagai respons kepada klien. 
1. **Membaca File html**: Menggunakan `fs::read_to_string` untuk membaca isi `hello.html` ke dalam sebuah string.
2. **Membuat Response HTTP**: Membuat string `response` yang memiliki baris status HTTP 200 OK (`HTTP/1.1 200 OK`), header `Content-Length` sesuai panjang isi file, baris kosong penutup header HTTP, diikuti dengan isi dokumen HTML itu sendiri. 
3. **Mengirim Response**: Menggunakan `stream.write_all(response.as_bytes())` untuk menulis data respons tersebut kembali melewari jalur TCP ke klien.

![Commit 2](commit2.png)

Commit 3 Reflection notes

### Validating Request and Selectively Responding
Pada tahap ini, web server telah dikembangkan untuk mengecek baris permintaan (request line) HTTP. Jika rute yang diminta adalah `"GET / HTTP/1.1"`, server akan merespons dengan menyertakan isi dari file `hello.html` beserta status `200 OK`. Jika klien meminta rute lain yang tidak terdaftar, server akan secara otomatis memecah aliran tanggapan dengan status `404 NOT FOUND` dan mengirimkan konten `404.html` (halaman peringatan bahwa halaman tidak ditemukan). Mekanisme ini adalah langkah pertama yang kuat menuju _routing_ web server sejati.

### Mengapa Refactoring Diperlukan?
Sebelum kode di-*refactor*, blok `if` dan `else` terlihat memiliki banyak sekali pengulangan kode. Keduanya sama-sama memuat dan membaca file, mendapatkan panjang kontennya, menyusun argumen panjang ke dalam string HTTP yang sama, dan menuliskannya ke dalam koneksi *stream*. Duplikasi semacam ini buruk karena dua hal: membuat rentan terjadinya eror jika salah satu diubah namun lupa mengubah bagian lain, dan membuat kode jadi panjang serta sulit dibaca.

Oleh karena itu, kode di-*refactor* sehingga bagian `if` dan `else` hanya mengembalikan **status line HTTP** dan **nama file tujuan** dalam bentuk nilai `tuple` (Tuple Destructuring). Bagian logika pengiriman respons (pembacaan file dan pengubahan/penempelan *Header*) ditarik keluar menjadi *unconditional code* di bawahnya. Hal ini membuat keseluruhan struktur menjadi jauh lebih pendek, bersih, elegan, dan lebih _maintainable_ untuk pembaruan berikutnya tanpa mengorbankan performa!

Commit 4 Reflection notes

### Simulation of slow request
Pada iterasi ini, ditambahkan sebuah *route* lambat, yakni `/sleep` (`GET /sleep HTTP/1.1`) menggunakan operator `match`. Perbedaannya dari route biasa (seperti root `/`) adalah adanya blok `thread::sleep(Duration::from_secs(10));` yang menahan jalannya eksekusi *thread* selama 10 detik sebelum akhirnya mengembalikan response sukses. 

**Tujuan Simulasi lambat:**
Mengingat web server ini saat ini berjalan secara **single-threaded** (hanya bekerja di iterasi perputaran koneksi dari `listener.incoming()`), simulasi _sleep_ ini menyoroti sebuah kelemahan fundamental kode kita jika berada di *environment* nyata (production). 
Eksperimen membuktikannya jika kita membuka `127.0.0.1:7878/sleep` di window browser pertama, dan **kemudian tepat disaat bersamaan** kita membuka `127.0.0.1:7878/` di window browser kedua. Window kedua tersebut juga **akan ikut macet memuat (loading) selama kurang lebih 10 detik** hingga browser pertama selesai.

**Penjelasan kenapa ini terjadi:** 
Fungsi `main` memproses setiap koneksi TCP dengan `handle_connection(stream)` di thread yang sama secara *synchronous* satu persatu. Ketika server tertunda 10 detik memproses request `/sleep` dari Browser A, loop `for stream in listener.incoming()` tidak akan dapat berlanjut ke perputaran (iterasi blok scope) berikutnya untuk menyambar permohonan koneksi stream dari Browser B meskipun _request_-nya adalah `/` (ringan dan biasa). Server tersebut menjadi *bottleneck*, di mana satu pengguna lambat dapat membuat semua pengguna lain di antrean ikut menjadi mangkrak tak terlayani. Oleh karena itu, kita memerlukan konkurensi (Concurrency) agar setiap stream dapat diproses oleh banyak _threads_ mandiri secara simultan (Multithreading/ThreadPool).

Commit 5 Reflection notes

### Multithreaded server using Threadpool
Untuk mengatasi masalah single-threaded *bottleneck* sebelumnya, kita mengimplementasikan `ThreadPool` untuk server ini dengan kapasitas awal terbatas (contoh: 4 threads). Alih-alih membuat thread baru secara tidak terbatas per setiap *request* yang bisa menghabiskan memori sistem (rentan DoS attack), ThreadPool menyediakan jumlah thread pekerja (*workers*) yang tetap dan terus menunggu pekerjaan (job). 

**Cara kerja Threadpool:**
Server web mendengarkan koneksi lalu mengirim *closure* pemrosesan (fungsi `handle_connection(stream)`) ke ThreadPool menggunakan `pool.execute(...)`. Di dalam manajemen internal ThreadPool, *closure* tersebut dikemas menjadi `Job` (sebuah trait object bertipe `Box<dyn FnOnce() + Send + 'static>`) lalu dikirimkan via *channel transmitter* (`mpsc::Sender`).

Di ujung lainnya, *receiver channel* digunakan bersama oleh kumpulan 4 `Worker` menggunakan mekanisme antrean *thread-safe* yaitu `Arc<Mutex<Receiver<Job>>>`. Adanya `Mutex` (Mutual Exclusion) menjaga agar di satu waktu hanya ada SATU Worker yang bisa menarik job dari antrean channel (menjalankan satu _request_ TCP tertentu), sisa job dari _clients_ lain akan ditarik oleh Workers lain yang tengah diam (*idle*). Konsep elegan di atas memungkinkan server untuk menangani maksimal 4 _request stream_ (`N`) sekaligus di latar belakang (asynchronous processing), sehingga simulasi hambatan lambat _request_ `/sleep` tidak akan membekukan permintaan cepat lain selama Workers mumpuni (tersedia).

Commit Bonus Reflection notes

### Function improvement
Sebagai bonus perbaikan ("Function improvement"), web server ini dirancang untuk memiliki metode alternatif dari fungsi inisiasi awal `new` milik `ThreadPool` menjadi fungsi yang lebih aman, yaitu fungsi `build()`.

**Mengapa ini lebih baik?**
`ThreadPool::new` menggunakan makro pengecekan keras yaitu `assert!(size > 0);`. Jika di skenario nyata (misal data konfigurasi keliru meminta _0 threads_), *program akan langsung crash atau panik secara kasar*. Sedangkan, kita ingin program kita menjadi se-tangguh tipe I/O parser di `Chapter 12` Rust Book, yang bisa pulih atau setidaknya menutup secara elegan ("*Graceful Shutdown*"). 

Fungsi `build`, alih-alih melempar _panic_, membungkus respons baliknya dengan Tipe Data `Result<ThreadPool, PoolCreationError>`. Jika parameter `size == 0`, fungsi ini tidak mematikan program, namun mengembalikan `Err` custom yang berisi *Trait Error String* yang bernilai `"ThreadPool size must be greater than zero"`. Di `main.rs`, pemanggilan inisialisasi tersebut ditangani oleh adapter `unwrap_or_else` sehingga admin server dapat melihat pesan _error stderr_ yang informatif lalu program membatalkan diri (`exit(1)`) ketimbang crash yang mengerikan. Error-handling ini jauh lebih elegan dan siap produksi (*production-ready*).

