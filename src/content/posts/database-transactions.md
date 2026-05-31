---
title: "Database Transaction Apakah Itu Penting?"
published: 2026-03-11
tags: [Software Architecture, Blogging, Backend, Database Concept]
category: 'Database'
draft: false
---

### Transaction: Handle Proses Yang Mengharuskan Semua Query Harus Berhasil

Dalam pembuatan sebuah service dalam pemrograman, terkadang ada beberapa proses database yang harus berhasil secara bersamaan.

Contohnya seperti transfer saldo, mengurangi stok barang, atau proses booking kursi. Jika salah satu proses gagal di tengah jalan, data bisa menjadi tidak sesuai, misalnya saldo berkurang tetapi tujuan tidak menerima uang, atau stok barang menjadi minus, dan itu akan membuat data menjadi inkonsisten dan berantakan,

Pada blog kali ini aku bakal bahas gimana caranya agar kesalahan tersebut tidak terulang lagi dan bisa di tanggulangi, langsung aja kita lanjut ke next section,

## Yang pertama itu ada konsep ACID ( Atomicity, Consistency, Isolation, Durability )

Konsep acid ini merupakan sekumpulan prinsip pada database transactions yang di gunakan untuk data agar tetap aman dan konsisten ketika sebuah proses logika dijalankan, sekarang kita coba lihat flow dibawah ini 

<div align="center">
  <img src="/contents/FlowAcidSample.png" alt="Flow Acid Sample" width="600"/>
</div>

di flow diatas kita sudah mendapatkan gimana gambaran flow dari acid itu sendiri di jalankan, jadi blog selesai, eits, belom dong

jadi gini dalam diagram diatas kita bisa lihat, kalo salah satu proses gagal, database dapat membatalkan seluruh perubahan agar data tetap aman dan tidak menjadi corrupt.

Lalu sebenarnya bagaimana cara kerja masing-masing konsep ACID tersebut?
Mari kita bahas satu per satu.

# 1. Atomicity

Atomicity itu bisa kita artikan sebagai satu kesatuan sebuah proses,

Artinya, jika ada beberapa query yang dijalankan dalam satu flow, maka semua query tersebut harus berhasil sampai akhir, dan jika salah satu query gagal, maka semua query yang sudah terjadi sebelumnya akan di kembalikan ke value sebelumnya.

<div align="center">
  <img src="/contents/petrik.jpg" alt="Petrik Kok Bisa Bang" width="600"/>
</div>

Woiya bisa dong, di beberapa database seperti PostgreSQL, itu sangat memungkikan untuk melakukan proses ini, biasanya kita menggunakan bahasa sql prosedural,

Misalnya kita punya tabel wallets:

```sql
CREATE TABLE wallets (
  id SERIAL PRIMARY KEY,
  user_id INT NOT NULL,
  balance NUMERIC NOT NULL DEFAULT 0
);
```

Contoh datanya:

```sql
INSERT INTO wallets (user_id, balance)
VALUES
  (1, 100000),
  (2, 50000);
```

Lalu kita buat function untuk transfer saldo:

```sql
CREATE OR REPLACE FUNCTION transfer_balance(
  sender_id INT,
  receiver_id INT,
  amount NUMERIC
)
RETURNS VOID AS $$
BEGIN
  UPDATE wallets
  SET balance = balance - amount
  WHERE user_id = sender_id;

  UPDATE wallets
  SET balance = balance + amount
  WHERE user_id = receiver_id;
END;
$$ LANGUAGE plpgsql;
```

Function tersebut bisa dijalankan seperti ini:

```sql
SELECT transfer_balance(1, 2, 25000);
```

Di PostgreSQL, function PL/pgSQL berjalan di dalam transaction. Jadi jika salah satu query di dalam function gagal, seluruh proses akan dibatalkan.

Namun, function di atas masih punya masalah. Kita belum memastikan saldo pengirim cukup.

Versi yang lebih aman bisa seperti ini:

