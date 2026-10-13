# Modul Praktikum 4: Inisialisasi Proyek Next.js & Integrasi API Autentikasi (Login)

## 1. Tujuan Pembelajaran

Setelah menyelesaikan modul ini, mahasiswa diharapkan mampu:

* Melakukan instalasi dan konfigurasi awal proyek Next.js menggunakan App Router.
* Mengelola dependensi pihak ketiga untuk manajemen *cookies* di sisi klien.
* Membangun form login menggunakan antarmuka berbasis *Client Component* (`"use client"`).
* Melakukan HTTP POST request ke API Laravel untuk autentikasi.
* Menyimpan token autentikasi (Bearer Token) ke dalam *browser cookies* secara aman untuk digunakan pada *request* selanjutnya.

## 2. Prasyarat Praktikum

* **API Backend (Laravel) sudah berjalan** di lokal (contoh: `http://laraweb.test`) atau server *staging*.
* API harus memiliki endpoint `POST /api/login` yang menerima payload `email` dan `password`, serta mengembalikan data berupa `token` (atau `access_token`).
* Node.js versi 18.x atau lebih baru sudah terinstal.

---

## Bagian 1: Persiapan Proyek Next.js

**Langkah 1.1: Membuat Proyek Baru**

Buka terminal dan jalankan perintah berikut untuk menginisialisasi proyek Next.js baru. Kita akan menggunakan Tailwind CSS untuk styling.

```bash
npx create-next-app@latest nextweb

```

Saat muncul *prompt*, pilih konfigurasi berikut:

* Would you like to use **TypeScript**? Yes
* Would you like to use **ESLint**? Yes
* Would you like to use **Tailwind CSS**? Yes
* Would you like to use **`src/` directory**? No (kita gunakan root `app/`)
* Would you like to use **App Router**? Yes
* Would you like to customize the default **import alias**? No

Masuk ke direktori proyek yang baru dibuat:

```bash
cd nextweb

```

**Langkah 1.2: Instalasi Library Tambahan**

Untuk menyimpan token autentikasi yang diberikan oleh API Laravel, kita akan menggunakan *cookies*. *Cookies* lebih direkomendasikan daripada `localStorage` di Next.js karena *cookies* dapat dibaca oleh *Server Components* dan *Middleware* di modul selanjutnya.

Jalankan perintah instalasi library `js-cookie`:

```bash
npm install js-cookie
npm install -D @types/js-cookie

```

**Langkah 1.3: Konfigurasi Environment URL API**

Buat sebuah file baru bernama `.env.local` di *root* folder (sejajar dengan `package.json`). Tambahkan URL dasar API Laravel Anda:

```env
NEXT_PUBLIC_API_URL=http://laraweb.test/api

```

---

## Bagian 2: Membangun Antarmuka & Logika Login

Dalam Next.js App Router, komponen secara *default* berjalan di server (*Server Components*). Karena form login membutuhkan interaksi pengguna dan state (seperti mengetik di form), kita harus mendeklarasikan komponen ini sebagai *Client Component* dengan direktif `"use client"`.

**Langkah 2.1: Membuat Halaman Login**

Buat folder dan file baru di `app/login/page.tsx`, lalu tambahkan kode berikut:

```tsx
"use client";

import { useState } from "react";
import { useRouter } from "next/navigation";
import Cookies from "js-cookie";

export default function LoginPage() {
  const router = useRouter();
  
  // State untuk menyimpan inputan user
  const [email, setEmail] = useState("");
  const [password, setPassword] = useState("");
  
  // State untuk handling loading dan error
  const [loading, setLoading] = useState(false);
  const [errorMsg, setErrorMsg] = useState("");

  const handleLogin = async (e: React.FormEvent) => {
    e.preventDefault();
    setLoading(true);
    setErrorMsg("");

    try {
      const response = await fetch(`${process.env.NEXT_PUBLIC_API_URL}/login`, {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
          "Accept": "application/json",
        },
        body: JSON.stringify({ email, password }),
      });

      const data = await response.json();

      if (!response.ok) {
        // Menangkap pesan error dari API Laravel (misal: kredensial salah)
        throw new Error(data.message || "Gagal melakukan login");
      }

      // Asumsi respons API memiliki properti: { token: "1|xyz..." }
      // Simpan token ke dalam cookies. 
      // Opsi `expires: 1` berarti cookie akan kadaluarsa dalam 1 hari.
      Cookies.set("token", data.token, { expires: 1 });

      // Redirect ke halaman dashboard (akan kita buat di modul selanjutnya)
      router.push("/dashboard");

    } catch (error: any) {
      setErrorMsg(error.message);
    } finally {
      setLoading(false);
    }
  };

  return (
    <main className="min-h-screen flex items-center justify-center bg-gray-100">
      <div className="bg-white p-8 rounded-xl shadow-md w-full max-w-md">
        <h1 className="text-2xl font-bold text-center mb-6 text-gray-800">
          Login Sistem
        </h1>

        {errorMsg && (
          <div className="bg-red-100 text-red-700 p-3 rounded-md mb-4 text-sm text-center">
            {errorMsg}
          </div>
        )}

        <form onSubmit={handleLogin} className="space-y-4">
          <div>
            <label className="block text-sm font-medium text-gray-700 mb-1">
              Email
            </label>
            <input
              type="email"
              value={email}
              onChange={(e) => setEmail(e.target.value)}
              className="w-full border border-gray-300 rounded-md px-4 py-2 focus:outline-none focus:ring-2 focus:ring-blue-500"
              placeholder="admin@example.com"
              required
            />
          </div>

          <div>
            <label className="block text-sm font-medium text-gray-700 mb-1">
              Password
            </label>
            <input
              type="password"
              value={password}
              onChange={(e) => setPassword(e.target.value)}
              className="w-full border border-gray-300 rounded-md px-4 py-2 focus:outline-none focus:ring-2 focus:ring-blue-500"
              placeholder="••••••••"
              required
            />
          </div>

          <button
            type="submit"
            disabled={loading}
            className={`w-full py-2 px-4 rounded-md text-white font-semibold transition ${
              loading ? "bg-blue-300 cursor-not-allowed" : "bg-blue-600 hover:bg-blue-700"
            }`}
          >
            {loading ? "Memproses..." : "Masuk"}
          </button>
        </form>
      </div>
    </main>
  );
}

```

**Langkah 2.2: Menjalankan Aplikasi**

Buka terminal dan jalankan server pengembangan Next.js:

```bash
npm run dev

```

Buka browser dan akses URL `http://localhost:3000/login`.
Pastikan server API Laravel Anda juga sedang menyala. Cobalah masukkan kredensial yang valid dari API Anda, buka tab *Application > Cookies* di *Developer Tools* browser, dan pastikan `token` berhasil tersimpan saat login berhasil.

---

## 3. Tugas Praktikum

Setelah selesai praktikum, kerjakan tugas berikut:

1. **Validasi Form Client-Side**: Modifikasi kode di atas agar tombol "Masuk" tidak bisa diklik jika format email tidak valid (tambahkan regex atau pengecekan *string* sederhana) atau jika password kurang dari 6 karakter.
2. **Halaman Dummy Dashboard**: Buat file `app/dashboard/page.tsx` sederhana yang menampilkan tulisan "Selamat Datang di Dashboard". Halaman ini adalah target dari fungsi `router.push('/dashboard')` setelah login sukses.
