# WUP Coach Booking System

Sistema di prenotazione centralizzato e anti-caos per sessioni WUP (Wake Up Call) con coach/sales.

---

## 📐 Architettura

```
┌─────────────────────────────────────────────────────────────────┐
│                         CLIENTE (Browser)                        │
│  index.html → seleziona coach                                    │
│  booking.html → vede slot disponibili → compila form → prenota  │
└─────────────────────────┬───────────────────────────────────────┘
                          │ HTTPS fetch
┌─────────────────────────▼───────────────────────────────────────┐
│              APPS SCRIPT WEB APP (doGet / doPost)                │
│  - Api.gs        → routing endpoint                              │
│  - Calendar.gs   → legge/scrive calendari "Managed"             │
│  - Sheets.gs     → legge/scrive Google Sheets Master            │
│  - Email.gs      → invio email conferma/cancellazione           │
│  - Utils.gs      → lock, validazione, logging                   │
│  - Setup.gs      → inizializza sheets, crea fogli/header        │
└──────────┬──────────────────────┬───────────────────────────────┘
           │                      │
┌──────────▼────────┐   ┌─────────▼──────────────────────────────┐
│  GOOGLE CALENDAR  │   │         GOOGLE SHEETS "Master"          │
│  (WUP Admin acct) │   │  Tab: Coaches / Bookings / Audit       │
│                   │   └────────────────────────────────────────┘
│ Calendar Managed  │
│ per ogni coach:   │
│  - WUP – NomeCoach│
│    (Managed)      │
│  - Admin = OWNER  │
│  - Coach = guest  │
│    (no modifica)  │
└───────────────────┘
```

### Principi chiave

| Principio | Implementazione |
|-----------|-----------------|
| Source of truth centralizzata | Google Sheets "Master" + calendari "Managed" di proprietà Admin |
| Coach non possono rompere il sistema | Calendari Managed owned by Admin; coach ricevono solo invito come guest |
| No doppie prenotazioni | LockService Google Apps Script attorno alla creazione evento |
| Timezone coerente | Tutto in `Europe/Rome` |
| Scalabile a decine di coach | Coach gestiti via tab "Coaches" in Sheets |

---

## 📁 Struttura cartelle

```
wup-coach-booking/
├── README.md                   ← questo file
├── apps-script/
│   ├── Code.gs                 ← entry point Web App (doGet/doPost)
│   ├── Config.gs               ← costanti e configurazione globale
│   ├── Api.gs                  ← routing e handler API
│   ├── Calendar.gs             ← logica disponibilità e creazione eventi
│   ├── Sheets.gs               ← lettura/scrittura Google Sheets
│   ├── Email.gs                ← template e invio email
│   ├── Utils.gs                ← LockService, validazione, logging
│   └── Setup.gs                ← funzione setup() per primo avvio
└── web/
    ├── index.html              ← pagina selezione coach
    ├── booking.html            ← pagina calendario + form prenotazione
    ├── app.js                  ← logica frontend (fetch API)
    └── style.css               ← stile mobile-first
```

---

## 🚀 Setup passo-passo

### FASE 0 — Prerequisiti

- Account Google Workspace (Admin) dedicato al progetto (es. `wup-admin@tuodominio.com`)
- Accesso a Google Drive, Sheets, Calendar, Apps Script di quell'account
- Lista coach con: `nome`, `cognome`, `email`, `ruolo`, `foto_url` (fornita da Juan Pablo)

---

### FASE 1 — Google Sheets "Master Coaching"

