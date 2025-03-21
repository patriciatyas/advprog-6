# Module 6 - Concurrency

## Commit 1 Reflection

### Kode Lama
Dalam kode pertama:

```rust
use std::net::TcpListener;

fn main() {
    let listener = TcpListener::bind("127.0.0.1:7878").unwrap();
    for stream in listener.incoming() {
        let stream = stream.unwrap();
        println!("Connection established!");
    }
}
```

Kode tersebut hanya menerima koneksi TCP dan mencetak pesan saat ada klien yang terhubung, tanpa membaca atau memproses data yang dikirim klien.

### Penjelasan kode lama
`TcpListener::bind("127.0.0.1:7878")` -> Membuat server yang mendengarkan koneksi pada port 7878

`for stream in listener.incoming()` -> Loop untuk menangani setiap koneksi yang masuk.

`let stream = stream.unwrap();` -> Mengambil koneksi yang berhasil dibuat.

`println!("Connection established!");` -> Mencetak pesan ketika klien terhubung.

### Kelemahan dari kode lama tersebut adalah:
- Tidak membaca atau memproses data yang dikirim oleh klien.
- Hanya mencetak pesan tanpa interaksi lebih lanjut.

### Kode Baru

```rust
use std::{
    io::{prelude::*, BufReader},
    net::{TcpListener, TcpStream},
};

fn main() {
    let listener = TcpListener::bind("127.0.0.1:7878").unwrap();

    for stream in listener.incoming() {
        let stream = stream.unwrap();
        handle_connection(stream);
    }
}

fn handle_connection(mut stream: TcpStream) {
    let buf_reader = BufReader::new(&mut stream);
    let http_request: Vec<_> = buf_reader
        .lines()
        .map(|result| result.unwrap())
        .take_while(|line| !line.is_empty())
        .collect();
    
    println!("Request: {:#?}", http_request);
}
```

Kode tersebut membaca permintaan HTTP dari klien dan mencetaknya dalam format terstruktur.

Dalam kode baru tersebut, fungsi `handle_connection` adalah membaca permintaan HTTP menggunakan `BufReader`, memproses data baris per baris menggunakan `.lines()`, berhenti membaca saat menemukan baris kosong ("") yang menandai akhir header HTTP, mencetak permintaan HTTP ke terminal.

Perubahan dari kode lama yang dapat kita lihat adalah sekarang kode tersebut membaca data dari klien, bukan hanya mencetak pesan koneksi. Kode tersebut juga menggunakan `BufReader` untuk membaca baris demi baris. Selain itu, ditambahkan juga `.take_while(|line| !line.is_empty())` untuk menangani akhir header HTTP.

### Output Program (`cargo run`)
Saat menjalankan server (`cargo run`) jika ada klien (misalnya browser) yang mengakses ke `http://127.0.0.1:7878` outputnya akan menampilkan permintaan HTTP:

```
Request: [
    "GET / HTTP/1.1",
    "Host: 127.0.0.1:7878",
    "Connection: keep-alive",
    "Cache-Control: max-age=0",
    "sec-ch-ua: \"Chromium\";v=\"134\", \"Not:A-Brand\";v=\"24\", \"Google Chrome\";v=\"134\"",
    "sec-ch-ua-mobile: ?0",
    "sec-ch-ua-platform: \"Windows\"",
    "Upgrade-Insecure-Requests: 1",
    "User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/134.0.0.0 Safari/537.36",
    "Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7",
    "Sec-Fetch-Site: cross-site",
    "Sec-Fetch-Mode: navigate",
    "Sec-Fetch-User: ?1",
    "Sec-Fetch-Dest: document",
    "Accept-Encoding: gzip, deflate, br, zstd",
    "Accept-Language: en-US,en;q=0.9,id;q=0.8",
    "Cookie: csrftoken=p5UjAl9jZM4x44ZFu77JVHA2CeXc3zk7",
]
```

### Penjelasan output:
1. `GET / HTTP/1.1`
- Menunjukkan bahwa browser mengirimkan permintaan HTTP GET ke `/` (root).

