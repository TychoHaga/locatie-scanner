# Overdracht v2: sync van Qlik Cloud naar de Inventory Scan Hub

Plak dit hele bericht als eerste bericht in een nieuwe sessie (Claude Code op claude.ai/code, met de Qlik- en Lovable-connectors aan en ingelogd).

## Opdracht

Stel de automatische sync in tussen Qlik Cloud (bron: Easy-Trade van Autotaal) en de Inventory Scan Hub. Gebruik de Qlik-connector voor Qlik Cloud en de Lovable-connector voor de app. Werk stap voor stap en vraag mij om bevestiging voordat je iets aanmaakt of wijzigt. Ik communiceer in het Nederlands.

Er zijn twee syncs:

| | Wat | Scope | Wanneer |
|---|---|---|---|
| **Dagsync** | Alle artikelen die in Easy-Trade een locatie hebben | `located` | Elke nacht om 02:00 (Europe/Amsterdam) |
| **Weeksync** | Alle artikelen (±2,1 miljoen) | `full` | Eén keer per week, bijvoorbeeld zondagnacht |

Na elke dagsync wordt in de app een verschillenrapport gemaakt. Dat is te downloaden als Excel onder Beheer → ERP Sync-status, met de kolommen Artikelnummer, Locatie Easy-Trade, Locatie app, Groep en Soort verschil.

## Achtergrond

- **De app:** Inventory Scan Hub, een Lovable-project.
  - Project-id: `ab46552c-dacf-44f9-9207-e6ceb5971feb`
  - Workspace: "Tycho's Lovable" (`FsKqMj7urzq0msYTstQB`)
  - Live: https://inventory-scan-hub.lovable.app
  - Supabase-project: `yzuxtgqwpmylzmrfsxrp`
  - Edge function URL: `https://yzuxtgqwpmylzmrfsxrp.supabase.co/functions/v1/inventory-sync`
- **De database** (25-10-2026): 2.095.020 rijen in `inventory_items`, met 2.094.480 unieke artikelnummers. 0 artikelen hebben `location_set_in_app_at`.
  - Locaties: 2.086.434 rijen hebben `-` (geen locatie), 8.341 hebben een echte locatie in de vorm W.01.01 of A.10.02.C, en 245 hebben een andere waarde.
  - De dagsync gaat dus over ongeveer 8.600 artikelen. Filter in Qlik `-` en lege locaties eruit. In het verschillenrapport tellen `-` en `ONBEKEND` niet mee. Er is één eerdere testsync geweest (29-07-2026, start en daarna abort), dus de functie en het secret hebben toen gewerkt.
- **De sessie van 25-10:** de bug in finish (`.rpc(...).catch` gaf een crash nadat de stale-delete al was uitgevoerd) is hersteld en gedeployed. Daarna zijn scope, verschillenrapport en download gebouwd. Controleer met `read_file` op `supabase/functions/inventory-sync/index.ts` hoe het nu precies werkt.

## Edge function `inventory-sync`

- **Header:** `x-api-key` met de waarde van het secret `INVENTORY_SYNC_API_KEY`. Dat staat in Lovable onder Cloud → Secrets en ik vul het zelf in. Zet het nooit in de chat, in code of in platte tekst in Qlik. Gebruik in Qlik Automate een veilige plek, zoals een connection of credential.
- **Body (JSON):**
  1. `{"mode":"start","scope":"located"|"full"}` geeft `{sync_id, started_at}` terug. Zonder `scope` wordt het `full`.
  2. `{"mode":"upsert","sync_id":"…","items":[{article_number, ean_code, ean_code_2, location, description, group}, …]}` met maximaal 5000 items per keer. Upsert gebeurt op (`article_number`, `location`).
  3. `{"mode":"finish","sync_id":"…"}`
     - Bij **full**: **verwijdert alle artikelen die tijdens deze sync niet zijn aangeraakt.**
     - Bij **located**: verwijdert geen artikelen die ontbreken. Wel verdwijnen oude locaties van gemelde artikelen (verhuizing), behalve locaties die in de app zijn ingeboekt. Daarna wordt het verschillenrapport gemaakt.
  4. `{"mode":"abort","sync_id":"…","error":"…"}` breekt de sync af zonder iets te verwijderen.

## Plan

1. **Qlik:** zoek de app met de Easy-Trade-voorraad op. Controleer de velden (artikelnummer, EAN, 2e EAN, locatie, omschrijving, groep) en de aantallen: hoeveel artikelen in totaal en hoeveel met een locatie. Kijk ook hoe laat de app 's nachts herladen wordt; dat moet vóór 02:00 zijn.
2. **Dagsync (Qlik Automate):** maak een straight table met alleen artikelen die een locatie hebben. "Get Straight Table Data" haalt maximaal 100.000 regels op. Controleer dat het aantal daaronder blijft. De stappen:
   - Call URL voor start (`scope: located`)
   - Get Straight Table Data
   - Loop batch (1000) met Call URL voor upsert
   - Controle of het aantal verstuurde regels gelijk is aan het aantal in Qlik
   - Call URL voor finish
   - Bij een fout: abort
3. **Weeksync:** ±2,1 miljoen regels passen niet in Get Straight Table Data. Zoek een andere route, bijvoorbeeld: het reload-script schrijft een CSV of QVD weg die via een veilige URL bereikbaar is, en een (nieuwe) edge function haalt die zelf op en verwerkt hem in delen. Leg mij eerst de opties voor. **Een onvolledige weeksync gevolgd door finish wist artikelen uit de app.**
4. **Test** eerst de dagsync één keer handmatig. Controleer daarna ERP Sync-status, het verschillenrapport en een paar artikelen in de app. Zet pas dan de planning aan. Doe daarna hetzelfde voor de weeksync.

## Waar je op moet letten

- Preview en live app delen dezelfde database.
- Het verschillenrapport "Alleen in app" kan bij de eerste dagsync erg groot zijn, omdat catalogusartikelen een locatie kunnen hebben die Easy-Trade niet meldt. Bespreek dat met mij na de eerste test.
- Roep finish alleen aan als de lijst gegarandeerd compleet is. Anders altijd abort.
