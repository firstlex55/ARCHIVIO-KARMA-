# ARCHIVIO TARIFFE — Project Documentation

**Repository:** `firstlex55/ARCHIVIO-KARMA-`  
**URL:** `https://firstlex55.github.io/ARCHIVIO-KARMA-/`  
**File principale:** `index.html` (219 KB, single-file app)  
**Versione app:** v1.0.0  
**Azienda:** Pro Trasporti Srl  

---

## Descrizione

App HTML single-file per la gestione di tariffe di trasporto e prezzi di prodotti biomassa/legno. Funziona completamente offline tramite localStorage, con sync opzionale su Google Drive. Sviluppata per uso mobile (Android Chrome), ottimizzata per schermo piccolo.

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
- ExcelJS via CDN per export Excel
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
- Verificare sempre con `node --check` dopo modifiche JS

### Struttura HTML
```
<head>
  <meta charset> + PWA meta tags
  <style> ... </style>           ← CSS (~538 coppie {})
  <script src="xlsx CDN">        ← libreria Excel
</head>
<body>
  <!-- HTML interfaccia -->
  <!-- Modal dialogs (15+ modal) -->
  <script>
    // JS principale (~136KB)
    var PDF_DATA = [...];        ← 189 prezzi tariffe
    var KARMA_DATA = [...];      ← 43 prezzi prodotti
    // ... tutto il codice app
    init();
  </script>
</body>
```

**ATTENZIONE:** Il tag `<script>` del main JS si trova a posizione fissa nel file. Modifiche che inseriscono HTML dentro il blocco `<script>` rompono l'app (il codice appare visibile nella pagina invece di essere eseguito).

---

## Database (localStorage)

**Chiave:** `archivio-tariffe-v3`

```json
{
  "trasporti": [...],   // tariffe trasporto
  "prodotti": [...],    // prezzi prodotti
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
  "data": "",
  "source": "karma"
}
```

