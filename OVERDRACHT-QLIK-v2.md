# Overdracht v3: sync van Qlik Cloud naar de Inventory Scan Hub

Stand van zaken op 26-10-2026. Plak dit in een nieuwe sessie (Claude Code met de Qlik- en Lovable-connectors aan en ingelogd) om verder te gaan. Werk stap voor stap en vraag om bevestiging voordat je iets aanmaakt of wijzigt. De gebruiker communiceert in het Nederlands.

**Niet aanraken:** de automatisering "Autotaal app reload" en de app "HAGA Dashboard in-en verkoop". Daar komt alle Easy-Trade-data vandaan.

## Wat er staat en werkt

### Dagsync (klaar en getest)

**Qlik-app "Inventory Scan Hub - Sync"** (`d7a02a15-065c-4887-9788-e0531d41fb08`, persoonlijke ruimte)
- Leest alleen QVD's uit `lib://QVD_TRANSFORMED:DataFiles`:
  - de meest recente `HAGA_FACTS_Voorraad_DD-MM-YYYY.qvd` (filiaal 0, velden `Vrd_Loc_Picking` en `Vrd_Loc_Extra`)
  - `HAGA_DIM_Artikelen.qvd`
- Er komt geen STORE in het script.
- Tabel `SyncLocaties` bevat één regel per artikel + locatie, met de kolommen `article_number`, `ean_code`, `ean_code_2`, `description`, `location` en `group`. `group` is `%ArtGrp_Key`, dus Artikel_HoofdgroepNummer.
- EAN (sinds 26-10): per **artikelnummer** over alle Easy-Trade-records (één artikelnummer kan meerdere interne keys hebben). `ean_code` = MaxString, `ean_code_2` = een tweede, afwijkende EAN. Onbruikbare waarden worden gefilterd via de variabele `vEanOk`: Excel-notatie zoals `8,71235E+12`, `#N/B`, `0`/`0000` en `EAN`.
- Locaties worden in hoofdletters gezet zonder spaties.
- Overgeslagen als "geen locatie": `-`, `0`, `NAN` en `GEEN`.
- Geldige vormen:
  - `X.00.00`, `X.00.00.X`, `XX.00.00` en `XX.00.00.X` (dus ook SL/BL)
  - de speciale codes `BULK`, `AB1` en `CD2`
- Tabel `OngeldigeLocaties` toont typefouten om in Easy-Trade te verbeteren. Dat waren er 28 op 25-10.
- Sheet "Sync naar Inventory Scan Hub" (`8b1621ae-41e6-49e3-b407-9205ae00f25d`) bevat de tabel **SyncLocaties v2** (`417fd54a-0d56-45e7-8702-5ec79593e5d5`, met ean_code_2). De oude tabel `a70141a9-…` wordt niet meer gebruikt. De dimensies zijn article_number en location; de measures zijn `MaxString(x) & ''`.

**Qlik-automatisering "Inventory Scan Hub - Dagsync locaties"** (`8184d090-8989-43e3-95a2-bc364ddfa3eb`)
- Stappen:
  1. Do Reload van de sync-app
  2. Get Straight Table Data
  3. API Key Raw Request `start` (`scope: located`)
  4. één `upsert` met alle items (`{json: {$.getStraightTableData}}`)
  5. `finish`
- Alle blokken staan op "stop on error". Bij een fout wordt finish dus niet aangeroepen. Een sync die blijft hangen, wordt bij de volgende start na 3 uur automatisch op "failed" gezet.
- De sleutel staat in de Qlik-connectie "API Key" (`9527f95b-3c97-47ad-bacf-b3fc2fcfda2a`): header `x-api-key`, base URL `yzuxtgqwpmylzmrfsxrp.supabase.co/functions/v1/`. Dit is een billable connector.
- Planning: zie hieronder.
- Testrun 25-10 19:51 duurde 1,5 minuut:
  - 9.819 regels verwerkt
  - 1.323 oude locaties verwijderd (verhuizingen)
  - 339 regels "Alleen in app", 0 regels "Andere locatie"
  - totaal 2.095.562 rijen

**Lovable / edge function `inventory-sync`**
- `start` accepteert `scope` (`full` of `located`) en zet syncs die langer dan 3 uur op "running" staan op "failed".
- `upsert`:
  - maximaal 25.000 items per aanroep, intern verwerkt in stukken van 500 (de PostgREST-limiet is 8 seconden)
  - lege `ean_code`, `ean_code_2`, `description` en `group` overschrijven geen bestaande waarden
