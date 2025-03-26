# Reflection Publisher-1

## 1. Kebutuhan Interface/Trait untuk Subscriber

### Analisis Kode Saat Ini

```rust
#[derive(Debug, Clone, Deserialize, Serialize)]
pub struct Subscriber {
    pub url: String,
    pub name: String,
}
```

### Observasi Pola Observer Klasik

Pada implementasi Observer pattern umumnya:
- Publisher mengelola daftar Observer yang mengimplementasikan interface tertentu
- Interface biasanya memiliki method seperti `update()` untuk notifikasi

### Kenapa Struct Cukup di Kasus Ini?
1. **Uniform Behavior**  
   Semua subscriber hanya membutuhkan penyimpanan data (URL dan nama) tanpa perlu implementasi logika khusus. Tidak ada kebutuhan untuk polymorphism.
   
2. **Simplifikasi Desain**  
   Tidak ada method yang perlu di-enforce melalui trait (e.g., tidak ada `send_notification()` yang harus diimplementasikan berbeda tiap subscriber).
   
3. **Kebutuhan Spesifik BambangShop**  
   Subscriber bertindak sebagai passive receiver dimana notifikasi dikirim melalui HTTP request eksternal. Mekanisme pengiriman sudah dihandle oleh sistem secara terpusat.
   
4. **Perbandingan dengan OOP**  
   Dalam konteks Rust yang lebih functional, trait akan diperlukan jika kita perlu:
   ```rust
   trait Subscriber {
       fn notify(&self, notification: Notification);
   }
   ```
   Tapi pattern ini tidak terlihat di implementasi saat ini.

### Kesimpulan
Penggunaan struct cukup memadai untuk kasus BambangShop karena tidak ada variasi perilaku antar subscriber. Interface/trait akan menjadi over-engineering.

---

## 2. Vec vs DashMap untuk Uniqueness Constraints

### Struktur Data Saat Ini

```rust
static ref SUBSCRIBERS: DashMap<String, DashMap<String, Subscriber>>;
```

### Analisis Persyaratan
- `id` di Program dan `url` di Subscriber harus unique
- Operasi yang sering dilakukan: add, delete, lookup by URL

### Perbandingan Performa

| Operasi          | Vec   | DashMap |
|------------------|-------|---------|
| Insert           | O(1)* | O(1)    |
| Delete by URL    | O(n)  | O(1)    |
| Lookup           | O(n)  | O(1)    |
| Concurrency      | Not thread-safe | Thread-safe |

*Asumsi push ke akhir list, tapi tidak menjamin uniqueness

### Contoh Kasus Nyata
Untuk 1000 subscriber:
- Delete dengan Vec: Perlu iterasi 1000 elemen untuk cari URL
- Delete dengan DashMap: Langsung access bucket hash

### Keuntungan DashMap
1. **Enforcement of Uniqueness**  
   Mekanisme hash key mencegah duplikasi URL secara native.
   
2. **Concurrent Access**  
   Bisa handle multiple requests secara paralel tanpa race condition.
   
3. **Efisiensi Spasial**  
   Tidak perlu menyimpan data duplikat untuk validasi.

### Risiko Jika Pakai Vec

```rust
// Contoh implementasi tidak efisien
fn add_subscriber(subscribers: &mut Vec<Subscriber>, new: Subscriber) {
    if !subscribers.iter().any(|s| s.url == new.url) {
        subscribers.push(new);
    }
}
```

Kode ini akan memiliki kompleksitas O(n) untuk setiap operasi insert.

### Kesimpulan
DashMap adalah pilihan tepat untuk menjamin uniqueness dan efisiensi operasi CRUD dalam lingkungan konkuren.

---

## 3. DashMap vs Singleton Pattern

### Implementasi Saat Ini

```rust
lazy_static! {
    static ref SUBSCRIBERS: DashMap<...> = DashMap::new();
}
```

### Karakteristik DashMap
1. **Thread-Safe by Design**  
   Menggunakan Shard Locking (banyak Mutex) untuk konkurensi tinggi.
   
2. **API Atomic**  
   Operasi seperti insert, remove, dan get sudah atomic.

### Jika Pakai Singleton Manual

Contoh implementasi alternatif:

```rust
struct SubscriberRepo {
    inner: RwLock<HashMap<String, HashMap<String, Subscriber>>>,
}

impl SubscriberRepo {
    fn add(&self, product_type: &str, subscriber: Subscriber) {
        let mut map = self.inner.write().unwrap(); // Blocking write
        // ... logic
    }
}
```

### Masalah yang Muncul
1. **Lock Contention**  
   RwLock akan menjadi bottleneck saat high traffic.
   
2. **Manual Sync Complexity**  
   Harus manage lock lifetime secara manual.
   
3. **Error Prone**  
   Risiko deadlock jika lupa melepas lock.
   
4. **Granularity**  
   Lock seluruh map vs DashMap yang lock per shard.

### Benchmark Performa
Dalam skenario 100 concurrent requests:
- **DashMap**: Lock hanya pada shard yang relevan.
- **Singleton RwLock**: Lock global seluruh map.

