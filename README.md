/*
Website Karang Taruna 
Karsel 10 - Single-file React component

Instructions:
- This is a single-file React component (default export) using Tailwind classes.
- To run locally:
  1. Create a new React app (e.g. `npm create vite@latest my-karsel --template react`)
  2. Install Tailwind (optional for styling). If Tailwind isn't available, basic styling still works.
  3. Replace App.jsx with this file content and run `npm run dev`.
- Deployment: Build and push to GitHub, then enable GitHub Pages or deploy to Netlify.

Features included:
- Public site: Home, Kegiatan (activities), Jadwal Rapat, Kas (cash ledger), Kontak
- Admin panel (client-side) to add/edit kegiatan, rapat, kas masuk/keluar
- Simple password-protected admin (password stored in localStorage; default: "admin123")
- Data persisted in localStorage so you can edit from admin and it's saved
- Export/Import JSON and CSV export for kas
- Elegant layout using Tailwind utility classes

Notes / limitations:
- This is a client-side demo (no server). For multi-user real admin, you'll need a backend or Firestore.
- Change default password in Admin > Settings after first login.
*/

import React, { useEffect, useState } from "react";

const STORAGE_KEY = "karsel10_data_v1";
const DEFAULT_ADMIN_PASSWORD = "admin123";

const initialData = {
  meta: {
    name: "Karsel 10 - Karang Taruna",
    address:
      "Kedondong Kidul 1 RT 10 RW 06, Kec. Tegalsari, Kel. Tegalsari, Surabaya",
    tagline: "Bersatu, Berkarya, Berbagi",
  },
  kegiatan: [
    {
      id: 1,
      title: "Pembersihan Lingkungan",
      date: "2025-09-07",
      description: "Gotong royong pembersihan lingkungan sekitar RW.",
    },
  ],
  rapat: [
    {
      id: 1,
      title: "Rapat Koordinasi Bulanan",
      date: "2025-09-10",
      time: "19:00",
      notes: "Bahas program kerja bulan depan",
    },
  ],
  kas: [
    // type: 'in' or 'out'
    { id: 1, date: "2025-08-01", desc: "Iuran anggota", type: "in", amount: 200000 },
    { id: 2, date: "2025-08-15", desc: "Beli alat kebersihan", type: "out", amount: 50000 },
  ],
  settings: {
    adminPassword: DEFAULT_ADMIN_PASSWORD,
  },
};

function saveToStorage(data) {
  localStorage.setItem(STORAGE_KEY, JSON.stringify(data));
}
function loadFromStorage() {
  try {
    const raw = localStorage.getItem(STORAGE_KEY);
    if (!raw) return initialData;
    return JSON.parse(raw);
  } catch (e) {
    return initialData;
  }
}

function formatCurrency(n) {
  return new Intl.NumberFormat("id-ID").format(Number(n || 0));
}

