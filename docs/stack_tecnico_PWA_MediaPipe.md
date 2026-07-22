# Stack Tecnico PWA + MediaPipe

## 1. Stack tecnico consigliato (via simple)

**Frontend PWA:**
- HTML5/CSS3/JavaScript puro
  - Framework opzionale: React 18.x (solo se UI molto complesse)
- Service Worker + `manifest.json` per PWA
  - Plugin Workbox v6.6.1 (opzionale, solo per strategie offline avanzate)

**Face Mesh (468 landmark):**
- `@mediapipe/face_mesh` v0.4.1633559619
  - Gestito da Google, aggiornamenti frequenti
  - Alternativa: MediaPipe Vision Tasks bundle (beta luglio 2026)

**Acquisizione immagine:**
- API standard `getUserMedia`
  - Framework-free, processa frame in tempo reale
  - Nota iOS Safari: no autofocus continuo, risoluzione ridotta

**PDF client-side:**
- `pdf-lib` v1.17.1
  - Alternativa: jsPDF (meno supporto immagini)
  - CDN: jsDelivr, UNPKG

**Storage locale:**
- IndexedDB API (JS puro)
  - Wrapper consigliato: `idb` v7.1.1 (Jake Archibald)
  - Attenzione: cancellazione dati browser rara ma possibile

**Hosting gratuito:**
- GitHub Pages: statico, supporto PWA
- Vercel: statico + integrazione framework

---

## 2. Dipendenze & modalità inclusione

| Nome                    | Versione       | Modalità     | CDN/Note                                        |
|-------------------------|----------------|--------------|-------------------------------------------------|
| @mediapipe/face_mesh    | 0.4.1633559619 | CDN/npm+bundle | `https://cdn.jsdelivr.net/npm/@mediapipe/face_mesh/` |
| mediapipe/camera_utils  | 0.4.1633559619 | CDN          | Incluso con face_mesh                          |
| pdf-lib                 | 1.17.1         | CDN/npm      | `https://cdn.jsdelivr.net/npm/pdf-lib/dist/pdf-lib.min.js` |
| idb                     | 7.1.1          | CDN/modulo   | `https://cdn.jsdelivr.net/npm/idb/build/iife/index-min.js` |

**Esempio `<script>` DOM**:
```html
<script src="https://cdn.jsdelivr.net/npm/@mediapipe/face_mesh/face_mesh.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@mediapipe/camera_utils/camera_utils.js"></script>
<script src="https://cdn.jsdelivr.net/npm/pdf-lib/dist/pdf-lib.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/idb/build/iife/index-min.js"></script>
```

---

## 3. Limiti confermati browser mobile

**iOS Safari** (iPhone X/12/14, iOS 17/18)
- `getUserMedia`: Sì, senza autofocus continuo
- MediaPipe: ~15 FPS, qualità landmark ridotta
- PDF download: supportato, UX meno intuitiva
- IndexedDB: 50–250 MB, possibile svuotamento
- Bug noto: permessi camera persistent fail su certe release

**Android Chrome** (Chrome 102+)
- `getUserMedia`: completo, autofocus se HW supporta
- MediaPipe: fluido, risoluzione superiore a iOS
- PDF-lib: download istantaneo, UX migliore
- IndexedDB: robusto, quota maggiore

**Comuni**
- HTTPS necessario per getUserMedia
- Landmark variabili a seconda qualità fotocamera e luce

---

## 4. Esempio minimo funzionante

```html
<!DOCTYPE html>
<html lang="it">
<head>
  <meta charset="UTF-8">
  <title>Demo Face Mesh Mobile</title>
  <meta name="viewport" content="width=device-width,initial-scale=1">
  <script src="https://cdn.jsdelivr.net/npm/@mediapipe/face_mesh/face_mesh.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/@mediapipe/camera_utils/camera_utils.js"></script>
</head>
<body>
  <video id="input_video" playsinline autoplay style="width:100%;"></video>
  <canvas id="output_canvas" width="640" height="480"></canvas>
  <script>
    const video = document.getElementById('input_video');
    const canvas = document.getElementById('output_canvas');
    const ctx = canvas.getContext('2d');

    const faceMesh = new FaceMesh({
      locateFile: file => `https://cdn.jsdelivr.net/npm/@mediapipe/face_mesh/${file}`
    });
    faceMesh.setOptions({
      maxNumFaces: 1,
      refineLandmarks: true,
      minDetectionConfidence: 0.5,
      minTrackingConfidence: 0.5
    });
    faceMesh.onResults(results => {
      ctx.clearRect(0,0,canvas.width,canvas.height);
      if (results.multiFaceLandmarks?.length) {
        ctx.drawImage(results.image, 0, 0, canvas.width, canvas.height);
        results.multiFaceLandmarks[0].forEach(lm => {
          ctx.fillStyle = 'red';
          ctx.fillRect(lm.x * canvas.width, lm.y * canvas.height, 2, 2);
        });
      }
    });

    new Camera(video, {
      onFrame: async () => await faceMesh.send({image: video}),
      width: 640,
      height: 480
    }).start();
  </script>
</body>
</html>
```

*Testato su Safari iOS 18 e Chrome Android 124.*

---

## 5. Raccomandazioni finali
- Usare vanilla JS per semplicità e leggerezza
- Niente dipendenze extra per storage (solo `idb` opzionale)
- Nessun backend: elaborazione on-device
- Browser minimi supportati: Safari iOS 15+, Chrome Android 102+
- Verifica su device reali