2. Header Utama
- `"Host: 127.0.0.1:7878"` -> Server yang dituju.
- `"User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) ..."` -> Browser yang digunakan.
- `"Accept: text/html,application/xhtml+xml,application/xml;...` -> Format yang diterima oleh browser.
- `"Cookie: csrftoken=p5UjAl9jZM4x44ZFu77JVHA2CeXc3zk7"` -> Cookie yang dikirimkan (misalnya untuk sesi login).

3. Permintaan Berulang
- Jika permintaan GET dikirim berkali-kali oleh browser, output akan muncul beberapa kali karena browser sering mengirim permintaan tambahan untuk caching dan koneksi keep-alive. Hal ini menjelaskan mengapa permintaan GET bisa muncul lebih dari sekali dalam log terminal.


## Commit 2 Reflection

![Commit 2 screen capture](/assets/images/commit2.png)

```rust
use std::{
    fs,
    ...
};

...

fn handle_connection(mut stream: TcpStream) {
    ...

    let status_line = "HTTP/1.1 200 OK";
    let contents = fs::read_to_string("hello.html").unwrap();
    let length = contents.len();

    let response = format!("{status_line}\r\nContent-Length: {length}\r\n\r\n{contents}");
    stream.write_all(response.as_bytes()).unwrap();
}

```

### Perubahan yang dilakukan
Pada commit 2, Pada commit ini, kita mengembangkan fungsi `handle_connection` agar server tidak hanya membaca HTTP request tetapi juga mengirimkan respons HTTP yang berisi file HTML (`hello.html`). Berikut adalah detail perubahan yang dilakukan:

Kode baru di `handle_connection`:
```rust
use std::{fs, io::prelude::*, net::TcpStream};

fn handle_connection(mut stream: TcpStream) {
    let buf_reader = std::io::BufReader::new(&mut stream);
    let http_request: Vec<_> = buf_reader
        .lines()
        .map(|result| result.unwrap())
        .take_while(|line| !line.is_empty())
        .collect();

    println!("Request: {:#?}", http_request);

    let status_line = "HTTP/1.1 200 OK";
    let contents = fs::read_to_string("hello.html").unwrap();
    let length = contents.len();

    let response = format!("{status_line}\r\nContent-Length: {length}\r\n\r\n{contents}");
    stream.write_all(response.as_bytes()).unwrap();
}
```

Fungsi `handle_connection` sekarang membaca request HTTP menggunakan `BufReader::new(&mut stream)`, yang membungkus `stream` agar dapat dibaca baris per baris. Kemudian, `.lines()` digunakan untuk mengambil setiap baris dari request yang dikirim klien, diikuti dengan `.map(|result| result.unwrap())` untuk mengurai hasilnya menjadi string. Proses pembacaan dihentikan dengan `.take_while(|line| !line.is_empty())`, yang memastikan kita hanya mengambil header HTTP hingga baris kosong (`\r\n`). Hasilnya adalah vektor (`Vec<_>`) berisi seluruh header HTTP, yang kemudian ditampilkan ke terminal menggunakan `println!("Request: {:#?}", http_request);`.

Setelah request diproses, server menyusun respons HTTP dengan membaca file HTML menggunakan `fs::read_to_string("hello.html").unwrap()`. Jika file ditemukan, isinya disimpan dalam variabel `contents`, dan panjang kontennya dihitung menggunakan `contents.len()`. Respons kemudian dirangkai dalam format standar HTTP menggunakan `format!("{status_line}\r\nContent-Length: {length}\r\n\r\n{contents}")`, di mana `status_line` berisi `"HTTP/1.1 200 OK"`, `Content-Length` menunjukkan ukuran file, dan bagian terakhir adalah isi dari `hello.html`.

Akhirnya, respons dikirim ke klien melalui `stream.write_all(response.as_bytes()).unwrap();`. Metode `write_all()` memastikan bahwa seluruh data dikirim melalui TCP stream, sementara `as_bytes()` mengubah string menjadi byte array agar bisa dikirim melalui jaringan. Dengan perubahan ini, jika server dijalankan menggunakan `cargo run` dan diakses melalui `http://127.0.0.1:7878`, browser akan menerima halaman HTML yang berisi:

```
<!DOCTYPE html>
<html lang="en">
    <head>
        <meta charset="utf-8">
        <title>Hello!</title>
    </head> 
    <body>
        <h1>Hello!</h1>
        <p>Hi from Rust, running from Patricia's machine.</p>
    </body>
</html>
```

Dengan ini, server Rust sekarang bisa mengirimkan file HTML sebagai respons HTTP. Penggunaan fungsi `fs::read_to_string()` membuatnya lebih fleksibel karena konten halaman dapat diperbarui tanpa perlu mengubah kode Rust. Header `Content-Length` juga ditambahkan untuk memastikan respons dikirim dengan benar. 

Ketika menjalankan `cargo run`, muncul peringatan berikut:

```
PS C:\adpro\module-6\hello> cargo run
warning: unused variable: `http_request`
  --> src\main.rs:18:9
   |
18 |     let http_request: Vec<_> = buf_reader
   |         ^^^^^^^^^^^^ help: if this is intentional, prefix it with an underscore: `_http_request`
   |
   = note: `#[warn(unused_variables)]` on by default

warning: `hello` (bin "hello") generated 1 warning
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.01s
     Running `target\debug\hello.exe`
```

Peringatan ini muncul karena variabel `http_request` dideklarasikan tetapi tidak digunakan di dalam kode. Compiler menyarankan untuk menambahkan awalan underscore jika memang variabel tersebut sengaja tidak digunakan. Meskipun ada peringatan tersebut, program tetap berhasil di-compile dan dijalankan dalam mode development dengan profil debug yang unoptimized.

## Commit 3 Reflection

![Commit 3 screen capture](/assets/images/commit3.png)

Dalam commit 3, terdapat beberapa perubahan pada fungsi `handle_connection`, yaitu:

```rust
fn handle_connection(mut stream: TcpStream) {
    let buf_reader = BufReader::new(&stream);
    let request_line = buf_reader.lines().next().unwrap().unwrap();

    if request_line == "GET / HTTP/1.1" {
        let status_line = "HTTP/1.1 200 OK";
        let contents = fs::read_to_string("hello.html").unwrap();
        let length = contents.len();

        let response = format!(
            "{status_line}\r\nContent-Length: {length}\r\n\r\n{contents}"
        );

        stream.write_all(response.as_bytes()).unwrap();
    } else {
        let status_line = "HTTP/1.1 404 NOT FOUND";
        let contents = fs::read_to_string("404.html").unwrap();
        let length = contents.len();

        let response = format!(
            "{status_line}\r\nContent-Length: {length}\r\n\r\n{contents}"
        );

        stream.write_all(response.as_bytes()).unwrap();
    }
}
```

Dengan perubahan tersebut, server kini dapat membedakan request yang valid dan tidak valid dengan memeriksa baris pertama dari permintaan HTTP. Jika permintaan adalah `GET / HTTP/1.1`, server akan merespons dengan status `200 OK` dan mengirimkan konten dari `hello.html`. Tetapi jika permintaan berbeda (seperti mengakses halaman yang tidak ada), maka server akan merespons dengan status `404 NOT FOUND` dan mengirimkan konten dari `404.html`. Hal ini memungkinkan server untuk memberikan respons yang sesuai berdasarkan jenis _request_ yang diterima dan meningkatkan fungsionalitas dari server HTTP yang kita buat dibandingkan dengan sebelumnya yang hanya bisa mengirimkan satu jenis respons untuk seluruh permintaan.

Dengan _refactoring_, kode menjadi lebih mudah dibaca dan kode `handle_connection` menjadi lebih ringkas:
```rust
fn handle_connection(mut stream: TcpStream) {
    let buf_reader = BufReader::new(&stream);
    let request_line = buf_reader.lines().next().unwrap().unwrap();

    let (status_line, filename) = if request_line == "GET / HTTP/1.1" {
        ("HTTP/1.1 200 OK", "hello.html")
    } else {
        ("HTTP/1.1 404 NOT FOUND", "404.html")
    };

    let contents = fs::read_to_string(filename).unwrap();
    let length = contents.len();

    let response =
        format!("{status_line}\r\nContent-Length: {length}\r\n\r\n{contents}");

    stream.write_all(response.as_bytes()).unwrap();
}
```

## Commit 4 Reflection
Terdapat beberapa peningkatan dalam fungsi `handle_connection`, terutama dalam cara menangani permintaan HTTP. Salah satu perubahan utama adalah penggunaan ekspresi `match` untuk memproses *request line*, menggantikan struktur `if-else` sebelumnya. Dengan pendekatan ini, server dapat lebih mudah mengelola berbagai rute.

Selain menangani rute utama (`GET / HTTP/1.1`), kini terdapat rute baru, yaitu `GET /sleep HTTP/1.1`. Rute ini secara khusus dirancang untuk menunda pengiriman respons selama 10 detik menggunakan `thread::sleep(Duration::from_secs(10));`, mensimulasikan situasi di mana server membutuhkan waktu lebih lama untuk merespons.

Jika permintaan tidak sesuai dengan rute yang telah ditentukan, klausa `_` pada `match` berfungsi sebagai *catch-all*, yang secara otomatis menangani permintaan yang tidak dikenali dengan mengembalikan respons 404 Not Found dan menampilkan halaman `404.html`.

Setelah menentukan status dan file yang akan dikirim, server membaca kontennya menggunakan `fs::read_to_string(filename).unwrap();`, menghitung panjangnya, dan menyusun respons HTTP yang berisi status line, header Content-Length, serta isi dari file HTML. Terakhir, respons dikirim ke klien melalui `stream.write_all(response.as_bytes()).unwrap();`.

Dengan perubahan ini, server menjadi lebih modular dan mampu menangani berbagai permintaan dengan lebih fleksibel, termasuk rute dengan *delay*.

```rust
 fn handle_connection(mut stream: TcpStream) {
     let buf_reader = BufReader::new(&stream);
     let request_line = buf_reader.lines().next().unwrap().unwrap();
 
     let (status_line, filename) = match &request_line[..] {
         "GET / HTTP/1.1" => ("HTTP/1.1 200 OK", "hello.html"),
         "GET /sleep HTTP/1.1" => {
             thread::sleep(Duration::from_secs(10));
             ("HTTP/1.1 200 OK", "hello.html")
         }
         _ => ("HTTP/1.1 404 NOT FOUND", "404.html"),
     };
 
     let contents = fs::read_to_string(filename).unwrap();
     let length = contents.len();
 
     let response =
         format!("{status_line}\r\nContent-Length: {length}\r\n\r\n{contents}");
 
     stream.write_all(response.as_bytes()).unwrap();
 }
 ```

## Commit 5 Reflection
Terdapat perubahan pada `main.rs` dan terdapat file baru bernama `lib.rs`:
 
```rust
 // main.rs
 use hello::ThreadPool;
 
 fn main() {
     let listener = TcpListener::bind("127.0.0.1:7878").unwrap();
     let pool = ThreadPool::new(4);
 
     for stream in listener.incoming() {
         let stream = stream.unwrap();
 
         pool.execute(|| {
             handle_connection(stream);
         });
     }
 }
 ```
 
 ```rust
 // lib.rs
 use std::{
     sync::{mpsc, Arc, Mutex},
     thread::{self, JoinHandle},
 };
 
 pub struct ThreadPool {
     workers: Vec<Worker>,
     sender: mpsc::Sender<Job>,
 }
 
 type Job = Box<dyn FnOnce() + Send + 'static>;
 
 impl ThreadPool {
     pub fn new(size: usize) -> ThreadPool {
         assert!(size > 0);
 
         let (sender, receiver) = mpsc::channel();
         let receiver = Arc::new(Mutex::new(receiver));
 
         let mut workers = Vec::with_capacity(size);
 
         for id in 0..size {
             workers.push(Worker::new(id, Arc::clone(&receiver)));
         }
 
         ThreadPool { workers, sender }
     }
 
     pub fn execute<F>(&self, f: F)
     where
         F: FnOnce() + Send + 'static,
     {
         let job = Box::new(f);
 
         self.sender.send(job).unwrap();
     }
 }
 
 struct Worker {
     id: usize,
     thread: thread::JoinHandle<()>,
 }
 
 impl Worker {
     fn new(id: usize, receiver: Arc<Mutex<mpsc::Receiver<Job>>>) -> Worker {
         let thread = thread::spawn(move || loop {
             let job = receiver.lock().unwrap().recv().unwrap();
             println!("Worker {id} got a job; executing.");
             job();
         });
 
         Worker { id, thread }
     }
 }
 ```
 
Berdasarkan kedua kode tersebut, server telah diperbarui menjadi model multithreaded dengan implementasi Thread Pool untuk menangani _concurrency_. Pada `main.rs`, perubahan utama adalah pembuatan instance `ThreadPool` dengan 4 thread worker dan penggunaan metode `pool.execute()` untuk membagi penanganan koneksi ke thread pool. Hal ini memungkinkan server untuk menangani beberapa koneksi secara bersamaan tanpa blocking satu sama lain.
 
File `lib.rs` sendiri mengimplementasikan infrastruktur thread pool dengan struktur `ThreadPool` yang berisikan kumpulan `Worker` dan sistem komunikasi berbasis channel. Setiap `Worker` memiliki thread yang berjalan dalam _infinite loop_, menunggu pekerjaan dari channel yang dibagikan menggunakan `Arc<Mutex<>>`. Ketika method `execute()` dipanggil, pekerjaan akan dibungkus dalam `Box` dan dikirim melalui channel ke salah satu worker yang tersedia. Implementasi ini memanfaatkan beberapa fitur dalam Rust seperti ownership untuk menciptakan sistem _concurrency_ yang aman dan efisien. Hal ini dibuktikan dengan tidak adanya delay ketika mengakses page lain ketika membuka rute `/sleep`.

## Commit Bonus Reflection
 
Terdapat perbaikan dan peningkatan pada implementasi Thread Pool, di `main.rs` dan `lib.rs`:
 
```rust
fn main() {
     ...
 
     for stream in listener.incoming() {
         let stream = stream.unwrap();
         ...
     }
}
```
 
```rust
pub struct PoolCreationError {
     reason: String,
}
 