```sql
CREATE OR REPLACE FUNCTION transfer_balance(
  sender_id INT,
  receiver_id INT,
  amount NUMERIC
)
RETURNS VOID AS $$
DECLARE
  sender_balance NUMERIC;
BEGIN
  SELECT balance
  INTO sender_balance
  FROM wallets
  WHERE user_id = sender_id;

  IF sender_balance < amount THEN
    RAISE EXCEPTION 'Saldo tidak cukup';
  END IF;

  UPDATE wallets
  SET balance = balance - amount
  WHERE user_id = sender_id;

  UPDATE wallets
  SET balance = balance + amount
  WHERE user_id = receiver_id;
END;
$$ LANGUAGE plpgsql;
```

Dengan function ini, proses transfer hanya akan dilanjutkan jika saldo pengirim mencukupi.

Jika saldo tidak cukup, PostgreSQL akan menjalankan RAISE EXCEPTION, lalu seluruh proses dalam transaction akan gagal dan perubahan tidak akan disimpan.

Inilah contoh penerapan Atomicity:

Semua proses transfer berhasil, atau tidak ada perubahan sama sekali.

# 2. Consistency

Setelah sebelumnya kita membahas tentang Atomicity, sekarang kita lanjut ke konsep berikutnya yaitu Consistency.

Consistency berarti data harus tetap valid sebelum dan sesudah transaction dijalankan.

Misalnya:

- Saldo nya gakboleh jadi minus.
- Stock ga boleh jadi minus.
- User Gaboleh duplikat,
- Segala sesuatu tindakan yang mengubah value data menjadi anomali seperti sosok monyet ijo banyumas

Nah simple nya gini deh kita coba lanjut ke kasus permasalahanya

Bayangkan kita punya tabel products seperti ini:

```sql
CREATE TABLE products (
  id SERIAL PRIMARY KEY,
  product_name TEXT NOT NULL,
  stock INT NOT NULL
);
```

Contoh datanya:

```sql
INSERT INTO products (product_name, stock)
VALUES
  ('Keyboard Mechanical', 5);
```

Sekarang kita coba membuat function untuk mengurangi stok barang ketika user checkout.

```sql
CREATE OR REPLACE FUNCTION reduce_stock(
  product_id INT,
  total INT
)
RETURNS VOID AS $$
BEGIN
  UPDATE products
  SET stock = stock - total
  WHERE id = product_id;
END;
$$ LANGUAGE plpgsql;
```

Kelihatannya aman...

Tapi coba bayangkan jika ada request seperti ini:

```sql
SELECT reduce_stock(1, 10);
```

Padahal stok awal cuma 5.

Akibatnya?

stock = -5

<div align="center">
  <img src="/contents/pura-pura-gak-liat.png" alt="Github Actions Screenshot" width="600"/>
</div>

Nah, data seperti ini sudah tidak nguwawor rek ini udah ngelanggar konsep flow bisnis dan bisa di parani user nanti wkwkwkwkwk


## Terus Gimana Bang Caranya? 

Untuk menjaga data tetap konsisten, kita perlu melakukan validasi sebelum perubahan dilakukan seperti contoh dibawah ini:

```sql
CREATE OR REPLACE FUNCTION reduce_stock(
  product_id INT,
  total INT
)
RETURNS VOID AS $$
DECLARE
  current_stock INT;
BEGIN
  SELECT stock
  INTO current_stock
  FROM products
  WHERE id = product_id;

  IF current_stock < total THEN
    RAISE EXCEPTION 'Stock tidak mencukupi';
  END IF;

  UPDATE products
  SET stock = stock - total
  WHERE id = product_id;
END;
$$ LANGUAGE plpgsql;
```

Sekarang flow transaksi akan menolak proses checkout jika stok tidak mencukupi guys, Dengan begitu:

stok tidak akan minus
data tetap valid
aturan bisnis tetap terjaga

Inilah inti dari konsep Consistency untuj membantu memastikan bahwa setiap transaction tetap mengikuti aturan yang sudah ditentukan oleh rencana flow bisnis,

# 3. Isolation

