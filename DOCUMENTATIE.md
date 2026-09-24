# Excel Viewer — Technische Documentatie

## Overzicht

`excel-viewer.html` is een **standalone, client-side** webapplicatie (single HTML-bestand, ~4400 regels) die Excel- en CSV-bestanden inleest, omzet naar JSON, en een interactieve data-verkenner biedt met facetfilters, zoeken, sorteren, groepering en export. Geen server, geen uploads — alles draait in de browser.

## Architectuur

```
excel-viewer.html
├── SheetJS library (inline, r7-31)      ~1000 regels geminificeerde xlsx.js
├── CSS (r32-1037)                        Volledige styling, Excel-look
├── HTML body (r1039-1228)                3 schermen + overlays + merge-dialog
└── Applicatie-JS (r1229-4372)            IIFE met alle logica
```

### Bestandsgrootte
Het bestand is ~1MB doordat de complete **SheetJS/xlsx.js** library inline staat (regels 7-31). Dit maakt het bestand volledig zelfstandig — geen CDN of externe dependencies.

## Drie schermen (flow)

| Scherm | Element | Doel |
|--------|---------|------|
| **1. Upload** | `#dropScreen` | Drag & drop of bestandskiezer. Ondersteunt `.xlsx`, `.xls`, `.csv`. Keuze horizontaal/verticaal tabelindeling. |
| **2. Configuratie** | `#configScreen` | Per kolom instellen: doorzoekbaar, facetfilter (ja/nee + type), zichtbaar in tabel. Rij-groepering en kleurprofiel kiezen. |
| **3. Viewer** | `#viewerScreen` | Twee-paneel layout: links facetfilters, rechts datatable met zoekbalk, sortering, paginering en groepering. |

## Kernfeatures

### Bestand inlezen (`handleFile`, `loadSheet`)
- Leest het bestand via `FileReader.readAsArrayBuffer`
- Parst met `XLSX.read()` (SheetJS) naar een workbook
- Bij meerdere tabbladen: sheet-picker
- Ondersteunt **verticale tabellen** (veldnamen in kolom A) via `transposeMatrix()`
- Automatische **header-detectie** (`findHeaderRowIndex`): slaat lege/titeltekst-regels boven de eigenlijke header over
- Converteert cellenmatrix naar array-of-objects (`rawRows`)

### Kolomdetectie (`detectColumns`)
Analyseert alle waarden per kolom en detecteert automatisch:

| Wat | Hoe |
|-----|-----|
| **Datatype** | getal / datum / boolean / tekst |
| **BAG-ID's** | 16-cijferige codes → klikbare links naar adresverkenner.nl |
| **E-mailadressen** | user@domein.nl → facetfilter op domeinnaam |
| **URL's** | http(s)/www → klikbare links |
| **Energielabels** | A+ t/m G → gekleurde badges |
| **Financiele velden** | prijs/bedrag/kosten → euro-opmaak |
| **Bouwjaarperiodes** | 4-cijferige jaartallen → voorgedefinieerde klassen |
| **Komma-gescheiden waarden** | "waarde1, waarde2" → multi-select facet |
| **Basis(specificatie)** | "woning (boven)" → parts-facet |
| **Adresvelden** | straat/huisnr/postcode/plaats → samengesteld adres |

### Facetfilters (`buildFacetPanel`, `renderFacetBody`)
Zes facet-typen:

| Type | Werking |
|------|---------|
| `values` | Checkbox-lijst van unieke waarden met tellingen |
| `multi` | Voor komma-gescheiden waarden: elke token apart filterbaar |
| `range` | Min/max slider (numeriek of datum), met preset-knoppen (bv. bouwjaarklassen) |
| `buckets` | Voorgedefinieerde periodeklassen als checkboxes (bv. bouwjaarklassen met beschrijving, of "Mini/Klein/Groot" voor aantal woningen). Gebruikt `RANGE_PRESETS` |
| `parts` | Basis + specificatie apart filterbaar ("woning (boven)" → filter op "woning" of op "boven") |
| `emailDomain` | Extraheert het domein uit e-mailadressen (bv. "jan@gmail.com" → "gmail.com") en toont als checkbox-lijst |

Facetten tonen **live tellingen** die meebewegen met andere actieve filters (cross-filter). Elk facet heeft een zoekbalk, in-/uitklapbaar, en een "wis"-knop.

**Edit-modus** (potlood-icoon): facetten toevoegen, verwijderen, of van type veranderen. Standaard verborgen om het filterpaneel opgeruimd te houden.

**Supergroepen**: facetten worden gegroepeerd weergegeven (bv. "Locatie", "Gebouw", "Energie") op basis van de `customGroup`/`groupLevel`-configuratie of de automatische `computeDisplayGroups()`-classificatie.