- `finish`:
  - loopt via een directe DB-verbinding (`SUPABASE_DB_URL`, statement_timeout 10 minuten)
  - `located` roept `finish_located_sync` aan: verhuizingen, en daarna het verschillenrapport in `inventory_sync_diffs`
  - `full` voert de stale-delete uit
- Beheer → ERP Sync-status toont per sync "Week (volledig)" of "Dag (locaties)" met de tellingen, en een knop "Verschillen downloaden (Excel)".
- Reservekopie van vóór de eerste sync: tabel `backup_inventory_items_20261025` (31.251 rijen: echte locaties plus rijen met ean_code_2 of omschrijving). Die mag weg zodra alles een paar dagen goed loopt.

### Weeksync

**Qlik-app "Inventory Scan Hub - Sync Volledig"** (`afe6d268-f4ed-43b4-928c-d996a0b1b479`)
- Alle artikelen uit `HAGA_DIM_Artikelen`. Artikelen met een geldige locatie krijgen een regel per locatie (dezelfde regels als de dagsync). Een artikelnummer zonder enige locatie krijgt één regel `-`; heeft het ergens een locatie, dan komt er geen extra `-`-regel.
- EAN per artikelnummer met `ean_code_2`, net als de dagsync.
- Tabel `SyncAlles`: 2.266.237 regels op 25-10, met het veld `Blok` (blokken van 10.000, 227 blokken). `SyncInfo.Sync_AantalRegels` bevat het verwachte totaal.
- Sheet `5f104544-c4ba-4b1f-817d-65847b8a85b0` met tabel **SyncAlles v2** (`2d61891e-ca7c-4e00-a60c-deea7683cbf8`, met ean_code_2). De oude tabel `6aee2a14-…` wordt niet meer gebruikt.

**Qlik-automatisering "Inventory Scan Hub - Weeksync volledig"** (`070c1ec1-7cbc-4a65-9d90-732445ade115`)
- Stappen:
  1. Do Reload
  2. Get Expression Value `=Max({1} Sync_AantalRegels)` (blok `verwachtAantal`)
  3. `start` (`scope: full`)
  4. List Values Of Field `Blok`, met in de loop: Select Field Value `{$.blokken.item.qText}`, Get Straight Table Data, `upsert` (met `"block":"{$.blokken.item.qText}"`, on error = ignore) en `syncUpsertHerhaal` (dezelfde aanroep, on error = stop)
  5. `finish` met `expected_count: {$.verwachtAantal}`
- Planning: elke zondag om 03:00 Europe/Amsterdam (eerste run 01-11-2026).

- Waarom de herhaling: Qlik Automate gaf twee keer midden in de run "cURL error 52: Empty reply from server" (blok 151 en blok 184). Elk blok wordt nu twee keer verstuurd; de app verwerkt een blok maar één keer (tabel `inventory_sync_blocks`), dus de tweede keer antwoordt hij `already_done`.

**Edge function (stand na 26-10)**
- Uitrollen: een push naar GitHub zet de edge function **niet** live. Vraag de Lovable-agent kort om `inventory-sync` ongewijzigd te deployen (kost weinig credits). Supabase-MCP heeft geen rechten.
- `upsert` met `block`: idempotent via `inventory_sync_blocks` (PK sync_id + block).
- EAN's uit Easy-Trade gaan door `public.clean_ean()`: onbruikbare waarden tellen als leeg en overschrijven dus nooit een goede EAN.
- Na elke `finish` (beide scopes) draait `public.fix_ean_quality()`; de tellingen komen in `inventory_sync_state.counts`.
- `upsert` loopt via een directe DB-verbinding: één INSERT … ON CONFLICT met COALESCE, en alleen als er iets veranderd is.
- Seen-regels worden voor alle scopes bijgehouden.
- Full-`finish` vereist `expected_count`. Het veiligheidsslot controleert dat `received == expected_count` en dat het aantal geziene regels minstens 90% is van de app. Daarna worden rijen verwijderd die niet in seen staan; beschermde rijen blijven.
- Let op: `erp-bridge.py` (oud script) stuurt geen `expected_count` mee.

