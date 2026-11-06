# Modul Praktikum 4: Implementasi CRUD Data Mahasiswa

## 1. Tujuan Pembelajaran

Setelah menyelesaikan modul ini, mahasiswa mampu:

* Membuat *RESTful Resource API* yang diproteksi pada Laravel.
* Menampilkan daftar data (Read) menggunakan *Server Components* di Next.js App Router.
* Membangun form interaktif (Create & Update) menggunakan *Client Components*.
* Mengimplementasikan aksi penghapusan data (Delete) dan menyegarkan (*refresh*) antarmuka secara dinamis.
* Mengelola *Dynamic Routes* (contoh: `/mahasiswa/[id]/edit`) pada Next.js.

## 1. Implementasi Frontend (Next.js)

Kita akan membuat struktur folder berikut di dalam proyek Next.js Anda:

```text
app/
 └── dashboard/
      └── mahasiswa/
           ├── page.tsx               (Read - Tabel Data)
           ├── create/
           │    └── page.tsx          (Create - Form Tambah)
           └── components/
                └── DeleteButton.tsx  (Delete - Tombol Hapus)

```

### Bagian 1.1: Read - Menampilkan Data Mahasiswa

Halaman ini adalah *Server Component* yang akan melakukan *fetch* data ke endpoint `/api/mahasiswa` dengan menyertakan token dari *cookies*.

Buka/buat file `app/dashboard/mahasiswa/page.tsx`:

```tsx
import Link from "next/link";
import { cookies } from "next/headers";
import DeleteButton from "./components/DeleteButton";

// Tipe data Mahasiswa
export interface Mahasiswa {
  id: number;
  nim: string;
  nama: string;
}

// Fetching data di Server
async function getMahasiswas(): Promise<Mahasiswa[]> {
  const cookieStore = await cookies();
  const token = cookieStore.get("token")?.value;

  const res = await fetch(`${process.env.NEXT_PUBLIC_API_URL}/mahasiswa`, {
    headers: {
      Authorization: `Bearer ${token}`,
      Accept: "application/json",
    },
    cache: "no-store", // Jangan di-cache agar data CRUD selalu realtime
  });

  if (!res.ok) throw new Error("Gagal mengambil data mahasiswa");
  
  const json = await res.json();
  return json.data; // Sesuaikan dengan struktur response Laravel (misal Resource API)
}

export default async function MahasiswaPage() {
  const mahasiswas = await getMahasiswas();

  return (
    <div className="bg-white p-6 rounded-lg shadow-sm border border-gray-200">
      <div className="flex justify-between items-center mb-6">
        <h1 className="text-2xl font-bold text-gray-800">Data Mahasiswa</h1>
        <Link 
          href="/dashboard/mahasiswa/create" 
          className="bg-blue-600 text-white px-4 py-2 rounded-md hover:bg-blue-700"
        >
          + Tambah Data
        </Link>
      </div>

      <div className="overflow-x-auto">
        <table className="min-w-full divide-y divide-gray-200">
          <thead className="bg-gray-50">
            <tr>
              <th className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">NIM</th>
              <th className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Nama</th>
              <th className="px-6 py-3 text-center text-xs font-medium text-gray-500 uppercase tracking-wider">Aksi</th>
            </tr>
          </thead>
          <tbody className="bg-white divide-y divide-gray-200">
            {mahasiswas.map((mhs) => (
              <tr key={mhs.id}>
                <td className="px-6 py-4 whitespace-nowrap text-sm text-gray-900">{mhs.nim}</td>
                <td className="px-6 py-4 whitespace-nowrap text-sm text-gray-900">{mhs.nama}</td>
                <td className="px-6 py-4 whitespace-nowrap text-sm font-medium text-center space-x-2">
                  <Link href={`/dashboard/mahasiswa/${mhs.id}/edit`} className="text-indigo-600 hover:text-indigo-900">
                    Edit
                  </Link>
                  <DeleteButton id={mhs.id} />
                </td>
              </tr>
            ))}
          </tbody>
        </table>
      </div>
    </div>
  );
}

```

### Bagian 3.2: Delete - Komponen Tombol Hapus

Karena tombol hapus memerlukan aksi `onClick`, kita pisahkan menjadi *Client Component*.
Buat file `app/dashboard/mahasiswa/components/DeleteButton.tsx`:

```tsx
"use client";

import { useRouter } from "next/navigation";
import Cookies from "js-cookie";

export default function DeleteButton({ id }: { id: number }) {
  const router = useRouter();

  const handleDelete = async () => {
    if (!confirm("Apakah Anda yakin ingin menghapus data ini?")) return;

    const token = Cookies.get("token");

    try {
      const res = await fetch(`${process.env.NEXT_PUBLIC_API_URL}/mahasiswa/${id}`, {
        method: "DELETE",
        headers: {
          Authorization: `Bearer ${token}`,
          Accept: "application/json",
        },
      });

      if (res.ok) {
        // Refresh router untuk memuat ulang data tabel di Server Component
        router.refresh();
      } else {
        alert("Gagal menghapus data");
      }
    } catch (error) {
      console.error(error);
    }
  };

  return (
    <button 
      onClick={handleDelete} 
      className="text-red-600 hover:text-red-900 cursor-pointer"
    >
      Hapus
    </button>
  );
}

```

### Bagian 3.3: Create - Form Tambah Data

Buat file untuk halaman form di `app/dashboard/mahasiswa/create/page.tsx`:

```tsx
"use client";

import { useState } from "react";
import { useRouter } from "next/navigation";
import Cookies from "js-cookie";
import Link from "next/link";

export default function CreateMahasiswaPage() {
  const router = useRouter();
  const [formData, setFormData] = useState({ nim: "", nama: "" });
  const [loading, setLoading] = useState(false);

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    setLoading(true);
    const token = Cookies.get("token");

    try {
      const res = await fetch(`${process.env.NEXT_PUBLIC_API_URL}/mahasiswa`, {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
          Authorization: `Bearer ${token}`,
          Accept: "application/json",
        },
        body: JSON.stringify(formData),
      });

      if (res.ok) {
        router.push("/dashboard/mahasiswa"); // Kembali ke tabel
        router.refresh(); // Segarkan data tabel
      } else {
        const errorData = await res.json();
        alert(errorData.message || "Terjadi kesalahan validasi");
      }
    } catch (error) {
      console.error(error);
    } finally {
      setLoading(false);
    }
  };

  return (
    <div className="max-w-2xl mx-auto bg-white p-6 rounded-lg shadow-sm border border-gray-200">
      <h1 className="text-2xl font-bold text-gray-800 mb-6">Tambah Mahasiswa</h1>
      
      <form onSubmit={handleSubmit} className="space-y-4">
        <div>
          <label className="block text-sm font-medium text-gray-700">NIM</label>
          <input
            type="text"
            required
            value={formData.nim}
            onChange={(e) => setFormData({ ...formData, nim: e.target.value })}
            className="mt-1 w-full border border-gray-300 rounded-md p-2 focus:ring-blue-500 focus:border-blue-500"
          />
        </div>
        <div>
          <label className="block text-sm font-medium text-gray-700">Nama Lengkap</label>
          <input
            type="text"
            required
            value={formData.nama}
            onChange={(e) => setFormData({ ...formData, nama: e.target.value })}
            className="mt-1 w-full border border-gray-300 rounded-md p-2"
          />
        </div>
        
        <div className="flex space-x-3 pt-4">
          <button
            type="submit"
            disabled={loading}
            className="bg-blue-600 text-white px-4 py-2 rounded-md hover:bg-blue-700 disabled:bg-blue-300"
          >
            {loading ? "Menyimpan..." : "Simpan Data"}
          </button>
          <Link 
            href="/dashboard/mahasiswa"
            className="bg-gray-200 text-gray-700 px-4 py-2 rounded-md hover:bg-gray-300"
          >
            Batal
          </Link>
        </div>
      </form>
    </div>
  );
}

```

---

## 4. Tugas Evaluasi / Praktikum Lanjutan

1. **Tampilkan Data (Read):** Tampilkan data-data lain yang sesuai dengan yang ada di Rest API.
2. **Fitur Edit (Update):** Berdasarkan konsep *Create* di atas, buatlah halaman `app/dashboard/mahasiswa/[id]/edit/page.tsx`. Halaman ini harus melakukan `GET /api/mahasiswa/{id}` terlebih dahulu menggunakan `useEffect` (atau langsung dari Server Component dan mem- *pass* prop ke Client Form) untuk mengisi nilai awal form, lalu melakukan request `PUT /api/mahasiswa/{id}` saat di- *submit*.
3. **Global Loading State:** Tambahkan file `loading.tsx` di dalam folder `app/dashboard/mahasiswa/` untuk memberikan indikator visual (seperti *spinner* atau teks "Memuat data...") saat Next.js sedang menunggu balasan dari API Laravel di belakang layar.
