# Performance-analyse v2 — de 30+ seconden hang (herzien)

**Site:** www.competencefactory.nl
**Datum:** 06-07-2026
**Status:** Vervangt de hoofdconclusie van v1. De developer had gelijk dat "session-locking" niet het hele verhaal is.

---

## Belangrijkste conclusie (kort)

**Ja, dit is een server-side probleem.** De 30+ seconden hang is géén netwerk-, front-end- of asset-probleem, en (in tegenstelling tot v1) ook niet primair PHP-sessielocking.

Het is een **intermitterende server-stall van ~51 seconden** die af en toe **één losse request** volledig laat hangen, terwijl de rest van de site op dat moment gewoon snel is (~2s). De vertraging zit **volledig in de Time-To-First-Byte** — de server staat ~51s "na te denken" vóórdat er ook maar één byte terugkomt.

De duur clustert opvallend strak rond **~50–52 seconden**. Dat is exact de standaardwaarde van **`innodb_lock_wait_timeout` in MySQL/MariaDB (50s)**. Dit is de handtekening van een **database-lock die ~50 seconden vastgehouden wordt**: een request wacht op een rij/tabel die door een andere transactie gelocked is, en komt pas los rond de 50s-grens.

> **Kernpunt voor de developer:** zoek naar een **query/transactie die een lock ~50s vasthoudt** (of een lock-wait timeout van 50s). Niet naar de sessie-driver.

---

## Wat er precies gemeten is

Alle metingen zijn black-box tegen de live site met `curl` (volledige timing-breakdown), met echte Chrome-desktop-headers. Elke test verandert één variabele om de oorzaak te isoleren.

> **NB over de browser-test:** ik heb ook geprobeerd met een echte headless Chromium rond te klikken, maar in deze testomgeving worden de browserverbindingen door de tussenliggende proxy na ~12,7s geforceerd gereset (een artefact van de omgeving — `curl` naar dezelfde URL geeft gewoon 200 in ~2s). De browser-cijfers hieronder zijn dus buiten beschouwing gelaten; alle conclusies komen uit de betrouwbare `curl`-metingen.

### 1. Eén bezoeker, rustig klikken → nooit een probleem
Sequentieel 60 pagina-loads: TTFB steeds 1,3–5s. De hang komt **nooit** voor bij één request tegelijk.

### 2. Servercapaciteit is op zich prima
Oplopende gelijktijdige belasting met **losse** sessies (= verschillende bezoekers), pagina `/nieuws`:

| Gelijktijdig | mediaan | max |
|---|---|---|
| 5 | 2,30s | 2,57s |
| 10 | 2,04s | 2,39s |
| 20 | 2,35s | **50,77s** ⚠ |
| 30 | 3,26s | 4,22s |
| 40 | 3,18s | 4,12s |

De server verwerkt 40 gelijktijdige requests moeiteloos (~3s). Dus **geen** klassieke capaciteits-/worker-uitputting. Maar bij 20 gelijktijdig sprong ineens één request naar **50,77s**.

### 3. De stall is reproduceerbaar en clustert rond ~51s
20 rondes van 18–20 gelijktijdige **losse-sessie**-requests. Elke stall (>8s) gelogd:

| Ronde | Pagina | TTFB | Totaal | HTTP |
|---|---|---|---|---|
| — | /nieuws | 50,72s | **50,77s** | 200 |
| 2 | /trainingen | 51,99s | **52,04s** | 200 |
| — | /trainingen | 51,62s | **51,81s** | 200 |
| — | /nieuws | 51,24s | **51,45s** | 200 |
| — | /trainingen | 51,57s | **51,62s** | 200 |
| — | /trainingen | 54,38s | **54,42s** | 200 |

Zes treffers, allemaal tussen **50,7 en 54,4s** (mediaan ~51,6s). De rest van elke burst (17–19 requests) was telkens gewoon ~2s.

**Wat dit betekent:**

- **Het is server-side.** De ~51s zit volledig in de TTFB — de server heeft nog niets teruggestuurd. Uitgesloten: netwerk, DNS, TLS, front-end JS, afbeeldingen, Lottie-animaties, third-party widgets (die laden allemaal in <1s, apart getest).
- **Het is géén sessielocking.** Elke request had een **eigen, verse sessie**. Er is dus geen gedeelde sessie om op te wachten. (De sessielock uit v1 bestaat wél en maakt same-session bursts trager, maar verklaart deze 50s-hangs niet.)
- **Het is geen algemene overbelasting.** Slechts ~1 op de 18–20 requests hangt; de andere zijn snel. Eén request wacht op een **specifieke gedeelde resource**, de rest niet.
- **De strakke clustering rond ~50s** wijst op een **vaste timeout**, niet op willekeurige wachtrijen (die zouden 5s/15s/30s/45s door elkaar geven). ~50s = `innodb_lock_wait_timeout` (MySQL/MariaDB default). Dat de request daarna alsnog **200** teruggeeft, past bij: de blokkerende transactie laat de lock rond 50s los, waarna de wachtende request alsnog slaagt.
- **`/trainingen` springt eruit** (4 van de 6). Die pagina doet vermoedelijk het meeste database-werk (trainingencatalogus, filters/keuzehulp) en/of een schrijf-actie, en is daardoor het vaakst de klos.

---

## Meest waarschijnlijke oorzaak

Een **database-lock die ~50 seconden wordt vastgehouden**. Concreet, in volgorde van waarschijnlijkheid:

1. **Een langlopende transactie die rij- of tabel-locks vasthoudt.** Andere requests die diezelfde rijen/tabel nodig hebben, blokkeren tot `innodb_lock_wait_timeout` (50s). Klassiek patroon: een request opent een transactie en doet er iets traags in.

2. **Een externe/trage aanroep *binnen* een open database-transactie.** Bijv. tijdens het renderen een synchrone call naar een externe dienst (mail, betaal-API, reviews/feedbackcompany, een `test-nieuw.competencefactory.nl`/`pipeline.competencefactory.nl`-endpoint) terwijl er nog locks openstaan. Hangt die call ~50s, dan blokkeert hij alle requests die op diezelfde rijen wachten.

3. **Een `MyISAM`-tabel die veel geschreven wordt** (bijv. een teller voor paginaweergaves, een log-/sessietabel). MyISAM kent alleen **tabel-locks**: één trage schrijf blokkeert alle andere lezers/schrijvers van die hele tabel. Onder gelijktijdigheid geeft dat precies zulke incidentele lange stalls.

Wat het niet is: netwerk, front-end, assets, CDN, of te weinig servercapaciteit in het algemeen.

---

## Hoe de developer dit in ~15 minuten bevestigt en lokaliseert

Draai dit **op het moment dat er belasting is** (of reproduceer met de load-test onderaan):

1. **Vang de blokkade live.** Tijdens een stall, in de database:
   ```sql
   SHOW ENGINE INNODB STATUS\G      -- sectie "TRANSACTIONS": zoek naar 'LOCK WAIT' en welke query wacht/blokkeert
   SHOW FULL PROCESSLIST;           -- zoek queries met hoge Time + State 'Waiting for ... lock'
   SELECT * FROM information_schema.INNODB_TRX;               -- lopende transacties + hoe lang
   SELECT * FROM performance_schema.data_lock_waits;         -- wie blokkeert wie (MySQL 8)
   ```
2. **Zet de slow query log aan** met `long_query_time = 2` en `log_slow_extra = ON`. De hangende query verschijnt met ~50s lock/wait-tijd.
3. **Controleer de storage-engines:** `SELECT table_name, engine FROM information_schema.tables WHERE table_schema = DATABASE();` — staat er nog `MyISAM` tussen, zeker op tabellen die per request geschreven worden (sessions, views, cart, logs)? Zet die om naar InnoDB.
4. **Zoek in de code naar externe calls binnen een transactie:** `DB::transaction(...)` of `beginTransaction()` met daarbinnen een `Http::`, `curl`, `file_get_contents`, mail-verzending of andere I/O. Haal die I/O buiten de transactie.
5. **Check waar `/trainingen` en `/nieuws` schrijven naar de DB** tijdens een GET (view-counter, cache-warmup, laatst-bekeken, filter-state). Een schrijf tijdens een GET is vaak de boosdoener.

---

## Verband met v1 (session-locking)

v1 is niet onjuist, maar wél ondergeschikt:
- **Klopt nog steeds:** same-session bursts (autocomplete per toetsaanslag, hover-AJAX) serialiseren door de PHP-sessielock en maken het klikken trager. Debouncen/sessie-driver blijft zinvolle winst.
- **Maar:** dat verklaart de **30–50s hangs die gebruikers melden niet**, want die treffen ook losse sessies. De hoofdoorzaak is de database-lock hierboven.

**Aanpak-advies:** los éérst de ~50s DB-lock op (grootste, meest zichtbare pijn) en pak daarna pas de sessie-/front-end-optimalisaties uit v1.

---

## Waarom vooral op desktop?

De ~51s-stall wordt getriggerd door **gelijktijdigheid**. Een desktopbezoeker vuurt per paginabezoek veel meer gelijktijdige requests af dan mobiel: hover-AJAX (`/subtraining-ophalen`, `/reviews-ophalen`), autocomplete per toetsaanslag, browser-prefetch en meer parallelle verbindingen, plus sneller doorklikken. Hoe meer gelijktijdige requests, hoe groter de kans dat er eentje net de gelockte resource raakt en de volle ~50s incasseert. Op mobiel zijn het er veel minder → veel kleinere kans per paginabezoek.

---

## Reproductie-script (voor de developer)

```bash
# Vuur herhaald bursts van 20 gelijktijdige requests met ELK een eigen sessie af.
# Losse sessies = het is GEEN sessielock. Een enkele ~50s uitschieter = de DB-lock.
for round in $(seq 1 12); do
  tmp=$(mktemp)
  for i in $(seq 1 20); do
    curl -s -o /dev/null -w "%{time_total}\n" \
      https://www.competencefactory.nl/trainingen >> "$tmp" &
  done; wait
  echo "ronde $round  max=$(sort -n "$tmp" | tail -1)s"
  rm -f "$tmp"
done
# Verwacht: de meeste rondes ~2-3s, af en toe één ronde met max ~50-54s.
# Draai tegelijk SHOW FULL PROCESSLIST / SHOW ENGINE INNODB STATUS om de dader te zien.
```

---

*Methode: black-box load-simulatie met `curl` (per-request timing, echte Chrome-headers), oplopende gelijktijdigheid en herhaalde bursts met losse sessies om server-capaciteit, sessielocking en lock-contention los van elkaar te toetsen. Zes onafhankelijke reproducties van de ~51s-stall. Exacte oorzaak in de servercode/DB is niet in te zien vanaf buiten; de diagnose (~50s lock-wait) is gebaseerd op de sterk clusterende timing en is met de bovenstaande DB-queries in minuten te bevestigen.*
