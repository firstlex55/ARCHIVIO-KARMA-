# ARCHIVIO TARIFFE — PROJECT.md
*Ultimo aggiornamento: 31/03/2026*

---

## Identità del progetto

**App:** Archivio Tariffe — gestione tariffe trasporti e prezzi prodotti per Pro Trasporti Srl (Ferrara)  
**File:** `archivio-tariffe.html` — single-file HTML/CSS/JS, hosted su GitHub Pages  
**URL:** `firstlex55.github.io/archivio-tariffe.html`  
**Versione:** v1.0.0  
**localStorage key:** `archivio-tariffe-v3`  
**Drive Client ID:** `107091966360-vbepp0lmghbck14vv89et30acl34d8a9.apps.googleusercontent.com`  
**Drive filename:** `archivio-tariffe-data.json` (appDataFolder)

---

## Dati incorporati (costanti JS globali)

```js
var PDF_DATA   // 189 prezzi tariffe trasporti REV 03/2026 (Ferrara 16/03/2026)
               // 18 trasportatori, 63 tratte
var KARMA_DATA // 41 record Listino KarMa 03/2026
               // formato: { prodotto, fornitore, provenienza, destinazione,
               //            acquirente, costoAcquisto, prezzoVendita, unita }
```

Entrambe dichiarate con `var` (non `const`/`let`) per compatibilità Android Chrome.

---

## Struttura DB

```js
var DB = {
  trasporti: [],   // [{from, to, carrier, price, unit, conditions, rev, provisional, negotiable}]
  prodotti:  [],   // [{prodotto, tipo, entity, luogo, prezzo, unita, note, data}]
  lastUpdate: null
}
```

**Migrazione automatica in init():** COAP Bientina→Verolavecchia: price=32.5, unit='€/ton'

---

## Architettura JS — ordine critico

```
1.  var PDF_DATA = [...]          ← dati incorporati
2.  var KARMA_DATA = [...]
3.  Variabili stato globali       ← var (non let/const) per Android
    _drivePanel, _isDirty, _dirtyTimer, activePill, editingTratta,
    activeProdottoFilter, importMode, currentView, currentProdView,
    _autoSaveTimer, DB, driveAccessToken, driveFileId,
    DRIVE_CLIENT_ID, DRIVE_SCOPE, DRIVE_FILENAME, APP_VERSION
4.  Drive panel functions         ← toggleDrivePanel, markDirty, loadGISandRun
5.  Autocomplete (acSearch, acOpen, acRender, acSelect, acClose)
6.  Core functions                ← save, refreshAll, refreshStats, refreshProdotti, filterRoutes
7.  abbrev(), normalizeDest()
8.  function renderRapida()       ← stub che chiama _renderRapidaPremium
9.  var _renderRapidaPremium      ← DEVE essere prima di init()
10. var _renderConfronto
11. var _renderCarrierPillsPremium
12. function init()               ← chiamata esplicita dopo
13. init();                       ← chiamata esplicita
14. renderViewProdotto, renderViewFornitore, renderViewCliente
15. openEditSingle, saveEditTratta, deleteTratta, removePriceRow
16. importTrasporti, importProdotti, reloadBuiltinData, loadKarmaData
17. handleFileImport              ← usa readAsArrayBuffer + decode_cell
18. Drive functions               ← handleDriveClick, driveLogin, driveDisconnect,
                                     setDriveStatus, driveSave, driveLoad, saveAndSync
19. Export functions              ← openExportModal, doExport, exportToExcel, printA3
20. Modal prodotto autocomplete   ← mpAcSearch, mpAcOpen, mpAcSelect, mpAcClose,
                                     mpAcRender, mpAutoFillFromProdotto, mpAutoFillFromEntity
21. openEntityModal
22. window.addEventListener("load") ← NO GIS automatico, solo flag visivo Drive
```

**Regola critica:** `_renderRapidaPremium` definita PRIMA di `init()`. La stub `renderRapida()` controlla `typeof _renderRapidaPremium === 'function'` e delega.

---

## Regole di compatibilità Android

