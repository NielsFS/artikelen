# Performance-analyse: intermitterende 20+ seconden laadtijd

**Site:** www.competencefactory.nl
**Datum onderzoek:** 27-06-2026
**Symptoom (gerapporteerd):** Bij ~1 minuut rondklikken op desktop ontstaat soms een onverklaarbaar lange laadtijd van 20+ seconden. Op mobiel niet waargenomen.

---

## TL;DR voor de developer

De 20+ seconden stalls worden **niet** veroorzaakt door een trage server of te weinig capaciteit, maar door **PHP session-lock contention** (serialisatie van gelijktijdige requests binnen dezelfde sessie).

De site draait op **Laravel met `SESSION_DRIVER=file`** (PHP 7.4.33 / Apache). De file-session-handler zet bij elke request een **exclusieve lock** op het sessiebestand voor de **volledige duur** van die request. Verzoeken binnen dezelfde browsersessie kunnen daardoor niet parallel verwerkt worden — ze gaan **één voor één** in de wachtrij.

Omdat een enkele pagina al **2–4 seconden** kost om te renderen (geen caching), stapelt dit razendsnel op:

> 8 gelijktijdige verzoeken in dezelfde sessie → laatste pas klaar na **20,3 seconden**.

Op desktop vuurt de site veel meer gelijktijdige same-session verzoeken af (hover-AJAX, autocomplete-per-toetsaanslag, meer parallelle connecties, sneller doorklikken) dan op mobiel — daarom zie je het **alleen op desktop**.

**Belangrijkste fix:** zet `SESSION_DRIVER` van `file` naar `redis` (of `database`/`cookie`), en/of zorg dat read-only AJAX-routes de sessie niet locken. Dit haalt de serialisatie er in één klap uit.

---

## Hoe het gereproduceerd is

Alle metingen zijn gedaan tegen de live site met `curl` (volledige timing-breakdown). De testopzet isoleert de oorzaak door telkens één variabele te veranderen.

### Stap 1 — Baseline: sequentieel klikken (1 verzoek tegelijk)

10 pagina's, 6 rondes, steeds netjes één verzoek na het andere binnen dezelfde sessie.

| Pagina | TTFB (typisch) |
|---|---|
| `/` (home) | ~2,0 s |
| `/trainingen` | ~1,5 s |
| `/inspiratie` | ~2,8 – 4,9 s |
| `/nieuws` | ~3,4 – 4,0 s |
| `/over-ons` | ~1,6 s |
| `/cart` | ~1,3 s |

**Conclusie:** sequentieel verschijnt de 20s-stall **nooit**. Wel is de baseline al hoog (2–4 s) door het ontbreken van caching (zie hieronder). De stall vereist dus *gelijktijdigheid*.

### Stap 2 — Gelijktijdige verzoeken in DEZELFDE sessie (de reproductie)

Dezelfde pagina (`/nieuws`), meerdere verzoeken tegelijk, allemaal met dezelfde sessie-cookie (precies wat één desktopgebruiker doet die rondklikt terwijl AJAX/prefetch nog loopt):

| Gelijktijdige verzoeken | Afrondtijden per verzoek | Laatste klaar na |
|---|---|---|
| 4 | 3,6 → 6,1 → 8,6 → 10,9 s | **10,9 s** |
| 6 | 3,8 → 6,0 → 8,2 → 10,7 → 13,0 → 15,1 s | **15,1 s** |
| 8 | 3,3 → 5,4 → 8,2 → 10,3 → 13,0 → 15,7 → 17,8 → 20,3 s | **20,3 s** |

De karakteristieke **trapvorm** (elk verzoek ~2,4 s later klaar dan het vorige) is de handtekening van lock-serialisatie: elk verzoek moet wachten tot het vorige de sessielock loslaat.

### Stap 3 — Controletest: zelfde belasting, maar LOSSE sessies

Identiek aan stap 2 (8× `/nieuws` gelijktijdig), maar nu met elk een eigen sessie-cookie:

| | 8× `/nieuws` gelijktijdig | Resultaat |
|---|---|---|
| **Gedeelde** sessie | 3,3 → 5,4 → … → **20,3 s** (trap) | geserialiseerd |
| **Losse** sessies | allemaal **~3,6–3,9 s** | volledig parallel |

**Dit is het sluitende bewijs.** Zelfde server, zelfde pagina, zelfde aantal gelijktijdige verzoeken. Het enige verschil is of ze een sessie delen. De server kán 8 verzoeken prima parallel aan (~3,8 s elk) — de vertraging zit **uitsluitend** in de gedeelde sessielock, niet in CPU, geheugen of database.

### Stap 4 — Realistische desktop-trigger: de zoek-autocomplete

`main.js` koppelt een autocomplete met `minLength: 1` aan de zoekbalk: **elke toetsaanslag** vuurt een verzoek naar `/zoeken/autocomplete?q=...`. Het woord "marketing" intypen = 9 verzoeken in dezelfde sessie:

```
q=m         1,03 s
q=ma        1,16 s
q=mar       1,79 s
...          (trapvorm)
q=market    2,08 s
```

Ook hier weer serialisatie. Tijdens het typen houden deze verzoeken de sessielock vast; klikt de gebruiker dan op een link, dan staat die navigatie achteraan in de wachtrij.

---

## Waarom alleen op desktop?

De oorzaak is hoeveel **gelijktijdige verzoeken binnen dezelfde sessie** worden afgevuurd. Op desktop is dat structureel meer:

1. **Hover-events** — `main.js` bevat `mouseenter`/`mouseover`-handlers en AJAX-routes (`/subtraining-ophalen`, `/reviews-ophalen`). Hover bestaat niet op touch/mobiel. Over kaartjes bewegen kan op de achtergrond verzoeken afvuren die de lock pakken.
2. **Autocomplete per toetsaanslag** (`minLength: 1`) — desktopgebruikers typen sneller en meer; elke aanslag is een apart, lock-houdend verzoek.
3. **Meer parallelle connecties & sneller doorklikken** — desktopbrowsers openen meer gelijktijdige verbindingen en de gebruiker klikt vaak door vóór de vorige pagina klaar is. Die overlappende navigaties delen de sessie.
4. **Browser-prefetch** — desktop Chrome doet eerder speculatieve prefetch/prerender van links; elke prefetch is een volwaardig, lock-houdend PHP-verzoek.

Op mobiel: geen hover, minder/trager typen, minder parallelle connecties → zelden genoeg overlap om de wachtrij te laten oplopen tot 20 s.

---

## Bijdragende factoren (versterken het probleem)

Deze veroorzaken de stall niet zelf, maar maken elke "trede" in de trap hoger:

- **Geen caching.** Response-headers: `Cache-Control: no-store, no-cache, must-revalidate, private`. Elke paginaweergave wordt volledig opnieuw door Laravel gerenderd. Daardoor is de lock-houdtijd per verzoek 2–4 s in plaats van milliseconden. Hoe langer elk verzoek de lock vasthoudt, hoe sneller de wachtrij naar 20 s loopt.
- **`minLength: 1` op autocomplete, zonder (zichtbare) debounce.** Maximaal aantal lock-houdende verzoeken tijdens het typen.
- **PHP 7.4.33** — sinds eind 2022 end-of-life (geen security-updates). PHP 8.x is bovendien fors sneller, wat de lock-houdtijd direct verkort.
- **Hoge baseline-TTFB (2–4 s)** — ook zónder de stall voelt de site traag. Dit is een apart, op zichzelf staand aandachtspunt.

---

## Aanbevolen oplossingen (op prioriteit)

### 1. Sessielock wegnemen — *grootste impact, lost de 20s-stall op*

**Optie A (aanbevolen): wissel van session-driver.**
Zet in `.env` `SESSION_DRIVER=redis` (of `database`, of `cookie`).
De file-driver gebruikt blocking file-locks; de Redis-driver doet dat standaard niet, dus gelijktijdige verzoeken in dezelfde sessie serialiseren niet meer. Dit is meestal een wijziging van één regel + een Redis-instance.

**Optie B: laat read-only AJAX de sessie niet locken.**
Routes zoals `/zoeken/autocomplete`, `/reviews-ophalen`, `/subtraining-ophalen` hoeven niets naar de sessie te schrijven. Haal ze uit de `web`-middlewaregroep (geen `StartSession`), of geef ze een aparte stateless-route/middleware. Dan pakken ze de lock niet en blokkeren ze de echte navigatie niet.

> Tip om te verifiëren dat de file-lock de boosdoener is: herhaal stap 2 vs. stap 3 hierboven na de wijziging. De trapvorm hoort te verdwijnen.

### 2. Render-tijd per verzoek verlagen — *verlaagt elke trede én de baseline*

- Productie-caching aanzetten: `php artisan config:cache route:cache view:cache` (of `php artisan optimize`).
- **OPcache** aanzetten/controleren in de PHP-config.
- Volledige-pagina-/responsecaching voor publieke pagina's (bijv. `responsecache`, of een reverse-proxy/Cloudflare-cache) — let op dat de huidige `no-store`-headers dit nu actief blokkeren.
- Trage database-queries opsporen (Laravel Debugbar / Telescope / `DB::enableQueryLog()` op `/inspiratie` en `/nieuws`, de traagste pagina's).

### 3. Aantal gelijktijdige same-session verzoeken verminderen

- Autocomplete: `minLength` naar 2–3 én een **debounce** van ~250 ms toevoegen.
- AJAX op hover (`/subtraining-ophalen`, `/reviews-ophalen`) niet op `mouseenter` afvuren, maar pas op klik / lazy-load, en resultaten client-side cachen.

### 4. Onderhoud

- Upgrade **PHP 7.4 → 8.2/8.3** (sneller + weer security-updates).

---

## Snelle verificatie achteraf

Na het doorvoeren van fix #1 kun je dit één-op-één natrekken met dezelfde test die het probleem aantoonde:

```bash
# 8 gelijktijdige verzoeken in DEZELFDE sessie naar een zware pagina
JAR=$(mktemp); curl -s -c "$JAR" -o /dev/null https://www.competencefactory.nl/
for i in $(seq 1 8); do
  curl -s -b "$JAR" -o /dev/null \
    -w "req#$i total=%{time_total}s\n" \
    https://www.competencefactory.nl/nieuws &
done; wait
```

**Vóór de fix:** trapvorm tot ~20 s.
**Na de fix:** alle 8 ongeveer gelijktijdig klaar (~render-tijd van één pagina).

---

*Methode: black-box reproductie tegen de live site met `curl` (timing-breakdown per verzoek) en statische analyse van de uitgeleverde `main.js`/HTML. Geen toegang tot servercode; aanbevelingen zijn gebaseerd op het waargenomen gedrag (Laravel + `file` sessions + ontbrekende caching) en zijn waar mogelijk via een controletest geverifieerd.*
