# ARCHIVIO TARIFFE — Project Documentation

**Repository:** `firstlex55/ARCHIVIO-KARMA-`  
**URL:** `https://firstlex55.github.io/ARCHIVIO-KARMA-/`  
**File principale:** `index.html` (single-file app, ~5450 righe)  
**Versione app:** v1.0.0  
**Azienda:** Pro Trasporti Srl  

---

## File nel Repository

| File | Descrizione |
|------|-------------|
| `index.html` | App completa (single-file) |
| `sw.js` | **Service worker** (giro 4), **opzionale e NON in uso** (Fil ha deciso di non usarlo, dal giro 5 non si consegna più). Se manca l'app funziona uguale (la registrazione fallisce in silenzio), solo non si apre offline |
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
- **Nessun Service Worker** (la versione precedente di questo file diceva il contrario, ma nel codice non c'è): l'app NON funziona offline perché SheetJS e Google si caricano dal web. Da fare (vedi backlog).

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
  "karmaHidden": { "<chiave KarMa>": 1 },
  "lastUpdate": null
}
```

- `pdfLoaded`: le tariffe incorporate si caricano **solo al primo avvio**. Se l'utente svuota, resta vuoto.
- `karmaRev`: il listino KarMa si (ri)applica solo quando questa non coincide con `KARMA_REV` nel codice (nuova revisione del listino), mai perché "mancano dei record".
- `karmaHidden`: chiavi (`karmaKey`) dei prezzi KarMa cancellati dall'utente; il listino non li rimette. Viene salvato anche su Drive.

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
`rev` vale **`"Manuale"`** per ogni tariffa inserita **o modificata a mano** (dal giro 5 anche la "Modifica tratta"/"Modifica prezzo" la imposta, ma solo se un campo è davvero cambiato: `trSig()`). Le tariffe `Manuale` sono protette dal "Ricarica REV 03/2026" e dall'import Excel (mai sovrascritte). ⚠️ Le modifiche fatte **prima** del giro 5 non hanno questo segno e non si distinguono da quelle incorporate.

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
  "source": "karma",
  "kk": "cippato|acquisto|ardenghi|dosolo (mn)|",
  "edited": true
}
```

**source**:
- `"karma"` — dal listino incorporato `KARMA_DATA`. Ha `kk` (chiave stabile del listino, `karmaKey()`); `edited:true` se l'utente ne ha cambiato i valori (un aggiornamento di revisione non lo sovrascrive).
- `"manual"` — inserito dal modulo. I record vecchi senza `source` vengono marcati così da `normalizeDB()`.
- `"import"` — importato da Excel (`importProdotti`).

Manuali e importati **non vengono mai toccati** dal ricaricamento del listino KarMa.  
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
- `builtinTariffe()` (primo avvio e `reloadBuiltinData()`): tutti i record ricevono `data: '2026-03-01'`, destinazioni normalizzate
- `reloadBuiltinData()` chiede conferma e **conserva le tariffe con `rev:'Manuale'`** (inserite o modificate a mano); le altre (incorporate o importate da Excel) vengono sostituite

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

**Layout chip (sessione 11-06) — dal Giro 7 vale solo per le viste Fornitore e Cliente; la vista Prodotto è descritta nel Giro 7**
- Struttura identica ai chip della tab Trasporti (stesso layout, stesse dimensioni testo)
- **Acquisto:** verde `#34d399` — bordo sinistro `rgba(52,211,153,.6)`, sfondo `rgba(52,211,153,.07)`, section header verde
- **Vendita:** ambra `#fb923c` — bordo sinistro `rgba(251,146,60,.6)`, sfondo `rgba(251,146,60,.07)`, section header ambra
- Entity 13px font-weight 700, loc 11px blu `#7eb8f7`, note 11px `#93c5fd` corsivo
- Prezzo acquisto verde, prezzo vendita ambra
- **Pill data:** `prezzo del GG/MM/AA` con pallino — identica alla pill dei chip tratte
- **Niente avatar** — rimosso il quadrato con iniziali
- Best acquisto badge `★ min`, best vendita badge `★ max`

**Vista Prodotto** (default)
- **Dal Giro 7**: schede prodotto chiuse/aperte con icone SVG, due schede interne "Dove compro" / "A chi vendo", €/ton e €/mc in gruppi separati, data "prezzo del …" su ogni riga
- Tocca riga → **menu dettagli** (`openProdSheet`) → Modifica / Condividi / Elimina; la modifica è sempre il modal `openEditProdotto(idx[, idxs])`
- Stat cards colorate: Prodotti=verde, Fornitori=rosso, Clienti=ambra

**Vista Fornitore / Vista Cliente**
- Raggruppa per entità

**Modal Aggiunta Prodotto (+)**
- Campi: Prodotto, Tipo, Fornitore/Cliente, Luogo, Prezzo, Unità, Note, Data

**Modal Modifica Prodotto** (`openEditProdotto(idx)`, `saveEditProdotto()`, `deleteEditProdotto()`)
- Campi: Prodotto, Tipo, Fornitore/Cliente, Luogo, Prezzo, Unità, 📅 Data rilevazione, Note
- Pulsante 🗑 Elimina

### Tab Import

- Import Excel trasporti **con controllo prima di scrivere** (giro 5, vedi sezione "Giro 5"): finestra "📋 Controllo del file" con Nuove / Già presenti / Prezzo diverso / Non lette; si aggiungono le nuove e, per le tariffe già presenti con prezzo diverso, c'è la scelta in cima alla finestra **"Mantieni le mie" (di partenza) / "Sovrascrivi"**. Le tariffe che nel file non ci sono non si toccano mai. Con "Sovrascrivi" le tariffe inserite/modificate a mano (`rev:'Manuale'`) e quelle con data più recente del file restano comunque com'erano, a meno che si spunti la casella "Sovrascrivi anche le N inserite o modificate da me, o più recenti". Il vecchio selettore Aggiungi/Aggiorna non c'è più
- Import Excel prodotti (⚠️ **superato dal giro 9**: ora si controlla prima e si può sovrascrivere; il testo che segue descrive la versione di prima — invariato: **solo aggiunge**, non sovrascrive mai; record con `source:'import'`, `prodKey` contro i doppioni). ⚠️ Provato sul listino `Listino_KarMa_03.2026.xls` sopra ai prezzi KarMa già incorporati: ne aggiunge ~42 doppi (le voci incorporate hanno in `note` la destinazione "→ …"/"da X (luogo)", quindi `prodKey` non coincide) e di "56+M%" tiene solo 56. Non corretto (non richiesto): vedi Backlog
- Import prodotti: record con `source:'import'`; un file ripetuto non crea doppioni (`prodKey`)
- 🔄 Ricarica tariffe REV 03/2026 (da PDF_DATA, setta `data:'2026-03-01'`; chiede conferma, conserva le tariffe a mano)
- 🌾 Carica listino KarMa (da KARMA_DATA; chiede conferma: i prezzi KarMa modificati/cancellati tornano originali, manuali e importati restano)
- ⬇️ Export Excel — modal con selezione tipo:
  - **🚛 Tariffe Trasporti**: filtri da/a/trasportatori, formato Excel o A3
  - **🌾 Prezzi Prodotti**: sheet "Tutti i prezzi" + sheet per ogni prodotto
