<!DOCTYPE html>
<html lang="id">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0"/>
  <title>Aplikasi Kas Anggota</title>
  <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.2/dist/css/bootstrap.min.css" rel="stylesheet">
</head>
<body class="bg-light">
  <div class="container py-5">
    <div class="card shadow rounded-4 p-4">
      <h2 class="mb-4 text-center">Aplikasi Kas Anggota</h2>

      <!-- Form Input -->
      <form id="kasForm" class="mb-4">
        <div class="mb-3">
          <label for="nama" class="form-label">Nama Anggota</label>
          <input type="text" class="form-control" id="nama" required />
        </div>
        <div class="mb-3">
          <label for="jumlah" class="form-label">Jumlah Simpanan (Rp)</label>
          <input type="number" class="form-control" id="jumlah" required />
        </div>
        <div class="mb-3">
          <label for="bukti" class="form-label">Link Bukti (opsional)</label>
          <input type="url" class="form-control" id="bukti" placeholder="https://drive.google.com/..." />
        </div>
        <button type="submit" class="btn btn-primary w-100">Simpan</button>
      </form>

      <!-- Riwayat -->
      <h4 class="mb-3">Riwayat Simpanan</h4>
      <ul id="riwayat" class="list-group"></ul>
    </div>
  </div>

  <script>
    const scriptURL = "PASTE_URL_WEB_APP_MU_DISINI"; // Ganti dengan URL Web App Google Apps Script

    // Kirim data ke Spreadsheet
    document.getElementById("kasForm").addEventListener("submit", function(e) {
      e.preventDefault();
      const nama = document.getElementById("nama").value;
      const jumlah = document.getElementById("jumlah").value;
      const bukti = document.getElementById("bukti").value;

      fetch(scriptURL, {
        method: 'POST',
        body: new URLSearchParams({ nama, jumlah, bukti })
      })
      .then(() => {
        alert("✅ Data berhasil disimpan!");
        document.getElementById("kasForm").reset();
        loadRiwayat();
      })
      .catch(err => alert("❌ Gagal menyimpan: " + err));
    });

    // Ambil riwayat dari Spreadsheet
   function loadRiwayat() {
  fetch(scriptURL + "?action=read")
    .then(res => res.json())
    .then(data => {
      const riwayat = document.getElementById("riwayat");
      riwayat.innerHTML = "";

      data.reverse().forEach((row, index) => {
        const item = document.createElement("div");
        item.className = "accordion-item";

        item.innerHTML = `
          <h2 class="accordion-header" id="heading${index}">
            <button class="accordion-button collapsed" type="button" data-bs-toggle="collapse" data-bs-target="#collapse${index}" aria-expanded="false" aria-controls="collapse${index}">
              ${row.nama} - Rp${row.jumlah} (${row.tanggal})
            </button>
          </h2>
          <div id="collapse${index}" class="accordion-collapse collapse" aria-labelledby="heading${index}" data-bs-parent="#riwayat">
            <div class="accordion-body">
              <p><strong>Nama:</strong> ${row.nama}</p>
              <p><strong>Jumlah:</strong> Rp${row.jumlah}</p>
              <p><strong>Tanggal:</strong> ${row.tanggal}</p>
              ${row.bukti ? `<p><a href="${row.bukti}" target="_blank" class="btn btn-sm btn-outline-primary">🔗 Lihat Bukti</a></p>` : "<p>Tidak ada bukti.</p>"}
            </div>
          </div>
        `;
        riwayat.appendChild(item);
      });
    });
}
    
    // Load saat pertama kali
    loadRiwayat();
  </script>
</body>
</html>
