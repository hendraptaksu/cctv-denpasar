# CCTV ATCS Denpasar — Viewer

Program sederhana untuk melihat semua CCTV dari
https://atcs.denpasarkota.go.id/streaming dalam satu halaman grid
(164 kamera per snapshot terakhir, sumber: API publik situs tersebut).

## Cara pakai (termudah)

1. Buka `index.html` langsung di browser (double-click),
   atau lebih disarankan lewat server lokal agar fallback `cctvs.json` bisa dibaca:

   ```sh
   cd cctv-oc
   python3 -m http.server 8000
   # buka http://localhost:8000
   ```

2. Ketik di kolom pencarian untuk memfilter simpang/lokasi.
3. Klik **▶ Putar** pada kartu untuk memulai stream (HLS).
4. Klik **⛶ Fullscreen** untuk tampilan besar, **🗺 Peta** untuk lokasi di Google Maps,
   **🔗 Copy URL** untuk menyalin URL `.m3u8` (bisa dibuka di VLC/MPV).
5. **📡 Cek status** menandai kamera READY/OFFLINE. **▶ Putar yang tampil**
   memutar maks. 12 stream pertama yang tampil (membatasi beban CPU/jaringan).

## File

- `index.html` — program utama (satu file, tanpa build, tanpa install).
- `cctvs.json` — snapshot daftar kamera; dipakai otomatis kalau API live tidak bisa diakses.
- `spbu.json` — 37 titik SPBU di Denpasar (sumber: OpenStreetMap via Nominatim,
  sudah diaudit per titik via kategori OSM; 3 titik non-SPBU — Instalasi Aspal,
  depot Pedungan, kantor Jl. Sugianyar — dibuang);
  nama generik diperkaya nama jalan dari alamat). Bisa diedit manual — format:
  `{"id":"SPBU-01","name":"...","lat":-8.68,"lon":115.18,"addr":"..."}`.
  Ketelitian setara data OSM; koreksi dipersilakan bila ada titik yang bukan SPBU umum.

## Fitur peta & filter SPBU

- Tab **🗺 Peta CCTV + SPBU**: peta Leaflet/OpenStreetMap berisi marker 📷 (CCTV)
  dan ⛽ (SPBU). Klik marker CCTV untuk memutar stream; klik marker SPBU untuk
  melihat daftar CCTV dalam 1 km + tombol "Tampilkan di grid".
- Filter SPBU di header: pilih SPBU tertentu + radius (0,5–3 km) untuk menampilkan
  hanya CCTV di sekitarnya (diurutkan dari terdekat, lingkaran radius digambar di peta),
  atau centang **CCTV dekat SPBU saja** untuk semua SPBU sekaligus.
- Setiap kartu grid menampilkan jarak ke SPBU terdekat, mis. "⛽ 0,6 km ke SPBU ...".
- Tombol **🗺 Peta** pada kartu melompat ke peta dan fokus ke kamera itu.
  Regenerasi:

  ```sh
  curl -s -H 'x-client-id: a194e6ae-d4dd-4b62-a0ac-388922f09303' \
       -H 'x-client-secret: f430fde38a031fb657a2a7d6f84644a9aed767a4c22314d4b7c565648acc2396' \
       'https://atcs.denpasarkota.go.id/api/v3/pv/ldevice?orderBy=id_lokasi&sortedBy=asc' \
  | python3 -c "
  import json,sys
  d=json.load(sys.stdin); out=[]
  for loc in d['data']:
      for dev in loc.get('tb_device_lokasi',[]):
          url=(dev.get('url_proxy_hls') or '').strip()
          if not url: continue
          if not url.endswith('/'): url+='/'
          out.append({'id_lokasi':loc['id_lokasi'],'nama_lokasi':loc.get('nama_lokasi'),
              'ket_lokasi':loc.get('ket_lokasi'),'lat':loc.get('lat_lokasi'),'lon':loc.get('lon_lokasi'),
              'nama':dev.get('nama'),'nama_alias':dev.get('nama_alias'),
              'stream_page':url,'hls':url+'index.m3u8',
              'poster':('https://atcs.denpasarkota.go.id/poster/'+dev['poster']) if dev.get('poster') else None})
  json.dump(out, open('cctvs.json','w'), ensure_ascii=False, indent=1)
  print(len(out), 'kamera')"
  ```

## Catatan teknis

- Daftar diambil dari `GET https://atcs.denpasarkota.go.id/api/v3/pv/ldevice`
  dengan header `x-client-id` / `x-client-secret` (sama seperti yang dipakai situs aslinya;
  CORS server terbuka `*` sehingga bisa di-fetch langsung dari browser).
- Setiap kamera di-stream via HLS: `{url_proxy_hls}/index.m3u8`, diputar dengan
  `hls.js` (Chrome/Firefox/Edge) atau HLS native (Safari).
