# Architettura PWA - PAC (Posture Analysis Capture)

## Panoramica

PAC è una Progressive Web App (PWA) offline-first per l'acquisizione e analisi della postura del capo. L'architettura è progettata per:
- **Funzionamento offline** dopo il primo accesso (con compromesso: primo carico richiede internet)
- **Nessuna sincronizzazione cloud**: tutti i dati restano sul dispositivo
- **Installabilità**: aggiunta a home screen come app nativa
- **Semplicità tecnologica**: stack "simple" senza framework pesanti

---

## Stack Tecnologico

### Core
- **HTML5**: struttura semantica
- **CSS3**: stili con supporto responsive (mobile-first)
- **Vanilla JavaScript (ES6+)**: logica applicativa, niente framework pesanti
- **MediaPipe Face Mesh**: rilevamento posizione testa (via CDN)

### Storage
- **IndexedDB**: database locale per dati strutturati (medici, pazienti, sedute)
- **Dexie.js**: wrapper per semplificare operazioni IndexedDB (opzionale, ~30KB)

### PWA Components
- **Service Worker**: gestione cache e offline
- **Web App Manifest**: metadati per installazione
- **Cache API**: storage risorse statiche

### Funzionalità Browser
- **getUserMedia API**: accesso camera frontale
- **Canvas API**: elaborazione immagini e disegno linee
- **File API / Blob**: gestione foto
- **Web Share API**: condivisione PDF (progressive enhancement)

---

## Architettura File

```
PAC/
├── index.html              # Entry point, shell applicazione
├── manifest.json           # Manifest PWA
├── sw.js                   # Service Worker
├── css/
│   ├── main.css           # Stili globali
│   ├── screens.css        # Stili per singole schermate
│   └── components.css     # Componenti riusabili
├── js/
│   ├── app.js             # Bootstrap app, routing
│   ├── db.js              # Wrapper IndexedDB (Dexie)
│   ├── auth.js            # Logica login/logout
│   ├── router.js          # Routing hash-based
│   ├── screens/
│   │   ├── login.js       # Schermata login
│   │   ├── pazienti.js    # Lista pazienti
│   │   ├── scheda.js      # Scheda paziente
│   │   ├── scatto.js      # Scatto foto
│   │   ├── risultato.js   # Risultato analisi
│   │   ├── confronto.js   # Confronto sedute
│   │   ├── pdf.js         # Generazione PDF
│   │   └── impostazioni.js# Impostazioni
│   ├── utils/
│   │   ├── camera.js      # Gestione camera
│   │   ├── mediapipe.js   # Integrazione MediaPipe
│   │   ├── pdf-gen.js     # Generazione PDF (jsPDF)
│   │   └── helpers.js     # Utility varie
│   └── vendor/
│       └── (librerie esterne se non da CDN)
├── assets/
│   ├── icons/             # Icone PWA (varie dimensioni)
│   ├── images/            # Immagini guida, placeholders
│   └── fonts/             # Font (opzionale, preferire system fonts)
└── README.md              # Documentazione sviluppatore
```

---

## Service Worker - Strategia Caching

### Strategia: **Cache-First con Network Fallback (per risorse statiche)**

**File**: `sw.js`