impl fmt::Display for PoolCreationError {
     fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
         write!(
             f,
             "Unable to create ThreadPool: {}", self.reason
         )
     }
}
 
impl fmt::Debug for PoolCreationError {
     fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
         write!(f, "{{ file: {}, line: {} }}", file!(), line!())
     }
}
 
impl ThreadPool {
     pub fn build(size: usize) -> Result<ThreadPool, PoolCreationError> {
         if size <= 0 {
             Err(PoolCreationError {
                 reason: "Thread count must be a positive nonzero integer".to_string()
             })
         } else {
             let (sender, receiver) = mpsc::channel();
             let receiver = Arc::new(Mutex::new(receiver));
 
             let mut workers = Vec::with_capacity(size);
 
             for id in 0..size {
                 workers.push(Worker::new(id, Arc::clone(&receiver)));
             }
 
             Ok(ThreadPool { workers, sender })
         }
     }
     
     ...
}
```
Saya meningkatkan cara kerja Thread Pool dengan menambahkan metode `build` di `ThreadPool`. Sebelumnya, pembuatan thread pool langsung dilakukan dengan `ThreadPool::new(4)`, tetapi sekarang menggunakan `ThreadPool::build(4)`, yang lebih aman karena bisa menangani error jika terjadi masalah saat inisialisasi.

Metode `build` menerima jumlah thread sebagai parameter dan mengembalikan `Result<ThreadPool, PoolCreationError>`. Jika jumlah thread yang diberikan kurang dari atau sama dengan nol, maka akan mengembalikan error `PoolCreationError` dengan alasan yang jelas. Untuk memudahkan debugging, saya juga menambahkan implementasi `Display` dan `Debug` pada `PoolCreationError`, sehingga informasi error bisa ditampilkan dengan lebih informatif.

Dengan perubahan ini, server menjadi lebih stabil dan aman karena kesalahan dalam konfigurasi thread pool bisa dideteksi lebih awal sebelum server benar-benar berjalan.