export default function App() {
  const [data, setData] = useState(() => loadFromStorage());
  const [route, setRoute] = useState("/home");
  const [isAdmin, setIsAdmin] = useState(false);
  const [showAdminModal, setShowAdminModal] = useState(false);

  useEffect(() => {
    saveToStorage(data);
  }, [data]);

  // compute kas summary
  const totalIn = data.kas.filter((k) => k.type === "in").reduce((s, it) => s + Number(it.amount), 0);
  const totalOut = data.kas.filter((k) => k.type === "out").reduce((s, it) => s + Number(it.amount), 0);
  const balance = totalIn - totalOut;

  // helpers to mutate
  function upsertKegiatan(item) {
    setData((d) => {
      const items = [...d.kegiatan];
      if (item.id) {
        const idx = items.findIndex((i) => i.id === item.id);
        if (idx >= 0) items[idx] = item;
      } else {
        item.id = Date.now();
        items.unshift(item);
      }
      return { ...d, kegiatan: items };
    });
  }

  function deleteKegiatan(id) {
    setData((d) => ({ ...d, kegiatan: d.kegiatan.filter((i) => i.id !== id) }));
  }

  function upsertRapat(item) {
    setData((d) => {
      const items = [...d.rapat];
      if (item.id) {
        const idx = items.findIndex((i) => i.id === item.id);
        if (idx >= 0) items[idx] = item;
      } else {
        item.id = Date.now();
        items.unshift(item);
      }
      return { ...d, rapat: items };
    });
  }
  function deleteRapat(id) {
    setData((d) => ({ ...d, rapat: d.rapat.filter((i) => i.id !== id) }));
  }

  function addKas(entry) {
    setData((d) => ({ ...d, kas: [{ ...entry, id: Date.now() }, ...d.kas] }));
  }
  function deleteKas(id) {
    setData((d) => ({ ...d, kas: d.kas.filter((k) => k.id !== id) }));
  }

  function exportJSON() {
    const blob = new Blob([JSON.stringify(data, null, 2)], { type: "application/json" });
    const url = URL.createObjectURL(blob);
    const a = document.createElement("a");
    a.href = url;
    a.download = "karsel10_data.json";
    a.click();
    URL.revokeObjectURL(url);
  }

  function importJSON(file) {
    const reader = new FileReader();
    reader.onload = (e) => {
      try {
        const parsed = JSON.parse(e.target.result);
        setData(parsed);
        alert("Data berhasil diimpor");
      } catch (err) {
        alert("Gagal membaca file: " + err.message);
      }
    };
    reader.readAsText(file);
  }

  function exportKasCSV() {
    const rows = ["Tanggal,Deskripsi,Tipe,Nominal"];
    data.kas.forEach((k) => rows.push(`${k.date},"${k.desc}",${k.type},${k.amount}`));
    const blob = new Blob([rows.join("\n")], { type: "text/csv" });
    const url = URL.createObjectURL(blob);
    const a = document.createElement("a");
    a.href = url;
    a.download = "kas_karsel10.csv";
    a.click();
    URL.revokeObjectURL(url);
  }

  function doAdminLogin(password) {
    if (password === data.settings.adminPassword) {
      setIsAdmin(true);
      setShowAdminModal(false);
      return true;
    }
    return false;
  }

  function changeAdminPassword(newPass) {
    setData((d) => ({ ...d, settings: { ...d.settings, adminPassword: newPass } }));
    alert("Password admin diubah. Simpan baik-baik.");
  }

  // small components
  const Nav = () => (
    <nav className="bg-white shadow p-4 rounded-2xl mb-6">
      <div className="max-w-5xl mx-auto flex items-center justify-between">
        <div>
          <h1 className="text-xl font-semibold">{data.meta.name}</h1>
          <p className="text-sm text-gray-500">{data.meta.address}</p>
        </div>
        <div className="flex items-center gap-3">
          <button onClick={() => setRoute("/home")} className={`px-3 py-2 rounded-lg ${route==='/home'?'bg-slate-100':''}`}>Home</button>
          <button onClick={() => setRoute("/kegiatan")} className={`px-3 py-2 rounded-lg ${route==='/kegiatan'?'bg-slate-100':''}`}>Kegiatan</button>
          <button onClick={() => setRoute("/rapat")} className={`px-3 py-2 rounded-lg ${route==='/rapat'?'bg-slate-100':''}`}>Rapat</button>
          <button onClick={() => setRoute("/kas")} className={`px-3 py-2 rounded-lg ${route==='/kas'?'bg-slate-100':''}`}>Kas</button>
          <button onClick={() => setRoute("/admin")} className="px-3 py-2 rounded-lg bg-slate-50 border" >Admin</button>
        </div>
      </div>
    </nav>
  );

  const HomeView = () => (
    <div className="max-w-5xl mx-auto">
      <div className="bg-gradient-to-r from-sky-50 to-white p-8 rounded-2xl shadow-lg mb-6">
        <h2 className="text-3xl font-bold mb-2">Selamat datang di {data.meta.name}</h2>
        <p className="text-gray-700 mb-4">{data.meta.tagline}</p>
        <div className="grid grid-cols-1 md:grid-cols-3 gap-4">
          <div className="p-4 rounded-xl bg-white shadow">
            <h3 className="font-semibold">Kegiatan Mendatang</h3>
            <ul className="mt-2 text-sm text-gray-600">
              {data.kegiatan.slice(0,3).map(k => (
                <li key={k.id} className="py-1">{k.date} — {k.title}</li>
              ))}
              {data.kegiatan.length===0 && <li className="text-gray-400">Belum ada kegiatan</li>}
            </ul>
          </div>
          <div className="p-4 rounded-xl bg-white shadow">
            <h3 className="font-semibold">Rapat Berikutnya</h3>
            <ul className="mt-2 text-sm text-gray-600">
              {data.rapat.slice(0,3).map(r => (
                <li key={r.id} className="py-1">{r.date} {r.time?`(${r.time})`:''} — {r.title}</li>
              ))}
              {data.rapat.length===0 && <li className="text-gray-400">Belum ada jadwal rapat</li>}
            </ul>
          </div>
          <div className="p-4 rounded-xl bg-white shadow">
            <h3 className="font-semibold">Saldo Kas</h3>
            <p className="text-2xl font-bold mt-2">Rp {formatCurrency(balance)}</p>
            <p className="text-sm text-gray-500">Masuk: Rp {formatCurrency(totalIn)} • Keluar: Rp {formatCurrency(totalOut)}</p>
          </div>
        </div>
      </div>

      <section className="grid md:grid-cols-2 gap-6">
        <div className="p-4 rounded-xl bg-white shadow">
          <h3 className="font-semibold mb-2">Tentang Kami</h3>
          <p className="text-gray-700 text-sm">{data.meta.name} — alamat: {data.meta.address}. Kami bergerak di bidang sosial dan lingkungan untuk memberdayakan pemuda di wilayah kami.</p>
        </div>
        <div className="p-4 rounded-xl bg-white shadow">
          <h3 className="font-semibold mb-2">Kontak</h3>
          <p className="text-sm text-gray-700">Untuk informasi lebih lanjut, hubungi Ketua Karang Taruna atau gunakan admin untuk menambahkan kontak.</p>
        </div>
      </section>
    </div>
  );

  const KegiatanView = () => (
    <div className="max-w-5xl mx-auto">
      <h2 className="text-2xl font-semibold mb-4">Kegiatan</h2>
      <div className="grid gap-4">
        {data.kegiatan.map(k => (
          <div key={k.id} className="p-4 rounded-xl bg-white shadow flex justify-between items-start">
            <div>
              <div className="text-sm text-gray-500">{k.date}</div>
              <div className="font-semibold">{k.title}</div>
              <div className="text-sm text-gray-600 mt-1">{k.description}</div>
            </div>
            {isAdmin && <div className="flex gap-2">
              <button onClick={() => {
                const title = prompt('Ubah judul', k.title);
                if (!title) return;
                const date = prompt('Ubah tanggal (YYYY-MM-DD)', k.date) || k.date;
                const desc = prompt('Ubah deskripsi', k.description) || k.description;
                upsertKegiatan({ ...k, title, date, description: desc });
              }} className="px-3 py-1 border rounded">Edit</button>
              <button onClick={() => { if(confirm('Hapus kegiatan ini?')) deleteKegiatan(k.id); }} className="px-3 py-1 bg-red-50 border rounded">Hapus</button>
            </div>}
          </div>
        ))}
        {data.kegiatan.length===0 && <div className="p-4 bg-white rounded shadow text-gray-500">Belum ada kegiatan.</div>}
      </div>

    </div>
  );

  const RapatView = () => (
    <div className="max-w-5xl mx-auto">
      <h2 className="text-2xl font-semibold mb-4">Jadwal Rapat</h2>
      <div className="grid gap-4">
        {data.rapat.map(r => (
          <div key={r.id} className="p-4 rounded-xl bg-white shadow flex justify-between items-start">
            <div>
              <div className="text-sm text-gray-500">{r.date} {r.time?`• ${r.time}`:''}</div>
              <div className="font-semibold">{r.title}</div>
              <div className="text-sm text-gray-600 mt-1">{r.notes}</div>
            </div>
            {isAdmin && <div className="flex gap-2">
              <button onClick={() => {
                const title = prompt('Ubah judul', r.title);
                if (!title) return;
                const date = prompt('Ubah tanggal (YYYY-MM-DD)', r.date) || r.date;
                const time = prompt('Ubah waktu (HH:MM)', r.time) || r.time;
                const notes = prompt('Ubah notes', r.notes) || r.notes;
                upsertRapat({ ...r, title, date, time, notes });
              }} className="px-3 py-1 border rounded">Edit</button>
              <button onClick={() => { if(confirm('Hapus rapat ini?')) deleteRapat(r.id); }} className="px-3 py-1 bg-red-50 border rounded">Hapus</button>
            </div>}
          </div>
        ))}
        {data.rapat.length===0 && <div className="p-4 bg-white rounded shadow text-gray-500">Belum ada jadwal rapat.</div>}
      </div>
    </div>
  );

  const KasView = () => {
    const [form, setForm] = useState({ date: '', desc: '', type: 'in', amount: '' });
    return (
      <div className="max-w-5xl mx-auto">
        <h2 className="text-2xl font-semibold mb-4">Buku Kas</h2>
        <div className="grid md:grid-cols-3 gap-4 mb-6">
          <div className="p-4 rounded-xl bg-white shadow">
            <h4 className="font-semibold">Saldo</h4>
            <div className="text-2xl font-bold mt-2">Rp {formatCurrency(balance)}</div>
            <div className="text-sm text-gray-500">Masuk: Rp {formatCurrency(totalIn)} • Keluar: Rp {formatCurrency(totalOut)}</div>
          </div>
          <div className="p-4 rounded-xl bg-white shadow">
            <h4 className="font-semibold">Ekspor / Impor</h4>
            <div className="flex gap-2 mt-2">
              <button onClick={exportKasCSV} className="px-3 py-2 border rounded">Export CSV</button>
              <button onClick={exportJSON} className="px-3 py-2 border rounded">Export JSON</button>
              <label className="px-3 py-2 border rounded cursor-pointer">
                Import JSON
                <input type="file" accept="application/json" onChange={e => importJSON(e.target.files[0])} className="hidden" />
              </label>
            </div>
          </div>
          <div className="p-4 rounded-xl bg-white shadow">
            <h4 className="font-semibold">Tambah Kas (Admin)</h4>
            {isAdmin ? (
              <div className="mt-2">
                <input value={form.date} onChange={e=>setForm({...form,date:e.target.value})} placeholder="YYYY-MM-DD" className="w-full mb-2 p-2 border rounded" />
                <input value={form.desc} onChange={e=>setForm({...form,desc:e.target.value})} placeholder="Deskripsi" className="w-full mb-2 p-2 border rounded" />
                <div className="flex gap-2 mb-2">
                  <select value={form.type} onChange={e=>setForm({...form,type:e.target.value})} className="flex-1 p-2 border rounded">
                    <option value="in">Masuk</option>
                    <option value="out">Keluar</option>
                  </select>
                  <input value={form.amount} onChange={e=>setForm({...form,amount:e.target.value})} placeholder="Nominal" className="w-40 p-2 border rounded" />
                </div>
                <div className="flex gap-2">
                  <button onClick={()=>{ if(!form.date||!form.desc||!form.amount){alert('Isi semua field');return;} addKas(form); setForm({date:'',desc:'',type:'in',amount:''}); }} className="px-3 py-2 bg-slate-50 border rounded">Tambah</button>
                </div>
              </div>
            ) : (
              <div className="text-sm text-gray-500">Login sebagai admin untuk menambah transaksi kas.</div>
            )}
          </div>
        </div>

        <div className="bg-white rounded-xl shadow p-4">
          <table className="w-full text-sm">
            <thead>
              <tr className="text-left text-gray-600">
                <th className="py-2">Tanggal</th>
                <th>Deskripsi</th>
                <th>Tipe</th>
                <th className="text-right">Nominal</th>
                {isAdmin && <th />}
              </tr>
            </thead>
            <tbody>
              {data.kas.map(k => (
                <tr key={k.id} className="border-t">
                  <td className="py-2">{k.date}</td>
                  <td>{k.desc}</td>
                  <td>{k.type==='in'? 'Masuk':'Keluar'}</td>
                  <td className="text-right">Rp {formatCurrency(k.amount)}</td>
                  {isAdmin && <td className="text-right"><button onClick={()=>{ if(confirm('Hapus transaksi?')) deleteKas(k.id); }} className="px-2 py-1 border rounded">Hapus</button></td>}
                </tr>
              ))}
            </tbody>
          </table>
        </div>
      </div>
    );
  };

  const AdminView = () => {
    const [pw, setPw] = useState("");
    const [showSettings, setShowSettings] = useState(false);
    const [metaEdit, setMetaEdit] = useState(data.meta);

    if (!isAdmin) {
      return (
        <div className="max-w-3xl mx-auto p-6 bg-white rounded-xl shadow">
          <h3 className="text-xl font-semibold mb-2">Login Admin</h3>
          <p className="text-sm text-gray-500 mb-4">Masukkan password admin untuk mengedit data.</p>
          <input value={pw} onChange={e=>setPw(e.target.value)} type="password" className="w-full p-2 border rounded mb-2" placeholder="Password" />
          <div className="flex gap-2">
            <button onClick={()=>{ if(doAdminLogin(pw)){ setPw(''); } else alert('Password salah'); }} className="px-3 py-2 bg-slate-50 border rounded">Login</button>
            <button onClick={()=>{ setShowAdminModal(true); }} className="px-3 py-2 border rounded">Buka Bantuan</button>
          </div>
          <div className="mt-4 text-sm text-gray-500">Default password: <strong>{data.settings.adminPassword}</strong> (Silakan ubah setelah login)</div>
        </div>
      );
    }

    return (
      <div className="max-w-5xl mx-auto">
        <div className="bg-white rounded-xl shadow p-4 mb-6 flex justify-between items-center">
          <div>
            <h3 className="text-xl font-semibold">Panel Admin</h3>
            <p className="text-sm text-gray-500">Anda masuk sebagai admin. Semua perubahan disimpan di browser (localStorage).</p>
          </div>
          <div className="flex gap-2">
            <button onClick={()=>{ setIsAdmin(false); }} className="px-3 py-2 border rounded">Logout</button>
            <button onClick={()=>{ exportJSON(); }} className="px-3 py-2 border rounded">Backup JSON</button>
          </div>
        </div>

        <div className="grid md:grid-cols-2 gap-6">
          <div className="p-4 bg-white rounded-xl shadow">
            <h4 className="font-semibold mb-2">Tambah Kegiatan</h4>
            <KegiatanForm onSave={upsertKegiatan} />
          </div>

          <div className="p-4 bg-white rounded-xl shadow">
            <h4 className="font-semibold mb-2">Tambah Rapat</h4>
            <RapatForm onSave={upsertRapat} />
          </div>
        </div>

        <div className="p-4 bg-white rounded-xl shadow mt-6">
          <h4 className="font-semibold mb-2">Pengaturan Situs</h4>
          <div className="grid gap-2">
            <input value={metaEdit.name} onChange={e=>setMetaEdit({...metaEdit,name:e.target.value})} className="p-2 border rounded" />
            <input value={metaEdit.address} onChange={e=>setMetaEdit({...metaEdit,address:e.target.value})} className="p-2 border rounded" />
            <input value={metaEdit.tagline} onChange={e=>setMetaEdit({...metaEdit,tagline:e.target.value})} className="p-2 border rounded" />
            <div className="flex gap-2 mt-2">
              <button onClick={()=>{ setData(d=>({...d, meta: metaEdit})); alert('Meta disimpan'); }} className="px-3 py-2 border rounded">Simpan</button>
              <button onClick={()=>{ setShowSettings(!showSettings); }} className="px-3 py-2 border rounded">Password Admin</button>
            </div>

            {showSettings && (
              <div className="mt-2">
                <input placeholder="Password baru" className="p-2 border rounded w-full mb-2" onKeyDown={e=>{ if(e.key==='Enter'){ changeAdminPassword(e.target.value); } }} />
                <div className="text-sm text-gray-500">Tekan Enter untuk menyimpan password baru.</div>
              </div>
            )}
          </div>
        </div>

        <div className="mt-6 p-4 bg-white rounded-xl shadow">
          <h4 className="font-semibold mb-2">Impor / Ekspor Data</h4>
          <div className="flex gap-2">
            <button onClick={exportJSON} className="px-3 py-2 border rounded">Backup JSON</button>
            <label className="px-3 py-2 border rounded cursor-pointer">Import JSON<input type="file" accept="application/json" onChange={e => importJSON(e.target.files[0])} className="hidden"/></label>
            <button onClick={()=>{ if(confirm('Reset semua data ke default?')){ localStorage.removeItem(STORAGE_KEY); setData(initialData); } }} className="px-3 py-2 border rounded">Reset ke Default</button>
          </div>
        </div>
      </div>
    );
  };

  return (
    <div className="min-h-screen bg-gray-50 p-6">
      <Nav />

      <main className="mb-12">
        {route === '/home' && <HomeView />}
        {route === '/kegiatan' && <KegiatanView />}
        {route === '/rapat' && <RapatView />}
        {route === '/kas' && <KasView />}
        {route === '/admin' && <AdminView />}
      </main>

      <footer className="max-w-5xl mx-auto text-center text-sm text-gray-500">
        © {new Date().getFullYear()} {data.meta.name} • {data.meta.address}
      </footer>

      {/* admin help modal */}
      {showAdminModal && (
        <div className="fixed inset-0 bg-black bg-opacity-30 flex items-center justify-center p-4">
          <div className="bg-white rounded-xl shadow p-6 max-w-lg w-full">
            <h4 className="font-semibold mb-2">Bantuan Admin</h4>
            <p className="text-sm text-gray-600">Masukkan password admin (default tertera pada halaman login). Setelah masuk, Anda dapat menambah kegiatan, rapat, dan kas. Data tersimpan di browser (localStorage).</p>
            <div className="mt-4 text-right">
              <button onClick={()=>setShowAdminModal(false)} className="px-3 py-2 border rounded">Tutup</button>
            </div>
          </div>
        </div>
      )}
    </div>
  );
}

