# Declara Webb — Arkitektur & referens

> Syfte: en snabb genväg för Claude (eller Henric) att förstå hur appen hänger ihop,
> utan att behöva härleda det på nytt varje session. Uppdatera den här filen när
> arkitekturen förändras i grunden — inte för varje liten ändring.
>
> Senast uppdaterad: 2026-06-20

## Teknisk stack

- **Frontend:** React, skrivet som en enda fil (`declara.jsx`), kompilerad med esbuild
  (klassisk JSX-transform mot global `React`, `react`-importen pekar mot `window.React`
  via en liten shim) till en självständig `declara.html`.
- **Backend:** Supabase (Postgres + Auth + RLS), Stockholm-region.
- **Hosting:** Vercel, `app.declara.se` via Loopia DNS.
- **GitHub:** `HenricMalmvall/Declara-webb`.

### HTML-bygget i korthet
React 18 UMD + ReactDOM 18 UMD + Supabase JS v2 laddas via CDN i `<head>`. Eftersom bara
`useState`/`useRef`/`useCallback` importeras explicit i `declara.jsx`, men `useEffect`
(och ibland `React.Component`/`React.useState`/`React.Fragment`) används som bara globaler
på flera ställen, **måste** HTML-skalet definiera globala hook-alias
(`var useEffect = React.useEffect` osv.) innan den bundlade koden körs — annars
`ReferenceError`. Supabase-uppgifter läses från `window.__SUPABASE_URL__` /
`window.__SUPABASE_ANON_KEY__`.

## Appens routing (`App()` → vilken panel visas)

`App()` wrappar allt i `AuthGate`, som via en render-prop skickar ut
`{ tenantInfo, schoolData, versionId, isReadOnly, onSignOut, role }`. Baserat på det:

- `role === "owner"` eller `tenantInfo.tenants.type === "owner"` → **`OwnerAdmin`** (Declara AB, ser alla kunder)
- `tenantInfo.tenants.type === "principal"` → **`PrincipalAdmin`** (huvudman, ser bara sina skolor)
- annars → **`AppInner`** (rektor/skola), som i sin tur grenar till **`GYAppInner`** om `school_type === "gymnasieskola"`

## Datamodell (Supabase-tabeller)

| Tabell | Nyckelfält | Beskrivning |
|---|---|---|
| `tenants` | `id`, `type` (`school`/`principal`), `parent_id` (skola→huvudman), `school_type` | Kunder. Ägarkonton hanteras inte här, sätts manuellt via `profiles.role="owner"`. |
| `profiles` | `id` (=auth user id), `tenant_id`, `role` (`editor`/`reader`/`admin`/`owner`) | Kopplar inloggade användare till en tenant. |
| `school_data` | `tenant_id`, `school_year`, `data` (jsonb — hela DTV-staten), `version_id`, `version_name`, `is_primary` | Skolans faktiska DTV-data. **OBS: kolumnen heter `data`, inte `state`** — det har varit en återkommande bugg (se nedan). |
| `principal_templates` | `tenant_id` (=huvudmannens id), `school_year`, `settings` (jsonb), `locked_fields` (array) | Huvudmannens mall + lås-inställningar. |
| `tenant_invites` | `email`, `tenant_id`, `role` | Väntande inbjudan, konsumeras automatiskt vid första inloggning (matchas på e-post i `AuthGate`). |
| `tenant_payments` | `tenant_id`, `status` | Betalstatus. |
| `system_updates` | `target_type`, `target_id` | Nyheter/notiser till tenants. |

### Versionshantering (infördes 2026-06-20, Runda 1)

`school_data` har tre nya fält: `version_id` (uuid), `version_name` (text),
`is_primary` (bool). En skola kan i framtiden ha flera versioner av sin DTV för
samma läsår, men **exakt en är alltid `is_primary = true`** (skyddat av ett
partial unique index `(tenant_id, school_year) WHERE is_primary = true`).

Designbeslut som redan är tagna för version-systemet (gäller när Runda 2/3 byggs):
- Huvudmannens lås/"Skicka ut" påverkar **bara huvudversionen**, inga andra versioner.
- Huvudmannapanelen (skollistor, analyser, kostnader) pekar alltid på huvudversionen.
- Rektorn kan själv utse vilken version som ska vara huvudversion.
- Huvudversionen får bara raderas om en annan version först utses till huvud.
- En version-väljare (öppna befintlig / ny tom / mall från huvudman) ska visas
  vid **varje** inloggning, oavsett om skolan bara har en version.

## Spara/ladda-flödet (en enda sanningskälla)

- `loadSchoolDataFull(tenantId, schoolYear)` / `loadSchoolData(...)` — läser alltid
  raden där `is_primary = true`. `Full`-varianten returnerar även `versionId`/`versionName`.
- `saveSchoolData(tenantId, schoolYear, data, versionId?)` — skriver till huvudversionen.
  Om `versionId` utelämnas slår funktionen själv upp (eller skapar) rätt version_id
  och **returnerar** det använda id:t, så anroparen kan cacha det.
