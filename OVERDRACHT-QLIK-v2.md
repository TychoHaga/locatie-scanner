# Overdracht v3: sync van Qlik Cloud naar de Inventory Scan Hub

Stand van zaken op 25-10-2026. Plak dit in een nieuwe sessie (Claude Code met de Qlik- en Lovable-connectors aan en ingelogd) om verder te gaan. Werk stap voor stap en vraag om bevestiging voordat je iets aanmaakt of wijzigt. De gebruiker communiceert in het Nederlands.

**Niet aanraken:** de automatisering "Autotaal app reload" en de app "HAGA Dashboard in-en verkoop". Daar komt alle Easy-Trade-data vandaan.

## Wat er staat en werkt

### Dagsync (klaar en getest)

**Qlik-app "Inventory Scan Hub - Sync"** (`d7a02a15-065c-4887-9788-e0531d41fb08`, persoonlijke ruimte)
- Leest alleen QVD's uit `lib://QVD_TRANSFORMED:DataFiles`:
  - de meest recente `HAGA_FACTS_Voorraad_DD-MM-YYYY.qvd` (filiaal 0, velden `Vrd_Loc_Picking` en `Vrd_Loc_Extra`)
  - `HAGA_DIM_Artikelen.qvd`
- Er komt geen STORE in het script.
- Tabel `SyncLocaties` bevat één regel per artikel + locatie, met de kolommen `article_number`, `ean_code`, `description`, `location` en `group`. `group` is `%ArtGrp_Key`, dus Artikel_HoofdgroepNummer.
- Locaties worden in hoofdletters gezet zonder spaties.
- Overgeslagen als "geen locatie": `-`, `0`, `NAN` en `GEEN`.
- Geldige vormen:
  - `X.00.00`, `X.00.00.X`, `XX.00.00` en `XX.00.00.X` (dus ook SL/BL)
  - de speciale codes `BULK`, `AB1` en `CD2`
- Tabel `OngeldigeLocaties` toont typefouten om in Easy-Trade te verbeteren. Dat waren er 28 op 25-10.
- Sheet "Sync naar Inventory Scan Hub" (`8b1621ae-41e6-49e3-b407-9205ae00f25d`) bevat de tabel (`a70141a9-daed-4857-8cba-dd71766d973b`). De dimensies zijn article_number en location; de measures zijn `MaxString(x) & ''`.

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

## Openstaand

1. **Weeksync (scope `full`, ±2,4 miljoen artikelen).** Get Straight Table Data haalt maximaal 100.000 cellen op, dus deze route werkt niet. Opties uitzoeken en eerst aan de gebruiker voorleggen. Let op: een full-finish verwijdert alles wat niet is aangeraakt, dus de lijst moet gegarandeerd compleet zijn.
2. **Artikelnummers met komma's.** 12.092 rijen in de app hebben een artikelnummer als `447,050` (waarschijnlijk een oude Excel-import). Daarvan hebben 100 een locatie, en die staan nu allemaal in "Alleen in app". 24 daarvan hebben ook een schone variant (`447050`). Met de gebruiker bespreken of dit opgeschoond moet worden.
3. 582 gesynchroniseerde regels hebben geen EAN in Easy-Trade.
