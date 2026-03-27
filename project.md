# ARCHIVIO TARIFFE — Documentazione Progetto

## Panoramica
App HTML single-file per Pro Trasporti Srl (Ferrara). Permette di consultare, modificare ed esportare tariffe di trasporto e prezzi di acquisto/vendita prodotti (cippato, segatura, lolla, pula, sfarinato).

**URL produzione:** `firstlex55.github.io` (GitHub Pages)  
**File:** `archivio-tariffe.html` — singolo file HTML autocontenuto  
**localStorage key:** `archivio-tariffe-v3`  
**Google Drive Client ID:** `107091966360-vbepp0lmghbck14vv89et30acl34d8a9.apps.googleusercontent.com`

---

## Stack Tecnico
- **HTML/CSS/JS** puro — nessun framework
- **XLSX.js** (CDN `cdnjs.cloudflare.com/ajax/libs/xlsx/0.18.5/xlsx.full.min.js`) — export Excel
- **Google Identity Services** — caricato in modo lazy (1.5s dopo l'avvio) per evitare flash bianco su Android
- **Target:** Android-first, Chrome mobile

---

## Struttura HTML
```
<head>
  <script src="xlsx.full.min.js">        ← CDN XLSX
  <!-- GIS loaded lazily on Drive click -->
  <style> ... </style>                   ← CSS inline (~300 regole)
</head>
<body>
  <div id="app-header">                  ← Header con logo, badge tratte, pulsante Drive
  <div id="drivePanel">                  ← Pannello Drive slide-down
  <div id="unsavedBanner">               ← Banner modifiche non salvate
  <div class="tabs">                     ← Tab Trasporti | Prodotti | Import
  <div id="page-trasporti">             ← Vista trasporti
  <div id="page-prodotti">              ← Vista prodotti
  <div id="page-import">                ← Vista import
  <!-- modali: modalEditTratta, modalExport, modalAddTariffa, modalAddProdotto -->
  <div id="toast">                       ← Notifiche toast
  <script> ... </script>                 ← Tutto il JS inline
</body>
```

---

## Dati Incorporati (costanti JS)

### `PDF_DATA` — 189 record tariffe trasporti
Fonte: `Tariffe_trasporti_2026_rev_03_16_03_2026.xlsx` (REV 03/2026, Ferrara 16/03/2026)

Struttura record:
```js
{
  from: "Bientina (PI)",
  to: "Verolavecchia (BS)",
  carrier: "COAP",
  price: 32.5,
  unit: "€/ton",          // oppure "€/vg" (€/viaggio)
  conditions: "BB 90 gg FM",
  rev: "REV 03/2026"
}
```

**18 trasportatori:** A.L.B. Srl TRASPORTI, ALBA TRASPORTI, ASCHIERI TRASPORTI, AVIO TRASPORTI, BRANCHINI, CEVOLO Trasporti, CIRIONI Trasporti, C.L.P. Trasporti, C.M. TRASPORTI, COAP, CONECO, CONSAR 2026, FRAULINI TRASPORTI, PADANA TRASPORTI, RUFFINI, STEGAGNO, UDERZO, UNITRAG

**63 tratte** — basi di carico principali: Bientina (PI), Canossa (RE), Caprarola (VT), Castelfranco Emilia (MO), Castel S. Nicolo-Cetica (AR), Castelvetro (PC), Colle Val d'Elsa (SI), Cortemilia (CN), Cremona (CR) cantieri, Dosolo (MN), Firenzuola (FI), Fossacaprara (CR), Grezzana (VR), Imola (BO), Jolanda di Savoia (FE), Mantova (MN) cantieri, Mercatino Conca (PU), Mezzocorona (TN), Mondovì (CN), Ravenna-LLOYDD, S.Anna di Alfaedo (VR), S.Felice sul Panaro (MO), San Giorgio di Piano (BO), Tirano (SO), Vione-Stadolina (BS)

### `KARMA_DATA` — 41 record listino KarMa 03/2026
Fonte: `Listino_KarMa_03_2026.xlsx`

Struttura record:
```js
{
  prodotto: "Segatura",
  tipo: "acquisto",        // o "vendita"
  entity: "Ardenghi",      // fornitore o cliente
  luogo: "Dosolo (MN)",
  prezzo: 85,
  unita: "€/ton",          // o "€/mc"
  note: ""
}
```

**Prodotti:** Cippato, Segatura, Lolla di Farro, Lolla di Riso, Pula/lolla mix, Sfarinato cereali, Sfarinato sorghum

---

## DATA STORE (localStorage)
```js
let DB = {
  trasporti: [],    // array record tariffe (PDF_DATA + modifiche manuali)
  prodotti: [],     // array record listino KarMa + inserimenti manuali
  pdfLoaded: false,
  pdfRev: '',
  lastUpdate: ''    // ISO timestamp ultimo salvataggio
}
```

**Key localStorage:** `archivio-tariffe-v3`

**Migrazione automatica in `init()`:**
- COAP Bientina→Verolavecchia: 32.7 €/vg → 32.5 €/ton
- COAP Mercatino Conca→Ponzano: €/vg → €/ton

---

## Variabili di Stato Globali
```js
var activePill = '';              // trasportatore selezionato nei pills
var editingTratta = null;         // {from, to} o {from, to, singleIdx}
var activeProdottoFilter = '';    // filtro prodotto attivo
var importMode = 'add';           // 'add' | 'replace'
var currentView = 'rapida';       // 'rapida' | 'confronto'
var currentProdView = 'prodotto'; // 'prodotto' | 'fornitore' | 'cliente'
var driveAccessToken = null;
var driveFileId = null;
var _isDirty = false;
var _drivePanel = false;
var _dirtyTimer = null;
```

---

## Flusso di Avvio (`init()`)
1. Rimuove vecchia cache `archivio-tariffe-old`
2. Carica `archivio-tariffe-v3` da localStorage
3. Normalizza destinazioni sinonime (`normalizeDest`)
4. Applica migrazioni prezzi (COAP fix)
5. Se `DB.trasporti.length === 0` → precarica `PDF_DATA`
6. Imposta data oggi in form prodotti (con null-check)
7. Chiama `refreshAll()` → `refreshStats()` + `refreshProdotti()` + `filterRoutes()`
8. Dopo 1.5s carica GIS in background e tenta auto-reconnect Drive

---

## Tab TRASPORTI

### Vista ⚡ Rapida
- Card per ogni tratta (`.rrow`) con bordo ambra sinistro
- Sezioni separate `€/VIAGGIO` e `€/TON`
- Ogni riga trasportatore (`.price-row`) è cliccabile → apre `openEditSingle(globalIdx)`
- Il best-price è evidenziato in ambra (stella ★)
- Badge provvisorio ⚠ e trattabile 🤝 visibili inline

### Vista 📊 Confronto
- Select tratta + barre orizzontali proporzionali
- Auto-selezione se From/To già compilati nel filtro di ricerca
- Quando si passa a Confronto con campi già compilati, la tratta viene auto-selezionata

### Filtri
- **Partenza / Destinazione:** input testo con autocomplete custom DOM-based (funziona su Android)
  - `acSearch(field)` — filtra mentre scrivi, con highlight verde del testo cercato
  - `acOpen(field)` — mostra tutte le opzioni al focus
  - `acSelect(field, value)` — seleziona e chiude
- **Pill trasportatori:** click su pill → filtra per quel trasportatore + banner arancione "TRATTE DI [carrier]"
- **Reset:** pulsante 🔄

### Modifica
- **`openEditSingle(idx)`** — apre modal con UNA sola riga (prezzo, unità, condizioni, ⚠️ da confermare, 🤝 trattabile)
- **`openEditTratta(from, to)`** — apre modal con tutti i trasportatori della tratta (accesso da pulsante ✏️)
- **`saveEditTratta()`** — salva entrambe le modalità (controlla `editingTratta.singleIdx`)
- **`addPriceRowInEdit()`** — aggiunge riga nuova trasportatore (DOM-based, no innerHTML)
- **`deleteTratta()`** — elimina tutti i record from+to

### Export / Stampa
- Pulsante 🖨️ → `openExportModal()` con filtri partenza, destinazione, trasportatori (checkbox), formato
- **Excel** (`exportToExcel`): struttura identica a REV 03/2026 — trasportatori in colonne, tratte in righe
- **Stampa A3** (`printA3`): apre finestra ottimizzata per stampa orizzontale A3, prezzi migliori evidenziati

---

## Tab PRODOTTI

### Vista 🌿 Prodotto
- Card per prodotto con sezioni acquisto (rosso) / vendita (verde)
- Filtro per tipo prodotto (pill tabs)
- Campo ricerca libero

### Vista 🏭 Fornitore
- `renderViewFornitore(data)` — schede premium con avatar 🏭, bordo rosso in cima
- Grid 3 colonne: nome prodotto | prezzo (rosso, 18px) + unità | luogo

### Vista 🤝 Cliente
- `renderViewCliente(data)` — schede premium con avatar 🤝, bordo verde in cima
- Se un prodotto ha più prezzi mostra range (es. 70–120 €/ton)
- Grid 3 colonne: nome prodotto | prezzo (verde) + unità | luogo

**Tab rimossi:** Margine (era presente, ora eliminato)

---

## Tab IMPORT

### Import Trasporti
- Trascina o seleziona file XLSX/XLS
- `handleFileImport` usa `readAsArrayBuffer` + `XLSX.read({type:'array'})` (compatibilità Android)
- Auto-detect formato KarMa → redirect a `importProdotti`
- Pulsante **"🌾 Carica Listino KarMa 03/2026"** — carica direttamente da `KARMA_DATA` incorporato

### Import Prodotti
- Parser KarMa dedicato — rileva colonne Costo Acquisto / Prezzo Vendita
- Modalità: Aggiungi (default) o Sostituisci

---

## Google Drive

### Pannello Drive (slide-down)
```
[dot stato] Drive connesso / Drive non connesso
[☁️ SALVA / su Drive]  [📥 CARICA / da Drive]
[🔗 Connetti Google Drive]  ← visibile solo se disconnesso
```

### Stato dot
- 🟢 verde pulsante = connesso
- ⚫ grigio = disconnesso
- 🟡 ambra lampeggiante = sync in corso

### Pulsanti disabilitati (opacità 45%) quando non connesso

### Funzioni principali
- `toggleDrivePanel()` — apre/chiude pannello
- `updateDrivePanelStatus()` — aggiorna dot + testo + opacità pulsanti
- `handleDriveClick()` / `driveLogin()` — OAuth token
- `driveSave()` — salva JSON su Drive (appDataFolder)
- `driveLoad()` — carica JSON da Drive
- `markDirty()` — segna modifiche non salvate → dopo 3s mostra banner ambra
- `tryAutoReconnect()` — auto-riconnessione al caricamento (GIS lazy)

### GIS Lazy Loading
Google Identity Services NON è caricato nell'`<head>` — si carica in modo lazy dopo 1.5s per evitare la schermata bianca su Android Chrome.

---

## CSS — Palette e Variabili

```css
--bg: #0b0f16
--s1: #111927    (surface 1)
--s2: #0f1520    (surface 2)
--s3: #1a2436    (surface 3)
--b1: rgba(255,255,255,.07)  (border)
--b2: rgba(255,255,255,.12)
--t1: #f0f6ff    (text primario)
--t2: #8ba3c0    (text secondario)
--t3: #4a6080    (text terziario)
--green: #34d399
--green2: #059669
--gdim: rgba(52,211,153,.12)
--amber: #fb923c
--amberdim: rgba(251,146,60,.12)
--red: #f87171
--reddim: rgba(248,113,113,.12)
--r: 12px        (border-radius default)
```

---

## Componenti CSS Principali

| Classe | Descrizione |
|--------|-------------|
| `.rrow` | Card tratta trasporti, bordo ambra sinistro |
| `.price-section` | Sezione €/viaggio o €/ton dentro rrow |
| `.price-section-hd` | Header sezione (etichetta + icona) |
| `.price-row` | Riga singolo trasportatore, cliccabile |
| `.price-row.best` | Miglior prezzo, sfondo + bordo ambra |
| `.erow` | Riga edit nel modal modifica |
| `.erow-prov-label` | Label checkbox "da confermare" / "trattabile" |
| `.entity-card.forn` | Scheda fornitore, bordo rosso in cima |
| `.entity-card.clie` | Scheda cliente, bordo verde in cima |
| `.entity-avatar` | Avatar quadrato 44px con icona |
| `.entity-row` | Grid 3 colonne: prodotto | prezzo | luogo |
| `.ac-dropdown` | Dropdown autocomplete ricerca |
| `.ac-item` | Voce dropdown, `<mark>` per highlight |
| `.drive-panel` | Pannello Drive slide-down |
| `.dp-btn` | Pulsante Drive verticale (icona + label + sub) |
| `.pvbtn` | Tab switcher vista prodotti (icona + label) |
| `.cpill` | Pill filtro trasportatore |
| `.cpill.on` | Pill attiva (bordo ambra) |

---

## Funzioni JS Principali

### Core
| Funzione | Descrizione |
|----------|-------------|
| `init()` | Avvio app, carica dati, DOM setup |
| `save()` | Salva DB in localStorage |
| `saveAndSync()` | save() + driveSave() se connesso |
| `refreshAll()` | refreshStats + refreshProdotti + filterRoutes |
| `refreshStats()` | Aggiorna contatori tratte/trasportatori/prezzi |
| `refreshProdotti()` | Aggiorna vista prodotti in base a currentProdView |
| `filterRoutes()` | Filtra tratte per from/to/carrier e renderizza |

### Trasporti
| Funzione | Descrizione |
|----------|-------------|
| `renderRapida(routes, filterCarrier)` | Render vista rapida (override tramite window.renderRapida) |
| `renderConfronto()` | Render vista confronto con barre |
| `renderCarrierPills()` | Render pill filtro trasportatori (DOM-based) |
| `openEditSingle(idx)` | Apre modal edit SINGOLA riga |
| `openEditTratta(from, to)` | Apre modal edit TUTTI i trasportatori della tratta |
| `saveEditTratta()` | Salva modifiche (gestisce sia single che full) |
| `addPriceRowInEdit()` | Aggiunge nuovo trasportatore nel modal (DOM-based) |
| `normalizeDest(dest)` | Normalizza sinonimi destinazioni |
| `getUniqueRoutes()` | Restituisce array tratte uniche con prezzi |

### Autocomplete
| Funzione | Descrizione |
|----------|-------------|
| `acSearch(field)` | Filtra lista mentre si digita |
| `acOpen(field)` | Mostra tutte le opzioni al focus |
| `acSelect(field, value)` | Seleziona un'opzione |
| `acClose(field)` | Chiude dropdown |
| `acRender(field, items, q)` | Renderizza dropdown con highlight |

### Prodotti
| Funzione | Descrizione |
|----------|-------------|
| `setProdView(view)` | Cambia vista prodotti ('prodotto'/'fornitore'/'cliente') |
| `renderViewFornitore(data)` | Render schede fornitore |
| `renderViewCliente(data)` | Render schede cliente con range prezzi |
| `setProdottoFilter(p)` | Filtra per tipo prodotto |

### Drive
| Funzione | Descrizione |
|----------|-------------|
| `toggleDrivePanel()` | Apre/chiude pannello |
| `updateDrivePanelStatus()` | Aggiorna UI pannello (dot, testo, opacità) |
| `handleDriveClick()` / `driveLogin()` | OAuth login |
| `driveSave()` | Salva su Drive (appDataFolder) |
| `driveLoad()` | Carica da Drive |
| `markDirty()` | Segna modifiche non salvate |
| `tryAutoReconnect()` | Auto-reconnect al caricamento |
| `loadGISandRun(fn)` | Lazy-load GIS poi esegue callback |

### Export
| Funzione | Descrizione |
|----------|-------------|
| `openExportModal()` | Apre modal export con filtri |
| `doExport()` | Esegue export in base al formato selezionato |
| `exportToExcel(rows, carriers)` | Genera XLSX con struttura REV 03/2026 |
| `printA3(rows, carriers)` | Genera pagina stampa A3 orizzontale |

---

## Problemi Risolti in Sessione

### Critici
1. **`</style>` corrotto in `le>`** — causa schermata nera. Fix: `str.replace('le>', '</style>')`
2. **`init()` chiamata due volte** — il file base e il patch block chiamavano entrambi `init()`. Fix: rimossa la prima
3. **`activePill` e `editingTratta` non dichiarate** — rimosse per errore durante deduplication. Fix: reinserite prima di DATA STORE
4. **`statTrasportatori` null crash** — `refreshStats()` non aveva null-check su tutti gli elementi. Fix: early-return + null-check completi
5. **`refreshProdotti` leggeva `prodTipo`/`prodSearch` null** — Fix: null-check + early-return su `prodottiContainer`
6. **GIS flash bianco Android** — Fix: rimosso `<script>` GIS dall'head, caricato lazy dopo 1.5s
7. **COAP Bientina→Verolavecchia in sezione €/vg sbagliata** — Fix: migrazione in `init()` che corregge la cache localStorage

### Design
8. **Drive panel senza CSS** — il blocco CSS era fuori dal `<style>`. Fix: reinserito nel blocco corretto
9. **Checkbox "da confermare" non togglava** — div con onclick doppio. Fix: sostituito con `<label for="...">`
10. **Autocomplete non visibile** — `overflow:hidden` sulla search-card tagliava il dropdown. Fix: `overflow:visible`

---

## Architettura File

Il file è strutturato in questo ordine:
```
1. <!DOCTYPE html> + <head> (XLSX CDN, style)
2. CSS (~300 regole)
3. <body> HTML (header, panel Drive, tabs, pagine, modali, toast)
4. <script>
   a. Variabili stato globali (activePill, editingTratta, ...)
   b. Drive panel functions (_isDirty, toggleDrivePanel, markDirty, ...)
   c. Autocomplete functions (acSearch, acOpen, acRender, ...)
   d. DATA STORE (DB, PDF_DATA 189 records, KARMA_DATA 41 records)
   e. Core functions (save, refreshAll, refreshStats, refreshProdotti)
   f. Trasporti functions (filterRoutes, getUniqueRoutes, normalizeDest)
   g. renderRapida (originale, ora stub che chiama window.renderRapida)
   h. renderConfronto, renderCarrierPills
   i. Edit functions (openEditSingle, openEditTratta, saveEditTratta)
   j. Import functions (handleFileImport, importTrasporti, importProdotti)
   k. Drive functions (handleDriveClick, driveSave, driveLoad, ...)
   l. Export functions (openExportModal, exportToExcel, printA3)
   m. Prod views (setProdView, renderViewFornitore, renderViewCliente)
   n. openEditSingle (modifica singola riga trasportatore)
   o. window.renderRapida = function(...) { ... }  ← override/patch
   p. init() call
   q. window.addEventListener('load', ...)
5. </script></body></html>
```

---

## Note per Sviluppo Futuro

- **Aggiornamento dati:** quando arriva REV 04/2026, sostituire `PDF_DATA` e aggiornare `pdfRev`. La migrazione automatica in `init()` va aggiornata di conseguenza.
- **KarMa 04/2026:** sostituire `KARMA_DATA` e il pulsante "Carica Listino KarMa 03/2026"
- **Quote conflict JS:** le stringhe HTML nel JS vanno sempre costruite con DOM (`createElement`/`appendChild`) quando contengono attributi con virgolette, NON con concatenazione `html += '<div onclick="...">'`
- **Test runtime:** prima di ogni deploy, eseguire il test Node.js con stubs DOM per verificare zero crash su init()
- **Backup:** il file non ha versioning automatico. Sempre fare backup prima di modifiche strutturali.
