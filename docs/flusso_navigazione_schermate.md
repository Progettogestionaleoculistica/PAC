# Flusso Navigazione Schermate - PAC (Posture Analysis Capture)

## Diagramma Flusso

```
┌─────────────────────┐
│  1. LOGIN           │
│  - Selezione medico │
│  - Password         │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│  2. LISTA PAZIENTI  │◄─────────────────────┐
│  - Ricerca          │                      │
│  - Nome/Cognome/DN  │                      │
└──────────┬──────────┘                      │
           │                                 │
           ├──► [+ Nuovo Paziente]           │
           │                                 │
           ▼                                 │
┌─────────────────────┐                      │
│  3. SCHEDA PAZIENTE │                      │
│  - Dati anagrafici  │                      │
│  - Fino a 3 sedute  │                      │
│  - Etichette        │                      │
└──────────┬──────────┘                      │
           │                                 │
           ├──► [+ Nuova Seduta]             │
           │         │                       │
           │         ▼                       │
           │   ┌─────────────────────┐       │
           │   │  4. SCATTO FOTO     │       │
           │   │  - Sagoma guida     │       │
           │   │  - Avviso distanza  │       │
           │   │  - Camera frontale  │       │
           │   └──────────┬──────────┘       │
           │              │                  │
           │              ▼                  │
           │   ┌─────────────────────┐       │
           │   │  5. RISULTATO       │       │
           │   │  - Foto elaborata   │       │
           │   │  - Linee inclinaz.  │       │
           │   │  - Angoli YPR       │       │
           │   │  - Note             │       │
           │   └──────────┬──────────┘       │
           │              │                  │
           │              └─────► [Salva] ───┤
           │                                 │
           ├──► [Confronta Sedute]           │
           │         │                       │
           │         ▼                       │
           │   ┌─────────────────────┐       │
           │   │  6. CONFRONTO       │       │
           │   │  - 2 foto affiancate│       │
           │   │  - Angoli           │       │
           │   │  - Differenze       │       │
           │   │  - Note comparative │       │
           │   └──────────┬──────────┘       │
           │              │                  │
           │              └─────► [PDF] ─────┤
           │                       │         │
           │                       ▼         │
           │              ┌─────────────────────┐
           │              │  7. GENERA/CONDIV.  │
           │              │  PDF                │
           │              │  - Anteprima        │
           │              │  - Salva/Condividi  │
           │              └─────────────────────┘
           │                                 │
           └──► [Impostazioni]               │
                     │                       │
                     ▼                       │
              ┌─────────────────────┐        │
              │  8. IMPOSTAZIONI    │        │
              │  - Gestione medici  │        │
              │  - Logo studio      │        │
              │  - Export dati      │        │
              └─────────────────────┘        │
                     │                       │
                     └───────────────────────┘
```

## Dettaglio Schermate

### 1. Login
**Scopo**: Autenticazione medico  
**Input**: Selezione medico da dropdown + password  
**Output**: Accesso a lista pazienti  
**Navigazione**: Unica schermata di ingresso → Lista pazienti

### 2. Lista Pazienti
**Scopo**: Visualizzare e cercare pazienti  
**Elementi**:
- Campo ricerca (nome/cognome/data nascita)
- Lista pazienti con nome, cognome, ultima seduta
- Pulsante "+ Nuovo Paziente"
- Pulsante "Impostazioni" (top-right)
**Navigazione**:
- Click paziente → Scheda paziente
- "+ Nuovo Paziente" → Scheda paziente vuota
- "Impostazioni" → Schermata impostazioni

### 3. Scheda Paziente
**Scopo**: Visualizzare dati paziente e storico sedute  
**Elementi**:
- Dati anagrafici (nome, cognome, data nascita)
- Lista sedute (max 3 visualizzate, con etichetta e data)
- Pulsante "+ Nuova Seduta"
- Pulsante "Confronta Sedute" (se ≥2 sedute)
**Navigazione**:
- "← Indietro" → Lista pazienti
- "+ Nuova Seduta" → Scatto foto
- Click seduta → Risultato (visualizzazione)
- "Confronta Sedute" → Confronto

### 4. Scatto Foto
**Scopo**: Acquisire foto frontale con guida  
**Elementi**:
- Preview camera frontale
- Sagoma guida trasparente (testa + spalle)
- Avviso distanza approssimativo (es. "Avvicinati" / "OK" / "Allontanati")
- Pulsante scatto
- Campo etichetta seduta (opzionale, es. "Prima visita", "Controllo 3 mesi")
**Navigazione**:
- "← Annulla" → Scheda paziente
- Scatto → Elaborazione → Risultato

### 5. Risultato
**Scopo**: Visualizzare foto elaborata con dati angoli  
**Elementi**:
- Foto con linee inclinazione sovrapposte
- Valori angoli: Yaw (rotazione dx/sx), Pitch (avanti/indietro), Roll (inclinazione laterale)
- Campo note testuale
- Pulsante "Salva"
- Pulsante "Riscatta" (ripete acquisizione)
**Navigazione**:
- "Salva" → Scheda paziente (seduta salvata)
- "Riscatta" → Scatto foto
- "← Annulla" → Scheda paziente (niente salvato)

### 6. Confronto Due Sedute
**Scopo**: Confrontare visivamente e numericamente due sedute  
**Elementi**:
- Selezione seduta 1 e seduta 2 (dropdown con etichetta+data)
- Due foto affiancate (o una sopra l'altra su mobile)
- Tabella angoli affiancati (Yaw, Pitch, Roll)
- Colonna differenze (Δ Yaw, Δ Pitch, Δ Roll)
- Campo note comparative
- Pulsante "Genera PDF"
**Navigazione**:
- "← Indietro" → Scheda paziente
- "Genera PDF" → Generazione/Condivisione PDF

### 7. Generazione/Condivisione PDF
**Scopo**: Creare e condividere report  
**Elementi**:
- Anteprima PDF (logo studio, dati paziente, foto sedute, angoli, note)
- Pulsante "Salva PDF" (download locale)
- Pulsante "Condividi" (Web Share API se disponibile)
**Navigazione**:
- "← Indietro" → Schermata chiamante (Confronto o Risultato)
- "Salva PDF" → Salva file → Conferma → Indietro

### 8. Impostazioni
**Scopo**: Configurazione app e gestione dati  
**Elementi**:
- **Gestione medici**: lista medici, aggiungi/rimuovi/modifica password
- **Logo studio**: carica immagine logo per PDF
- **Export dati**: esporta tutti i dati in JSON (backup manuale)
- **Info app**: versione, credits
**Navigazione**:
- "← Indietro" → Lista pazienti

## Note Implementative

- **Routing**: utilizzare hash-based routing (#/login, #/pazienti, #/paziente/:id, ecc.) per compatibilità offline
- **Stato**: mantenere stato navigazione in sessionStorage per ripristino dopo refresh
- **Breadcrumb**: mostrare percorso navigazione per orientamento utente
- **Conferme**: chiedere conferma prima di uscire da Scatto/Risultato senza salvare
- **Loading**: indicatori caricamento per elaborazione foto e generazione PDF