Isolation ini berarti beberapa transaction yang berjalan secara bersamaan tidak boleh saling mengganggu yang lain.

Hal ini penting karena dalam aplikasi nyata, sering kali ada banyak user yang mengakses data yang sama di waktu bersamaan dan bisa menyebabkan yang namanya **Race Condition**.

## Contoh Kasus

Bayangkan ada 2 user kita sebut aja mas amba dan mas rusdi yang ingin booking kursi yang sama secara bersamaan.

Misalnya kita punya tabel:

```sql
CREATE TABLE seats (
  id SERIAL PRIMARY KEY,
  seat_number TEXT,
  is_booked BOOLEAN DEFAULT FALSE
);
```

Lalu mas amba dan mas rusdi menjalankan proses booking hampir di waktu yang sama.

Tanpa Isolation yang baik, keduanya bisa saja membaca data:

```txt
is_booked = false
```

Akibatnya:

* Mas Amba berhasil booking
* Mas Rusdi juga berhasil booking

<div align="center">
  <img src="/contents/informasi-palsu.jpg" alt="Github Actions Screenshot" width="600"/>
</div>

## Solusi Dengan Locking

PostgreSQL menyediakan `FOR UPDATE` untuk mengunci data selama transaction berjalan.

```sql
CREATE OR REPLACE FUNCTION book_seat(
  seat_id INT
)
RETURNS VOID AS $$
DECLARE
  seat_status BOOLEAN;
BEGIN
  SELECT is_booked
  INTO seat_status
  FROM seats
  WHERE id = seat_id
  FOR UPDATE;

  IF seat_status THEN
    RAISE EXCEPTION 'Kursi sudah dibooking';
  END IF;

  UPDATE seats
  SET is_booked = TRUE
  WHERE id = seat_id;
END;
$$ LANGUAGE plpgsql;
```

Dengan `FOR UPDATE`, PostgreSQL akan mengunci baris data tersebut sampai transaction selesai.

Jadi ketika user lain mencoba booking kursi yang sama, mereka harus menunggu terlebih dahulu.

Inilah inti dari **Isolation**:

Transaction yang berjalan bersamaan harus tetap aman dan tidak saling merusak data.

# 4. Durability

Konsep terakhir adalah **Durability**.

Durability bisa kita artikan kalo data yang sudah berhasil di-*commit* atau di proses oleh sistem, maka tidak boleh hilang, bahkan jika terjadi crash, listrik mati, atau server restart.

Simpelnya gini kalo transaction sudah dianggap berhasil, maka data harus tetap tersimpan.

## Contoh Kasus

Bayangkan kalo Mas Gatot berhasil melakukan transfer saldo.

```sql
SELECT transfer_balance(1, 2, 25000);
```

Setelah proses selesai, PostgreSQL akan melakukan:

```sql
COMMIT;
```

Artinya seluruh perubahan sudah dianggap valid dan resmi disimpan ke database.

Walaupun setelah itu:

* server tiba-tiba mati
* aplikasi crash
* listrik padam

data transfer tersebut tetap tidak boleh hilang.

## Bagaimana PostgreSQL Bisa Ngeproses ini?

PostgreSQL menggunakan mekanisme bernama **WAL (Write-Ahead Logging)**.

Sebelum data benar-benar ditulis ke database, PostgreSQL akan lebih dulu mencatat perubahan ke log.

Jadi ketika server crash, PostgreSQL masih bisa melakukan recovery menggunakan log tersebut.

Karena itulah database modern tetap bisa menjaga data tetap aman meskipun terjadi gangguan sistem.

Inilah inti dari **Durability**:

Data yang sudah berhasil disimpan harus tetap aman dan tidak boleh hilang

## Penutup

Oke barusan kita sudah membahas secara ringkas Konsep Transaction ACID, Gimana caranya agar data tetap aman dan konsisten ketika sebuah proses logika dijalankan, semoga blog ini bisa bermanfaat, bagi kalian yang suka dengan blog ini, bisa follow atmint loh ya, see you in next blog kawann.
