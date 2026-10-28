<!DOCTYPE html>
<html lang="id">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>Input Packing Online</title>
<style>
  *{box-sizing:border-box}
  body{font-family:system-ui,Arial,sans-serif;background:#f1f5f4;margin:0;padding:16px}
  .card{max-width:560px;margin:auto;background:#fff;border-radius:14px;padding:22px;box-shadow:0 2px 12px rgba(0,0,0,.08)}
  h2{margin:0 0 16px;color:#0f7b4a}
  label{display:block;font-size:13px;font-weight:600;margin:12px 0 4px;color:#333}
  input,select{width:100%;padding:11px;border:1px solid #cfd8d5;border-radius:8px;font-size:15px}
  input:focus,select:focus{outline:2px solid #0f7b4a33;border-color:#0f7b4a}
  .row{display:grid;grid-template-columns:1fr 1fr;gap:12px}
  .fotos{display:grid;grid-template-columns:1fr 1fr;gap:12px}
  .foto{border:2px dashed #b7c7c1;border-radius:10px;padding:8px;text-align:center;font-size:12px;color:#666}
  .foto img{width:100%;height:100px;object-fit:cover;border-radius:6px;margin-top:6px;display:none}
  .foto input{padding:6px;font-size:12px;border:none}
  button{width:100%;margin-top:20px;padding:13px;border:0;border-radius:10px;background:#0f7b4a;color:#fff;font-size:16px;font-weight:600;cursor:pointer}
  button:disabled{background:#9bb;cursor:wait}
  #msg{margin-top:14px;padding:10px;border-radius:8px;display:none;font-size:14px}
  .ok{background:#e3f6ec;color:#0a6b3f}.err{background:#fde8e8;color:#b32121}
</style>
</head>
<body>
<div class="card">
  <h2>📦 Input Packing Online</h2>
  <form id="f">
    <div class="row">
      <div><label>Tanggal</label><input type="date" id="tanggal" required></div>
      <div><label>Nama Packing</label>
        <select id="nama" required>
          <option value="">-- Pilih --</option>
          <option>ADI</option><option>AMIR</option><option>FERRY</option>
          <option>TANTO</option><option>ARIFIN</option>
        </select>
      </div>
    </div>

    <label>No. Invoice</label>
    <input id="invoice" placeholder="INV123456789" required autocomplete="off">

    <div class="row">
      <div><label>Pengiriman</label>
        <select id="pengiriman">
          <option value="">-- Pilih --</option>
          <option>JNE</option><option>SPX</option><option>J&T</option>
          <option>SICEPAT</option><option>ANTERAJA</option><option>GOSEND</option>
        </select>
      </div>
      <div><label>Jumlah Koli</label><input type="number" id="koli" min="1" value="1"></div>
    </div>

    <label>Foto (maks 4)</label>
    <div class="fotos">
      <div class="foto">Foto 1<input type="file" accept="image/*" capture="environment" data-i="0"><img></div>
      <div class="foto">Foto 2<input type="file" accept="image/*" capture="environment" data-i="1"><img></div>
      <div class="foto">Foto 3<input type="file" accept="image/*" capture="environment" data-i="2"><img></div>
      <div class="foto">Foto 4<input type="file" accept="image/*" capture="environment" data-i="3"><img></div>
    </div>

    <button id="btn" type="submit">Simpan</button>
    <div id="msg"></div>
  </form>
</div>

<script>
// ====== KONFIGURASI ======
const API_URL = 'GANTI_DENGAN_WEB_APP_URL_/exec';
const SECRET  = 'rahasia123'; // samakan dengan Code.gs

const fotos = [null, null, null, null];
const $ = id => document.getElementById(id);

$('tanggal').valueAsDate = new Date();
// ingat nama packing terakhir
$('nama').value = localStorage.getItem('nama') || '';

// Kompres foto agar upload cepat
function compress(file, max = 1280, q = 0.7) {
  return new Promise(res => {
    const img = new Image();
    const url = URL.createObjectURL(file);
    img.onload = () => {
      const s = Math.min(1, max / Math.max(img.width, img.height));
      const c = document.createElement('canvas');
      c.width = img.width * s; c.height = img.height * s;
      c.getContext('2d').drawImage(img, 0, 0, c.width, c.height);
      const dataUrl = c.toDataURL('image/jpeg', q);
      URL.revokeObjectURL(url);
      res({ data: dataUrl.split(',')[1], type: 'image/jpeg', preview: dataUrl });
    };
    img.src = url;
  });
}

document.querySelectorAll('.foto input').forEach(inp => {
  inp.addEventListener('change', async e => {
    const i = +e.target.dataset.i, file = e.target.files[0];
    const img = e.target.parentElement.querySelector('img');
    if (!file) { fotos[i] = null; img.style.display = 'none'; return; }
    const r = await compress(file);
    fotos[i] = { data: r.data, type: r.type };
    img.src = r.preview; img.style.display = 'block';
  });
});

function show(text, ok) {
  const m = $('msg');
  m.textContent = text; m.className = ok ? 'ok' : 'err'; m.style.display = 'block';
}

$('f').addEventListener('submit', async e => {
  e.preventDefault();
  const btn = $('btn');
  btn.disabled = true; btn.textContent = 'Menyimpan...';
  $('msg').style.display = 'none';

  const payload = {
    key: SECRET,
    tanggal: $('tanggal').value,
    nama: $('nama').value,
    invoice: $('invoice').value.trim(),
    pengiriman: $('pengiriman').value,
    koli: $('koli').value,
    fotos: fotos
  };

  try {
    // text/plain supaya tidak kena preflight CORS
    const resp = await fetch(API_URL, {
      method: 'POST',
      headers: { 'Content-Type': 'text/plain;charset=utf-8' },
      body: JSON.stringify(payload)
    });
    const r = await resp.json();
    if (r.status !== 'success') throw new Error(r.message);

    localStorage.setItem('nama', payload.nama);
    show('✅ Tersimpan! ID: ' + r.id + ' (baris ' + r.row + ')', true);
    $('invoice').value = ''; $('koli').value = 1; $('pengiriman').value = '';
    document.querySelectorAll('.foto input').forEach(i => i.value = '');
    document.querySelectorAll('.foto img').forEach(i => i.style.display = 'none');
    fotos.fill(null);
    $('invoice').focus();
  } catch (err) {
    show('❌ Gagal: ' + err.message, false);
  } finally {
    btn.disabled = false; btn.textContent = 'Simpan';
  }
});
</script>
</body>
</html>
