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