1. Vai su [Google Sheets](https://sheets.google.com) con l'account WUP Admin
2. Crea un nuovo Spreadsheet → nominalo **"Master Coaching"**
3. Copia l'ID dello Spreadsheet dall'URL (la parte tra `/d/` e `/edit`):
   ```
   https://docs.google.com/spreadsheets/d/SPREADSHEET_ID/edit
   ```
4. Tieni questo ID: ti servirà in `Config.gs`

I fogli verranno creati automaticamente dalla funzione `setup()` (vedi Fase 3).

---

### FASE 2 — Google Apps Script

1. Vai su [script.google.com](https://script.google.com)
2. Clicca **"Nuovo progetto"**
3. Rinomina il progetto in **"WUP Coach Booking"**
4. Per ogni file `.gs` nella cartella `apps-script/`:
   - Clicca **"+"** → **"Script"**
   - Nominalo esattamente come il file (es. `Config`, `Api`, `Calendar`, ecc.)
   - Incolla il contenuto del file corrispondente
5. Aggiorna `Config.gs` con i valori reali:
   - `SPREADSHEET_ID`
   - `ADMIN_EMAIL`
   - `APP_NAME`

> **Nota:** il file principale si chiama `Code.gs` (già presente di default).

---

### FASE 3 — Primo avvio: funzione setup()

1. Nell'editor Apps Script, seleziona la funzione `setup` dal menu a tendina
2. Clicca **"Esegui"**
3. Accetta le autorizzazioni OAuth richieste (Calendar, Sheets, Mail)
4. Verifica che in Google Sheets siano apparsi i fogli: `Coaches`, `Bookings`, `Audit`
5. Popola il foglio `Coaches` con i dati dei coach (vedi schema sotto)

---

### FASE 4 — Creazione calendari "Managed" per ogni coach

Per ogni coach:

1. Vai su [Google Calendar](https://calendar.google.com) con account WUP Admin
2. Barra sinistra → **"Aggiungi altri calendari"** → **"Crea nuovo calendario"**
3. Nomina il calendario: `WUP – NomeCoach (Managed)`
4. Crea il calendario
5. Vai nelle impostazioni del calendario → **"Condividi con persone specifiche"**
6. Aggiungi l'email del coach con permesso: **"Visualizza tutti i dettagli degli eventi"** (NO modifica)
7. Copia il **Calendar ID** (nelle impostazioni, sezione "Integra calendario")
8. Incolla il Calendar ID nella colonna `calendar_managed_id` del foglio `Coaches`

> **Permessi coach:** Il coach vede gli eventi ma non può modificarli o eliminarli perché il calendario è owned dall'Admin. Riceverà un invito come `guest` per ogni prenotazione.

---

### FASE 5 — Deploy Web App

1. In Apps Script: **"Distribuisci"** → **"Nuova distribuzione"**
2. Tipo: **"App web"**
3. Impostazioni:
   - Esegui come: **Me (WUP Admin)**
   - Chi ha accesso: **Chiunque** (per permettere prenotazioni pubbliche)
4. Clicca **"Distribuisci"**
5. Copia l'**URL della Web App**
6. Incolla l'URL in `web/app.js` come valore di `API_URL`

---

### FASE 6 — Deploy Frontend

Il frontend è HTML/CSS/JS statico puro. Puoi hostarlo in due modi:

**Opzione A — GitHub Pages (consigliato)**
```bash
cd wup-coach-booking/web
git init && git add . && git commit -m "init"
# push su GitHub → abilita Pages
```

**Opzione B — Google Sites**
- Carica i file HTML/CSS/JS come allegati su un Google Site

**Opzione C — Qualsiasi hosting statico**
- Netlify, Cloudflare Pages, qualsiasi server web

---

### FASE 7 — Test con coach pilota

1. Popola `Coaches` con 1 coach di test:
   ```
   id: COACH001
   nome: Mario
   cognome: Rossi
   email: mario.rossi@example.com
   ruolo: Senior Sales Coach
   bio: Esperto di tecniche WUP con 5 anni di esperienza.
   foto_url: https://...
   calendar_managed_id: abc123@group.calendar.google.com
   working_hours_start: 09:00
   working_hours_end: 18:00
   slot_duration_min: 60
   active: TRUE
   ```
2. Apri `index.html` (o l'URL deployato)
3. Seleziona il coach → verifica che appaia la pagina di booking
4. Scegli uno slot → compila il form → prenota
5. Verifica:
   - [ ] Email arriva al cliente
   - [ ] Email arriva al coach
   - [ ] Riga appare in Sheets tab "Bookings"
   - [ ] Evento appare nel calendario "WUP – Mario Rossi (Managed)"
   - [ ] Il coach riceve invito calendario

---

## 📊 Schema Google Sheets

### Tab: `Coaches`

| Colonna | Tipo | Descrizione |
|---------|------|-------------|
| `id` | Testo | ID univoco coach (es. COACH001) |
| `nome` | Testo | Nome |
| `cognome` | Testo | Cognome |
| `email` | Email | Email coach (per inviti) |
| `ruolo` | Testo | Ruolo/titolo |
| `bio` | Testo | Bio breve (max 200 char) |
| `foto_url` | URL | Link foto profilo |
| `calendar_managed_id` | Testo | ID calendario Managed (es. xxx@group.calendar.google.com) |
| `working_hours_start` | Ora | Inizio disponibilità (es. 09:00) |
| `working_hours_end` | Ora | Fine disponibilità (es. 18:00) |
| `slot_duration_min` | Numero | Durata slot in minuti (es. 60) |
| `active` | Booleano | TRUE = coach visibile sul sito |

### Tab: `Bookings`

| Colonna | Tipo | Descrizione |
|---------|------|-------------|
| `booking_id` | Testo | ID univoco prenotazione |
| `created_at` | DateTime | Data/ora creazione |
| `coach_id` | Testo | Riferimento ID coach |
| `coach_name` | Testo | Nome completo coach |
| `coach_email` | Email | Email coach |
| `client_name` | Testo | Nome cliente |
| `client_surname` | Testo | Cognome cliente |
| `client_email` | Email | Email cliente |
| `client_phone` | Testo | Telefono cliente |
| `start_datetime` | DateTime | Inizio prenotazione (Europe/Rome) |
| `end_datetime` | DateTime | Fine prenotazione (Europe/Rome) |
| `timezone` | Testo | Sempre "Europe/Rome" |
| `notes` | Testo | Note opzionali cliente |
| `status` | Testo | CONFIRMED / CANCELLED |
| `cancel_token` | Testo | Token per link cancellazione |
| `event_id` | Testo | ID evento Google Calendar |
| `calendar_id` | Testo | ID calendario Managed usato |

### Tab: `Audit`

| Colonna | Tipo | Descrizione |
|---------|------|-------------|
| `timestamp` | DateTime | Quando è accaduto |
| `level` | Testo | INFO / WARN / ERROR |
| `action` | Testo | Azione (BOOKING_CREATED, CANCEL, ecc.) |
| `booking_id` | Testo | Riferimento prenotazione (se applicabile) |
| `message` | Testo | Descrizione estesa |
| `payload` | Testo | JSON raw della richiesta (per debug) |

---

## ✅ Checklist pre-evento

- [ ] Foglio `Coaches` popolato con tutti i coach attivi
- [ ] Calendari "Managed" creati per tutti i coach
- [ ] Calendar ID inseriti nel foglio `Coaches`
- [ ] Web App re-deployata con ultima versione del codice
- [ ] URL Web App aggiornato in `app.js`
- [ ] Test end-to-end completato con coach pilota
- [ ] Email di conferma verificate (mittente, oggetto, contenuto)
- [ ] Test cancellazione con link tokenizzato
- [ ] Test doppia prenotazione simultanea (LockService)
- [ ] Accesso frontend verificato da mobile

---

## 🔒 Strategia permessi calendari

### Perché i coach NON possono rompere il sistema

Google Calendar non permette a un guest di modificare o eliminare un evento su un calendario di cui non è owner. Poiché i calendari "Managed" sono di proprietà dell'account WUP Admin:

- Il coach riceve un **invito** come guest → vede l'evento nel suo calendario
- Il coach **NON può eliminare** l'evento dal calendario Managed (può solo rimuoverlo dalla propria vista)
- Il coach **NON può modificare** data/ora/dettagli dell'evento
- Il coach può solo rispondere all'invito (Accetta/Rifiuta/Forse) — questo NON cancella l'evento

### Audit failsafe

Il sistema controlla periodicamente (trigger ogni ora) che gli eventi prenotati esistano ancora. Se un evento viene cancellato/modificato dall'esterno:
- Logga in tab `Audit` con livello `WARN`
- Invia email di alert all'Admin

### Limitazioni note

- Se il coach rimuove l'evento dalla **propria** vista (Elimina solo per me), l'evento rimane nel calendario Managed — nessun problema.
- L'unico modo in cui un coach potrebbe creare problemi è se avesse accesso diretto all'account Admin — il che non deve mai accadere.

---

## 📧 Contatti progetto

- **Owner:** Luca Sammarco
- **Supporto tecnico:** Alessandro Terracciano
- **Lista coach:** fornita da Juan Pablo Pliego
