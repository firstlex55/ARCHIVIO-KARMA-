# ARCHIVIO TARIFFE — Project Documentation

**Repository:** `firstlex55/ARCHIVIO-KARMA-`  
**URL:** `https://firstlex55.github.io/ARCHIVIO-KARMA-/`  
**File principale:** `index.html` (single-file app, ~4300 righe)  
**Versione app:** v1.0.0  
**Azienda:** Pro Trasporti Srl  

---

## File nel Repository

| File | Descrizione |
|------|-------------|
| `index.html` | App completa (single-file) |
| `manifest.json` | Manifest PWA per installazione come app |
| `icon-tariffe.png` | Icona 192×192 per schermata home |
| `icon-tariffe-512.png` | Icona 512×512 per PWA |
| `PROJECT.md` | Questo file |

---

## Struttura Tecnica

### Stack
- HTML/CSS/JS vanilla (no framework)
- SheetJS (XLSX) via CDN per export Excel
- Google Drive API (GIS) per sync cloud
- localStorage come database locale
- Service Worker inline per PWA offline

### Compatibilità Android (regole critiche)
- **Zero** arrow functions (`=>`) — usare `function(x){}`
- **Zero** template literals (`` ` ``) — usare concatenazione `'...' + var`
- **Zero** `let`/`const` globali — usare `var`
- **Zero** spread operator (`[...arr]`, `{...obj}`)
- **Zero** shorthand object properties (`{from, to}`)
- **Zero** parametri default (`function(x, y='val')`)
- **Zero** `Array.from()`, `.find()`, `.findIndex()` su array
- **Zero** `Array.prototype.includes()` su array (ok su stringhe)
- Verificare sempre con `node --check` dopo modifiche JS

### Struttura HTML
```
<head>
  <meta charset> + PWA meta tags
  <style> ... </style>           ← CSS (~600+ regole)
  <script src="xlsx CDN">        ← libreria Excel (SheetJS)
</head>
<body>
  <!-- Ricerca globale (sopra i tab) -->
  <!-- Tab Trasporti / Prodotti / Import -->
  <!-- Modal dialogs (15+ modal) -->
  <script>
    var PDF_DATA = [...];        ← 189 prezzi tariffe
    var KARMA_DATA = [...];      ← 43 prezzi prodotti
    // ... tutto il codice app
    init();
  </script>
</body>
```

**ATTENZIONE:** Il tag `<script>` del main JS si trova a posizione fissa nel file. Modifiche che inseriscono HTML dentro il blocco `<script>` rompono l'app.

---

## Database (localStorage)

**Chiave:** `archivio-tariffe-v3`

```json
{
  "trasporti": [...],
  "prodotti": [...],
  "pdfLoaded": true,
  "pdfRev": "REV 03/2026",
  "karmaRev": "KarMa 03/2026",
  "lastUpdate": null
}
```

### Schema record trasporto
```json
{
  "from": "Bientina (PI)",
  "to": "Verolavecchia (BS)",
  "carrier": "COAP",
  "price": 31,
  "unit": "€/ton",
  "conditions": "BB 90 gg FM",
  "provisional": false,
  "negotiable": false,
  "note": "",
  "data": "2026-03-01",
  "rev": "REV 03/2026"
}
```

### Schema record prodotto
```json
{
  "prodotto": "Cippato",
  "tipo": "acquisto",
  "entity": "Ardenghi",
  "luogo": "Dosolo (MN)",
  "prezzo": 38,
  "unita": "€/ton",
  "note": "",
  "data": "2026-03-01",
  "source": "karma"
}
```

**source** può essere `"karma"` (da KARMA_DATA) o `"manual"` (aggiunto dall'utente).  
**data** formato `YYYY-MM-DD` — visualizzato come `GG/MM/AA` tramite `formatData()`.

---

## Migrazioni localStorage

| Chiave | Operazione |
|--------|-----------|
| `tariffe-migration-v2` | Prima deduplicazione prodotti |
| `tariffe-migration-v3` | Deduplicazione con chiave base |
| `tariffe-migration-v4` | Deduplicazione con chiave estesa (include note) — **ULTIMA** |

---

## Dati Incorporati

### PDF_DATA — Tariffe Trasporto (REV 03/2026)
- **189 prezzi** su **63 tratte** per **18 trasportatori**
- Unità: `€/vg` (viaggio), `€/ton`, `€/mc`
- Trasportatori: A.L.B. Srl, ALBA, ASCHIERI, AVIO, BRANCHINI, C.L.P., C.M. TRASPORTI, CEVOLO, CIRIONI, COAP, CONECO, CONSAR 2026, FRAULINI, PADANA, RUFFINI, STEGAGNO, UDERZO, UNITRAG
- Al ricarico via `reloadBuiltinData()`: tutti i record ricevono `data: '2026-03-01'` automaticamente

### KARMA_DATA — Prezzi Prodotti (KarMa 03/2026)
- **43 record** (dopo deduplicazione)
- Prodotti: Cippato, Segatura, Lolla di Farro, Lolla di Riso, Pula/lolla mix cereali, Sfarinato cereali mix, Sfarinato sorgo/b
- Include 9 record vendita AGROGI

---

## Funzionalità

### Ricerca Globale (sopra i tab)
- Campo sempre visibile, cerca in tempo reale su trasporti E prodotti
- Attiva con 2+ caratteri, mostra max 5 risultati per sezione
- Click risultato trasporto → salta tab, imposta **sia `searchFrom` che `searchTo`** e filtra
- Click risultato prodotto → salta tab e apre modal modifica
- Funzioni: `doGlobalSearch()`, `closeGlobalSearch()`, `globalJumpTrasporto(from, to)`, `globalJumpProdotto(idx)`

### Bug ricerca corretti (sessione 11-06)
1. **`globalJumpTrasporto`** ora riceve e imposta anche `to` (prima azzerava sempre searchTo)
2. **`acSearch`** ora suggerisce trasportatori come fallback anche nel campo Destinazione (prima solo in Partenza)
3. **`doGlobalSearch`** ora passa sia `from` che `to` al click sul risultato tratta

### Tab Trasporti

**Vista Rapida**
- Card per tratta (`rrow`) con bordo sinistro colorato deterministicamente per base di carico (`fromColor(from)` — palette 10 colori)
- Prezzo migliore in evidenza con badge sfondo ambra (`.rrow-best-price`)
- Chip per trasportatore con condizioni pagamento, data tariffa pill (`tariffa del GG/MM/AA`) con pallino, note
- Badge `⏰` su chip se data > 90 giorni
- **Pill data chip:** pill arrotondata con pallino `●` + "tariffa del GG/MM/AA" — stile `.chip-date` con `border-radius:20px`, sfondo `rgba(36,48,72,.7)`, testo `#6a82a0`
- Pill filtro trasportatori: bordo 2px + glow quando attiva
- Tap feedback su card (`:active` → `scale(0.985)`) e chip (`:active` → `scale(0.98)`)
- Click su chip → modal modifica singola tariffa
- Click su header card → vista Confronto per quella tratta
- Autocomplete: Campo DA e A cercano anche per nome trasportatore (fallback su entrambi i campi)
- Funzione `fromColor(from)` per colore deterministico bordo

**Vista Confronto**
- Barre proporzionali per confronto visivo prezzi
- Dropdown selezione tratta

**Modal Aggiunta Tariffa (+)**
- Campi: Da, A, Trasportatore, Condizioni, Prezzo, Unità, Note, 📅 Data tariffa
- Auto-fill condizioni dal trasportatore più frequente
- Se esiste già: modal scelta "➕ Aggiungi" / "✏️ Aggiorna"

**Modal Modifica Singola** (`openEditSingle(globalIdx)`)
- Campi: Prezzo, Unità, Condizioni, 📅 Data tariffa, Note, ⚠️ Da confermare, 🤝 Trattabile
- In fondo: confronto altri trasportatori stessa tratta

**Modal Modifica Tratta (✏️ Modifica)**
- Tutti i trasportatori con campi modificabili
- Pulsante ➕ aggiungi trasportatore (usa `Object.keys` invece di `Array.from(new Set(...))`)

### Tab Prodotti

**Layout chip (aggiornato sessione 11-06)**
- Struttura identica ai chip della tab Trasporti (stesso layout, stesse dimensioni testo)
- **Acquisto:** verde `#34d399` — bordo sinistro `rgba(52,211,153,.6)`, sfondo `rgba(52,211,153,.07)`, section header verde
- **Vendita:** ambra `#fb923c` — bordo sinistro `rgba(251,146,60,.6)`, sfondo `rgba(251,146,60,.07)`, section header ambra
- Entity 13px font-weight 700, loc 11px blu `#7eb8f7`, note 11px `#93c5fd` corsivo
- Prezzo acquisto verde, prezzo vendita ambra
- **Pill data:** `prezzo del GG/MM/AA` con pallino — identica alla pill dei chip tratte
- **Niente avatar** — rimosso il quadrato con iniziali
- Best acquisto badge `★ min`, best vendita badge `★ max`

**Vista Prodotto** (default)
- Card per tipo prodotto con emoji automatica
- Sezioni ACQUISTO e VENDITA con colori sopra descritti
- Tocca riga → modal modifica prodotto (`openEditProdotto(globalIdx)`)
- Stat cards colorate: Prodotti=verde, Fornitori=rosso, Clienti=ambra

**Vista Fornitore / Vista Cliente**
- Raggruppa per entità

**Modal Aggiunta Prodotto (+)**
- Campi: Prodotto, Tipo, Fornitore/Cliente, Luogo, Prezzo, Unità, Note, Data

**Modal Modifica Prodotto** (`openEditProdotto(idx)`, `saveEditProdotto()`, `deleteEditProdotto()`)
- Campi: Prodotto, Tipo, Fornitore/Cliente, Luogo, Prezzo, Unità, 📅 Data rilevazione, Note
- Pulsante 🗑 Elimina

### Tab Import

- Import Excel trasporti (auto-detect colonne)
- Import Excel prodotti
- 🔄 Ricarica tariffe REV 03/2026 (da PDF_DATA, setta `data:'2026-03-01'`)
- 🌾 Carica listino KarMa (da KARMA_DATA, idempotente)
- ⬇️ Export Excel — modal con selezione tipo:
  - **🚛 Tariffe Trasporti**: filtri da/a/trasportatori, formato Excel o A3
  - **🌾 Prezzi Prodotti**: sheet "Tutti i prezzi" + sheet per ogni prodotto
- 🗑 Svuota manuali (`clearProdottiManuali()` — mantiene `source='karma'`)
- 🗑 Svuota tutto
- Import/Export JSON backup

**Export A3** (`printA3(rows, selectedCarriers)`)
- Apre anteprima in nuova finestra (no stampa automatica)
- Pulsante 🖨️ Stampa visibile, nascosto in stampa
- REV dinamica da `DB.pdfRev`
- Best price: min €/vg separato da min €/ton, cella gialla
- Gestione popup bloccato con toast

**Export Excel Tariffe** (`exportToExcel(rows, selectedCarriers)`)
- REV dinamica, freeze righe/colonne, flag ⚠️ su provvisori

**Export Excel Prodotti** (`exportProdottiExcel()`)
- Sheet "Tutti i prezzi" + uno sheet per prodotto
- Acquisti ordinati per prezzo crescente, vendite per decrescente

### Google Drive
- Client ID: `107091966360-vbepp0lmghbck14vv89et30acl34d8a9.apps.googleusercontent.com`
- GIS caricato lazy al primo click
- Auto-save debounce 2s dopo ogni modifica
- Salva/carica in `appDataFolder`
- Gestione conflitti con modal scelta locale/drive

---

## Funzioni Utility Chiave

| Funzione | Descrizione |
|----------|-------------|
| `formatData(d)` | `'2026-03-01'` → `'01/03/26'` |
| `isDataVecchia(d, giorni)` | true se data > N giorni fa (default 60, tariffe 90) |
| `fromColor(from)` | colore deterministico da stringa (palette 10 colori) |
| `normalizeDest(dest)` | normalizza sinonimi destinazioni (es. Migliaro/Fiscaglia) |
| `abbrev(carrier)` | abbreviazione trasportatore per display compatto |
| `saveAndSync()` | salva localStorage + trigger Drive sync |
| `refreshAll()` | aggiorna stats + ricerca + prodotti + confronto |
| `doGlobalSearch()` | ricerca globale su trasporti e prodotti |
| `globalJumpTrasporto(from, to)` | salta a tab trasporti con from e to impostati |
| `acSearch(field)` | autocomplete campi DA/A con fallback trasportatori su entrambi |

---

## Regole di Sviluppo

### Workflow
1. L'utente carica il file HTML completo nella chat
2. Claude applica modifiche chirurgiche al JS/CSS/HTML
3. Verifica con `node --check` dopo ogni modifica JS
4. Audit: zero arrow functions, spread, Array.from, template literals, ID duplicati
5. L'utente scarica e carica su GitHub

### Punti Critici
- **Non toccare mai** la struttura dei tag `<script>`
- **Non inserire HTML** dentro il blocco `<script>`
- **renderViewProdotto** usa `DB.prodotti.indexOf(p)` con fallback by-value
- **renderEditPricesList** idem per `DB.trasporti.indexOf(p)`
- **init()** chiamata esplicita alla fine del JS
- **KARMA_DATA** si ricarica automaticamente se `DB.karmaRev` non corrisponde
- **selCarriers** filtrato con loop `for`, mai `.includes()` su array

### CSS Chiavi
| Variabile/Classe | Uso |
|-----------------|-----|
| `--amber` | colore primario ambra (#fb923c) |
| `--green` | verde (#34d399) |
| `--t2`, `--t3` | testo secondario/terziario |
| `.prow-*` | classi chip prodotti (layout identico a `.chip-*`) |
| `.chip-*` | classi chip trasportatori |
| `.chip-date` | pill data tariffa: `border-radius:20px`, pallino, "tariffa del GG/MM/AA" |
| `.prow-date` | pill data prodotto: identica a `.chip-date`, "prezzo del GG/MM/AA" |
| `.rrow-*` | classi card tratta |
| `.global-*` | classi ricerca globale |
| `.cpill` | pill filtro trasportatori |
| `.stat-card.red/amber` | stat cards colorate tab prodotti |

---

## Feature da Implementare (Backlog)

- [ ] Campo Cliente separato nei record prodotto
- [ ] Vista Fornitore → Cliente con freccia
- [ ] Storico prezzi (confronto revisioni)
- [ ] Delta prezzi tra date (▲/▼ accanto al prezzo)
- [ ] Copia rapida chip con long-press
- [ ] Widget "ultima modifica" in fondo alla pagina
- [ ] Icona PWA su Android (problema manifest GitHub Pages)
- [ ] Ulteriori migliorie grafiche tab Prodotti (in corso)

---

## Note Sessioni Precedenti

- **Bug ricorrente**: HTML modal dentro `<script>` → codice visibile. Fix: sempre ricostruire dal file originale.
- **Doppioni prodotti**: risolto con migrazione v4.
- **Arrow functions Android**: `=>` causa crash. Convertire sempre in `function(){}`.
- **CSS duplicato**: blocchi `.prow-*` erano definiti 4-5 volte. Rimossi in sessione precedente.
- **Modal duplicati**: `modalTariffaChoice`, `modalEntity`, `modalExport` erano duplicati in fondo al file. Rimossi.
- **spread in doExport**: `[...querySelectorAll()]` convertito in loop `for`.
- **Array.from in addPriceRowInEdit**: convertito in `Object.keys({})`.
- **Drive Client ID** da non modificare: `107091966360-vbepp0lmghbck14vv89et30acl34d8a9.apps.googleusercontent.com`

## Modifiche Sessione 11-06-2026

### Grafiche
1. **Chip data tab Trasporti**: sostituita emoji `📅` con pill arrotondata — pallino + "tariffa del GG/MM/AA". Rimosso `⏰` condizionale (per ora). CSS `.chip-date` riscritto.
2. **Tab Prodotti — layout completo**: riscritto per essere identico ai chip tratte. Rimosso avatar con iniziali. Verde per acquisto, ambra per vendita. Pill data "prezzo del GG/MM/AA".

### Bug Fix Ricerca
1. `globalJumpTrasporto(from, to)` — ora imposta sia partenza che destinazione
2. `acSearch(field)` — fallback trasportatori attivo su entrambi i campi DA e A
3. `doGlobalSearch()` — click risultato tratta ora passa `from` e `to` alla funzione di salto