### Zoeken
- Vrije-tekst zoeken over alle als `searchable` gemarkeerde kolommen
- Voorberekende zoekindex (`searchIndex`: Map van rij → lowercase haystack)
- **Debounce** met visuele feedback (border kleurt groen tijdens wachttijd)
- Zoektermen worden gehighlight in de tabel (`<mark>`)

### Sorteren
- Klik op kolomkop: oplopend → aflopend → uit
- Ondersteunt datum-, numerieke en tekst-sortering (NL collator)
- Gesorteerde resultaten worden gecached (`sortedRowsCache`)

### Rij-groepering (`buildRowTree`, `renderGroupedRows`)
- Max 3 niveaus hiearchische groepering (bv. VvE → gemeente → blok)
- Uitklapbare boomstructuur met `+`/`−` toggles
- Aggregatiekolom: toont `Aantal` per groep, plus gemiddeld energielabel als van toepassing
- Progress-overlay bij grote groepen (>400 rijen)
- Cache op groepenboom (`groupTreeCache`) om onnodige herberekening te voorkomen

### Toetsenbordnavigatie
- **Pijltjes omhoog/omlaag**: navigeren door rijen
- **Pijl rechts**: groep uitklappen (of naar eerste kind)
- **Pijl links**: groep inklappen (of naar bovenliggende groep)
- **Spatie**: rijdetail-popup openen/sluiten
- **Esc**: popup sluiten
- Focus wordt hersteld na in-/uitklappen via `pendingFocusChildPath`

### Rijdetail-popup (`openDetailPopup`)
- Toont alle velden van de gefocuste rij in een modal
- Navigeerbaar met pijltjes (vorige/volgende rij)
- Opent ook voor groepsrijen (toont dan aggregatie-info)

### Paginering
- Standaard 50 rijen per pagina (`PAGE_SIZE`)
- Aparte paginering voor platte en gegroepeerde weergave
- "Alles uitklappen" / "Alles inklappen" knoppen bij groepering

## Menubalk (Excel-stijl)

| Menu | Item | Functie |
|------|------|---------|
| **Bestand** | Open... | Nieuw bestand laden (reset alles) |
| | Bestand toevoegen... | Extra bestand samenvoegen met huidige data (zie *Bestand toevoegen*) |
| | Opslaan als HTML | Download standalone HTML met data + config erin |
| | Export → Excel/CSV/JSON | Exporteert huidige (gefilterde) data |
| **Weergave** | Kolommen aanpassen | Terug naar configuratiescherm |
| | Filters aanpassen | Zet facet-edit-modus aan |
| | Opslaan | Alias voor "Opslaan als HTML" |

### Bestand toevoegen (`handleImportFile`)

Via **Bestand → Bestand toevoegen...** kan een tweede Excel- of CSV-bestand worden samengevoegd met de al geladen data. De app analyseert de kolommen van beide bestanden en biedt twee modi:

#### 1. Rijen toevoegen (append + deduplicatie)

Wanneer het importbestand dezelfde kolomstructuur heeft als de bestaande data (of er geen geschikte sleutelkolom is), worden de rijen samengevoegd. **Identieke rijen** worden automatisch overgeslagen op basis van een fingerprint van alle gedeelde kolommen (`rowFingerprint`). Na afloop toont een melding hoeveel rijen zijn toegevoegd en hoeveel zijn overgeslagen.

#### 2. Kolommen koppelen (LEFT JOIN)

Wanneer de app een **gedeelde sleutelkolom** herkent én het importbestand **nieuwe kolommen** bevat, verschijnt een keuze-dialog:

| Optie | Werking |
|-------|---------|
| **Rijen toevoegen** | Append + deduplicatie (zie boven) |
| **Kolommen koppelen** | LEFT JOIN: verrijkt bestaande rijen met kolommen uit het importbestand, gekoppeld op de gekozen sleutel |

**Voorbeeld**: dataset met PandID, Adres, Bouwjaar + importbestand met PandID, Energielabel, WOZ_waarde → na koppeling op PandID heeft elke rij alle 5 kolommen. Rijen zonder match krijgen lege waarden voor de nieuwe kolommen.

#### Sleuteldetectie (`detectSharedKeys`)

Een kolom wordt als potentiële sleutel herkend wanneer:

1. **Naamherkenning** — de kolomnaam matcht `KEY_NAME_HINTS`: id, code, nummer, number, identificatie, sleutel, key, bagid, pandid, vbo, pand, ligplaats, standplaats, postcode, kvk
2. **Uniekheid** — minstens 85% van de waarden in beide bestanden is uniek (`KEY_UNIQUENESS_MIN = 0.85`)
3. **Overlap** — er is daadwerkelijke overlap tussen de waarden in beide bestanden

De dialog toont per sleuteloptie hoeveel rijen matchen (bv. "PandID (4 van 5 matchen)").