```javascript
const CACHE_NAME = 'pac-v1';
const STATIC_CACHE = [
  '/',
  '/index.html',
  '/css/main.css',
  '/css/screens.css',
  '/css/components.css',
  '/js/app.js',
  '/js/db.js',
  '/js/auth.js',
  '/js/router.js',
  // ... tutti i JS
  '/assets/icons/icon-192.png',
  '/assets/icons/icon-512.png',
  '/manifest.json'
];

// Install: pre-cache risorse statiche
self.addEventListener('install', event => {
  event.waitUntil(
    caches.open(CACHE_NAME)
      .then(cache => cache.addAll(STATIC_CACHE))
      .then(() => self.skipWaiting())
  );
});

// Activate: pulizia vecchie cache
self.addEventListener('activate', event => {
  event.waitUntil(
    caches.keys().then(keys => 
      Promise.all(
        keys.filter(key => key !== CACHE_NAME)
            .map(key => caches.delete(key))
      )
    ).then(() => self.clients.claim())
  );
});

// Fetch: cache-first per static, network-only per API esterne
self.addEventListener('fetch', event => {
  const url = new URL(event.request.url);
  
  // MediaPipe e CDN esterni: network-first (con cache fallback)
  if (url.origin !== location.origin) {
    event.respondWith(
      fetch(event.request)
        .then(response => {
          const clone = response.clone();
          caches.open(CACHE_NAME).then(cache => cache.put(event.request, clone));
          return response;
        })
        .catch(() => caches.match(event.request))
    );
    return;
  }
  
  // Risorse locali: cache-first
  event.respondWith(
    caches.match(event.request)
      .then(cached => cached || fetch(event.request))
  );
});
```

### Compromesso Primo Accesso
- **Primo caricamento**: richiede connessione internet per scaricare risorse statiche e MediaPipe da CDN
- **Carichi successivi**: completamente offline
- **Aggiornamenti**: service worker rileva nuove versioni e aggiorna cache in background

---

## Manifest PWA

**File**: `manifest.json`

```json
{
  "name": "PAC - Posture Analysis Capture",
  "short_name": "PAC",
  "description": "Acquisizione e analisi postura del capo per professionisti medici",
  "start_url": "/",
  "display": "standalone",
  "orientation": "portrait",
  "background_color": "#ffffff",
  "theme_color": "#2563eb",
  "icons": [
    {
      "src": "/assets/icons/icon-72.png",
      "sizes": "72x72",
      "type": "image/png"
    },
    {
      "src": "/assets/icons/icon-96.png",
      "sizes": "96x96",
      "type": "image/png"
    },
    {
      "src": "/assets/icons/icon-128.png",
      "sizes": "128x128",
      "type": "image/png"
    },
    {
      "src": "/assets/icons/icon-144.png",
      "sizes": "144x144",
      "type": "image/png"
    },
    {
      "src": "/assets/icons/icon-152.png",
      "sizes": "152x152",
      "type": "image/png"
    },
    {
      "src": "/assets/icons/icon-192.png",
      "sizes": "192x192",
      "type": "image/png",
      "purpose": "any maskable"
    },
    {
      "src": "/assets/icons/icon-384.png",
      "sizes": "384x384",
      "type": "image/png"
    },
    {
      "src": "/assets/icons/icon-512.png",
      "sizes": "512x512",
      "type": "image/png",
      "purpose": "any maskable"
    }
  ],
  "categories": ["medical", "health", "productivity"],
  "screenshots": [
    {
      "src": "/assets/screenshots/screen1.png",
      "sizes": "540x720",
      "type": "image/png"
    }
  ]
}
```

### Installazione
- **Android/Chrome**: banner automatico dopo 2 visite entro 5 minuti + engagement
- **iOS/Safari**: manuale "Aggiungi a Home Screen" (nessun banner automatico)
- **Desktop**: icona nella barra URL per installazione come app desktop

---

## Routing

### Hash-Based Routing

Utilizziamo hash routing (`#/path`) per compatibilità offline senza configurazione server.

**File**: `js/router.js`

```javascript
class Router {
  constructor() {
    this.routes = new Map();
    this.currentRoute = null;
    
    window.addEventListener('hashchange', () => this.navigate());
    window.addEventListener('load', () => this.navigate());
  }
  
  register(path, handler) {
    this.routes.set(path, handler);
  }
  
  navigate(path) {
    if (path) {
      window.location.hash = path;
      return;
    }
    
    const hash = window.location.hash.slice(1) || '/login';
    const [route, ...params] = hash.split('/').filter(Boolean);
    
    const handler = this.routes.get(`/${route}`);
    if (handler) {
      this.currentRoute = route;
      handler(params);
    } else {
      this.navigate('/login');
    }
  }
}

// Utilizzo in app.js
const router = new Router();
router.register('/login', renderLogin);
router.register('/pazienti', renderPazienti);
router.register('/paziente', renderSchedaPaziente); // params[0] = pazienteId
router.register('/scatto', renderScatto);
router.register('/risultato', renderRisultato);
router.register('/confronto', renderConfronto);
router.register('/pdf', renderPDF);
router.register('/impostazioni', renderImpostazioni);
```