| ❌ Vietato | ✅ Usare invece |
|---|---|
| `const`/`let` globali | `var` |
| `const`/`let` dentro funzioni | `var` |
| Template literals `` ` `` | concatenazione `'...' + var + '...'` |
| `readAsBinaryString` | `readAsArrayBuffer` + `Uint8Array` |
| `sheet_to_json({header:1})` | `decode_range` + `encode_cell` cella per cella |
| `Object.entries(map)` | `Object.keys(map)` + `map[key]` |
| Destructuring params `({a,b})=>` | `function(obj){ var a=obj.a, b=obj.b; }` |
| Arrow functions in forEach | `function(x){...}` |
| Surrogati unicode `\ud83e\udd1d` | emoji UTF-8 diretti 🤝 |
| `window.renderRapida = fn` | `var _renderRapidaPremium = fn` (nome privato) |

---

## Tab e viste

### Tab Trasporti
- **Vista ⚡ Rapida** — card `.rrow` con chip per trasportatore, best price in ambra (`_renderRapidaPremium`)
- **Vista 📊 Confronto** — barre proporzionali (`_renderConfronto`)
- Autocomplete DOM-based partenza/destinazione (`acSearch`, `acOpen`)
- Pill filtro trasportatori (`_renderCarrierPillsPremium`)
- Click su card → `jumpToConfronto()` → apre Confronto sulla tratta
- ✏️ Modifica tratta → `openEditTratta(from, to)` → modal con tutti i trasportatori
- Click su singolo prezzo → `openEditSingle(idx)` → modal singola riga

### Tab Prodotti
- **🌾 Prodotto** — `renderViewProdotto()` — card con sezioni ACQUISTO/VENDITA, best evidenziato
- **🏭 Fornitore** — `renderViewFornitore()` — entity-card bordo rosso, click → `openEntityModal`
- **🚜 Cliente** — `renderViewCliente()` — entity-card bordo verde, click → `openEntityModal`
- Filtro pill prodotti scroll orizzontale (`.ptabs-wrap`)
- Autocomplete modal aggiunta prodotto (prodotto, entity, luogo con auto-fill)
- Radio button Acquisto/Vendita

### Tab Import
- `handleFileImport` → `readAsArrayBuffer` + `XLSX.utils.decode_cell` (matrice completa)
- `importTrasporti` — auto-detect righe carrier/condizioni/dati (scansione prime 5 righe)
  - Struttura file: R1=data+carriers, R2=labels, R3=condizioni, R4+=dati
  - Parser gestisce range "500-550" → prende il primo valore (500)
- `importProdotti` — crea UN record per ogni riga (acquisto E vendita separati)
  - Struttura KarMa: PRODOTTO, Fornitore, Provenienza, Destinazione, Acquirente, Costo Acquisto, Prezzo Vendita
  - record vendita include `note: 'da [fornitore]'` per tracciabilità
- Pulsante **"🔄 Carica Tariffe REV 03/2026"** → `reloadBuiltinData()` (carica PDF_DATA)
- Pulsante **"🌾 Carica Listino KarMa 03/2026"** → `loadKarmaData()` (carica KARMA_DATA)

---

## Google Drive

- GIS caricato **lazy** solo al click del pulsante Drive (no flash bianco su Android)
- `handleDriveClick()` → `loadGISandRun()` → `driveLogin()`
- `driveSave(silent)` — cerca file esistente, aggiorna o crea in `appDataFolder`
- Auto-save debounce 2s dopo ogni `save()` (solo se `driveAccessToken` presente)
- `driveLoad()` — scarica e merge dati da Drive
- Token NON persistito — richiede login ad ogni sessione

---

## CSS palette

```css
--bg:#0b0f16; --s1:#111927; --s2:#0f1520; --s3:#1a2436;
--green:#34d399; --amber:#fb923c; --red:#f87171; --blue:#60a5fa;
--t1:#f0f6ff; --t2:#8ba3c0; --t3:#4a6080;
--b1:rgba(255,255,255,.08);
```

### Componenti CSS chiave
`.rrow`, `.chip`, `.chips`, `.rrow-best`, `.rrow-head`, `.rrow-route`  
`.price-row`, `.entity-card.forn/.clie`, `.prow`, `.prow-head`, `.prow-section-hd`  
`.erow`, `.erow-prov-label`, `.pvbtn`, `.pvbtn.pv-prodotto/fornitore/cliente.active`  
`.drive-panel`, `.dp-btn`, `.mp-ac`, `.ac-dropdown`, `.entity-modal-row`  
`.prodotti-tab`, `.ptabs-wrap`, `.modal-backdrop`, `.modal`

---

## Modal attivi

| ID | Funzione apertura | Descrizione |
|---|---|---|
| `modalEditTratta` | `openEditTratta(from,to)` / `openEditSingle(idx)` | Modifica prezzi tratta |
| `modalProdotto` | `openAddProdottoModal(prodotto?)` | Aggiungi prezzo prodotto |
| `modalEntity` | `openEntityModal(name, tipo)` | Dettaglio fornitore/cliente |
| `modalExport` | `openExportModal()` | Export Excel/Stampa A3 |
| `drivePanel` | `toggleDrivePanel()` | Pannello Google Drive |

---

## Stato audit (31/03/2026)

- ✅ Zero funzioni mancanti (24+ funzioni critiche verificate)
- ✅ Zero duplicati
- ✅ `_renderRapidaPremium` prima di `init()`
- ✅ CSS bilanciato
- ✅ Sintassi JS pulita (Node --check)
- ✅ Runtime test zero crash
- ✅ 188 prezzi tariffe (1 anomalia risolta: CIRIONI range 500-550→500)
- ✅ 54 prezzi KarMa senza anomalie

---

## File Excel di riferimento

| File | Struttura |
|---|---|
| `Tariffe_trasporti_2026_rev_03_16_03_2026.xlsx` | R1=carriers, R2=labels, R3=condizioni, R4+=dati, 63 tratte × 18 carrier |
| `Listino_KarMa_03_2026.xlsx` | PRODOTTO/Fornitore/Provenienza/Destinazione/Acquirente/CostoAcquisto/PrezzoVendita |

---

## Pending

- [ ] Aggiornamento dati quando arriva REV 04/2026 (sostituire PDF_DATA e KARMA_DATA)
- [ ] Test modifica singola tratta su Android
- [ ] Verifica funzionamento Drive su mobile dopo lazy-load GIS
- [ ] Gestione prezzi con unità miste per stesso fornitore (€/mc + €/ton)