- 🗑 Svuota manuali (`clearProdottiManuali()` — toglie manuali + importati, mantiene `source='karma'`)
- 🗑 Svuota tutto (`clearAll()` — l'archivio **resta vuoto** anche dopo il riavvio; per ripristinare usare i due pulsanti di ricarico)
- Card **💾 Backup e annulla**: Salva backup ora (file JSON), Annulla ultima modifica, Ripristina da un file di backup (l'import chiede conferma e rifiuta file senza tariffe/prodotti)

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

### Google Drive (riscritto nel giro 4 — vedi "Giro 4" per i dettagli)
- Client ID: `107091966360-vbepp0lmghbck14vv89et30acl34d8a9.apps.googleusercontent.com`
- GIS caricato lazy al primo click (`loadGISandRun`); salva/carica in `appDataFolder`
- Auto-save 2 s dopo ogni modifica, **solo** se il permesso è valido (`driveTokenValid()`)
- `driveCheck(r)`: ogni risposta di Google deve essere `ok`, altrimenti è un errore (prima un 401 veniva mostrato come "✅ Salvato")
- Prima di ogni salvataggio si controlla la data del file su Drive (`driveSyncCheck`): se un altro dispositivo ha salvato nel frattempo → unione automatica o finestra di conflitto
- Il permesso dura ~1 ora: alla scadenza il chip diventa 🔑, un tocco riconnette senza password (`requestAccessToken({prompt:''})`); il token è salvato in `archivio-drive-token` e dopo un ricaricamento si riprende da solo
- Offline: chip 📴, le modifiche restano sul telefono (`archivio-drive-dirty`) e partono da sole al ritorno della rete
- Payload su Drive: `trasporti`, `prodotti`, `lastUpdate`, `karmaHidden`, `karmaRev`, `pdfLoaded`, `savedAt`

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
- **renderViewProdotto** (via `prodIndexOf`) usa `DB.prodotti.indexOf(p)` con fallback by-value; il menu dettagli tiene gli indici e le righe (`prodSheetValid` controlla che non siano cambiati)
- **renderEditPricesList** idem per `DB.trasporti.indexOf(p)`
- **init()** chiamata esplicita alla fine del JS
- **KARMA_DATA** si applica solo se `DB.karmaRev !== KARMA_REV` (nuova revisione) — mai in base al numero di record. Per una nuova revisione del listino: cambiare `KARMA_REV` e `KARMA_DATA`; `applyKarma(false)` rispetta i prezzi KarMa modificati/cancellati dall'utente
- **Testo nell'HTML**: ogni testo dell'utente (trasportatore, tratta, note, fornitore…) va passato da `esc()`; ogni stringa dentro `onclick="f('…')"` da `jq()`. Mai concatenare nomi grezzi (un apostrofo o una virgoletta rompe il pulsante o tronca il campo)
- **selCarriers** filtrato con loop `for`, mai `.includes()` su array
- **Colori**: mai esadecimali o `rgba(0,0,0,…)` fissi nel CSS dell'app; usare sempre le variabili del tema (altrimenti il tema chiaro/scuro si rompe). Eccezione voluta: il popup di stampa A3 (`printA3`) ha i suoi colori fissi, perché stampa su carta bianca
- **Pesi**: usare al massimo `font-weight:700` (600 per i titoli, 500 per le etichette). Mai 800/900: stancano e sono troppo marcati
- **Tema**: dopo ogni modifica grafica controllare entrambi i temi (scuro e chiaro) e rilanciare l'audit di contrasto (rapporto ≥ 4.5 per il testo normale)

### CSS Chiavi
| Variabile/Classe | Uso |
|-----------------|-----|
| `--bg`, `--s1`, `--s2`, `--b1`, `--b2` | sfondo pagina, superfici, bordi (cambiano col tema) |
| `--t1`, `--t2`, `--t3` | testo primario/secondario/terziario |
| `--amber`, `--green`, `--blue`, `--red` | colori semantici (prezzo/avviso, minimo/acquisto, info, errore) |
| `--amber-rgb`, `--green-rgb`, `--shade-rgb`… | stessi colori come canali "r,g,b": si usano come `rgba(var(--green-rgb),.15)` per sfumature |
| `--hd1`, `--hd2` | gradiente dell'header |
| `.prow-*` | classi chip prodotti (layout identico a `.chip-*`), usate dalle viste secondarie; la vista Prodotto usa `pc-*`, `pr-*`, `psh-*` (Giro 7) |
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
- [x] ~~Condividi tratta su WhatsApp~~ (giro 4)
- [x] ~~Annulla dopo le cancellazioni + backup in un tocco~~ (giro 4)
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

## Modifiche Sessione 19-10-2026

### Bug gravi corretti (perdita dati / prezzi mancanti)
1. **Cancellazione**: il pulsante Elimina del modal prezzo singolo e il 🗑 sulle card chiamavano `deleteTratta()`, che cancellava TUTTI i prezzi della tratta. Ora:
   - `deleteSinglePrice()` elimina solo il record aperto (conferma con trasportatore, prezzo e tratta)
   - `deleteTratta()` elimina l'intera tratta solo da "Modifica tratta" (conferma con il numero di prezzi)
   - `deleteFromEditModal()` sceglie da sola quale usare; l'etichetta del pulsante cambia ("Elimina prezzo" / "Elimina tratta")
   - rimosso il 🗑 dalle card della vista Rapida
2. **Prezzi €/mc nascosti**: le viste Rapida e Confronto mostravano solo `€/vg` e `€/ton`, quindi i prezzi `€/mc` (o altre unità) sparivano. Ora `groupByUnit()` raggruppa tutte le unità (ordine vg, ton, mc, altre). (Questa NON era la causa dei prezzi mancanti segnalati: vedi punto successivo.)
2b. **Tendina Da/A che non scorre e taglia le basi (causa vera dei prezzi mancanti)**. Tre difetti insieme:
   - `.search-card` aveva `overflow:hidden`: la tendina veniva ritagliata dopo poche righe → ora `overflow:visible` e `.search-card-bar` con angoli arrotondati propri
   - `acRender` mostrava solo 14 voci (`slice(0,14)`) ma le basi di carico sono 27: da Grezzana in poi non comparivano mai (stesso problema con `slice(0,10)` nelle tendine del modal Prodotto, dove i luoghi sono 43) → ora si mostrano tutte, con indicatore "N voci · scorri per vederle tutte"
   - ogni voce faceva `preventDefault()` su `touchstart`: al primo contatto col dito la voce veniva scelta e lo scroll bloccato → ora `acBindPick()`: tocco breve = seleziona, trascinamento = scorre
   - altezza adattata allo spazio visibile sopra la tastiera (`acFit`, `acReveal`, listener `visualViewport`); scroll fluido (`-webkit-overflow-scrolling`, `overscroll-behavior:contain`); `acSafeClose` non chiude la tendina mentre il dito scorre
   - suggerimenti trasportatore ("🚛 NOME"): il simbolo veniva copiato nel campo e la ricerca non trovava nulla → ora `acSelect` lo toglie
3. **Modal prezzo singolo**: aggiunta la lista di tutti i prezzi della tratta (tap su una riga per passare a quel prezzo). Era documentata ma nel codice non c'era.
4. **Unità convertite di nascosto**: i select unità dei modal avevano solo vg/ton, quindi modificare un prezzo `€/mc` lo trasformava in `€/vg` al salvataggio. Ora `unitOptionsHtml()` include `€/mc` e conserva qualunque unità esistente (anche in "Aggiungi tariffa").
5. Intestazione vista Rapida: ora mostra "N tratte · N prezzi" per controllare a colpo d'occhio che i conteggi tornino.

### Pulizia codice
- Rimosso tag CSS troncato `</sty` (invalidava la regola `.chip-section-label`)
- `resolveConflict` riscritta senza `async/await` e con `_applyDriveData()` (prima richiamava funzioni inesistenti)
- `new Set(...)` in `openEntityModal` sostituito con loop su oggetto

### Funzioni nuove
| Funzione | Descrizione |
|----------|-------------|
| `unitRank(u)` / `UNIT_ORDER` | ordine fisso delle unità |
| `unitOptionsHtml(cur)` | `<option>` per i select unità, conserva l'unità corrente |
| `groupByUnit(prices)` | raggruppa e ordina i prezzi per unità |
| `deleteSinglePrice()` | elimina un solo record |
| `deleteFromEditModal()` | instrada Elimina prezzo / Elimina tratta |

### Giro 2 — Audit del codice e correzioni sulla sicurezza dei dati
Audit completo con simulazione dei problemi sul codice vero (harness `vm` + Chromium/Playwright su profilo telefono). Problemi riprodotti e corretti:

1. **Cancellare un prezzo KarMa faceva ricaricare tutto il listino** (condizione `karmaInDB < karmaExpected` in `init`): il prezzo cancellato ricompariva e le modifiche agli altri prezzi KarMa venivano azzerate. Ora il listino si applica solo se cambia `KARMA_REV`; le cancellazioni sono ricordate in `DB.karmaHidden`, le modifiche segnate con `edited:true` (`applyKarma`, `forgetKarma`).
2. **Prodotti importati da Excel senza `source`** venivano persi a ogni ricarico del listino KarMa. Ora `source:'import'`; `normalizeDB()` marca come `manual` i record vecchi senza origine. L'import non crea più doppioni (`prodKey`).
3. **"Svuota tutto" non svuotava** (al riavvio tornavano le 189 tariffe): ora `pdfLoaded`/`karmaRev` impostati, restano vuoti. Stesso per "Elimina tutti i prodotti".
4. **"Carica tariffe incorporate" cancellava le tariffe inserite a mano**: ora conserva quelle con `rev:'Manuale'` (e se una tariffa PDF è stata modificata a mano, vince la modifica). Conferma con il numero esatto.
5. **Drive**:
   - un errore di Google (401/403/5xx) veniva mostrato come "✅ Salvato su Drive": ora `driveCheck`/`driveFail`
   - "Carica da Drive" sostituiva tutto senza chiedere (nel test: 190 → 1 tariffe): ora conferma con i numeri; rifiuta file senza dati
   - `_applyDriveData` e `loadJSON` non fanno più ricomparire i dati incorporati sopra i dati caricati; `loadJSON` chiede conferma e valida il file
6. **Apostrofi nei pulsanti**: cercando "colle" 4 risultati su 7 non rispondevano al tocco ("Colle Val d'Elsa (SI)"); stesso per i filtri prodotto e le pill. Nuovo `jq()` per le stringhe negli `onclick`.
7. **Testo non protetto nell'HTML**: nuovo `esc()` applicato alle viste principali (Rapida, Confronto, Prodotti per prodotto/fornitore/cliente, ricerca globale, modal modifica, export). Prima una virgoletta nelle condizioni di pagamento troncava il campo e al salvataggio troncava il dato.
8. Piccoli: tolto `maximum-scale=1.0` dal viewport (ora si può ingrandire); etichette import chiarite ("Aggiungi: solo le tariffe nuove" / "Aggiorna: sovrascrive i prezzi già presenti"; prima "Sostituisci — Cancella e reimporta" non era vero); le tariffe incorporate del primo avvio ora hanno la data come quelle ricaricate.

#### Funzioni nuove (giro 2)
| Funzione | Descrizione |
|----------|-------------|
| `esc(s)` / `jq(s)` | testo sicuro per HTML / per stringhe dentro `onclick` |
| `normalizeDB()` | rende coerente il DB dopo ogni caricamento (array, `karmaHidden`, `source`, `kk`) |
| `applyKarma(reset)` | applica il listino KarMa; `reset=true` = ricarica esplicita, `false` = aggiornamento di revisione |
| `karmaKey(k)` / `prodKey(p)` | chiave stabile di un prezzo KarMa / di un prodotto (anti-doppioni) |
| `forgetKarma(p)` | ricorda che un prezzo KarMa è stato cancellato |
| `builtinTariffe()` | tariffe incorporate pronte (data, destinazioni normalizzate) |
| `driveCheck(r)` / `driveFail(e, what)` | errori Drive veri e avvisi |

Costanti: `KARMA_REV`, `PDF_REV`.

### Giro 3 — Tema più morbido: scuro (predefinito), chiaro, automatico
> **Nota**: la palette "morbida" descritta qui sotto (blu-grafite `#151a22`) è stata **sostituita nel giro 6** — il tema scuro è tornato identico a prima del giro 3. Restano validi tutti i meccanismi (variabili, interruttore, attributi, anti-lampo, audit).

Motivo: nero pieno contro testo bianco pieno stanca la vista (alone intorno alle lettere), soprattutto sul telefono per ore.

- **Palette**: `:root` = scuro morbido (sfondo blu-grafite `--bg:#151a22`, testo non bianco puro); `:root[data-theme="light"]` = chiaro morbido (sfondo grigio-azzurro `--bg:#e4e7ec`, non bianco puro). Tutti i colori dell'app sono variabili (vedi "CSS Chiavi"); i colori esadecimali fissi sono stati sostituiti
- **Scelta dell'utente**: pulsante tondo nell'header (luna / sole / mezzo cerchio) che alterna **Scuro → Chiaro → Automatico** (segue il telefono). Salvata in `localStorage['archivio-theme']` (`dark` | `light` | `auto`)
- **Attributi su `<html>`**: `data-theme` = tema in uso (`dark`|`light`, già risolto), `data-theme-pref` = scelta (`dark`|`light`|`auto`; guida l'icona del pulsante)
- **Nessun lampo**: uno `<script>` piccolo subito dopo il meta viewport imposta gli attributi prima del primo disegno; `color-scheme` e `<meta name="theme-color">` seguono il tema (barra di stato di Android). In modalità automatica un listener `matchMedia('(prefers-color-scheme: light)')` aggiorna la pagina quando cambia il telefono
- **Contrasto**: audit automatico in Chromium su 3.723 testi — con i colori vecchi 670 sotto soglia, ora 0 in entrambi i temi (regola finale in fondo al CSS per le etichette ambra delle unità)
- **Stampa A3**: non cambia (colori propri, non usa le variabili)
- **Header stretto**: la scritta `v1.0.0` si nasconde sotto i 430px, il sottotitolo si stringe sotto 380/350px (verificato a 340, 360, 393, 412px). Le media query vanno **dopo** le regole `.h-*` base, altrimenti vengono sovrascritte

- **Pesi del carattere (variante "B", scelta da Fil)**: il grassetto 800/900 su tutto (circa 75 punti) risultava troppo pesante. Mappa applicata a tutto il CSS e agli stili inline: 900→700, 800→700, 700→600, 600→500, 500 e 400 invariati. Il carattere resta quello di sistema (`-apple-system, "SF Pro Display", "Segoe UI", sans-serif`). **Il popup di stampa A3 è escluso** (mantiene i suoi pesi 700/900). Regola per il nuovo codice: massimo `font-weight:700`, niente 800/900. Anteprime A/B/C/D confrontate in Chromium con Inter al posto del font del telefono; su telefono la resa può differire leggermente

#### Funzioni nuove (giro 3)
Costanti: `THEME_KEY` (`'archivio-theme'`), `THEME_NAMES`.

| Funzione | Descrizione |
|----------|-------------|
| `themePref()` | legge la scelta salvata (default `dark`) |
| `themeResolve(pref)` | `auto` → `dark`/`light` secondo il telefono |
| `applyTheme(pref)` | imposta gli attributi su `<html>`, `meta theme-color` e l'etichetta del pulsante (non salva) |
| `cycleTheme()` | pulsante header: Scuro → Chiaro → Automatico; salva la scelta e mostra un avviso |
| `themeInit()` | primo comando di `init()`: applica il tema e aggancia il listener `matchMedia` |

### Giro 4 — Annulla + backup, Condividi su WhatsApp, Drive affidabile, uso offline
Richiesta di Fil: "1, 2 e 8" dell'elenco di migliorie. Collaudo: 5 suite vm (test.js 13, test2.js 14, test3.js 16, test5.js 76, **test6.js 126**), **test_sw.js 18**, Chromium (browser_test 20, browser_theme_test 15, **browser_giro4_test 25** con service worker vero e rete spenta) e audit di contrasto delle nuove schermate (0 problemi nei due temi a 340 e 393 px).

#### 1. Annulla e backup
- **Avviso con pulsante** — `showToast(msg, type, act)`: `act = {label, fn}` aggiunge il pulsante dentro l'avviso (`.toast-act`, l'avviso diventa cliccabile, 9 s invece di 3,2 s)
- **Pila di annullamento** — prima di ogni operazione distruttiva si fa `var snap = undoSnapshot()`, dopo l'operazione `undoToast(etichetta, snap, messaggio, tipo)`: mostra l'avviso con **↩️ Annulla**. In memoria le ultime 10 (`_undoStack`), l'ultima è salvata anche in `archivio-tariffe-undo` (sopravvive al ricaricamento). Si può annullare anche dopo che l'avviso è sparito: Import → **Annulla ultima modifica** (`undoLast()`, mostra cosa e quando)
- **Operazioni con Annulla**: elimina prezzo/prodotto, modifica prodotto, svuota prodotti / manuali / tutto, carica KarMa, ricarica tariffe REV, ripristina JSON, importa tariffe (solo se qualcosa è cambiato), importa prodotti (solo se ci sono nuovi), elimina tratta, elimina singolo prezzo, salva tratta (modo singolo e multiplo), scelte nel conflitto Drive. **Regola per il nuovo codice**: ogni operazione che cancella o sovrascrive dati deve prendere `undoSnapshot()` prima e chiamare `undoToast()` dopo
- **Backup** — `exportData()` scarica `archivio-tariffe-backup-AAAAMMGG….json` (contiene `app` e `exportedAt`) e salva la data in `archivio-last-backup` (`BACKUP_KEY`); `backupInfo()` / `backupRefreshUI()` mostrano "Ultimo backup: GG/MM/AAAA (oggi / ieri / N giorni fa)" in ogni elemento `.backup-info` (in ambra se mai fatto o più vecchio di 14 giorni). Pulsante **Salva backup ora** nella card Import → "💾 Backup e annulla" e **Backup sul telefono (file)** nel pannello Drive. Ripristino: `importDataJSON()` apre il selettore file, `loadJSON()` legge e chiede conferma. I due vecchi pulsanti JSON della card "Gestione dati" sono stati tolti
- Se il telefono ha la memoria piena, `save()` avvisa ("❌ Memoria del telefono piena…") invece di perdere i dati in silenzio

#### 2. Condividi tratta (WhatsApp)
- Pulsante **📤 Condividi** su ogni scheda in vista Rapida e **📤 Condividi tratta** in vista Confronto → `openShareTratta(from, to)` apre `modalShare`
- Elenco dei prezzi della tratta con spunta (togli quelli che non vuoi mandare), opzioni **Nomi trasportatori / Condizioni / Data tariffa**, testo modificabile (`#shareText`). `shareBuild()` costruisce il testo (`*Da → A*`, un punto elenco per prezzo, `Tariffe del GG/MM/AA`; prezzi con virgola italiana, `€/vg` → `€/viaggio`)
- Invio: `shareSend('wa')` apre `https://wa.me/?text=…`; `shareSend('sys')` usa `navigator.share` (menu "Altre app"); `shareCopy()` copia (con ripiego `execCommand` se la clipboard non è permessa)
- Nel confronto, `_renderConfronto` ora passa le città da `esc()`/`jq()`

#### 3. Drive: conflitti, riconnessione, offline
**Chiavi localStorage**
| Chiave | Contenuto |
|--------|-----------|
| `driveConnected` | `'1'` se Drive è stato collegato su questo telefono |
| `archivio-drive-token` | `{t, e}`: token Google e scadenza (ms). `e = 0` = scadenza ignota, si considera valido finché Google non risponde 401 |
| `archivio-drive-sync` | `modifiedTime` dell'ultimo file Drive visto/scritto da questo telefono |
| `archivio-drive-dirty` | `'1'` = ci sono modifiche locali non ancora su Drive |
| `archivio-drive-base` | mappa di impronte dell'ultimo stato comune telefono↔Drive (base per l'unione a tre vie) |
| `archivio-tariffe-undo`, `archivio-last-backup` | vedi punto 1 |

**Stati del chip header / pannello** (`setDriveStatus(state)`, `_driveState`): `''` (non connesso) · `connected` ☁️ · `syncing` · `error` · `expired` 🔑 (permesso scaduto: un tocco riconnette) · `offline` 📴. Il chip è cliccabile (`driveChipClick()`): se scaduto riconnette subito, altrimenti apre il pannello.

**Flusso di salvataggio** (`save()` → autosave 2 s → `driveSave(silent)`):
1. `driveSyncCheck()` legge solo i metadati del file (`driveFetchMeta`: `id`, `modifiedTime`) e confronta con `archivio-drive-sync`
2. Drive invariato → carica (`driveDoUpload`, PATCH con `fields=id,modifiedTime`, poi `driveRemember`)
3. Drive cambiato da un altro dispositivo e **qui nessuna modifica** → adozione automatica (`driveAdopt`), con avviso
4. Drive cambiato **e** modifiche locali → se i due contenuti sono uguali (`driveSameContent`) si allinea; altrimenti **unione automatica** a tre vie; la finestra `modalDriveConflict` compare solo se serve scegliere (`driveShowConflict` / `resolveConflict('merge' | 'local' | 'drive' | 'later')`) — nella finestra "Unisci i due" è il pulsante consigliato
5. Ogni salvataggio manda `savedAt`; la sessione scaduta → chip 🔑, `archivio-drive-dirty` resta `'1'`; offline → chip 📴 e ripartenza da sola al ritorno della rete (listener `online`, `offline`, `visibilitychange`, con pausa di 60 s fra un controllo e l'altro)

**Unione a tre vie** (`driveMerge(remote)` → `{added, changed, removed}`, `mergeList`, `sigMap`, `driveSaveBase/driveLoadBase`)
- Le tariffe si riconoscono dalla chiave `tkey` = da + a + trasportatore + unità; i prodotti da `pkey`; `sigT`/`sigP` sono le impronte del contenuto
- Si confronta **base** (ultimo stato comune) con **locale** e **Drive**:
  - nuovo solo su un lato → si tiene
  - cancellato su un lato e non toccato sull'altro → **resta cancellato** (non ricompare)
  - cancellato su un lato ma modificato sull'altro → si tiene la modifica
  - modificato su un solo lato → vince quel lato
  - modificato su entrambi → vince la `data` più recente (a parità, questo telefono)
- Senza base (primo collegamento, nuovo telefono) l'unione è a due vie (unione dei due elenchi, in caso di differenza vince la data più recente); l'avviso dice quante tariffe sono state aggiunte, cambiate o tolte, e c'è sempre **Annulla**
- La base viene salvata dopo ogni allineamento riuscito (adozione, stesso contenuto, upload) e cancellata da `driveDisconnect`

**Funzioni** (blocco prima di `// ─── START ───`)
| Funzione | Descrizione |
|----------|-------------|
| `driveInUse()` / `driveIsDirty()` / `driveTokenValid()` | collegato su questo telefono / ci sono modifiche non salvate / token presente e non scaduto |
| `driveStoreToken`, `driveArmExpiry`, `driveExpire` | salva il token, programma la scadenza, passa a 🔑 |
| `driveLogin(reconnect)`, `driveReconnect`, `handleDriveClick`, `driveChipClick`, `driveDisconnect`, `driveRestore` | collegamento (con `prompt:''` = senza password se possibile), disconnessione, ripresa all'avvio da `init()` |
| `driveGetClient`, `driveOnToken`, `driveTokenError` | client GIS con `error_callback` (popup chiuso/bloccato → avviso chiaro) |
| `driveHeaders`, `driveFetchMeta`, `driveDownload`, `drivePayload`, `driveUploadFile`, `driveRemember` | chiamate all'API Drive (ognuna passa da `driveCheck`) |
| `driveSyncCheck(reason, silent)`, `driveDoUpload`, `driveAdopt`, `_applyDriveData`, `driveDiff`, `driveSameContent` | cuore della sincronizzazione |
| `driveSave(silent)`, `driveLoad()` | pulsanti del pannello (Salva / Carica); `driveLoad` chiede conferma e ha l'Annulla |
| `driveFail(e, what)`, `driveEnd`, `driveClean(seq)`, `driveFlashSaved` | errori, fine operazione, pulizia "non salvato" solo se non ci sono state altre modifiche nel frattempo (`_saveSeq`), lampo ✓ sul chip |
| `markDirty`, `showUnsavedBanner`, `hideUnsavedBanner`, `dismissUnsaved` | banner "modifiche non ancora su Drive" (ora compare anche se il permesso è scaduto) |

Variabili di stato: `_driveState`, `_driveBusy` (una sola operazione alla volta), `_driveRerun` (richiesta arrivata mentre era occupato → si ripete), `_driveSnooze` ("Decido dopo": niente popup finché non cambia qualcosa), `_driveConflict`, `_saveSeq`.

#### 4. Uso offline (service worker)
- **`sw.js`** (stessa cartella di `index.html`): all'installazione salva `index.html`, `manifest.json`, `icon-tariffe.png` (ognuno facoltativo: se uno manca non fallisce) e la libreria Excel da cdnjs
- **File dell'app** (`index.html`, manifest, icona): *prima la rete* (aggiornamenti subito), se non c'è rete o impiega più di 4 s → copia salvata. Se la rete risponde con un errore (500…) e c'è la copia, si usa la copia
- **SheetJS** (cdnjs): *prima la copia salvata*, poi la rete (`ignoreVary`). Non è più nel `<head>`: `loadXLSX(ok, fail)` la carica solo quando serve (import o export Excel); senza rete e senza copia spiega "Per esportare in Excel serve internet almeno una volta"
- **Google** (login e Drive): mai intercettato, passa sempre dalla rete. Anche POST e altri siti esterni non vengono toccati
- Registrazione: `swRegister()` all'evento `load`, solo su https o localhost. **Per forzare il rinnovo della memoria dopo aver cambiato `sw.js` cambiare `CACHE` (`archivio-tariffe-v1` → `v2`)**; le modifiche a `index.html` non lo richiedono
- Primo avvio con rete: l'app (e la libreria Excel) si salvano da sole; dalla seconda apertura in poi funziona anche in galleria/aereo
- **Deploy**: caricare su GitHub `index.html` **e** `sw.js` nella stessa cartella. Dopo il primo caricamento servono due aperture online perché il service worker prenda il controllo

#### Struttura dei test del giro 4 (cartella di lavoro, non nel repository)
`test6.js` (vm + finto server Drive: primo salvataggio, due dispositivi, conflitto, nuovo telefono, sessione scaduta, offline, corse tra richieste, unione a tre vie, loader Excel) · `test_sw.js` (service worker con cache e rete finte) · `browser_giro4_test.py` (Chromium vero: registra il service worker, spegne server e rete, ricarica; Annulla, Condividi, stati del chip, conflitto) · `audit_giro4.py` (contrasto nuove schermate, 340 e 393 px, scuro e chiaro). `test4.js` è vecchio e non funziona: ignorarlo.

#### Limiti noti / da sapere
- L'Annulla ripristina l'**intero** archivio com'era prima dell'operazione: se nel frattempo si sono fatte altre modifiche, vanno perse (l'avviso lo dice nel nome: "Annulla")
- Drive: se si modifica la stessa tariffa su due telefoni la data più recente vince; due telefoni con l'orologio sbagliato possono sbagliare vincitore
- Il pulsante "Carica Tariffe REV 03/2026 (incorporato)" a 340 px tocca il bordo destro (grafica vecchia, non toccata)

### Giro 5 — Import tariffe da Excel: controllo prima di scrivere
Richiesta di Fil: importare un file con molte tariffe, **non inserire quelle già presenti e a posto, inserire solo le nuove**. Il vecchio `importTrasporti` scriveva subito, confrontava con testo esatto (maiuscole, spazi, parentesi), segnava tutto "REV 03/2026" senza data, leggeva male alcune celle e con 3 trasportatori o meno non trovava la riga dei nomi.

**Flusso** (`handleFileImport` → `importTrasporti(data, filename, sheetName)` → finestra `modalImport` → `impApply()`):
1. Il file si legge (`sheetToMatrix`, max 5000 righe × 200 colonne). Se i fogli sono più d'uno si usa il primo in cui `impFindLayout` riconosce la tabella
2. `impParse(data)` produce un **piano** senza scrivere niente: prezzi letti, celle non lette, note
3. `impClassify(items)` confronta con l'archivio di adesso (si rifà anche in `impApply`, così un Drive che cambia i dati nel frattempo non crea errori)
4. La finestra mostra i conteggi e gli elenchi; **solo dopo "Importa"** si scrive, con `undoSnapshot()`/`undoToast()` (Annulla)

**Aggiunte dopo il collaudo sui file veri** (`Tariffe_trasporti_2026_rev_03_16.03.2026.xls` e `Listino_KarMa_03.2026.xls`, letti qui convertiti con LibreOffice → xlsx → matrice):
- **Richiesta esplicita di Fil**: *"non voglio che mi sovrascrivi quelli nuovi o che ho modificato"* → vedi "Protezione" qui sotto
- **Righe d'intestazione del file vero**: riga 1 = nomi dei 18 trasportatori (A1 = "Ferrara 16/03/2026", B1 = "REV 03/2026"), riga 2 = "Base di Carico | Luogo di Consegna" con note `MIN TAX 20`/`MIN TAX 18` sotto COAP e CONSAR, riga 3 = condizioni. Prima l'import prendeva la riga 2 come riga dei nomi (2 trasportatori finti "MIN TAX …")
- Le note (`impIsNote`: `MIN TAX …`, `* …`, `Nota…`) non sono mai nomi di trasportatori; `impFindLayout` sceglie fra il layout "per etichette" e quello "per conteggio" (riga con più nomi, escluse le righe di condizioni): vince quello con più colonne (`impLayoutByLabels` vs `impLayoutByCount`)
- **Intestazioni ripetute a ogni pagina** (righe con gli stessi nomi, es. righe 52-53 e 90 del file vero): ignorate in silenzio (`impIsHeaderRow`, nota nella finestra); la nota `* MIN TAX 9 ton` è riportata nella finestra ma non salvata
- **Asterisco di nota** sulle località (`Tarmassia IdS (VR) *`): tolto (`impStar`) per il confronto, come nell'archivio
- **Intervallo** `500-550 €`: si usa il **valore di mezzo** (525), come già fa l'archivio per CIRIONI Mercatino Conca → Ponzano R. (prima era il primo valore)
- **Numero senza unità sotto 100** (es. COAP `32,7`, `27,5`): letto come **€/ton** (stessa regola già usata da `builtinTariffe` per COAP); da 100 in su è €/vg; se l'unità è scritta vale quella. La finestra dice quanti
- **Risultato sul file vero**: 18 trasportatori, 189 prezzi, 0 non lette, **188 già presenti e 1 diverso** (COAP Bientina → Verolavecchia: file 32,7 €/ton, archivio 32,5 €/ton), 0 nuove

**Scelta "Mantieni le mie" / "Sovrascrivi"** (richiesta di Fil dopo il collaudo: *"quelle che ci sono già devono essere sovrascritte, le altre non devono essere toccate… mantieni tariffe oppure sovrascrivi tariffe"*). Due pulsanti affiancati sotto i riquadri dei conteggi (`impModeKeep`, `impModeOver`; compaiono solo se c'è almeno un prezzo diverso). Stato in `_impPlan.over` e `_impPlan.overProt` (si azzerano a ogni apertura: di partenza **Mantieni**, il pulsante dice "Niente da importare" se non ci sono nuove)
- **Mantieni le mie**: si aggiungono solo le nuove, i prezzi di adesso restano
- **Sovrascrivi**: le tariffe già presenti con prezzo (o condizioni) diverso prendono prezzo/condizioni/provvisorio/data/rev del file; **note e altri campi restano**. Quelle che nel file non ci sono non si toccano mai (l'import non cancella mai niente)
- La casella **"Sovrascrivi anche le N inserite o modificate da me, o più recenti"** compare solo con Sovrascrivi e se ci sono protette; spenta di default. Se acceso, anche quelle prendono il prezzo del file (il record sovrascritto è quello `Manuale`/più recente della chiave)
- Il pulsante dice "Importa: N nuove + M sovrascritte"; i titoli degli elenchi cambiano ("verranno sovrascritte" / "restano le tue" / "non toccate"); il riepilogo e l'Annulla dicono cosa è successo

**Elenchi della finestra** (grafica, richiesta di Fil dopo la consegna: "fai 4 e 5"):
- Ogni riga (`impRowHtml`/`impDiffRow`): a sinistra tratta e trasportatore (+ nota: condizioni cambiate, motivo 🔒), a destra il **prezzo nuovo in grassetto** e sotto "vecchio barrato ▲/▼ differenza". Il vecchio layout andava a capo su 3 righe a 340 px. Classi `.imp-rl`, `.imp-rr`, `.imp-rr1`, `.imp-rr2`, `.imp-r3`
- Gli elenchi (`impSecHtml(key,…)`, chiavi `diff`/`prot`/`new`/`skip`, id `impSec_<chiave>`) partono così: aperto solo "prezzo diverso" (serve una decisione), altrimenti "nuove"; il resto chiuso. Aprire/chiudere a mano toccando il titolo (`impSecClick`) si **ricorda** in `_impPlan.openSec` anche dopo un cambio di scelta o di data
- I riquadri **Nuove / Prezzo diverso / Non lette** sono pulsanti (`impTNew`, `impTDiff`, `impTSkip`): `impGoto(kind)` apre quell'elenco, chiude gli altri e ci scorre (`scrollIntoView` smooth). Se il numero è 0 il riquadro è spento (`disabled`). "Già presenti" non ha elenco. "Prezzo diverso" va alle "non toccate" se non ci sono altre differenze
- Titolo dell'elenco in modalità Mantieni: "N con prezzo diverso"; in Sovrascrivi: "N verranno sovrascritte"

**Protezione (le tariffe tue o più recenti si sovrascrivono solo se lo chiedi)** — `impProtect(list, refDate)` → `{why, rec}`, `impRefDate()`:
- Un "prezzo diverso" è **protetto 🔒** se, fra i record dell'archivio con la stessa chiave, **uno ha `rev==='Manuale'`** (inserita o modificata a mano) **oppure `data` più recente** (`>`, formato AAAA-MM-GG) della data del campo "📅 Data delle tariffe" (default: 1° del mese della REV del file, altrimenti oggi)
- Le protette sono in un elenco a parte con il motivo e il prezzo che resta; contano nel riquadro "Prezzo diverso" ma entrano nell'aggiornamento solo con la casella di cui sopra (`impToUpdate`). `impApply` aggiorna solo `impToUpdate(c, p)`
- Cambiando la data nella finestra (`onchange="impRender()"`) il confronto si rifà: una data più vecchia protegge di più, una più nuova di meno
- Le tariffe **uguali** non si toccano mai; le **nuove** (chiave assente) si aggiungono
- `saveEditTratta()` (modalità singola e multipla) ora marca `rev='Manuale'` le tariffe di cui cambia davvero prezzo/unità/condizioni/provvisorio/trattabile/nota/data (`trSig()`); prima non lo faceva, quindi le modifiche a mano non erano riconoscibili. Effetto collaterale voluto: anche "Ricarica REV 03/2026" ora le tiene (come dice già il suo messaggio)
- Limite: le modifiche fatte prima di questa versione non si possono distinguere; per quelle valgono anteprima, casella "Aggiorna" spenta di default e Annulla

**Confronto** — chiave `impKey` = base | destinazione | trasportatore | unità, normalizzata (`impNorm`: maiuscole, spazi doppi, spazi intorno alle parentesi). Il trasportatore del file si riporta all'ortografia già in archivio (es. `Branchini` → `BRANCHINI`). Esiti:
| Esito | Significato | Cosa succede |
|-------|-------------|--------------|
| **Già presente** | stessa chiave e stesso prezzo (±0,005); condizioni uguali oppure vuote nel file. Se in archivio la chiave ha più prezzi basta che uno coincida | non si tocca |
| **Nuova** | la chiave non c'è (anche: stessa tratta ma altra unità €/vg · €/ton · €/mc) | si aggiunge |
| **Prezzo diverso** | la chiave c'è, il prezzo (o le condizioni di pagamento) cambia | si mostra "vecchio → nuovo ▲/▼"; si sovrascrive **solo** scegliendo "Sovrascrivi" (di partenza è "Mantieni le mie") **e** la tariffa non è protetta |
| **🔒 Non toccata** | prezzo diverso ma la tariffa in archivio è `Manuale` o più recente del file | resta com'è, anche con "Sovrascrivi", salvo la casella "Sovrascrivi anche le mie…" |
| **Ripetuta nel file** | stessa chiave due volte | si tiene la prima, la nota dice quante |
| **Non letta** | cella con testo che non è un prezzo ("su richiesta"), riga senza destinazione/base | elencata con numero di riga, non importata |

**Record scritti**: nuove → `{from,to,carrier,price,unit,conditions,rev,data[,provisional]}`. `rev` = `REV MM/AAAA` letta dalle prime righe del file, altrimenti `Excel`; `data` = campo "📅 Data delle tariffe" della finestra (proposta: 1° del mese della REV, altrimenti oggi). Aggiornate → cambiano solo `price`, `conditions` (se il file le ha), `provisional`, `data`, `rev` (diventa quella del file: se era `Manuale` e l'utente ha chiesto di sovrascrivere anche le sue, non lo è più); **note e altri campi restano**, il record non si sostituisce più per intero

**Formati di tabella riconosciuti** (`impFindLayout`, `impFindCondRow`, `impFindRev`):
- riga "Base di Carico | Luogo di Consegna | trasportatori…" con le condizioni (`BB 60 gg FM`, `RIBA 30`) sotto, come scrive l'esportazione dell'app (il file esportato si reimporta: tutto "già presente")
- nomi sulla riga 1 e "Base di Carico | Luogo di Consegna | condizioni" sulla riga 2 (formato listino)
- nessuna riga di etichette (basta una riga di nomi con, sotto, righe di prezzi) e anche con soli 2–3 trasportatori
- base di carico **vuota** = celle unite: si usa quella della riga sopra (segnalato nella finestra)
- Se la riga ha condizioni e la riga sopra nomi, la maggioranza decide (`impCondLike`): un trasportatore con "RIBA" nel nome non viene scambiato per una condizione
- **Non riconosciuto** (nessuna riga di nomi, nessun prezzo): niente finestra, errore chiaro nel registro e avviso; non cambia niente

**Celle prezzo** (`impParseCell`, `impNum`): `550` · `550,00` · `550,00 €/vg` · `31 €/ton` · `12 €/mc` · `1.250,00` (migliaia con il punto) · `1,250.50` · due prezzi nella stessa cella separati da ` / `, `;` o a capo (**come li scrive l'esportazione**: `550,00 €/vg / 12,00 €/ton`) · `500-550` / `500-550 €` (si prende il valore di mezzo, segnalato) · numero solo sotto 100 = €/ton (segnalato) · `⚠️`/`prov` = provvisorio · vuoti e segni (`-`, `—`, `n.d.`, `x`, 0) ignorati senza errore · numeri negativi o testo senza numeri = non letti

| Funzione | Descrizione |
|----------|-------------|
| `impClean`, `impNorm`, `impKey`, `impNum`, `impToday` | utility di testo/numero |
| `impParseCell(raw)` | una cella → `{items:[{price,unit,prov}], bad, range, guess}` |
| `impFindLayout(data)` (= `impLayoutByLabels` / `impLayoutByCount`), `impFindCondRow`, `impFindRev`, `impCarrierCols`, `impLooksData`, `impCondLike`, `impIsNote`, `impStar`, `impIsHeaderRow` | riconoscimento della tabella |
| `impParse(data)` | piano completo (non scrive) |
| `impClassify(items, refDate)` | nuove / uguali / diverse / **protette** / ripetute rispetto a `DB.trasporti` |
| `impProtect(list, refDate)`, `impRefDate()` | protezione (Manuale o più recente) e data di riferimento |
| `impShow()` (apre) / `impRender()` (ridisegna: conteggi, scelta, elenchi con `esc()`, max 60 righe per elenco, note), `impRefreshBtn()` | finestra e pulsante "Importa: N nuove + M sovrascritte" |
| `impSetMode(over)`, `impSetProt(on)`, `impToUpdate(c, p)` | scelta Mantieni/Sovrascrivi, casella "anche le mie", elenco delle tariffe che verranno aggiornate |
| `impApply()`, `impCancel()` | scrive (con Annulla) / chiude senza toccare niente |
| `sheetToMatrix(ws)` | foglio SheetJS → righe × colonne |

Variabile di stato: `_impPlan`. `importMode` e `selectImportMode` sono stati eliminati.

**Collaudo**: `test7.js` (**159 prove**, con il file vero opzionale: `node test7.js app.js xl/layouts.json real/real.json`; sezioni Q = protezione, F = formato "Ferrara", M = Mantieni/Sovrascrivi, U = elenchi e riquadri), `browser_giro5c_test.py` (44 prove in Chromium: righe con prezzo a destra senza overflow, tocco sui riquadri e scorrimento, stato degli elenchi, area di tocco ≥ 44 px, 4 combinazioni tema/larghezza), `browser_giro5b_test.py` (48 prove in Chromium: protezione, scelta Sovrascrivi, cambio data, "Modifica tratta" → Manuale, file stile Ferrara, 4 combinazioni tema/larghezza), `test7.js` base (93 prove: celle, formati su file `.xlsx` veri creati con openpyxl da `gen_xlsx.py` — celle unite, sezioni, "n.d.", intervalli, ⚠️ —, confronto, doppioni, giro completo esporta → reimporta, scrittura/Annulla, 1600 prezzi in < 1,5 s, `esc()`), `browser_giro5_test.py` (Chromium: finestra, conteggi, import, Annulla, errore, 340 e 393 px, scuro e chiaro) e audit di contrasto (0 problemi). **Limite del collaudo**: nel sandbox non si può scaricare SheetJS, quindi il passaggio "file .xlsx → tabella" è stato fatto con openpyxl e con un finto `XLSX`; il resto del flusso è reale. Il file di Fil è stato poi fornito (.xls): letto convertendolo con LibreOffice e provato con `real_check.js`; il passaggio reale SheetJS su `.xls` resta non provato qui. Se un altro listino non venisse riconosciuto va mandato per aggiungere il formato

**Non supportato (ancora)**: elenco "piatto" con una riga per prezzo (colonne Da · A · Trasportatore · Prezzo · Unità)

### Giro 6 — Colori come prima (blu navy saturo)
Fil: *"i colori non mi piacciono, era meglio saturato com'era prima"* e, dopo una prima versione "ibrida" (accenti vivi su fondi grigio-azzurri morbidi): *"no fallo uguale a prima"*. **Il tema scuro è tornato identico alla palette precedente al giro 3** (blu notte `#0b0f16`, fondi blu navy, testi `#f5f8ff`). L'interruttore tema (scuro / chiaro / auto) resta, e il tema scuro resta quello predefinito.

| Token | scuro **ora** (= prima del giro 3) | note |
|---|---|---|
| `--bg` `--s1` `--s2` `--s3` | #0b0f16 · #111720 · #171f2e · #1e2840 | identici a prima |
| `--b1` `--b2` | #243048 · #2e3d5c | identici |
| `--t1` `--t2` | #f5f8ff · #c2cfea | identici |
| `--t3` | **#8197ba** (prima #7a90b4) | unico scostamento: +7 per canale, solo perché a #7a90b4 la data "tariffa del …" sui riquadri più chiari scendeva a 4,38:1 (soglia 4,5). A occhio non si vede |
| `--t4` | #4a5568 | usato solo per pallini decorativi |
| `--green` `--green2` | #34d399 · #10b981 | `--green-t` #86efac |
| `--amber` `--red` `--blue` `--gold` | #fb923c · #f87171 · #60a5fa · #fbbf24 | `--red-t` #fca5a5, `--blue-t` #7cb9ff |
| `--on-accent` | #0b0f16 | testo sui pulsanti verdi, come prima |
| `--hd1` `--hd2` `--hd3` | #0d1520 · #111c2e · #0d1a28 | intestazione: stesso gradiente di prima (nuovo `--hd3` per il terzo colore) |
| `--panel-rgb` `--pill-rgb` | 10,18,32 · 36,48,72 | righe prodotto e pill, come prima |
| `<meta theme-color>` | #0b0f16 | barra del browser, come prima (anche nel JS `applyTheme`) |

Altri ritocchi del giro 3 annullati nello scuro: la linea colorata sotto l'intestazione torna a piena intensità (nel chiaro resta a 0,55). Scostamenti residui rispetto al file originale (non visibili): alcuni sfondi inline quasi neri (#111927, #0f1622, #0f1520, #080c12) erano stati ricondotti a `--s1`/`--bg` nel giro 3 e lo restano (differenze di 2–6 livelli su 255); i grigi di testo molto scuri (#4a6480, #5a7898…) che nel file originale avevano contrasto 2,5–3:1 restano mappati a `--t3` e quindi sono **più leggibili** di prima.

**Tema chiaro** (non esisteva prima del giro 3): accenti più vivi del giro 3, scelti col colore più saturo che mantiene contrasto ≥ 4,8 sui fondi chiari (`g6/pick.py`): `--green` #046c49, `--amber` #883b07, `--red` #b8231e, `--blue` #185abf, `--gold` #815703 (+ `*-rgb` allineati).

**Come si ricostruisce**: `python3 $SP/g6/build_giro6b.py [t3]` legge `index.before-giro6.html` e scrive `index.html` (la versione "ibrida" è in `g6/build_giro6.py`, non più usata). Collaudo: audit contrasto 0 sotto soglia in scuro e chiaro (340 e 393 px, 51 192 testi), test 13/14/16/76/128/159, browser 42/48/44/20/15/25, contrasto modale import min 4,93 (scuro) / 5,26 (chiaro). Immagine di confronto prima/adesso: `cmp6/confronto_navy.png` (`cmp6b.py`).

**Misura dei corpi testo** (`fontstats.py`, 3 schermate su 393 px): il **57% dei caratteri visibili è ≤ 11 px** e il 17% ≤ 10 px (note, unità, luoghi, badge, titoli di sezione); i prezzi sono 26 px, i nomi 16 px. Proposta non ancora fatta: minimo 11 px, secondari 12–13, principali 14–15.

### Giro 7 — Grafica Prodotti (vista "Prodotto")
Fil: *"dobbiamo rivedere la grafica dei prodotti … più intuitiva e premium"*. Dopo 4 varianti disegnate (canvas "Prodotti — varianti grafiche": A, B, C, D) ha scelto la **A (Panoramica)** e l'ha rifinita in 3 passaggi. **Il margine non c'è, né nelle righe né nel menu: "c'è il trasporto che qui non viene contato".**

**Come si presenta** (solo la vista *Prodotto*; le viste Fornitore e Cliente non sono cambiate e aprono ancora il loro dettaglio):
- **Scheda chiusa** = icona + nome + `N acquisti · N vendite` + freccia. Nessun prezzo, barra o pallino. Icone SVG per prodotto (`pi-cip`, `pi-seg`, `pi-lfa`, `pi-lri`, `pi-pul`, `pi-sfm`, `pi-sfs`, `pi-gen` per i nomi nuovi), scelte con `prodIconId(nome)` (prima "pula", poi "lolla"). Lo sprite SVG è subito dopo `<body>`.
- **Scheda aperta**: due schede interne **Dove compro** | **A chi vendo** con il numero accanto, **una lista alla volta**. Se un lato è vuoto non ci sono le schede ma un titolo. Sotto, una riga che dice l'ordine (acquisti dal più basso, vendite dal più alto).
- **Unità separate**: dentro ogni lista un gruppo per unità, **€/ton prima, poi €/mc** (poi altre), ognuno con titolo "Prezzi in €/ton · N acquisti". Con una sola unità niente titolo. L'etichetta **Più basso / Più alto** vale dentro ogni gruppo, solo se nel gruppo ci sono almeno due prezzi diversi (a pari prezzo sono etichettate tutte).
- **Data "prezzo del …" su ogni riga**, sotto il prezzo a destra (`prodDataLabel`): giorno preciso (`prezzo del 14/10/26`) se il prezzo ha `data`; **per i prezzi del listino KarMa, che non hanno giorno, il mese del listino (`prezzo del 03/2026`, preso da `DB.karmaRev`)**; se non c'è nulla (prezzo a mano senza data) la riga non la mostra e il menu scrive "data non indicata".
- **Righe unite**: gli **acquisti** dello stesso fornitore, luogo, prezzo, unità e data diventano **una riga** con le destinazioni come etichette (Cippato: 7 record → 4 righe, "4 acquisti"). Le **vendite** restano una riga per record. Le destinazioni/provenienze si leggono dal campo Note (`prodRoute`): `→ A / B` = consegne, `da X (luogo)` = provenienza (nell'etichetta si toglie il luogo tra parentesi), qualsiasi altro testo = nota libera, mostrata sotto la riga e nel campo Note del menu.
- **Stato delle schede** (`prodUI`, solo in memoria): all'avvio chiuse; si aprono da sole con **un solo prodotto in vista, durante una ricerca o con un filtro prodotto** (cambiando ricerca/filtro lo stato torna automatico). Tocco sull'intestazione = apri/chiudi.

**Menu dettagli** (`openProdSheet`, finestra dal basso `#modalProdSheet`, si chiude con X, tocco fuori o Esc): prodotto · tipo, nome, luogo; prezzo grande con "prezzo del …" e fonte (Listino KarMa / Listino KarMa · modificato da te / Inserito a mano / Da import); **Carico e consegne** (acquisto: Carico = luogo del fornitore, Consegne = destinazioni con il prezzo di ciascuna; vendita: Carico = da dove arriva la merce, Consegna = luogo del cliente col prezzo); Note; **Modifica** (grande, verde), **Condividi**, **Elimina**. **Toccare una riga non apre più direttamente la modifica.**
- **Modifica**: apre il modal esistente. Se la riga unisce più consegne, il modal cambia **tutte insieme** prodotto/fornitore/luogo/prezzo/unità/tipo/data (titolo "… — 2 consegne insieme", campo Note disattivato e **note delle consegne intatte**); ogni consegna ha anche una **matita** per modificare solo quella (e se il prezzo diventa diverso la riga si divide). Annulla riporta tutto (una sola copia).
- **Elimina**: chiede conferma, toglie **tutte le voci della riga**, ricorda i KarMa cancellati (`forgetKarma`) e mostra "Annulla". `deleteEditProdotto` e il menu usano la stessa `prodDeleteIdxs`.
- **Condividi**: testo breve (prodotto · tipo, nome (luogo), prezzo e data, consegne o provenienza, note) col foglio di condivisione di Android; senza, copia negli appunti.
- Se i dati cambiano mentre il menu è aperto (`prodSheetValid`) i pulsanti non toccano niente e avvisano di riaprire la voce. Annulla chiude anche il menu.

**Codice**: `renderViewProdotto`, `prodMakeGroups`, `prodRoute`, `prodListHtml`/`prodRowsHtml`, `toggleProdCard`, `setProdTab`, `openProdSheet`/`closeProdSheet`, `prodSheetEditGroup`/`prodSheetEdit`/`prodSheetDelete`/`prodSheetShare`, `openEditProdotto(idx, idxs)`, `saveEditProdotto`, `prodDeleteIdxs`. Stile in un blocco "PRODOTTI — Giro 7" (classi `pc-*`, `pr-*`, `psh-*`, tutto con le variabili del tema; nessun colore fisso nel CSS). Le vecchie classi `.prow-*` restano (le usano ancora le altre viste). Costruzione riproducibile: `python3 $SP/g7/build_giro7.py` (legge `index.before-giro7.html`, `g7/css.txt`, `g7/sprite.txt`, `g7/js_view.txt`, `g7/js_edit.txt`).

**Contrasto**: nel chiaro il testo verde della scheda attiva usa `--green-t` e il numero nella scheda attiva è un contorno senza fondo (prima 4,29 e 3,6:1); `audit_giro7.py` (schede aperte, menu di acquisto/vendita/a mano, modal singolo e di gruppo, 340 e 393 px, scuro e chiaro) = **0 sotto soglia**.

**Collaudo**: `browser_giro7_test.py` (**186 prove** in Chromium su scuro 393, chiaro 393, scuro 340: conteggi e icone, ordine, date, etichette, destinazioni unite, gruppi €/ton / €/mc, virgola italiana, menu, Modifica di gruppo e di una sola consegna con Annulla, Elimina con conferma e Annulla, vendita, Condividi, voci a mano con e senza data e nota libera, ricerca e filtro, viste Fornitore/Cliente, nessuno scroll orizzontale, nessun errore JS), `browser_test.py` aggiornato (24: il primo tocco ora apre il menu, il cancellato è la riga con 2 consegne), tutti gli altri test invariati e verdi.

**Decisioni di Fil in questo giro**: Lolla di Farro acquistata a 1 €/ton è giusto (resta così). Condividi lasciato nel menu (Fil non ha chiesto di toglierlo).
**Non fatto**: etichetta "prezzo vecchio" oltre una soglia (tutti i KarMa hanno 7 mesi: sarebbe su ogni riga); le viste Fornitore e Cliente con la stessa grafica; font più grandi nel resto dell'app.

### Giro 8 — Aggiorna prezzi (aggiornare un listino scrivendo solo i prezzi cambiati)
Fil: *"l'inserimento mi rompe sempre tanto le palle… esiste un modo per velocizzarlo?"*. Delle 4 idee ha scelto **1 (incolla un messaggio), 2 (dettatura) e 3 (Aggiorna prezzi)**. Questo giro è la **3**; 1 e 2 sono lo stesso motore (un campo dove incolli o detti col microfono della tastiera → righe già compilate da confermare) e **non sono ancora fatte: servono 3-4 esempi veri di messaggi/frasi, e sapere se valgono per tariffe, prodotti o entrambi**.

**Dove si apre**: pulsante 💶 (`title="Aggiorna prezzi"`) nella barra di ricerca di Trasporti (dopo ⬇️, prima di ＋) e, su Prodotti, il pulsante "💶 Aggiorna" accanto alla ricerca (prima di ＋); schermata a tutto schermo `#modalAggiorna` (z-index 195, sotto i toast).

**Come funziona**
- Due schede **Trasporti | Prodotti** (si apre su quella da cui arrivi). Un menu "Quale trasportatore vuoi aggiornare?" / "Quale fornitore o cliente vuoi aggiornare?" (**nessuno manda i prezzi a Fil: li aggiorna lui**, quindi mai parole come "mandato" o "listino" nella schermata): trasportatori con il numero di voci; per i prodotti fornitori e clienti con il ruolo (`Ardenghi · fornitore (2)`).
- Per la persona scelta **una riga per ogni prezzo**: Trasporti = tratta + unità + condizioni; Prodotti = prodotto + acquisto/vendita + luogo, consegne e "prezzo del …". Ogni riga ha "ora N" e **una casella dove scrivi solo se il prezzo è cambiato** (il prezzo attuale è il suggerimento in grigio). Virgola o punto, come `impNum`.
- Sotto la casella la **differenza in euro** (`+3 €`, `-1 €`): **mai percentuali, Fil non le vuole** ("cambia la tariffa e basta, non con la percentuale"). Se il salto è almeno un quarto del prezzo di ora compare "⚠ +350 €: controlla" (arancione; la soglia non si vede), testo non valido = "non valido" in rosso e Salva resta spento; un valore uguale a quello di ora non conta come cambiato.
- **Data dei prezzi** (di oggi se non la tocchi); **Azzera** svuota le caselle. Invio passa alla riga dopo. (Il campo "Cambia tutti di (%)" + Applica c'era nella prima versione ed è stato **tolto** su richiesta di Fil.)
- **Salva N** (in alto a destra, spento finché non cambia qualcosa): scrive tutto in un colpo con `undoSnapshot()` + `undoToast('aggiornamento prezzi (chi)')` → toast "Aggiornati N prezzi (prezzi del GG/MM/AA)" con **Annulla**. Chiudere (X, Esc) o cambiare scheda/persona con prezzi scritti e non salvati **chiede conferma**. Annulla (`undoLast`) chiude anche la schermata.
- **Cosa viene scritto**: tariffe → `price`, `data`, `rev='Manuale'` (quindi un futuro import Excel non le sovrascrive), il resto (unità, condizioni, provvisoria, note) invariato. Prodotti → `prezzo`, `data`, e per i record KarMa `edited=true`; **la riga unita (es. Cippato di Ardenghi con 2 consegne) cambia prezzo a tutte le consegne insieme, note intatte**. Se i dati sono cambiati mentre la schermata era aperta, Salva si rifiuta e avvisa (riferimenti verificati).

**Codice**: `aggParse`, `aggFmtIn`, `aggItemsFor`, `aggWhoOptions`, `aggDirty`, `aggOpen`, `aggSetTab`, `aggWhoChange`, `aggRender`, `aggChanges`, `aggCount`, `aggInput`, `aggKey`, `aggClear`, `aggSave`, `aggClose`; stato `_agg`. Stile in un blocco "AGGIORNA PREZZI — Giro 8" (`agg-*`, solo variabili del tema; in tema chiaro il disabilitato usa `--t2` su `--s3`). Costruzione: `python3 $SP/g8/build_giro8.py` (legge `index.before-giro8.html`, `g8/css8.txt`, `g8/html8.txt`, `g8/js8.txt`).

**Voce nuova (＋), aggiunta dopo la consegna del giro** — Fil: *"hai altre modifiche da consigliare?"* → fai la 2 e la 3. Sotto l'elenco compare **"＋ Aggiungi una tratta nuova"** (Trasporti) o **"＋ Aggiungi un prodotto nuovo"** (Prodotti): si apre una scheda dentro la stessa schermata, senza uscire.
- **Tratta nuova** del trasportatore scelto: Da (con suggerimenti dalle tratte che ci sono), A (con `normalizeDest`, come il ＋ normale), Prezzo, Unità (€/viaggio predefinita, €/ton, €/mc), Condizioni facoltative. **Prodotto nuovo** del fornitore/cliente scelto: Prodotto (suggerimenti), "Che cosa fai con lui?" (Acquisto/Vendita: parte su Vendita se è solo cliente), Luogo facoltativo (precompilato col luogo più usato da quella persona), Prezzo, Unità (€/ton predefinita), Note facoltative.
- Sotto la scheda un messaggio: "Verrà aggiunta" (scheda evidenziata), "Manca: …", "Prezzo non valido", "Questa tratta c'è già (stessa unità): scrivi il prezzo nella riga sopra" (confronto senza maiuscole, stesso trasportatore+Da+A+unità), "C'è già una voce identica", "Hai già scritto questa … qui sopra". Una scheda tutta vuota non conta; una a metà blocca **Salva** con un avviso. ✕ toglie la scheda; Invio passa al campo dopo.
- **Salva N** conta prezzi cambiati + voci nuove pronte, tutto con **un solo Annulla**. Le tratte nuove: `{from,to,carrier,price,unit,conditions,provisional:false,note:'',data,rev:'Manuale'}`; i prodotti nuovi `{…, data, source:'manual'}`, data = "Data dei prezzi". Toast: "Aggiornati 1 prezzo e aggiunta 1 voce nuova (prezzi del GG/MM/AA)". Chiudere con una scheda scritta chiede conferma.
- Codice: `aggAddNew`, `aggRemoveNew`, `aggNewAll`, `aggNewStateT`, `aggNewStateP`, `aggNewInput`, `aggNewKey`, `aggFillLists`, `aggDefaults`, `aggNewReady`; stile `.agg-add`, `.agg-new`, `.agg-nh`, `.agg-nx`, `.agg-ng`, `.agg-nm`. Test: `browser_giro8b_test.py` (**160 prove**, 4 combinazioni tema/larghezza).

**⚠️ Difetto trovato e corretto (era nella versione del giro 8 già consegnata)**: con il pulsante 💶 la barra di **Trasporti** sbordava a destra a **320 e 340 px** (23 px a 340; a 360+ andava). Il controllo "niente scroll orizzontale" dei test era **sbagliato**: nell'emulazione del telefono la larghezza della pagina si allarga da sola e `scrollWidth <= innerWidth` passava sempre. **Ora tutti i test confrontano con `document.documentElement.clientWidth`** (che resta fisso) e `browser_giro8_test.py` controlla anche che l'ultimo pulsante della barra stia ≥ 4 px dentro il riquadro. Correzione in `css8.txt`: `@media (max-width:370px)` barra con gap 4 e padding 8, pillole Rapida/Confronto più strette; `@media (max-width:330px)` ancora più strette e pulsanti 30 px.

**Collaudo**: `browser_giro8_test.py` (**168 prove**, 4 combinazioni tema/larghezza, scuro 393 / chiaro 393 / scuro 340: ingressi, elenco, virgola, non valido, uguale, Invio, differenza in € (e mai %), ⚠ sui salti grandi, salvataggio con data → price/data/rev, resto dell'archivio intatto, Annulla identico, tutte le caselle riempite a mano e Azzera, conferme su chiusura e cambio scheda, dati cambiati, righe unite di Ardenghi, AGROGI 9 vendite, Esc, `undoLast`, nessuno scroll orizzontale, nessun errore JS); `audit_giro8.py` (barra Trasporti + schermata in tutti gli stati, 340/393, scuro e chiaro) = **0 sotto soglia** (minimo 5,06:1); tutti i test precedenti (node 13/14/16/76/128/159, browser 24/186/15/42/48/44/25, `audit_contrast`, `audit_giro4`, `audit_giro7`) invariati e a 0.
**Non fatto**: 1 e 2 (incolla / detta), vedi sopra; etichetta "prezzo vecchio"; aggiornare più persone in una volta.

### Giro 9 — Import prodotti: controllo prima di scrivere, niente doppioni, "+M%"
Fil: *"fai la 2 e la 3"* (la 3 = il difetto in backlog: importare il listino sopra ai KarMa incorporati duplicava ~42 voci e perdeva il +M%). L'import dei prodotti ora **si controlla prima, come quello delle tariffe** (`_ipPlan`, finestra `#modalImportProd`, funzioni `ip*`, stesse classi `imp-*`): riquadri **Nuovi / Già presenti / Prezzo diverso / Non lette** (toccabili, aprono l'elenco), scelta **Mantieni i miei | Sovrascrivi** (di partenza si mantiene tutto), casella per sovrascrivere anche le voci tue, campo **Data dei prezzi** (oggi), pulsante "Importa: 5 nuovi + 3 sovrascritti", un solo **Annulla**. Niente cambia finché non confermi.

**Come riconosce una voce già presente** (non più per la nota intera): *acquisto* = prodotto + fornitore + luogo (provenienza) + unità, e la **consegna** (`→ A / B` nella nota, confronto "largo": Ponzano = Ponzano RM, parentesi ignorate); *vendita* = prodotto + cliente + luogo (destinazione) + unità, e il **fornitore** (`da X (luogo)` o `da A / B / C`). Se nella voce in archivio la nota non dice consegna/fornitore, vale come "già presente". Stesso prezzo = già presente; prezzo diverso = "Prezzo diverso"; consegna o fornitore che nell'archivio non c'è = **nuova** (con la scritta "consegna nuova"/"fornitore nuovo").
- **Protette** (non si toccano, nemmeno con Sovrascrivi, finché non spunti la casella): voci `source:'manual'`, `edited` (cambiate con Aggiorna prezzi o a mano) o con data più recente del file. **Mai sovrascritte** le vendite il cui prezzo vale per **più fornitori insieme** (es. "da Carnovale / Incontrato / ToscoImballaggi" a 110): "cambialo a mano".
- Quando si sovrascrive: `prezzo`, `data`, e per i record KarMa `edited=true` (senza, il prossimo aggiornamento del listino rimetterebbe il prezzo vecchio); le note restano. Le voci nuove: `{…, note, data, source:'import'}` con la stessa forma del KarMa (acquisto `→ destinazione`, vendita `da fornitore (provenienza)`).
- **"+M%"**: `56+M%` / `44+M%` si importa come 56 / 44 e **nella nota resta scritto** (`da Aschieri (Fossacaprara (CR)) · 56+M%`, `→ Bagnolo di Po (RO) · 44+M%`); nella finestra una riga lo spiega ("prezzo base, margine da aggiungere") e c'è la nota col conteggio. Celle `-` o vuote = nessun prezzo; righe senza nessun prezzo = **Non lette** (con la riga del file).
- Intestazione cercata nelle prime 6 righe; errori chiari (niente "Prodotto", niente colonne dei prezzi, file vuoto); numeri all'italiana (`impNum`: `1.250,50`, `14,00 €/mc`).

**Risultato sul listino vero** (`real/karma.xlsx` = Listino_KarMa_03.2026 sopra ai 43 KarMa incorporati): **5 nuovi** (vendite Zignago Power ×3 con +M%, acquisti C.F.P. Srl ×2 con +M%), **49 già presenti, 0 doppioni** (prima ~42 doppi), **3 non lette** (Mancini ×2, Sicem Saga: senza prezzo). Rifatto due volte: tutto già presente.
**Collaudo**: `browser_giro9_test.py` (**152 prove**, scuro 393 / chiaro 393 / scuro 340 / chiaro 320: listino vero, nuovi/uguali, Annulla, secondo import, prezzi diversi con Mantieni e Sovrascrivi, voci tue e a più fornitori con la casella, consegna nuova, cella "-" e virgola, errori, tocco sui riquadri, nessuno scroll orizzontale); `audit_giro9.py` (pagina Import + finestra in tutti gli stati, scuro e chiaro) = 0 sotto soglia. **Test vecchi adattati**: `test5.js` e `browser_test.py` ora chiamano `ipApply()` dopo `importProdotti()` (l'import non è più immediato). Costruzione: `python3 $SP/g9/build_giro9.py` (legge `index.after-add.html` = giro 8 + voce nuova, `g9/js9.txt`, `g9/html9.txt`).
**Non fatto**: l'etichetta "prezzo vecchio"; le viste Fornitore/Cliente con la nuova grafica; font più grandi; **incolla un messaggio / detta a voce (1 e 2)**: aspettano gli esempi di Fil. Esempi ricevuti: trasporti *"Bientina Destinazione Mezzano € 420"*; prodotti *"Incontrato 42 vendita a Sicem Ponzano € 130"* (**"Incontrato" è un fornitore** — Bientina — e non va confuso con un prodotto; non è chiaro cosa sia il 42 e quale sia il prodotto: da chiedere). Fil vuole anche **un pulsante "?" accanto al campo che ricordi come si scrive la frase**.

### Backlog dall'audit (non ancora fatto)
- [x] ~~**Import prodotti sopra ai KarMa incorporati**: duplica ~42 voci (chiave `prodKey` con note diverse) e perde il "+M%" delle formule (`56+M%` → 56). Proposta: confronto per prodotto+tipo+fornitore/acquirente+luogo+prezzo+unità ignorando la nota "→/da", anteprima come per le tariffe, "+M%" in nota. Aspetta il via di Fil~~ — **fatto nel giro 9**
- [x] ~~**Backup in un tocco** e **annulla**~~ — fatto nel giro 4
- [x] ~~**Tema** chiaro/scuro con toni smorzati~~ — fatto nel giro 3. Resta da valutare: font leggermente più grandi
- [ ] **Schede tratta più compatte**: il prezzo migliore compare due volte (riga e barra), la pill data si ripete su ogni riga
- [ ] Grafica: barra di navigazione in basso, icone SVG al posto delle emoji, allineamento dei campi nei modal, Import più semplice
- [x] ~~**Offline vero**~~ — fatto nel giro 4 (service worker + SheetJS caricata solo quando serve). Resta: SheetJS in locale (oggi la prima importazione/esportazione Excel richiede internet una volta)
- [x] ~~Drive: conflitti fra dispositivi, riconnessione, banner~~ — fatto nel giro 4
- [ ] Record identificati dalla posizione nell'array (fragile): servirebbe un `id` stabile
- [ ] `esc()` non è ancora applicato a tutto (restano alcune viste secondarie: stampa A3, export)
- [ ] Ricerca/filtri ridisegnano tutto a ogni tasto
- [ ] Codice morto: `renderViewMargine`, `deleteProdotto`, `formatPrice`, `isDataVecchia`, `parsePrice` (alcune servono per funzioni future: badge prezzo vecchio). Il giro 4 ha eliminato `_showConflictModal`, `tryAutoReconnect` e le vecchie funzioni Drive
- [ ] CSS: 46 selettori definiti più volte, le regole successive sovrascrivono quelle documentate; blocco migrazione v3 mai raggiungibile
- [ ] `ws['!freeze']` nell'export Excel probabilmente non ha effetto (SheetJS CE non scrive i riquadri bloccati) — da verificare
- [ ] Storico prezzi con ▲/▼