### Rotte Principali

| Hash                  | Schermata          | Parametri                |
|-----------------------|--------------------|--------------------------|
| `#/login`             | Login              | -                        |
| `#/pazienti`          | Lista pazienti     | -                        |
| `#/paziente/:id`      | Scheda paziente    | id paziente              |
| `#/scatto/:id`        | Scatto foto        | id paziente              |
| `#/risultato/:id`     | Risultato          | id seduta (temp/saved)   |
| `#/confronto/:id`     | Confronto          | id paziente              |
| `#/pdf/:id1/:id2`     | PDF                | id sedute da confrontare |
| `#/impostazioni`      | Impostazioni       | -                        |

---

## Flusso Bootstrap App

**File**: `js/app.js`

```javascript
// 1. Registrazione Service Worker
if ('serviceWorker' in navigator) {
  navigator.serviceWorker.register('/sw.js')
    .then(reg => console.log('SW registered', reg))
    .catch(err => console.error('SW error', err));
}

// 2. Inizializzazione DB
import { initDB } from './db.js';
await initDB();

// 3. Check autenticazione
import { checkAuth } from './auth.js';
const isAuth = await checkAuth();

// 4. Inizializzazione router
import { router } from './router.js';
if (!isAuth) {
  router.navigate('/login');
} else {
  router.navigate(); // Naviga a hash corrente o default
}

// 5. Setup eventi globali
document.addEventListener('offline', () => {
  showToast('Modalità offline attiva');
});

document.addEventListener('online', () => {
  showToast('Connessione ripristinata');
});
```

---

## Gestione Stato Applicazione

### SessionStorage per Stato Temporaneo
- Autenticazione: `sessionStorage.setItem('medico_id', id)`
- Dati temporanei schermata (es. foto appena scattata prima del salvataggio)

### IndexedDB per Stato Persistente
- Tutti i dati utente (medici, pazienti, sedute)
- Impostazioni app

### In-Memory per UI
- Stato UI transitori (dropdown aperti, loading states)

---

## Gestione Errori e Fallback

### Errori IndexedDB
```javascript
try {
  await db.pazienti.add(paziente);
} catch (error) {
  if (error.name === 'QuotaExceededError') {
    showError('Spazio insufficiente. Elimina sedute vecchie.');
  } else {
    showError('Errore salvataggio dati.');
    console.error(error);
  }
}
```

### Errori Camera
```javascript
try {
  const stream = await navigator.mediaDevices.getUserMedia({ video: { facingMode: 'user' }});
} catch (error) {
  if (error.name === 'NotAllowedError') {
    showError('Permesso camera negato. Abilita dalle impostazioni del browser.');
  } else if (error.name === 'NotFoundError') {
    showError('Nessuna camera trovata.');
  }
}
```

### MediaPipe Non Disponibile
```javascript
if (!window.MediaPipe) {
  showError('Libreria analisi non caricata. Verifica connessione internet al primo avvio.');
  // Fallback: permetti scatto foto senza analisi angoli
}
```

---

## Performance e Ottimizzazioni

### Lazy Loading Schermate
- Caricare JS delle schermate solo quando navigato (dynamic import)
```javascript
const renderScatto = async () => {
  const module = await import('./screens/scatto.js');
  module.render();
};
```

### Compressione Immagini
- Salvataggio foto come JPEG con qualità 85%
- Ridimensionamento a max 1920×1080 prima del salvataggio

### Debounce Ricerca
- Input ricerca pazienti: debounce 300ms per ridurre query

### Virtual Scrolling (opzionale)
- Se lista pazienti > 100: implementare virtual scroll

---

## Sicurezza