#### JOIN-logica (`executeJoin`)

- **Type**: LEFT JOIN (alle bestaande rijen blijven behouden)
- Bouwt een index op het importbestand per sleutelwaarde (`normalizeKeyValue`: trim, lowercase)
- Loopt over bestaande rijen en voegt de nieuwe kolommen toe uit de gematchte importrij
- Niet-gematchte rijen krijgen `null` voor de nieuwe kolommen

#### Na import

- Kolommen worden opnieuw gedetecteerd (`detectColumns`)
- Nieuwe kolommen krijgen automatisch een `fieldConfig`-entry
- Alleen als er **nieuwe kolommen** zijn bijgekomen gaat de app naar het configuratiescherm; anders direct terug naar de viewer

### "Opslaan als HTML" (`buildConfiguredHtmlString`)
Slaat een complete kopie van de pagina op (`PRISTINE_HTML`, vastgelegd bij laden) met de huidige data + configuratie geserializeerd in het `#embeddedData` script-element. Bij heropenen wordt dit herkend door `tryRestoreEmbeddedData()` en springt de app direct naar de viewer.

## Export (`exportAs`)
- **XLSX**: maakt nieuw workbook via SheetJS
- **CSV**: met UTF-8 BOM (zodat Excel NL-tekens goed leest)
- **JSON**: pretty-printed
- Exporteert altijd de **gefilterde** rijen

## Kleurprofielen (`THEME_PROFILES`)

5 profielen die de CSS-variabelen `--accent`, `--accent-light`, `--accent-dark` overschrijven:

| Profiel | Accent |
|---------|--------|
| Groen (standaard) | `#1f8a5f` |
| Blauw | `#3457d5` |
| Grijs | `#55606b` |
| Oranje | `#d9730d` |
| Rood | `#c0392b` |

Keuze wordt opgeslagen in het HTML-export-bestand.

## Domeinspecifieke intelligentie

De app is geoptimaliseerd voor **VvE/vastgoed-data** (maar werkt met elk Excel-bestand):

- **BAG-identificaties**: 16-cijferige codes worden herkend en linken naar adresverkenner.nl (`?bagid=...`). Als de rij een gemeente/plaats-kolom bevat met "Haarlem", wordt `&haarlem` aan de URL toegevoegd
- **Energielabels**: A+ t/m G met kleurcodes (groen→rood)
- **Bouwjaarklassen**: voorgedefinieerde periodes gebaseerd op bouwbesluit-wijzigingen
- **Adresaggregatie**: losse straat/huisnr/postcode/plaats-kolommen worden samengevoegd tot een "Adres (samengesteld)"-kolom
- **Statutaire naam**: wordt automatisch als niveau-1 rijgroepering voorgesteld
- **E-mailadressen**: kolommen met e-mailadressen worden herkend, het domein (deel na @) wordt automatisch als facetfilter aangeboden
- **Financiele velden**: herkend op kolomnaam (prijs, bedrag, kosten, WOZ, etc.) → euro-opmaak

## State-variabelen

| Variabele | Type | Doel |
|-----------|------|------|
| `rawRows` | `object[]` | Alle rijen als key-value objecten |
| `columns` | `object[]` | Kolommetadata (type, uniques, min/max, suggest-flags) |
| `fieldConfig` | `object` | Per kolom: searchable, facet, visible, facetType, renderAs*, etc. |
| `facetState` | `object` | Per facet: `Set` van geselecteerde waarden of `{min,max}` |
| `searchQuery` | `string` | Huidige zoektekst |
| `sortState` | `{col, dir}` | Huidige sortering |
| `groupExpandedPaths` | `Set` | Paden van uitgeklapte groepen |
| `navigableRows` | `array` | Platte lijst van gerenderde rijen voor toetsenbordnavigatie |
| `themeProfile` | `string` | Actief kleurprofiel |

## CSS-architectuur

- **CSS-variabelen** op `:root` voor kleuren, border-radius
- **Excel (Windows) look**: `#viewerScreen` overschrijft met `--xl-*` variabelen (Calibri font, grid-borders)
- **Sticky headers**: kolomkoppen en rijnummers blijven zichtbaar bij scrollen
- **Responsive**: geen mobile-specifieke breakpoints (desktop-first, facetpaneel is fixed 260px breed)

## Beperkingen / aandachtspunten

- **Geen dark mode**: alleen light theme
- **Geen undo/redo** in configuratie
- **Geen virtual scrolling**: bij zeer grote bestanden (>10k rijen) kan rendering traag worden (paginering mitigeert dit)
- **Geen webworker**: parsing en filtering draaien op de main thread (met `nextTick`-scheduling voor UI-updates)
- **Single-file**: alle code in 1 HTML-bestand, geen build-stap, geen modules
- **Geen i18n**: interface is volledig Nederlands
