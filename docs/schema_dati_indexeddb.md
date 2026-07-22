# Schema Dati IndexedDB - PAC

## Database: `pac_db`
**Versione**: 1  
**Strategia**: Object Stores per entità principali, nessuna sincronizzazione cloud

---

## Object Store: `medici`

**keyPath**: `id` (auto-increment)

| Campo          | Tipo      | Obbligatorio | Descrizione                           |
|----------------|-----------|--------------|---------------------------------------|
| id             | number    | auto         | Identificativo univoco                |
| nome           | string    | sì           | Nome completo medico                  |
| username       | string    | sì           | Username per login (univoco)          |
| password_hash  | string    | sì           | Hash password (bcrypt o simile)       |
| creato_il      | timestamp | sì           | Data creazione record                 |

**Indici**:
- `username` (unique)

**Note**:
- Password memorizzata come hash per sicurezza base
- Username univoco per evitare duplicati

---

## Object Store: `pazienti`

**keyPath**: `id` (auto-increment)

| Campo          | Tipo      | Obbligatorio | Descrizione                           |
|----------------|-----------|--------------|---------------------------------------|
| id             | number    | auto         | Identificativo univoco                |
| nome           | string    | sì           | Nome paziente                         |
| cognome        | string    | sì           | Cognome paziente                      |
| data_nascita   | string    | sì           | Data nascita (formato ISO YYYY-MM-DD) |
| medico_id      | number    | sì           | ID medico che ha creato il paziente   |
| creato_il      | timestamp | sì           | Data creazione record                 |
| modificato_il  | timestamp | sì           | Data ultima modifica                  |

**Indici**:
- `cognome`
- `data_nascita`
- `medico_id`
- `[cognome, nome]` (compound index per ricerca)

**Note**:
- Ricerca ottimizzata per cognome+nome
- Un paziente appartiene al medico che lo ha creato
- Data nascita come stringa per compatibilità e semplicità parsing

---

## Object Store: `sedute`

**keyPath**: `id` (auto-increment)

| Campo          | Tipo      | Obbligatorio | Descrizione                                  |
|----------------|-----------|--------------|----------------------------------------------|
| id             | number    | auto         | Identificativo univoco                       |
| paziente_id    | number    | sì           | ID paziente                                  |
| medico_id      | number    | sì           | ID medico che ha eseguito la seduta          |
| data           | timestamp | sì           | Data e ora seduta                            |
| etichetta      | string    | no           | Etichetta descrittiva (es. "Prima visita")   |
| foto_blob      | Blob      | sì           | Immagine originale (JPEG/PNG)                |
| foto_elaborata | Blob      | no           | Immagine con linee sovrapposte (opzionale)   |
| pitch          | number    | sì           | Angolo pitch in gradi (tilt avanti/indietro) |
| yaw            | number    | sì           | Angolo yaw in gradi (rotazione dx/sx)        |
| roll           | number    | sì           | Angolo roll in gradi (inclinazione laterale) |
| note           | string    | no           | Note testuali libere                         |
| creato_il      | timestamp | sì           | Data creazione record                        |

**Indici**:
- `paziente_id`
- `medico_id`
- `data`
- `[paziente_id, data]` (compound index per storico paziente ordinato)

**Note**:
- Foto memorizzata come Blob per gestione ottimale memoria
- Angoli in gradi per leggibilità (range tipico: ±90°)
- Etichetta opzionale per identificazione rapida seduta
- `foto_elaborata` può essere generata on-the-fly e salvata per performance

---

## Object Store: `impostazioni`

**keyPath**: `chiave`

| Campo     | Tipo             | Obbligatorio | Descrizione                          |
|-----------|------------------|--------------|--------------------------------------|
| chiave    | string           | sì           | Identificativo impostazione          |
| valore    | any              | sì           | Valore impostazione (tipo variabile) |

**Record Predefiniti**:

| chiave              | tipo valore | Descrizione                              |
|---------------------|-------------|------------------------------------------|
| `logo_studio`       | Blob / null | Logo studio per PDF (PNG/JPEG)           |
| `nome_studio`       | string      | Nome studio medico per PDF               |
| `ultima_versione_db`| number      | Versione schema DB per migrazioni future |
| `primo_avvio`       | boolean     | Flag primo avvio per onboarding          |

