# Modul Praktikum 5: Middleware & Integrasi *Protected API*

## 1. Tujuan Pembelajaran

Setelah menyelesaikan modul ini, mahasiswa mampu:

* Memahami konsep dan siklus hidup middleware pada Next.js.
* Memproteksi rute (*route protection*) agar halaman tertentu hanya bisa diakses oleh pengguna yang sudah *login*.
* Mencegah pengguna yang sudah *login* untuk kembali ke halaman `/login`.
* Mengambil data dari endpoint API Laravel yang membutuhkan *Bearer Token* menggunakan *Server Components* (`next/headers`).

## 2. Prasyarat Praktikum

* Telah menyelesaikan Modul Praktikum 1 (Token berhasil disimpan di dalam *cookies* browser).
* API Laravel memiliki endpoint yang diproteksi oleh middleware `auth:sanctum` (contoh: `GET /api/user` yang mengembalikan profil user yang sedang *login*).

---

## Bagian 1: Memproteksi Halaman dengan Middleware

Middleware di Next.js berjalan di sisi server *sebelum* sebuah halaman di-*render* atau sebuah *request* diselesaikan. Ini adalah tempat yang tepat untuk mengecek keberadaan token *cookies*.

**Langkah 1.1: Membuat File Middleware**
Buat file baru bernama `middleware.ts` tepat di *root* proyek Anda (sejajar dengan folder `app/`, `package.json`, dan `.env.local`).

Tambahkan kode berikut:

```typescript
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';

export function middleware(request: NextRequest) {
  // 1. Ambil token dari cookies
  const token = request.cookies.get('token')?.value;

  // 2. Cek apakah user sedang mengakses rute /dashboard (dan sub-rutenya)
  const isDashboardRoute = request.nextUrl.pathname.startsWith('/dashboard');
  
  // 3. Cek apakah user sedang mengakses rute /login
  const isLoginRoute = request.nextUrl.pathname === '/login';

  // LOGIKA PROTEKSI:
  
  // Jika mencoba masuk ke /dashboard tapi tidak punya token, lempar ke /login
  if (isDashboardRoute && !token) {
    return NextResponse.redirect(new URL('/login', request.url));
  }

  // Jika mencoba masuk ke /login tapi sudah punya token, langsung ke /dashboard
  if (isLoginRoute && token) {
    return NextResponse.redirect(new URL('/dashboard', request.url));
  }

  // Jika kondisi aman, lanjutkan request
  return NextResponse.next();
}

// Menentukan di rute mana saja middleware ini dijalankan
export const config = {
  matcher: ['/dashboard/:path*', '/login'],
};

```

**Uji Coba Middleware:**
Pastikan server berjalan (`npm run dev`). Hapus *cookies* `token` melalui Inspect Element, lalu coba akses `http://localhost:3000/dashboard`. Anda seharusnya secara otomatis diarahkan ke halaman `/login`.

---

## Bagian 2: Mengonsumsi *Protected API* di Server Component

Setelah rute dilindungi, langkah selanjutnya adalah menampilkan data pengguna di `/dashboard`. Karena kita menggunakan App Router, *data fetching* sangat disarankan dilakukan di *Server Component* demi performa dan keamanan.

**Langkah 2.1: Membuat Halaman Dashboard Terproteksi**
Buka atau buat file di `app/dashboard/page.tsx` dan ubah isinya menjadi:

```tsx
import { cookies } from 'next/headers';
import { redirect } from 'next/navigation';

// Mendefinisikan tipe data balikan dari API
interface UserProfile {
  id: number;
  name: string;
  email: string;
}

// Fungsi asinkron untuk mengambil data dari backend Laravel
async function getUserProfile(): Promise<UserProfile | null> {
  // 1. Ambil token dari cookies di sisi server
  const cookieStore = await cookies();
  const token = cookieStore.get('token')?.value;

  if (!token) return null;

  // 2. Lakukan Fetch ke Endpoint API yang diamankan (auth:sanctum)
  const res = await fetch(`${process.env.NEXT_PUBLIC_API_URL}/user`, {
    headers: {
      'Authorization': `Bearer ${token}`, // Menyisipkan Bearer Token
      'Accept': 'application/json',
    },
    // Karena ini data user spesifik, jangan di-cache agar selalu up-to-date
    cache: 'no-store', 
  });

  if (!res.ok) {
    // Jika token tidak valid / expired, backend (Sanctum) akan mengembalikan 401
    if (res.status === 401) return null; 
    throw new Error('Gagal mengambil data profil');
  }

  return res.json();
}

// Komponen Utama
export default async function DashboardPage() {
  // Jalankan fungsi fetch sebelum merender UI
  const user = await getUserProfile();

  // Jika user bernilai null (karena 401 Unauthenticated), arahkan ke login
  if (!user) {
    redirect('/login');
  }

  return (
    <main className="min-h-screen bg-gray-50 p-8">
      <div className="max-w-4xl mx-auto bg-white rounded-lg shadow p-6">
        <h1 className="text-3xl font-bold text-gray-800 mb-4">
          Selamat Datang, {user.name}!
        </h1>
        
        <div className="mt-4 p-4 bg-blue-50 text-blue-800 rounded-md border border-blue-100">
          <h2 className="font-semibold text-lg mb-2">Informasi Akun</h2>
          <ul className="space-y-1">
            <li><strong>ID:</strong> {user.id}</li>
            <li><strong>Email:</strong> {user.email}</li>
            <li><strong>Status:</strong> Terautentikasi (Server-Side)</li>
          </ul>
        </div>
      </div>
    </main>
  );
}

```

**Penjelasan Kode Bagian 2:**

1. `next/headers`: Menyediakan fungsi `cookies()` untuk membaca token JWT/Sanctum saat halaman diproses di sisi server.
2. `Bearer ${token}`: Format standar Oauth2/Sanctum untuk mengirim kredensial di `Headers`.
3. `cache: 'no-store'`: Mencegah Next.js menyimpan hasil respons API di memori internalnya (*cache*). Ini memastikan bahwa setiap kali *reload*, Next.js benar-benar mengecek token tersebut ke Laravel.

---

## 3. Tugas Evaluasi / Praktikum Lanjutan

1. **Fungsi Logout:** Tambahkan sebuah tombol "Logout" di dalam `DashboardPage`. (Petunjuk: Anda perlu membuat Client Component terpisah, misalnya `<LogoutButton/>`, karena proses penghapusan *cookies* dengan `js-cookie` dan *event handler* `onClick` tidak dapat dilakukan di dalam *Server Component*.)
2. **API Logout:** Saat tombol diklik, pastikan aplikasi mengirimkan permintaan HTTP POST ke `/api/logout` di Laravel untuk menghapus token di *database* (revokasi), sebelum menghapus token di *browser cookies* dan mengarahkan *user* ke `/login`.