### Kenapa DashMap Lebih Unggul
1. **Zero-Cost Abstraction**  
   Tidak perlu implementasi thread safety manual.
   
2. **Scalability**  
   Performa tetap stabil saat data bertambah.
   
3. **Kompatibilitas dengan Ekosistem Rust**  
   DashMap adalah solusi standar untuk concurrent HashMap.

### Alternatif Lain
- **Mutex<HashMap>**: Lebih buruk dari DashMap.
- **AHashMap**: Tidak thread-safe.
- **Redis**: Overkill untuk in-memory storage.

### Kesimpulan
Penggunaan DashMap adalah pilihan optimal yang menyeimbangkan safety, performa, dan kemudahan implementasi. Singleton pattern dengan RwLock akan menambah kompleksitas tanpa keuntungan yang signifikan.

# BambangShop Publisher App
Tutorial and Example for Advanced Programming 2024 - Faculty of Computer Science, Universitas Indonesia

---

## About this Project
In this repository, we have provided you a REST (REpresentational State Transfer) API project using Rocket web framework.

This project consists of four modules:
1.  `controller`: this module contains handler functions used to receive request and send responses.
    In Model-View-Controller (MVC) pattern, this is the Controller part.
2.  `model`: this module contains structs that serve as data containers.
    In MVC pattern, this is the Model part.
3.  `service`: this module contains structs with business logic methods.
    In MVC pattern, this is also the Model part.
4.  `repository`: this module contains structs that serve as databases and methods to access the databases.
    You can use methods of the struct to get list of objects, or operating an object (create, read, update, delete).

This repository provides a basic functionality that makes BambangShop work: ability to create, read, and delete `Product`s.
This repository already contains a functioning `Product` model, repository, service, and controllers that you can try right away.

As this is an Observer Design Pattern tutorial repository, you need to implement another feature: `Notification`.
This feature will notify creation, promotion, and deletion of a product, to external subscribers that are interested of a certain product type.
The subscribers are another Rocket instances, so the notification will be sent using HTTP POST request to each subscriber's `receive notification` address.

## API Documentations

You can download the Postman Collection JSON here: https://ristek.link/AdvProgWeek7Postman

After you download the Postman Collection, you can try the endpoints inside "BambangShop Publisher" folder.
This Postman collection also contains endpoints that you need to implement later on (the `Notification` feature).

Postman is an installable client that you can use to test web endpoints using HTTP request.
You can also make automated functional testing scripts for REST API projects using this client.
You can install Postman via this website: https://www.postman.com/downloads/

## How to Run in Development Environment
1.  Set up environment variables first by creating `.env` file.
    Here is the example of `.env` file:
    ```bash
    APP_INSTANCE_ROOT_URL="http://localhost:8000"
    ```
    Here are the details of each environment variable:
    | variable              | type   | description                                                |
    |-----------------------|--------|------------------------------------------------------------|
    | APP_INSTANCE_ROOT_URL | string | URL address where this publisher instance can be accessed. |
2.  Use `cargo run` to run this app.
    (You might want to use `cargo check` if you only need to verify your work without running the app.)

## Mandatory Checklists (Publisher)
-   [ ] Clone https://gitlab.com/ichlaffterlalu/bambangshop to a new repository.
-   **STAGE 1: Implement models and repositories**
    -   [ ] Commit: `Create Subscriber model struct.`
    -   [ ] Commit: `Create Notification model struct.`
    -   [ ] Commit: `Create Subscriber database and Subscriber repository struct skeleton.`
    -   [ ] Commit: `Implement add function in Subscriber repository.`
    -   [ ] Commit: `Implement list_all function in Subscriber repository.`
    -   [ ] Commit: `Implement delete function in Subscriber repository.`
    -   [ ] Write answers of your learning module's "Reflection Publisher-1" questions in this README.
-   **STAGE 2: Implement services and controllers**
    -   [ ] Commit: `Create Notification service struct skeleton.`
    -   [ ] Commit: `Implement subscribe function in Notification service.`
    -   [ ] Commit: `Implement subscribe function in Notification controller.`
    -   [ ] Commit: `Implement unsubscribe function in Notification service.`
    -   [ ] Commit: `Implement unsubscribe function in Notification controller.`
    -   [ ] Write answers of your learning module's "Reflection Publisher-2" questions in this README.
-   **STAGE 3: Implement notification mechanism**
    -   [ ] Commit: `Implement update method in Subscriber model to send notification HTTP requests.`
    -   [ ] Commit: `Implement notify function in Notification service to notify each Subscriber.`
    -   [ ] Commit: `Implement publish function in Program service and Program controller.`
    -   [ ] Commit: `Edit Product service methods to call notify after create/delete.`
    -   [ ] Write answers of your learning module's "Reflection Publisher-3" questions in this README.

## Your Reflections
This is the place for you to write reflections:

### Mandatory (Publisher) Reflections

#### Reflection Publisher-1

#### Reflection Publisher-2

#### Reflection Publisher-3