**Note**:
- Struttura key-value flessibile per estensioni future
- Logo opzionale (null se non impostato)

---

## Relazioni

```
medici (1) ──┬── (N) pazienti
             └── (N) sedute

pazienti (1) ─── (N) sedute
```

**Regole**:
- Un medico può avere molti pazienti
- Un paziente appartiene a un solo medico (semplificazione)
- Un paziente può avere molte sedute
- Una seduta è sempre collegata a un paziente e al medico che l'ha eseguita

---

## Query Comuni

### 1. Lista pazienti del medico corrente (con ricerca)
```javascript
// Index: medico_id
const pazienti = await db.pazienti
  .where('medico_id').equals(medicoId)
  .filter(p => 
    p.nome.toLowerCase().includes(query) || 
    p.cognome.toLowerCase().includes(query) ||
    p.data_nascita.includes(query)
  )
  .sortBy('cognome');
```

### 2. Storico sedute di un paziente
```javascript
// Index: [paziente_id, data]
const sedute = await db.sedute
  .where('[paziente_id+data]')
  .between([pazienteId, minDate], [pazienteId, maxDate])
  .reverse()
  .toArray();
```

### 3. Ultima seduta di un paziente
```javascript
// Index: paziente_id
const ultima = await db.sedute
  .where('paziente_id').equals(pazienteId)
  .last();
```

### 4. Validazione login
```javascript
// Index: username
const medico = await db.medici
  .where('username').equals(username)
  .first();
// Confronta password_hash con hash della password inserita
```

---

## Gestione Versioni DB

Per future migrazioni dello schema, utilizzare `onupgradeneeded` di IndexedDB:

```javascript
const request = indexedDB.open('pac_db', 1);

request.onupgradeneeded = function(event) {
  const db = event.target.result;
  const oldVersion = event.oldVersion;
  
  if (oldVersion < 1) {
    // Creazione iniziale
    const medicoStore = db.createObjectStore('medici', { keyPath: 'id', autoIncrement: true });
    medicoStore.createIndex('username', 'username', { unique: true });
    
    const pazienteStore = db.createObjectStore('pazienti', { keyPath: 'id', autoIncrement: true });
    pazienteStore.createIndex('cognome', 'cognome');
    pazienteStore.createIndex('medico_id', 'medico_id');
    pazienteStore.createIndex('cognome_nome', ['cognome', 'nome']);
    
    const sedutaStore = db.createObjectStore('sedute', { keyPath: 'id', autoIncrement: true });
    sedutaStore.createIndex('paziente_id', 'paziente_id');
    sedutaStore.createIndex('medico_id', 'medico_id');
    sedutaStore.createIndex('paziente_data', ['paziente_id', 'data']);
    
    db.createObjectStore('impostazioni', { keyPath: 'chiave' });
  }
  
  // Future: if (oldVersion < 2) { ... }
};
```

---

## Stima Dimensioni

**Ipotesi**:
- Medici: ~10 record → ~1 KB
- Pazienti: ~500 record → ~50 KB
- Sedute: ~1500 record (media 3 per paziente)
  - Metadati: ~100 KB
  - Foto (JPEG 1920×1080, ~200 KB/foto): ~300 MB
- Impostazioni: ~10 KB
- **Totale stimato**: ~300 MB

**Limiti IndexedDB**:
- Chrome/Edge: generalmente >1 GB (6% storage disponibile)
- Firefox: ~50% storage disponibile
- Safari: ~1 GB

⚠️ Con ~1500 sedute si è ancora ampiamente sotto i limiti, ma monitorare crescita per ottimizzazioni future (es. compressione immagini, pulizia sedute vecchie).

---

## Sicurezza e Privacy

- **Locale-only**: dati MAI trasferiti in cloud
- **Crittografia**: non implementata (IndexedDB stesso accessibile solo dalla stessa origine)
- **Backup**: export JSON manuale da Impostazioni
- **Pulizia**: disinstallazione PWA elimina tutti i dati (warning necessario)

---

## Libreria Raccomandata

**Dexie.js** (wrapper IndexedDB):
- Sintassi più semplice rispetto a IndexedDB nativo
- Gestione promise/async-await
- Query simil-SQL intuitive
- Supporto TypeScript
- Leggero (~30 KB minified)

Alternativa: **IndexedDB nativo** se si preferisce zero dipendenze (complessità maggiore).