// Small forms used in admin
function KegiatanForm({ onSave }){
  const [title,setTitle]=useState('');
  const [date,setDate]=useState('');
  const [desc,setDesc]=useState('');
  return (
    <div>
      <input value={title} onChange={e=>setTitle(e.target.value)} placeholder="Judul kegiatan" className="w-full p-2 border rounded mb-2" />
      <input value={date} onChange={e=>setDate(e.target.value)} placeholder="YYYY-MM-DD" className="w-full p-2 border rounded mb-2" />
      <textarea value={desc} onChange={e=>setDesc(e.target.value)} placeholder="Deskripsi" className="w-full p-2 border rounded mb-2"></textarea>
      <div className="flex gap-2">
        <button onClick={()=>{ if(!title||!date){alert('Isi judul dan tanggal');return;} onSave({title,date,description:desc}); setTitle(''); setDate(''); setDesc(''); }} className="px-3 py-2 border rounded">Simpan</button>
      </div>
    </div>
  );
}

function RapatForm({ onSave }){
  const [title,setTitle]=useState('');
  const [date,setDate]=useState('');
  const [time,setTime]=useState('');
  const [notes,setNotes]=useState('');
  return (
    <div>
      <input value={title} onChange={e=>setTitle(e.target.value)} placeholder="Judul rapat" className="w-full p-2 border rounded mb-2" />
      <input value={date} onChange={e=>setDate(e.target.value)} placeholder="YYYY-MM-DD" className="w-full p-2 border rounded mb-2" />
      <input value={time} onChange={e=>setTime(e.target.value)} placeholder="HH:MM" className="w-full p-2 border rounded mb-2" />
      <textarea value={notes} onChange={e=>setNotes(e.target.value)} placeholder="Catatan" className="w-full p-2 border rounded mb-2"></textarea>
      <div className="flex gap-2">
        <button onClick={()=>{ if(!title||!date){alert('Isi judul dan tanggal');return;} onSave({title,date,time,notes}); setTitle(''); setDate(''); setTime(''); setNotes(''); }} className="px-3 py-2 border rounded">Simpan</button>
      </div>
    </div>
  );
}