- I `AppInner`: `const doSave = onSaveOverride || saveSchoolData;` — det här mönstret
  gör att samma `AppInner`-komponent kan återanvändas för helt andra syften genom att
  byta ut vart sparningen går, t.ex. huvudmannens "📝 Redigera mall" sparar via
  `onSaveOverride={saveTemplateDTV}` till `principal_templates` istället för `school_data`.
- `versionId` träs igenom: `AuthGate` laddar det vid inloggning → skickas som prop →
  `AppInner` cachar i en `ref` (`versionIdRef`) → skickas med i varje `doSave`-anrop
  och uppdateras från returvärdet.

## Modulsystemet

- `MODULES`-arrayen (i `AppInner`) definierar alla 21 moduler: `id`, `seq`, `label`,
  `group`, `Component`. Grupper: `grund` (Grundinställningar), `plan` (Tjänsteplanering),
  `analys` (Analys och rapporter), `io` (Import / Export).
- `LOCK_FIELDS`-arrayen (i `PrincipalAdmin`) definierar vilka kategorier en huvudman
  kan låsa: `calendar` (med granulär `calendarSubLocks`: terminsdatum/lov/helgdagar/
  studiedagar/planeringsdagar/schemabrytande dagar), `catTasks`, `nationalTimplan`,
  `lokalTimplan`, `meta.poTillagg`, `revenues`.

## Ägarens förhandsgranskningsgenvägar (redan byggda — fanns innan vi visste om dem!)

Inloggad som ägare kan man redan idag se in i andra paneler utan att logga ut:

- **"🏛 Öppna som huvudman"**-knappen på varje huvudmans rad i `OwnerAdmin` →
  `principalOverlay`-state → renderar `PrincipalAdmin` i en fullskärmsöverlay med
  syntetisk `tenantInfo`. `onSignOut` är omdirigerad till att bara stänga overlayn.
- **Klick på en skolas namn** i listan → `openSchoolTF` → `tfSchool`/`tfState` →
  renderar `AppInner` med `isReadOnly={!tfUnlocked}` och en "🔓 Lås upp för
  redigering"-knapp i en `ownerMode`-banner.

Bra att känna till innan man föreslår att bygga något liknande på nytt.

## Kända fallgropar / fixade buggar (logg)

- **`school_data`-kolumnen heter `data`, inte `state`.** Två ställen i
  `PrincipalAdmin` (`loadAll`, `loadSchoolTF`) läste fel kolumnnamn och fick tyst
  `undefined` tillbaka. Fixat 2026-06-20.
- **`generatePrincipalDemoData` var deklarerad två gånger** med olika signaturer
  (en parameterlös för ägarpanelens tre-skolors-demo, en med `baseState` för
  huvudmannapanelen). esbuilds modul-bundling tillåter inte detta (hård krasch vid
  byggning). Den parameterlösa döptes om till `generateOwnerDemoData`. Fixat 2026-06-20.
- **`GYAppInner` (gymnasiemodulen) har ingen spar-/autosave-koppling alls** mot
  `school_data` just nu — ett tidigare/ofärdigt skal. Separat, större projekt som
  ska byggas ut senare. Påverkas inte av versionshanteringsarbetet.
- **Huvudmannens "revenues"-lås är historiskt trasigt:** `pushToSchools` skrev bara
  över ett gammalt fält `revenues.grundskolepeng` som M8Revenues-modulen aldrig
  läser (riktig data ligger i `state.revenues2`, per årskurs). **`nationalTimplan`-låset
  hade ingen push-logik alls.** Båda måste fixas (planerat som Omgång 3a, ej påbörjat
  än) innan låsvisning i rektorns DTV kan byggas korrekt för dessa två kategorier.

## Projektstatus (2026-06-20)

- ✅ **Omgång 2** (huvudmannens lås/mall-funktion i huvudmannapanelen) — klar och godkänd.
- ✅ **Versionshantering — Runda 1** (databasmigrering: `version_id`/`version_name`/
  `is_primary`, alla läs-/skrivanrop i koden uppdaterade, beteende oförändrat i UI) — klar.
- ⏭️ **Nästa steg, i ordning:**
  1. Runda 2 — version-väljaren som landningsskärm (öppna befintlig / ny tom version).
  2. Runda 3 — koppla "Skicka ut" till huvudversionen specifikt, banner på Översikt
     när en mall väntar, "öppna mall som ny version"-flöde.
  3. Omgång 3a — laga synk-glappen för `nationalTimplan` och `revenues2` i
     `pushToSchools` (se fallgropar ovan).
  4. Omgång 3b — visuell låsvisning i rektorns DTV (gråade fält + 🔒 + klick-meddelande),
     hämtas live från mallen men syns först efter att huvudmannen tryckt "Skicka ut".

## Designregler

Se `declara_designregler.md` i projektet för färgpalett, layoutprinciper, knappstilar
m.m. (Navy `#1B3450` / Beige `#EAE5C8`, CSS Grid för listvyer, fasta kolumnbredder osv.)
