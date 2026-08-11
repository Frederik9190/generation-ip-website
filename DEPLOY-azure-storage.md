# Generation-IP — website publiceren in een Azure Storage account

Deze map bevat een volledig statische website. Er is geen webserver, geen build-stap
en geen serverside code nodig.

```
index.html          startpagina
404.html            foutpagina
robots.txt          instructies voor zoekmachines
sitemap.xml         paginaoverzicht voor zoekmachines
assets/             logo, iconen en achtergrondbeeld
```

---

## 1. Statische website inschakelen

1. Open je storage account in de Azure-portal.
2. Ga in het linkermenu onder **Data management** naar **Static website** en zet die op **Enabled**.
3. Vul in:
   - **Index document name:** `index.html`
   - **Error document path:** `404.html`
4. Klik **Save**.

Azure maakt nu automatisch een container **`$web`** aan. Alles wat daarin staat, wordt
publiek als website geserveerd. Na het opslaan verschijnt het **primary endpoint**,
in de vorm `https://<naam>.z6.web.core.windows.net` — dat is meteen je website-adres.

---

## 2. Bestanden uploaden

### Via de portal
Ga naar **Containers > $web** en sleep de inhoud van deze map erin. Let op dat je de
**inhoud** upload, niet de map zelf: `index.html` moet in de wortel van `$web` staan,
met daarnaast de map `assets`.

### Via Azure CLI
```bash
az storage blob upload-batch \
  --account-name <storageaccountnaam> \
  --auth-mode login \
  --destination '$web' \
  --source . \
  --overwrite
```

### Via AzCopy
```bash
azcopy sync "." "https://<naam>.blob.core.windows.net/\$web" --recursive
```

> **Belangrijk — content-type.** Als een `.html`-bestand als
> `application/octet-stream` wordt geüpload, gaat de browser het downloaden in plaats
> van tonen. De portal en `upload-batch` leiden het type meestal correct af. Controleer
> het na de upload op de blob-eigenschappen, of forceer het:
> ```bash
> az storage blob update --account-name <naam> --container-name '$web' \
>   --name index.html --content-type "text/html"
> ```

> **Hoofdlettergevoelig.** Bestandsnamen in de URL zijn hoofdlettergevoelig, ook al
> gaat het over HTTP. `Index.html` werkt niet als het bestand `index.html` heet — dat is
> de meest voorkomende oorzaak van een 404 na het uploaden.

---

## 3. Eigen domeinnaam koppelen

Je kan `www.generation-ip.com` rechtstreeks op het storage-endpoint laten wijzen, maar
let op deze beperking:

> **Azure Storage ondersteunt zelf géén HTTPS op een eigen domein.**
> Voor `https://www.generation-ip.com` heb je Azure CDN of Azure Front Door nodig.

Aanbevolen opzet:

1. Maak een **Azure Front Door**- of **Azure CDN**-profiel aan.
2. Zet als origin het **static website endpoint** (`<naam>.z6.web.core.windows.net`),
   dus niet het gewone blob-endpoint.
3. Voeg `www.generation-ip.com` toe als custom domain en laat Azure een beheerd
   certificaat uitreiken. HTTPS is daarmee geregeld, inclusief automatische vernieuwing.
4. Zet in je DNS een **CNAME** van `www` naar het Front Door- of CDN-endpoint.
5. Laat het kale domein `generation-ip.com` doorverwijzen naar `www` (bij de meeste
   DNS-providers via een ALIAS-, ANAME- of URL-redirectrecord).

Reken op tot 90 minuten voor een nieuw CDN-endpoint volledig is uitgerold.

---

## 4. Beveiligingsheaders (optioneel maar aan te raden)

De statische-websitefunctie kan zelf **geen HTTP-headers instellen**. Wil je headers
zoals `Strict-Transport-Security`, `X-Content-Type-Options` of een cachebeleid, dan
regel je dat in de **rules engine** van Front Door of CDN. Een zinvolle basis:

| Header | Waarde |
|---|---|
| `Strict-Transport-Security` | `max-age=31536000; includeSubDomains` |
| `X-Content-Type-Options` | `nosniff` |
| `Referrer-Policy` | `strict-origin-when-cross-origin` |
| `Cache-Control` op `assets/*` | `public, max-age=31536000, immutable` |
| `Cache-Control` op `*.html` | `public, max-age=300` |

---

## 5. Goed om te weten

- **Kostprijs.** De functie zelf is gratis; je betaalt enkel voor opslag en
  transacties. Deze site is minder dan 200 KB groot.
- **Toegang.** Bestanden in `$web` zijn altijd anoniem leesbaar. Er is geen
  ondersteuning voor aanmelden met Entra ID; dat is inherent aan een statische site.
- **CORS** wordt niet ondersteund op het static website endpoint. Deze site heeft het
  niet nodig.
- **Private endpoints.** Zet je een private endpoint op de blob-service, dan is de site
  extern niet meer bereikbaar tenzij je een apart private endpoint voor `web` maakt.
- **Alternatief.** Wil je later toch headers, authenticatie of een uitrol vanuit
  GitHub, dan is **Azure Static Web Apps** de logische volgende stap. De bestanden uit
  deze map kan je daar ongewijzigd naartoe verhuizen.

---

## 6. Site aanpassen

Alle opmaak zit in een `<style>`-blok bovenaan `index.html`. De huisstijlkleuren staan
als CSS-variabelen helemaal bovenaan:

```css
--brand:  #1e3328;   /* smaragdgroen */
--accent: #de6c2d;   /* oranje       */
```

De site past zich automatisch aan aan donkere modus, werkt op mobiel en print netjes af.
