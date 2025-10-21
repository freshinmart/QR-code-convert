<!doctype html>

<html lang="id">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width,initial-scale=1">
  <title>Pengubah Teks → QR Code (Single HTML)</title>
  <style>
    :root{--bg:#f7fafc;--card:#fff;--accent:#2563eb}
    body{font-family:Inter,ui-sans-serif,system-ui,-apple-system,"Segoe UI",Roboto,"Helvetica Neue",Arial; background:var(--bg); margin:0; display:flex; align-items:center; justify-content:center; min-height:100vh;}
    .card{background:var(--card); padding:20px; border-radius:12px; box-shadow:0 6px 20px rgba(2,6,23,0.08); width:360px;}
    h1{font-size:18px; margin:0 0 10px}
    label{display:block; font-size:13px; margin-top:8px}
    textarea,input[type=text],select{width:100%; padding:8px 10px; margin-top:6px; border-radius:8px; border:1px solid #e6edf3; font-size:14px; box-sizing:border-box}
    .row{display:flex; gap:8px; margin-top:12px}
    button{background:var(--accent); color:white; border:none; padding:8px 12px; border-radius:8px; cursor:pointer; font-weight:600}
    button:active{transform:translateY(1px)}
    .preview{display:flex; align-items:center; justify-content:center; margin-top:14px}
    .meta{font-size:12px; color:#6b7280; margin-top:8px;}
    .actions{display:flex; gap:8px; margin-top:10px}
    .small{font-size:12px; padding:6px 8px}
  </style>
</head>
<body>
  <div class="card">
    <h1>Ubah teks menjadi QR code</h1>
    <label for="text">Teks atau URL</label>
    <textarea id="text" rows="3" placeholder="Isi teks atau URL di sini">Halo dari ChatGPT!</textarea><label for="size">Ukuran (px)</label>
<select id="size">
  <option value="150">150 × 150</option>
  <option value="200" selected>200 × 200</option>
  <option value="300">300 × 300</option>
  <option value="400">400 × 400</option>
</select>

<div class="row">
  <button id="generate">Buat QR</button>
  <button id="download" class="small">Download PNG</button>
  <button id="copy" class="small">Salin data URI</button>
</div>

<div class="preview" id="preview">
  <!-- preview muncul di sini -->
  <img id="qrimg" alt="QR preview" style="display:none; border-radius:8px; border:1px solid #e6edf3">
</div>

<div class="meta">Metode: <strong id="method">Google Chart API (fallback)</strong></div>
<div style="margin-top:10px; font-size:12px; color:#374151">Catatan: file ini adalah satu berkas HTML. Jika Anda ingin menghasilkan QR offline tanpa memanggil layanan eksternal, beri tahu saya dan saya akan sertakan skrip generator QR kecil yang tertanam langsung (akan lebih panjang).</div>

  </div>  <script>
    // Elemen
    const textEl = document.getElementById('text');
    const sizeEl = document.getElementById('size');
    const genBtn = document.getElementById('generate');
    const qrimg = document.getElementById('qrimg');
    const preview = document.getElementById('preview');
    const downloadBtn = document.getElementById('download');
    const copyBtn = document.getElementById('copy');
    const methodLabel = document.getElementById('method');

    // Pilihan: metode default menggunakan Google Chart API yang sederhana dan cepat.
    // Jika Anda mendesain untuk penggunaan offline atau tanpa panggilan eksternal, minta versi dengan generator JS tertanam.

    function makeGoogleQR(text, size){
      const encoded = encodeURIComponent(text);
      // chart.googleapis.com is a simple way to create a QR image
      return `https://chart.googleapis.com/chart?chs=${size}x${size}&cht=qr&chl=${encoded}&choe=UTF-8`;
    }

    // Menghasilkan QR dan menampilkannya. Jika ingin menukar metode, bisa diubah di sini.
    genBtn.addEventListener('click', ()=>{
      const t = textEl.value.trim();
      if(!t){ alert('Masukkan teks atau URL terlebih dahulu.'); return; }
      const size = parseInt(sizeEl.value,10) || 200;

      // Gunakan Google Chart API
      const src = makeGoogleQR(t, size);
      qrimg.src = src;
      qrimg.width = size;
      qrimg.height = size;
      qrimg.style.display = 'block';
      methodLabel.textContent = 'Google Chart API';
    });

    // Download — konversi image (yang di-serve dari google) menjadi blob lalu download
    downloadBtn.addEventListener('click', async ()=>{
      if(!qrimg.src){ alert('Belum ada QR. Klik "Buat QR" lebih dulu.'); return; }
      try{
        const resp = await fetch(qrimg.src);
        if(!resp.ok) throw new Error('Gagal mengambil gambar.');
        const blob = await resp.blob();
        const url = URL.createObjectURL(blob);
        const a = document.createElement('a');
        a.href = url;
        a.download = 'qrcode.png';
        document.body.appendChild(a);
        a.click();
        a.remove();
        URL.revokeObjectURL(url);
      }catch(err){
        alert('Download gagal. Kemungkinan masalah CORS atau koneksi. Error: '+err.message);
      }
    });

    // Copy data URI — ambil canvas untuk menghindari masalah CORS; kalau gagal, fallback ke menyalin URL
    copyBtn.addEventListener('click', async ()=>{
      if(!qrimg.src){ alert('Belum ada QR. Klik "Buat QR" lebih dulu.'); return; }
      try{
        // coba ambil dan ubah ke dataURL via canvas
        const resp = await fetch(qrimg.src);
        const blob = await resp.blob();
        const imgBitmap = await createImageBitmap(blob);
        const canvas = document.createElement('canvas');
        canvas.width = imgBitmap.width; canvas.height = imgBitmap.height;
        const ctx = canvas.getContext('2d');
        ctx.drawImage(imgBitmap,0,0);
        const dataUrl = canvas.toDataURL('image/png');
        await navigator.clipboard.writeText(dataUrl);
        alert('Data URI QR disalin ke clipboard. Anda bisa menempelkannya langsung.');
      }catch(err){
        try{
          await navigator.clipboard.writeText(qrimg.src);
          alert('Gagal membuat data URI karena CORS. Saya menyalin URL gambar sebagai gantinya.');
        }catch(e){
          alert('Gagal menyalin ke clipboard: '+e.message);
        }
      }
    });

    // Shortcuts: generate on Ctrl+Enter
    textEl.addEventListener('keydown', (e)=>{ if(e.key === 'Enter' && (e.ctrlKey||e.metaKey)){ genBtn.click(); } });

    // Inisialisasi: buat preview awal
    genBtn.click();
  </script></body>
</html>