**Qlik Automate-workspace JSON (werkt, getest)**
- `StartBlock`, `SnippetBlock` met `snippet_guid`, `EndpointBlock` met `endpoint_guid`, `ShowBlock` (output met input-id `input`).
- Een list-blok krijgt `loopBlockId`; het item binnen de loop is `{$.blokNaam.item}`.
- Formules: `{json: …}` en `{count: …}`.
- On error per blok: instelling `blendr_on_error` = `stop` of `ignore`.
- Let op: `qlik_get_automation_by_id`/`qlik_update_automation` op de dagsync geven een validatiefout bij het teruglezen (lastRun.inputs is een lijst); de update zelf komt wel door. De workspace van de dagsync staat in deze sessie gereconstrueerd uit de eerdere create-aanroep.
- Schedules: `startAt` in de vorm `"YYYY-MM-DD HH:MM:SS"` en recurrence `"RRULE:…"`.

### EAN-kwaliteit (26-10)
- Tabel `ean_corrections`: elke EAN-wijziging door de app (`samengevoegd`, `variant`, `ongeldig_gewist`, `gescand_in_app`).
- `fix_ean_quality()`: onbruikbare EAN's wissen; dubbele regels van hetzelfde artikelnummer samenvoegen (de regel met locatie krijgt EAN, tweede EAN, omschrijving en groep); EAN overnemen van een schrijfvariant (`article_number_normalized`, alleen als die precies één EAN heeft); overbodige `-`/`ONBEKEND`-regels van artikelen met een locatie verwijderen.
- `set_article_ean(p_article_number, p_ean)`: knop "EAN scannen" bij een zoekresultaat zonder (bruikbare) EAN, plus een `+` voor een extra EAN.
- Views `ean_corrections_export` en `items_located_without_ean`; Beheer → ERP & Easy-Trade → kaart "EAN-correcties voor Easy-Trade" met twee Excel-downloads.
- `fix_ean_quality()` werkt alleen op de ~10k locatieregels (tijdelijke tabel `_loc` + partiële index `idx_inventory_items_real_location`): ±4 s per run. `clean_ean` en `is_real_location` hebben bewust géén `SET search_path`, anders kan Postgres ze niet inline uitvoeren (de eerste versie liep daardoor 10 min vast).
- Eerste run 26-10: 28.381 kapotte EAN's gewist (19 op een locatie), 21 EAN's van een schrijfvariant. Nog 420 locatieregels zonder EAN.
- Reservekopie van vóór de EAN-opschoning: `backup_inventory_items_20261026_ean` (38.183 rijen).
- Een EAN in wetenschappelijke notatie (`8,71235E+12`) is niet terug te rekenen: Excel heeft de laatste 7 cijfers weggegooid. Alleen vervangen uit een andere bron of opnieuw scannen.

### Zoeken (26-10)
- Kolom `search_codes` (trigger `trg_set_search_codes`, GIN-index): Bosch-filtercode uit de omschrijving (`OLIEFILTER P7166` bij F 026 407 166: letter + laatste 4 cijfers van het artikelnummer) en NGK-nummer tussen haakjes bij (gloei)bougies (`BOUGIE (5788)` bij LFR 6 C-11). ±4.575 artikelen.
- `search_inventory_normalized`: een code telt als EAN-treffer; deel-treffers op het artikelnummer (SP85020 bij P85020) alleen als er geen exacte treffer is.
- Zoekscherm: één blok per artikel met de locatie groot en de extra locatie(s) eronder. De volgorde is alfabetisch; de app weet (nog) niet welke de picking-locatie is.

### Weeksync 26-10 (eerste volledige run)
- 2.265.437 regels ontvangen = verwacht, 468.879 gewijzigd, 83.999 verwijderd (niet meer in Easy-Trade), totaal 2.261.815. Duur 42 min.

## Openstaand
1. Reservekopieën `backup_inventory_items_20261025`, `backup_inventory_items_full_20261025` en `backup_inventory_items_20261026_ean` mogen weg na een paar goede nachten.
2. **Artikelnummers met komma's:** 12.092 rijen zoals `447,050` bestaan niet in Easy-Trade (ook niet zonder komma) en verdwijnen bij de eerste weeksync. Komma's in echte Easy-Trade-nummers (zoals `M20X1,5`) zijn wel geldig.
3. 420 regels met locatie hebben geen EAN: zie de download "Locatie zonder EAN" in Beheer.
4. Of de dagsync de nieuwe tabel `SyncLocaties v2` (met ean_code_2) gebruikt, is niet zeker: de update gaf een validatiefout bij het teruglezen. De weeksync stuurt ean_code_2 in elk geval mee. Controleer in Qlik Automate of blok `getStraightTableData` naar object `417fd54a-…` wijst.
5. Eventueel: het soort locatie (picking / extra) uit Easy-Trade meesturen, zodat de picking-locatie bovenaan staat.