**source** può essere `"karma"` (da KARMA_DATA) o `"manual"` (aggiunto dall'utente).

---

## Migrazioni localStorage

| Chiave | Operazione |
|--------|-----------|
| `tariffe-migration-v2` | Prima deduplicazione prodotti |
| `tariffe-migration-v3` | Deduplicazione con chiave base |
| `tariffe-migration-v4` | Deduplicazione con chiave estesa (include note) — **ULTIMA** |

Le migrazioni si eseguono una sola volta al primo avvio dopo aggiornamento.

---

## Dati Incorporati

### PDF_DATA — Tariffe Trasporto (REV 03/2026)
- **189 prezzi** su **63 tratte** per **18 trasportatori**
- Unità: `€/vg` (viaggio), `€/ton`, `€/mc`
- Trasportatori: A.L.B. Srl, ALBA, ASCHIERI, AVIO, BRANCHINI, C.L.P., C.M. TRASPORTI, CEVOLO, CIRIONI, COAP, CONECO, CONSAR 2026, FRAULINI, PADANA, RUFFINI, STEGAGNO, UDERZO, UNITRAG

### KARMA_DATA — Prezzi Prodotti (KarMa 03/2026)
- **43 record** (dopo deduplicazione)
- Prodotti: Cippato, Segatura, Lolla di Farro, Lolla di Riso, Pula/lolla mix cereali, Sfarinato cereali mix, Sfarinato sorgo/b
- Include 9 record vendita AGROGI (Segatura, prezzi separati per fornitore)

---

## Funzionalità

### Tab Trasporti

**Vista Rapida**
- Card per tratta con prezzo migliore in evidenza (ambra)
- Chip per trasportatore con condizioni pagamento e note
- Sezioni separate: 🚛 €/viaggio · ⚖️ €/tonnellata · 📦 €/metro cubo
- Pill filtro trasportatori in cima (scrollabile orizzontalmente)
- Click su chip → modal modifica singola tariffa
- Click su header card → vista Confronto per quella tratta
- Autocomplete partenza/destinazione con info contestuale:
  - Campo DA: mostra n° destinazioni disponibili
  - Campo A: mostra prezzo minimo disponibile per quella tratta

**Vista Confronto**
- Barre proporzionali per confronto visivo prezzi
- Dropdown selezione tratta

**Modal Aggiunta Tariffa (+)**
- Campi: Da, A, Trasportatore, Condizioni, Prezzo, Unità, Note
- **Auto-fill condizioni**: scrivendo il trasportatore, le condizioni si compilano automaticamente con quelle usate più spesso per quel trasportatore
- Se esiste già una tariffa per stessa tratta+trasportatore+unità: modal scelta "➕ Aggiungi tratta" / "✏️ Aggiorna tratta"
- Campo Note: appare sul chip solo se presente

**Modal Modifica Singola**
- Aperto toccando il chip del trasportatore
- Campi: Prezzo, Unità, Condizioni, Note, ⚠️ Da confermare, 🤝 Trattabile
- In fondo: confronto altri trasportatori stessa tratta (verde=più economico, rosso=più caro)

**Modal Modifica Tratta (✏️ Modifica)**
- Mostra tutti i trasportatori della tratta con campi modificabili
- Pulsante ➕ aggiungi trasportatore

### Tab Prodotti

**Vista Prodotto** (default)
- Card per tipo prodotto con emoji automatica (🪵 Cippato, 🪚 Segatura, 🌾 Lolla, 🌿 altro)
- Sezioni ACQUISTO (blu) e VENDITA (ambra)
- Layout B4: fornitore grande, luogo in blu, prezzo in ambra 26px
- ★ min sul prezzo più basso acquisto
- Tocca riga → modal modifica prodotto

**Vista Fornitore / Vista Cliente**
- Raggruppa per entità con avatar iniziali (da implementare)
- Card con lista prodotti per fornitore/cliente

**Modal Aggiunta Prodotto (+)**
- Campi: Prodotto, Tipo (acquisto/vendita), Fornitore/Cliente, Luogo, Prezzo, Unità, Note, Data

**Modal Modifica Prodotto**
- Aperto toccando qualsiasi riga nella vista Prodotti
- Campi: Prodotto, Prezzo, Unità, Fornitore/Cliente, Luogo, Note
- Pulsante 🗑 Elimina

### Tab Import

- Import Excel trasporti (auto-detect colonne)
- Import Excel prodotti
- 🔄 Ricarica tariffe REV 03/2026 (da PDF_DATA)
- 🌾 Carica listino KarMa (da KARMA_DATA, idempotente)
- Export Excel filtrato
- Gestione Dati: Svuota tutto / Svuota prodotti
- Import/Export JSON backup

### Google Drive
- Client ID: `107091966360-vbepp0lmghbck14vv89et30acl34d8a9.apps.googleusercontent.com`
- GIS caricato lazy al primo click
- Auto-save debounce 2s dopo ogni modifica
- Salva/carica in `appDataFolder` (invisibile all'utente)
- Gestione conflitti con modal scelta locale/drive

---

## Regole di Sviluppo

### Workflow
1. L'utente carica il file HTML completo nella chat
2. Claude applica modifiche chirurgiche al JS/CSS/HTML
3. Verifica con `node --check` dopo ogni modifica JS
4. L'utente scarica e carica su GitHub

### Punti Critici
- **Non toccare mai** la struttura dei tag `<script>` — il main JS è tra posizioni fisse nel file
- **Non inserire HTML** dentro il blocco `<script>` (causa il bug del codice visibile)
- **renderViewProdotto** usa `DB.prodotti.indexOf(p)` con fallback by-value
- **renderEditPricesList** idem per `DB.trasporti.indexOf(p)`
- **init()** chiamata esplicita alla fine del JS (non dentro window.onload)
- **KARMA_DATA** si ricarica automaticamente all'avvio se `DB.karmaRev` non corrisponde

### CSS Chiavi
- `--amber`: colore primario ambra (#fb923c)
- `--green`: verde (#34d399)  
- `--t2`: testo secondario
- `.prow-*`: classi card prodotti
- `.chip-*`: classi chip trasportatori in vista rapida
- `.rrow-*`: classi card tratta

---

## Feature da Implementare (Backlog)

- [ ] Avatar iniziali colorati per trasportatori (stile B preferito — quadrato arrotondato con bordo)
- [ ] Campo Cliente separato nei record prodotto (oltre al campo entity)
- [ ] Vista Fornitore → Cliente con freccia nella lista prodotti
- [ ] Storico prezzi (confronto revisioni)
- [ ] Icona PWA che funziona su Android (problema: Chrome non legge manifest da GitHub Pages correttamente)

---

## Note Sessioni Precedenti

- **Bug ricorrente**: HTML del modal finisce dentro `<script>` tag → codice visibile nella pagina. Causa: inserimento di HTML vicino al punto di apertura del tag script. Fix: sempre ricostruire dal file originale caricato dall'utente.
- **Doppioni prodotti**: causati da caricamento KarMa multiplo senza deduplicazione. Risolto con migrazione v4.
- **Arrow functions Android**: `=>` causa crash su Android vecchio. Convertire sempre in `function(){}`.
- **Drive Client ID** da non modificare: `107091966360-vbepp0lmghbck14vv89et30acl34d8a9.apps.googleusercontent.com`
