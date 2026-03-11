---
title: Mengenal Konsep Design System: Clean Architecture
published: 2026-03-11
tags: [Software Architecture, Blogging, Backend, System Design]
category: System Design
draft: false
---


# Clean Architecture: Cara Biar Project Tidak Jadi “Spaghetti Code”

Sering pusing karena project kamu kelihatan tidak efisien, susah di-maintain, dan semua logic tercampur di satu tempat?

File controller penuh dengan query database, business logic bercampur dengan validasi, bahkan kadang helper random ikut nimbrung di situ juga.

Kalau projectnya masih kecil dan kita kerjakan sendiri, mungkin masih bisa diatasi. Kamu masih ingat semua alur kodenya di kepala.

Namun bagaimana jika project tersebut dikerjakan bersama tim?

Mulai muncul pertanyaan seperti:

* “Logic ini ada di mana ya?”
* “Kalau ubah fitur ini nanti efeknya ke mana saja?”
* “Kenapa function ini dipanggil dari tiga tempat berbeda?”

Akhirnya kita sebagai developer harus bolak-balik bertanya ke orang lain yang handle kode tsb hanya untuk memahami struktur project.

Dan di sinilah konsep **Clean Architecture** mulai terasa penting.

Clean Architecture membantu kita membuat sistem yang **terstruktur, mudah dipahami, dan lebih mudah di-maintain dalam jangka panjang.**

---

# Apa Itu Clean Architecture?

Clean Architecture adalah konsep arsitektur software yang diperkenalkan oleh Robert C. Martin (Uncle Bob).

Tujuan utamanya sederhana:

> Memisahkan tanggung jawab dalam aplikasi agar setiap bagian memiliki peran yang jelas.

Dengan pemisahan ini kita mendapatkan beberapa keuntungan:

* Code lebih mudah dipahami
* Mudah melakukan testing
* Tidak bergantung pada framework
* Lebih mudah melakukan perubahan di masa depan

---

# Masalah Umum Tanpa Arsitektur yang Jelas

Kita coba lihat contoh struktur project yang sering terjadi.

```
controllers/
    userController.js
    orderController.js

models/
    userModel.js
    orderModel.js

routes/
```

Masalahnya?

Sering kali controller berisi:

* Query database
* Validasi
* Business logic
* Response API

Contohnya:

```js
async function createUser(req, res) {
   const user = await db.query("INSERT INTO users ...")

   if(user.age < 18){
       return res.json({error: "Too young"})
   }

   sendEmail(user.email)

   return res.json(user)
}
```

Semua logic tercampur di satu tempat.

Akibatnya:

* Susah di test
* Susah di reuse
* Susah di maintain

Semakin besar project, semakin terasa dampaknya.

---

# Konsep Dasar Clean Architecture

Clean Architecture biasanya dibagi menjadi beberapa layer:

```
Entities
Use Cases
Interface Adapters
Frameworks & Drivers
```

Atau jika disederhanakan:

```
Domain
Usecase
Repository
Handler / Controller
```

Mari kita bahas satu per satu.

---

# 1. Entities (Domain)

Ini adalah inti dari aplikasi.

Berisi:

* Struct / Model
* Business rules utama

Contoh:

```go
type User struct {
    ID    string
    Name  string
    Email string
}
```

Layer ini **tidak boleh bergantung pada database, framework, atau API**.

Tujuannya supaya domain logic tetap bersih.

---

# 2. Usecase (Business Logic)

Layer ini berisi logic aplikasi.

Contohnya:

* Create user
* Login
* Calculate price
* Process order

Contoh sederhana:

```go
type UserUsecase struct {
    repo UserRepository
}

func (u *UserUsecase) CreateUser(user User) error {
    return u.repo.Create(user)
}
```

Usecase hanya tahu **apa yang harus dilakukan**, bukan **bagaimana implementasinya**.

---

# 3. Repository

Repository adalah layer yang bertugas berkomunikasi dengan database.

Contoh:

```go
type UserRepository interface {
    Create(user User) error
    FindByEmail(email string) (*User, error)
}
```

Implementasinya bisa berbeda:

* PostgreSQL
* MySQL
* MongoDB
* bahkan API

Usecase tidak perlu tahu.

---

# 4. Handler / Controller

Layer ini menangani request dari luar.

Contoh:

* HTTP API
* CLI
* gRPC

Contoh sederhana:

```go
func (h *UserHandler) CreateUser(c *fiber.Ctx) error {
    var req CreateUserRequest

    if err := c.BodyParser(&req); err != nil {
        return err
    }

    err := h.usecase.CreateUser(req.ToDomain())

    return c.JSON("success")
}
```

Controller hanya bertugas:

* menerima request
* memanggil usecase
* mengembalikan response

---

# Prinsip Penting Clean Architecture

Ada satu aturan utama:

> **Dependency harus mengarah ke dalam (inward).**

Artinya:

```
Controller -> Usecase -> Domain
```

Bukan sebaliknya.

Domain tidak boleh tahu tentang:

* database
* framework
* HTTP
* library eksternal

Dengan begitu, domain tetap stabil walaupun teknologi berubah.

---

# Contoh Struktur Project

Contoh struktur project:

```
internal/

    domain/
        user.go

    repository/
        user_repository.go
        user_repository_impl.go

    usecase/
        user_usecase.go

    handler/
        user_handler.go
```

Struktur seperti ini membuat code lebih mudah dipahami oleh developer lain.

---

# Kapan Clean Architecture Dibutuhkan?

Clean Architecture sangat berguna ketika:

* Project mulai membesar
* Banyak developer dalam satu project
* Sistem memiliki banyak business logic
* Project diharapkan bertahan lama

Namun untuk project kecil atau prototype, arsitektur sederhana mungkin sudah cukup.

---

# Penutup

Clean Architecture bukan sekadar membuat folder yang banyak.

Tujuannya adalah **memisahkan tanggung jawab dalam sistem** agar kode tetap rapi, scalable, dan mudah dipahami.

Dengan menerapkan arsitektur yang baik sejak awal, kita bisa menghindari masalah klasik seperti:

* spaghetti code
* sulit testing
* sulit maintain

Dan yang paling penting, developer lain tidak perlu lagi kebingungan mencari logic di dalam project, 

Mungkin sampai disini saja yang bisa dapat saya bagikan, dan semoga yang saya bagikan dapat bermanfaat kedepanya :)

Sekian dan Nuhunn ~