### Password Hashing
```javascript
// Utilizzo Web Crypto API (nativo) o libreria bcrypt.js
import bcrypt from 'bcryptjs';
const hash = await bcrypt.hash(password, 10);
const isValid = await bcrypt.compare(password, hash);
```

### Content Security Policy (CSP)
```html
<meta http-equiv="Content-Security-Policy" 
      content="default-src 'self'; 
               script-src 'self' https://cdn.jsdelivr.net; 
               style-src 'self' 'unsafe-inline'; 
               img-src 'self' blob: data:;">
```

### Sanitizzazione Input
- Validare e sanitizzare tutti gli input utente prima del salvataggio
- Escape HTML in output dinamico per prevenire XSS

---

## Testing Strategy

### Unit Test
- Funzioni pure (calcoli angoli, helpers)
- Operazioni DB (mock IndexedDB)

### Integration Test
- Flussi navigazione
- Interazione camera + MediaPipe

### E2E Test
- Playwright o Cypress per flussi completi utente

### Manual Test
- Test su dispositivi fisici (Android, iOS)
- Test offline: airplane mode dopo primo carico

---

## Deployment

### Hosting Statico
- **GitHub Pages** (gratis, HTTPS automatico)
- **Netlify / Vercel** (gratis, deploy automatico da Git)
- **Server locale**: nginx/Apache con redirect SPA

### Configurazione Server
- HTTPS obbligatorio per Service Worker
- Header `Cache-Control` per risorse statiche
- Servire `index.html` per tutte le rotte (per hash routing non strettamente necessario)

### Update Workflow
1. Modifica codice
2. Incrementa versione cache in `sw.js` (`pac-v1` → `pac-v2`)
3. Deploy
4. Service Worker rileva nuova versione al prossimo caricamento
5. Notifica utente: "Nuova versione disponibile, ricarica per aggiornare"

---

## Compatibilità Browser

| Feature            | Chrome | Firefox | Safari | Edge |
|--------------------|--------|---------|--------|------|
| Service Worker     | ✅     | ✅      | ✅     | ✅   |
| IndexedDB          | ✅     | ✅      | ✅     | ✅   |
| getUserMedia       | ✅     | ✅      | ✅     | ✅   |
| Web App Manifest   | ✅     | ✅      | ⚠️     | ✅   |
| Web Share API      | ✅     | ❌      | ✅     | ✅   |

**Note Safari iOS**:
- Manifest PWA supportato da iOS 14.5+
- Installazione solo manuale
- Limiti storage più stringenti (~1GB)

---

## Roadmap Architetturale Futura

1. **Sync opzionale**: plugin per sincronizzazione cloud (Google Drive, Dropbox)
2. **Multi-device**: esportazione/importazione dati tra dispositivi
3. **WebAssembly**: elaborazione immagini più veloce
4. **Background Sync**: upload PDF in background quando torna online

---

## Decisioni Architetturali Chiave

### Perché Hash Routing?
- Zero configurazione server
- Funziona offline out-of-the-box
- Semplicità implementazione

### Perché IndexedDB invece di LocalStorage?
- Limiti storage molto maggiori (LocalStorage ~5-10MB, IndexedDB ~50MB-1GB+)
- Supporto Blob per immagini
- Query strutturate più efficienti

### Perché Niente Framework (React/Vue)?
- Vincolo "simple"
- Riduzione bundle size
- Zero build step necessario
- Vanilla JS moderno (ES6+) è molto potente

### Perché CDN per MediaPipe?
- Libreria ~20MB, troppo grande per bundle app
- Google CDN ha cache globale e performance eccellenti
- Compromesso: primo carico richiede internet

---

## Conclusione

L'architettura PWA di PAC bilancia:
- ✅ **Offline-first**: dati e funzionamento locale
- ✅ **Semplicità**: stack "simple" senza complessità inutili
- ✅ **Performance**: cache aggressiva, lazy loading
- ⚠️ **Compromesso**: primo accesso richiede internet per MediaPipe

Questa architettura garantisce un'app installabile, veloce e affidabile anche senza connessione, mantenendo lo stack tecnologico semplice e manutenibile.
