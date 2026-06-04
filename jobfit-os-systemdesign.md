# Jobfit OS – systemdesign og arkitekturoversikt

> Basert på faktisk kodebase i `/root/projects/jobfit` per dagens repo-tilstand.
> Dette dokumentet beskriver hvordan systemet er bygget nå, ikke bare ønsket målarkitektur.

---

## 1. Kort oppsummert

**Jobfit OS** er et internt drifts- og oppfølgingssystem bygget som en **Next.js App Router-applikasjon** med **TypeScript**, **React 19**, **Supabase Auth + Database**, og deploymodell som passer direkte til **Vercel**.

Systemet har to hovedflater:

1. **Dashboard** – for admin og coacher
2. **Portal** – for klient-/deltakerbrukere

I tillegg finnes:

- innlogging (`/auth`)
- offentlig survey-flyt (`/survey/[token]`)
- server actions for CRUD og intern forretningslogikk
- Supabase-migrasjoner for schemautvikling

Arkitekturen er i praksis en **monolittisk webapp**:

- frontend og server-rendering ligger i samme Next.js-prosjekt
- auth og database ligger i Supabase
- deploy/runtime ligger naturlig i Vercel
- forretningslogikk ligger spredt mellom server components, server actions og hjelpebiblioteker i `src/lib`

---

## 2. Teknologistack

Fra `package.json`:

### Rammeverk og språk
- **Next.js 16.2.6**
- **React 19.2.4**
- **TypeScript 5.9**

### Data og auth
- **@supabase/supabase-js**
- **@supabase/ssr**

### UI
- **Tailwind CSS 4** via `@tailwindcss/postcss`
- **lucide-react** for ikoner

### Kvalitetsverktøy
- Typecheck: `tsc --noEmit`
- ESLint: `next/core-web-vitals` + `next/typescript`
- Test: Node test runner via `tsx`

### NPM-scripts
```json
{
  "dev": "next dev --turbopack",
  "build": "next build",
  "start": "next start",
  "typecheck": "tsc --noEmit",
  "check": "npm run typecheck && npm run build",
  "lint": "next lint",
  "test": "node --import tsx --test ./src/app/dashboard/admin/repair-email.test.ts"
}
```

**Vurdering:**
- moderne stack
- enkel drift
- lav deploy-friksjon
- testdekningen ser foreløpig smal ut i repoet (én eksplisitt testscript-target i `package.json`)

---

## 3. Overordnet systemarkitektur

```text
Bruker
  ↓
Vercel-hostet Next.js-app
  ↓
Next App Router (SSR + Server Actions + Client Components)
  ↓
Supabase Auth (sessions/cookies)
  ↓
Supabase Postgres
```

### Arkitekturkarakter
Dette er ikke delt opp i klassiske separate tjenester som:
- eget backend-API
- egen frontend-app
- egen auth-tjeneste

I stedet er løsningen bygget som en **integrert webapplikasjon** hvor:
- UI, routing og SSR håndteres av Next.js
- mange dataoperasjoner går direkte fra server-side kode til Supabase
- auth-kontroll skjer både i request-proxy og på sidenivå

Dette gir:
- rask produktutvikling
- færre moving parts
- enkel hosting

Men også:
- sterk kobling mellom UI og database
- mye ansvar lagt i service-role-operasjoner
- risiko for at tilgangsmodell blir vanskeligere å stramme inn senere

---

## 4. Mappestruktur og kodeorganisering

## Rotnivå
Viktige filer:
- `package.json`
- `tsconfig.json`
- `next.config.ts`
- `postcss.config.mjs`
- `eslint.config.mjs`
- `supabase/config.toml`
- `supabase/migrations/*.sql`

## Appkode
Hovedkode ligger under:
- `src/app` – sider, layouts, server actions
- `src/lib` – auth, Supabase-klienter, typer, domenehjelpere
- `src/components/ui` – gjenbrukbare UI-komponenter

### Faktiske page-ruter i `src/app`
Basert på kodebasen finnes blant annet:

- `/`
- `/auth`
- `/dashboard`
- `/dashboard/admin`
- `/dashboard/calendar`
- `/dashboard/cashflow`
- `/dashboard/chat`
- `/dashboard/coaches`
- `/dashboard/coaches/[id]`
- `/dashboard/companies`
- `/dashboard/companies/[id]`
- `/dashboard/contacts`
- `/dashboard/documents`
- `/dashboard/kpi`
- `/dashboard/leads`
- `/dashboard/leads/bedrift`
- `/dashboard/leads/privat`
- `/dashboard/measurements`
- `/dashboard/meetings`
- `/dashboard/meetings/bedrift`
- `/dashboard/meetings/privat`
- `/dashboard/mine-kunder`
- `/dashboard/packages`
- `/dashboard/participants`
- `/dashboard/partnere`
- `/dashboard/private-clients`
- `/dashboard/private-clients/[id]`
- `/dashboard/profile`
- `/dashboard/reports`
- `/dashboard/rutiner`
- `/dashboard/sessions`
- `/dashboard/tasks`
- `/portal`
- `/portal/chat`
- `/portal/profile`
- `/portal/progresjon`
- `/portal/sessions`
- `/survey/[token]`

### Tolkning
Det betyr at systemet allerede dekker flere domener:
- salg / pipeline
- møter
- kundeoppfølging
- coach-/teamflate
- deltaker/portal
- oppfølging/tasks
- målinger/progresjon
- meldinger/chat
- cashflow/økonomi
- admin/brukerdrift

---

## 5. Frontend- og UI-design

## Designretning
Koden viser en tydelig premium mørk UI-retning:
- mørke bakgrunner (`bg-black`, `bg-zinc-900`, `bg-zinc-950`)
- lyse tekster
- korte dashboard-kort
- sidebar-drevet navigasjon
- visuelle badges/statusindikatorer

Dette stemmer godt med en intern operations-plattform mer enn en offentlig markedsføringsflate.

## Stylingmodell
Det finnes:
- `src/app/globals.css`
- `postcss.config.mjs`
- `next.config.ts`
- ingen egen `tailwind.config.*` i repo-roten per nå

`globals.css` bruker:
```css
@import "tailwindcss";
```

`next.config.ts` er liten, men viktig, og setter:
```ts
experimental: {
  serverActions: {
    bodySizeLimit: '8mb',
  },
}
```

Det peker mot en relativt enkel Tailwind 4-basert setup uten mye ekstra custom config, kombinert med et bevisst løft av body-grensen for server actions.

## Gjenbrukbare UI-komponenter
Eksempler i `src/components/ui`:
- `SidebarNav.tsx`
- `PortalSidebar.tsx`
- `NotificationBell.tsx`
- `ChatWindow.tsx`
- `FollowUpPanel.tsx`
- `SessionEditDrawer.tsx`
- `AdminSurface.tsx`
- `NewsFeed.tsx`
- `PageLoading.tsx`
- `PageError.tsx`

**Vurdering:** UI-laget er relativt lettvekts og håndskrevet, ikke bygget rundt tung ekstern komponentpakke. Det gjør systemet fleksibelt, men stiller høyere krav til konsistens i eget designarbeid.

---

## 6. Routing, request-flyt og adgangskontroll

En viktig fil er `src/proxy.ts`.

Denne fungerer som request-gate foran store deler av appen, og gjør blant annet:
- oppretter Supabase SSR-klient
- leser auth user fra cookie/session
- slipper gjennom offentlige ruter som `/survey/*`
- lar server actions fullføre uten å trigge stygge redirect-feil
- sender ikke-innloggede brukere til `/auth`
- sender klientbrukere til `/portal`
- sender ikke-klientbrukere bort fra `/portal` til `/dashboard`

### Viktig designvalg
Proxyen bruker **rolle fra auth metadata** for tidlig routing, i stedet for å slå opp `profiles` i middleware/proxy.

Det er eksplisitt begrunnet i koden med at ødelagte RLS-policyer på `profiles` ellers kan skape redirect-loops før sidebeskyttelsen får kjørt.

Det er et smart og pragmatisk valg.

---

## 7. Auth-modellen

Auth ligger i praksis i to lag:

### Lag 1: request/proxy
I `src/proxy.ts` avgjøres første redirect basert på session og rollemetadata.

### Lag 2: side-/servernivå
I `src/lib/auth.ts` finnes funksjoner som:
- `requireUser()`
- `requireUserWithProfile()`
- `requirePortalUser()`
- `requireDashboardUser()`
- `requireAdminUser()`

### Logikken er omtrent:
- hvis ingen bruker: send til `/auth`
- hvis rolle = `client`: bruk portal
- hvis rolle != `client`: bruk dashboard
- hvis admin kreves: må være `admin`

### Viktig observasjon
`requireUserWithProfile()` slår opp profil i `profiles` via **admin client**:
```ts
const admin = createAdminClient()
const { data: profile } = await admin
  .from('profiles')
  .select('id, full_name, role, email, phone, avatar_url, company_id')
  .eq('id', user.id)
  .single()
```

Det betyr at applikasjonen i stor grad stoler på:
1. autentisert Supabase-bruker
2. profilrad i `profiles`
3. rollefelt i profil og/eller auth metadata

### Styrke
- tydelig rolleoppdeling
- enkel login-flyt
- lett å forstå

### Risiko
- auth metadata og `profiles.role` kan i teorien drive fra hverandre
- flere steder bruker service-role/admin client, som gjør at appen ikke i like stor grad er “tvunget” av RLS til å være riktig

---

## 8. Brukermodell og roller

Basert på kode og navigasjon fremstår rollene som minst:
- `admin`
- `coach`
- `client`

### Reell oppførsel i systemet
- **Admin**: full dashboard-tilgang, adminflate, brukerdrift
- **Coach**: dashboard-tilgang med egen coach-nav og operative flater
- **Client**: portalflate for egen bedrift/progresjon/sessions/chat/profil

### Superadmin-spesialregel
I `src/lib/config.ts`:
```ts
export const SUPER_ADMIN_EMAIL = 'lavrans.al@jobfit.no'
```

Og i admin-siden:
- innlogget bruker må være denne e-posten for å få tilgang til `/dashboard/admin`

Det betyr at systemet i dag har både:
- **rollebasert adgang**
- og en **hardkodet superadmin-e-post** for den mest sensitive adminflaten

### Konsekvens
Dette er praktisk og tydelig i en tidlig fase, men er også en form for applikasjonsnær policy som på sikt bør dokumenteres og kanskje flyttes til en mer eksplisitt tilgangsmodell.

---

## 9. Dashboard-flaten

`src/components/ui/SidebarNav.tsx` viser hovedinformasjonen om informasjonsarkitekturen.

### Hovedseksjoner i dashboard
- Oversikt
- Salg
  - Leads → Bedrift / Privat
  - Møter → Bedrift / Privat
- Team
  - Trenere
- Kunder
  - Bedrifter
  - Privatkunder
- Partnere
  - Bedriftsbooking
  - Privatbooking
  - Fordelspartnere
  - Utstyr
  - Reise
  - Lokaler
- Drift
  - Økter
  - Kalender
  - Meldinger
- Innsikt
  - Nøkkeltall
  - Kontantstrøm
- Innstillinger
  - Admin (for admin)

### Coach-spesifikk navigasjon
Coach-nav er enklere:
- Min oversikt
- Mine kunder
- Økter
- Meldinger
- Dokumenter
- Rutiner

### Tolkning
Dashboardet er først og fremst et **operativt kontrollsystem**, ikke bare et rapporteringssystem. Det kombinerer:
- salgspipeline
- kundehåndtering
- gjennomføring
- intern kommunikasjon
- økonomisk oversikt

---

## 10. Portal-flaten

Portal-layout og portalpage viser at klientbrukeren får en separat opplevelse.

### Portalens ansvar
- vise tilknyttet bedrift
- vise deltagere og målinger
- vise sessions
- vise progresjon
- vise chat og profil
- vise nyhetsfeed/oppdateringer fra Jobfit

I `src/app/portal/page.tsx` hentes blant annet:
- `companies`
- `participants`
- `measurements`
- `measurement_rounds`
- `sessions`

for brukerens `company_id`.

### Viktig produktmessig detalj
Hvis klientbrukeren ikke er knyttet til en bedrift, vises en kontrollert empty state i stedet for feil. Det er et godt tegn på at systemet håndterer onboarding/halvferdige kontoer pragmatisk.

---

## 11. Supabase-integrasjonen

Supabase brukes til tre ting samtidig:

1. **Auth**
2. **Postgres-database**
3. **SSR/session-integrasjon** via `@supabase/ssr`

### Klienttyper i koden
Det finnes tre tydelige klienter:

#### Browser client
`src/lib/supabase/client.ts`
- bruker `NEXT_PUBLIC_SUPABASE_URL`
- bruker `NEXT_PUBLIC_SUPABASE_ANON_KEY`
- brukes i client components som login-siden

#### Server SSR client
`src/lib/supabase/server.ts`
- koblet til cookies via `next/headers`
- brukes til å lese aktiv innlogget bruker i server context

#### Admin/service-role client
`src/lib/supabase/admin.ts`
- bruker `SUPABASE_SERVICE_ROLE_KEY`
- bypasser vanlige klientbegrensninger
- brukes svært mye i appen

### Viktig observasjon
Store deler av appens dataaksess går gjennom `createAdminClient()`.

Det betyr:
- appen er enkel å bygge videre på
- komplisert RLS kan omgås server-side
- men sikkerhetsmodellen hviler tungt på at applikasjonskoden selv alltid gjør riktige sjekker

Dette er en svært viktig designfakta.

---

## 12. Datamodell i Supabase

Fra `src/lib/types.ts` ser vi at `Database['public']['Tables']` minst inkluderer:

- `profiles`
- `coaches`
- `companies`
- `leads`
- `meetings`
- `private_clients`
- `partners`
- `participants`
- `sessions`
- `availability`
- `measurements`
- `measurement_rounds`
- `expenses`
- `conversations`
- `conversation_participants`
- `messages`
- `notifications`
- `follow_ups`
- `news_posts`
- `documents`
- `service_packages`

### Domeneinndeling

#### Personer og identitet
- `profiles`
- `coaches`
- `participants`
- `private_clients`

#### B2B-kunder
- `companies`
- bedriftsrelaterte `participants`, `measurements`, `sessions`

#### Salg og pipeline
- `leads`
- `meetings`
- `follow_ups`
- private leads / bedriftsleads-logikk i appen

#### Leveranse og drift
- `sessions`
- `availability`
- `documents`
- `service_packages`

#### Innsikt og helse/progresjon
- `measurements`
- `measurement_rounds`

#### Kommunikasjon
- `conversations`
- `conversation_participants`
- `messages`
- `notifications`
- `news_posts`

#### Økonomi/forretning
- `expenses`
- `cashflow`-relatert logikk i appen (selv om tabellnavn ikke fullt ut ble lest fra egne filer her)

### Tolkning
Datamodellen viser at Jobfit OS ikke bare er et CRM eller bare en coach-portal. Det er et hybrid system for:
- CRM/pipeline
- kundeoperasjon
- coach-operasjon
- deltakeroppfølging
- rapportering/progresjon
- intern kommunikasjon

---

## 13. Domene-konstanter og forretningsspråk

`src/lib/domain-options.ts` er viktig fordi den viser produktets språk og states.

Eksempler:

### Company status
- `lead`
- `pilot`
- `aktiv`
- `avsluttet`

### Programtyper
- `foundation`
- `efficiency`
- `executive`

### Lead status
- `ny`
- `kontaktet`
- `møte_booket`
- `tilbud_sendt`
- `forhandling`
- `vunnet`
- `tapt`

### Session status
- `planlagt`
- `gjennomført`
- `avlyst`
- `ikke_møtt`

### Partnerkategorier
- `b2b_booking`
- `privat_booking`
- `fordelspartner`
- `utstyr`
- `reise`
- `lokaler`

Dette er verdifullt fordi det viser at forretningslogikken allerede er ganske konkret, og at produktet har kommet lenger enn bare generiske CRUD-sider.

---

## 14. Server Actions og operativ logikk

Repoet bruker mange **Server Actions** (`'use server'`) i stedet for et stort separat API-lag.

Eksempler finnes i:
- `src/app/actions/chat.ts`
- `src/app/actions/operations.ts`
- `src/app/actions/news.ts`
- `src/app/actions/follow-ups.ts`
- `src/app/dashboard/.../actions.ts`
- `src/app/survey/[token]/actions.ts`

### Hva dette betyr arkitektonisk
I stedet for å bygge et klassisk REST- eller GraphQL-API for alt, ligger skriveoperasjoner og mye forretningslogikk tett på rutene/modulene som bruker dem.

### Fordeler
- rask utvikling
- mindre boilerplate
- lett å holde feature-logikk samlet

### Ulemper
- kan bli vanskeligere å standardisere når systemet vokser
- risiko for at samme validering/tilgangskontroll reimplementeres flere steder
- vanskeligere å gjenbruke som API for mobilapp eller tredjepartsintegrasjoner senere

Dette er relevant fordi du har et uttalt ønske om at systemet ikke skal gjøre fremtidig app-migrasjon vanskeligere.

---

## 15. Chat, varsler og intern kommunikasjon

Koden og tabellene viser et internt kommunikasjonslag med:
- samtaler (`conversations`)
- deltakere i samtaler (`conversation_participants`)
- meldinger (`messages`)
- varsler (`notifications`)
- nyhetsfeed (`news_posts`)

Dette er et viktig designsignal: systemet fungerer ikke bare som register, men også som arbeidsoverflate.

Det betyr at Jobfit OS beveger seg mot å være et faktisk operativsystem for teamet, ikke kun et administrasjonspanel.

---

## 16. Oppfølgingsmodul og robusthet

`src/lib/follow-ups.ts` er interessant fordi den viser en mer moden robusthetsstil.

Koden håndterer blant annet:
- manglende `follow_ups`-tabell
- policy-/RLS-relaterte profileringsfeil
- fallback-adferd med kontrollert degradering

Eksempel: hvis `follow_ups` ikke finnes i produksjonsbasen, returneres kontrollert melding i stedet for total krasj.

### Hvorfor dette er viktig
Det tyder på at prosjektet allerede har erfart produksjonsdrift hvor database og appkode ikke alltid er 100 % synkronisert, og at løsningen derfor begynner å bli mer operasjonelt robust.

---

## 17. Admin-flaten og brukerreparasjon

`/dashboard/admin` ser ut til å være mer enn en enkel brukerliste.

Admin-flaten:
- sammenligner `profiles` mot Supabase Auth-brukere
- ser etter manglende auth-brukere
- oppdager e-postmismatch
- sjekker om coach-profiler mangler tilhørende coach-rad
- viser driftsnøkkeltall

Dette er et sterkt trekk i designet: systemet prøver å være selvreparerende eller i det minste selvdiagnostiserende.

### Arkitektonisk betydning
Det reduserer behovet for manuell SQL-feilsøking ved vanlige kontoavvik.

---

## 18. Supabase-migrasjoner

Migrasjoner i repoet inkluderer blant annet:

- `20260530000000_create_follow_ups.sql`
- `20260601000000_create_partners.sql`
- `20260601010000_portal_round1.sql`
- `20260602000000_private_sales_split.sql`
- `20260602100000_repair_portal_round1_schema.sql`
- `20260603110000_add_postal_address_to_private_leads.sql`
- `20260603113000_backfill_private_lead_postal_addresses.sql`
- `20260603120000_add_lead_company_metadata.sql`
- `20260603153000_add_contact_phone_to_leads.sql`
- `20260604100000_add_role_to_private_leads.sql`
- `20260604113000_fix_meetings_type_to_channel.sql`
- `20260604113100_harden_sales_constraints.sql`

### Hva migrasjonsnavnene forteller
Prosjektet har nylig utviklet særlig disse områdene:
- follow-ups
- partners
- portal
- privat-salg vs bedriftssalg
- lead metadata
- møter/salgsregler/constraints

Det er et godt tegn: endringer skjer via migrasjoner og ikke bare via manuell SQL i produksjon.

---

## 19. Vercel-design og deploymodell

Det finnes **ingen `vercel.json`** i repoet per nå.

Det betyr sannsynligvis at prosjektet bruker **standard Vercel-konvensjoner**:
- repo koblet direkte til Vercel
- Next.js oppdages automatisk
- build kjøres med `next build`
- runtime er standard Next/Vercel-runtime
- miljøvariabler settes i Vercel dashboard

### Indikasjoner i koden
Det finnes fallback til produksjons-URL:
```ts
process.env.NEXT_PUBLIC_APP_URL ?? 'https://jobfit-vert.vercel.app'
```

Det peker mot at produksjonsdomenet er:
- `https://jobfit-vert.vercel.app`

### Praktisk Vercel-bilde
Sannsynlig deployoppsett er:
- pushes til `main` trigger produksjonsdeploy
- preview deploys kan brukes på branches/PR-er
- secrets/miljøvariabler ligger i Vercel
- frontend og server rendering deployes samlet

### Styrker ved dette valget
- veldig rask deploysyklus
- få infra-beslutninger å vedlikeholde
- god match for Next.js App Router

### Begrensninger
- mindre eksplisitt kontroll enn en helt custom infra-stack
- viktig å holde miljøvariabler og Supabase-prosjekter stramt organisert
- prosjektet blir tett koblet til Next/Vercel-måten å kjøre på

---

## 20. Miljøvariabler og secrets

Fra kodebasen er disse sentrale:

### Public
- `NEXT_PUBLIC_SUPABASE_URL`
- `NEXT_PUBLIC_SUPABASE_ANON_KEY`
- `NEXT_PUBLIC_APP_URL` (brukes enkelte steder)

### Server-only
- `SUPABASE_SERVICE_ROLE_KEY`

### Designmessig betydning
Denne delingen er riktig i prinsippet:
- browser får anon/public credentials
- server får service-role

Men siden service-role brukes mye i app-logikken, blir det ekstra viktig at:
- all sensitiv bruk skjer server-side
- auth guards er konsekvente
- ingen server action eksponerer bredere tilgang enn ment

---

## 21. Faktiske styrker i dagens design

### 1. Rask og pragmatisk produktutvikling
Monolitt med Next + Supabase + Vercel er svært effektivt.

### 2. Tydelig rolleoppdeling
Portal vs dashboard er enkel å forstå både teknisk og produktmessig.

### 3. Bra intern informasjonsarkitektur
Salg, kunder, team, drift, innsikt og admin er ryddig separert.

### 4. Typed domenegrunnlag
`src/lib/types.ts` og domene-konstanter gir relativt tydelig felles språk.

### 5. Operasjonell modenhet er på vei
Admin-repair, fallback-håndtering og migrasjonsdisiplin viser at systemet er i ferd med å bli robust, ikke bare funksjonelt.

### 6. God deploy-ergonomi
Vercel + standard Next-oppsett holder terskelen lav for hyppige pushes.

---

## 22. Viktigste risikoer og svakheter

## A. Tung avhengighet av service-role i appkoden
Mange sider og actions bruker `createAdminClient()` direkte.

**Risiko:**
- applikasjonen blir selv ansvarlig for tilgangskontroll
- RLS beskytter mindre enn den ellers kunne gjort
- feil i appkode kan få større konsekvens

## B. Rolleinformasjon finnes i flere lag
Det finnes både:
- Supabase auth metadata
- `profiles.role`
- superadmin-e-postregel

**Risiko:**
- drift mellom kildene
- edge cases ved onboarding eller reparasjon

## C. Forretningslogikk er delvis distribuert
Logikk ligger i:
- pages
- actions
- `src/lib`
- UI-komponenter

**Risiko:**
- vanskeligere vedlikehold over tid
- mer krevende å finne “source of truth”

## D. Begrenset testbilde i repoet
Det synlige testscriptet peker på én spesifikk test.

**Risiko:**
- regressjoner ved raske pushes til `main`
- viktig logikk kan være verifisert mest manuelt

## E. Fremtidig mobil-/API-ekspansjon kan bli dyrere
Når mye logikk er bundet til Server Actions og sidekontekst, kan det bli mer arbeid å:
- bygge native app
- eksponere stabilt API
- integrere med andre systemer

---

## 23. Anbefalt målarkitektur videre

Dette er ikke et argument for å rive dagens løsning. Tvert imot: dagens stack er god. Men jeg ville anbefalt følgende utviklingsretning:

### 1. Behold Next + Supabase + Vercel
Det er fortsatt riktig fundament.

### 2. Stram inn service-role-bruken gradvis
Flytt etter hvert mer lesing/skriving over til:
- brukerbundet klient der det er naturlig
- tydelige server-side domain services der service-role faktisk trengs

### 3. Samle domeneoperasjoner tydeligere
For eksempel egne server-side domeneområder som:
- `src/lib/domain/leads/*`
- `src/lib/domain/companies/*`
- `src/lib/domain/portal/*`
- `src/lib/domain/admin/*`

Da blir pages og actions tynnere.

### 4. Definer én sann kilde for roller
Helst eksplisitt dokumentert:
- auth metadata brukes for tidlig routing
- `profiles.role` brukes som appens operative rolle
- synk må håndheves gjennom admin tooling eller onboarding-flyt

### 5. Bygg ut testdekning på kritiske flyter
Først prioritet:
- auth/redirect-logikk
- admin user repair
- lead/meeting transitions
- participant/measurement mapping
- portaltilgang og company binding

### 6. Innfør mer eksplisitt deploydokumentasjon
Selv om `vercel.json` ikke er nødvendig, bør det finnes skriftlig oversikt over:
- required env vars
- branch → miljø
- deploy policy
- rollback policy
- migrasjonsrutine

---

## 24. Konkret vurdering av systemet slik det står nå

Hvis jeg skal beskrive prosjektet i én setning:

> **Jobfit OS er en raskt utviklet, allerede ganske kapabel operations-plattform for salg, kundeoppfølging, coacharbeid og deltakerportal – bygget som en Next.js/Supabase/Vercel-monolitt med tydelig rollemodell, men fortsatt med teknisk gjeld rundt service-role-bruk, rolle-synk og standardisering av domene-logikk.**

Det er i praksis et godt fundament for videre vekst, spesielt hvis målet er:
- høy utviklingshastighet
- aggressive pushes
- intern bruk først
- gradvis profesjonalisering

---

## 25. Konkrete filreferanser for videre arbeid

Hvis du eller andre skal forstå systemet raskt, er disse filene de viktigste startpunktene:

### App-shell og routing
- `src/app/page.tsx`
- `src/app/layout.tsx`
- `src/app/dashboard/layout.tsx`
- `src/app/portal/layout.tsx`
- `src/proxy.ts`

### Auth og tilgang
- `src/lib/auth.ts`
- `src/lib/config.ts`
- `src/app/auth/page.tsx`
- `src/app/auth/logout/route.ts`

### Supabase
- `src/lib/supabase/client.ts`
- `src/lib/supabase/server.ts`
- `src/lib/supabase/admin.ts`
- `src/lib/types.ts`
- `supabase/config.toml`
- `supabase/migrations/*.sql`

### Navigasjon og IA
- `src/components/ui/SidebarNav.tsx`
- `src/components/ui/PortalSidebar.tsx`

### Nøkkeldomener
- `src/lib/domain-options.ts`
- `src/lib/follow-ups.ts`
- `src/app/dashboard/admin/page.tsx`
- `src/app/portal/page.tsx`
- `src/app/dashboard/page.tsx`

---

## 26. Forslag til neste dokumenter

Dette systemdesigndokumentet bør ideelt følges av tre mer operative dokumenter:

1. **Databasemanual / ERD-dokument**
   - tabeller
   - relasjoner
   - formål per tabell
   - viktige constraints

2. **Deploy- og miljømanual**
   - Vercel-oppsett
   - miljøvariabler
   - Supabase-prosjekter
   - migrasjonsflyt
   - rollback-rutine

3. **Bruker- og tilgangsmanual**
   - roller
   - hvilke flater hver rolle har tilgang til
   - onboarding/offboarding
   - admin repair flows

---

## 27. Konklusjon

Jobfit OS er allerede mer enn et enkelt adminpanel. Det er en voksende intern plattform som binder sammen:
- salg
- leveranse
- coachdrift
- klientportal
- meldinger
- rapportering
- admin-reparasjon

Teknisk er det et fornuftig valg å bygge dette som:
- **Next.js App Router**
- **Supabase Auth + Postgres**
- **Vercel deploy**

Det som avgjør hvor bra systemet blir videre, er trolig ikke ny stack, men hvor godt dere:
- standardiserer tilgangskontroll
- samler domene-logikk
- bygger testdekning
- dokumenterer deploy og dataflyt

Per nå fremstår løsningen som **rask, pragmatisk, produktiv og ganske nær en ekte intern operations-plattform**, med neste naturlige steg i retning av mer robust struktur og tydeligere sikkerhets-/domenegrenser.

---

## 28. Visuelt arkitekturdiagram

Nedenfor er et tekstbasert arkitekturdiagram som kan rendres i mange Markdown-lesere med Mermaid-støtte:

```mermaid
graph TD
    U[Brukere
Admin / Coach / Client] --> V[Vercel-hostet Next.js app]
    V --> P[Proxy/auth-gating
src/proxy.ts]
    P --> A[/auth]
    P --> D[/dashboard]
    P --> O[/portal]
    P --> S[/survey/[token]]

    D --> DA[Dashboard pages + server actions]
    O --> PA[Portal pages]
    S --> SA[Survey-form + submit actions]

    DA --> L[Domenehjelpere i src/lib]
    PA --> L
    SA --> L

    L --> SSR[Supabase SSR client]
    L --> ADM[Supabase admin/service-role client]
    SSR --> AUTH[Supabase Auth]
    ADM --> DB[(Supabase Postgres)]
    SSR --> DB
    AUTH --> DB

    DB --> T1[Operative tabeller
companies, leads, meetings, sessions]
    DB --> T2[Portal og målinger
participants, measurement_rounds, measurements]
    DB --> T3[Støttedomener
partners, documents, service_packages, follow_ups]
    DB --> T4[Kommunikasjon
conversations, messages, notifications, news_posts]
```

### Arkitekturforklaring
- **Innlogging og redirect-logikk** starter i `src/proxy.ts`, som bestemmer om brukeren skal til `/auth`, `/dashboard` eller `/portal`.
- **Tidlig routing** bruker auth metadata for å unngå redirect-loops hvis `profiles` eller RLS/policies er i ubalanse.
- **Operativ rollevalidering** skjer i `src/lib/auth.ts`, som leser profil via server-side admin-klient og håndhever `admin` / `coach` / `client`.
- **Dataaksess** skjer både via vanlig SSR-klient og via service-role-klient, avhengig av om koden kjører på vegne av bruker eller som administrativ app-logikk.

---

## 29. Databasedokument (integrert appendiks)

Denne delen fungerer som et konsolidert databasekapittel inne i systemdesign-dokumentet. Den er basert på:
- `src/lib/types.ts`
- `src/app/**`-kall mot Supabase
- `supabase/migrations/*.sql`
- eksisterende notater i `docs/database/current-schema.md`

### 29.1 Databasenivå – hovedobservasjon
Jobfit OS er i praksis bygget rundt én Supabase/Postgres-database som dekker fire hovedområder:
1. **Brukere og tilgang**
2. **Salg og pipeline**
3. **Leveranse / coachdrift / portal**
4. **Støttedomener som dokumenter, pakker, varsler og oppfølging**

### 29.2 Tabeller appen bruker
| Tabell | Formål | Hovedbruk i appen |
|---|---|---|
| `profiles` | App-profiler koblet til Supabase Auth | rolle, navn, portalbinding, varsler, chat |
| `coaches` | Coach-spesifikke felter | kapasitet, satser, lønn, coachoversikter |
| `companies` | B2B-kunder | dashboard, portal, cashflow, coachtilknytning |
| `leads` | Bedriftsleads | salgsboard og møteflyt |
| `private_leads` | Privatleads | privat salgsflyt og møtebookinger |
| `meetings` | Salgs-/oppfølgingsmøter | bedrift, privat, kalender |
| `private_clients` | Privatkunder | coacharbeid, økonomi, sessions |
| `partners` | Partnerregister | partnerarbeidsflate og booking/fordeler |
| `participants` | Deltakere i bedrifter | målinger og portalgrunnlag |
| `sessions` | Planlagte/gjennomførte økter | kalender, drift, portal |
| `availability` | Coachtilgjengelighet | typegrunnlag for kapasitetsmodell |
| `measurement_rounds` | Målerunder per bedrift | survey-utsending og progresjon |
| `measurements` | Enkeltmålinger/svar | survey, progresjon, KPI |
| `expenses` | Kostnader | cashflow |
| `conversations` | Chat-tråder | meldingsinnboks |
| `conversation_participants` | Deltakere i tråder | chat access og mottakere |
| `messages` | Meldinger i samtaler | chat og portal |
| `notifications` | Varsler | dashboard/portal-bjelle |
| `follow_ups` | Neste steg / oppfølgingsmotor | leads, selskaper, privatkunder, privatleads |
| `news_posts` | Interne oppdateringer | portal feed / meldinger |
| `documents` | Dokumentmetadata | profil, klient, portal |
| `service_packages` | Pakke-/produktkatalog | pakkeadministrasjon |

### 29.3 Viktigste relasjoner
| Fra | Til | Betydning |
|---|---|---|
| `coaches.id` | `profiles.id` | Coach er en utvidelse av en profil/auth-bruker |
| `profiles.company_id` | `companies.id` | Knytter client-bruker til riktig portalbedrift |
| `companies.assigned_coach_id` | `coaches.id` | Ansvarlig coach på B2B-kunde |
| `participants.company_id` | `companies.id` | Deltaker tilhører én bedrift |
| `sessions.company_id` | `companies.id` | Gruppe-/B2B-session mot bedrift |
| `sessions.private_client_id` | `private_clients.id` | Privat-session mot privatkunde |
| `sessions.coach_id` | `coaches.id` | Utførende coach |
| `private_clients.coach_id` | `coaches.id` | Ansvarlig coach på privatkunde |
| `measurement_rounds.company_id` | `companies.id` | Målerunde tilhører bedrift |
| `measurements.participant_id` | `participants.id` | En måling tilhører en deltaker |
| `measurements.round_id` | `measurement_rounds.id` | En måling kan knyttes til en målerunde |
| `meetings.company_id` | `companies.id` | Møte koblet til bedrift |
| `meetings.lead_id` | `leads.id` | Møte koblet til bedriftslead |
| `meetings.private_lead_id` | `private_leads.id` | Møte koblet til privatlead |
| `documents.owner_user_id` | `profiles.id` | Dokument synlig for/tilordnet bruker |
| `documents.company_id` | `companies.id` | Dokument knyttet til bedrift |
| `documents.private_client_id` | `private_clients.id` | Dokument knyttet til privatkunde |
| `follow_ups.owner_user_id` | `profiles.id` | Eier av oppfølging |
| `messages.conversation_id` | `conversations.id` | Melding tilhører en samtale |
| `conversation_participants.user_id` | `profiles.id` | Bruker koblet til samtale |

### 29.4 Viktige domene-/constraint-observasjoner
- `src/proxy.ts` bruker **auth metadata role** for tidlig redirect, mens `src/lib/auth.ts` bruker **`profiles.role`** for operativ tilgangskontroll.
- `SUPER_ADMIN_EMAIL` i `src/lib/config.ts` setter en egen ekstra gate for `/dashboard/admin`.
- `measurements.token` driver offentlig survey-flyt via `/survey/[token]`.
- `follow_ups` er designet som en felles oppfølgingsmotor, og eksisterende notater peker på unik åpen oppfølging per objekt.
- `private_leads` brukes aktivt i kode (`private-leads`, `meetings`, `follow-ups`), men ligger **ikke** i `Database.public.Tables`-blokken i `src/lib/types.ts`. Det er en konkret type-/schema-mismatch som bør rettes.

### 29.5 Migrasjoner som finnes i repoet
| Migrasjon | Hva navnet tilsier |
|---|---|
| `20260530000000_create_follow_ups.sql` | etablerer oppfølgingsmotoren |
| `20260601000000_create_partners.sql` | innfører partnerdomenet |
| `20260601010000_portal_round1.sql` | første portal-runde |
| `20260602000000_private_sales_split.sql` | skiller privat-salg fra bedriftssalg |
| `20260602100000_repair_portal_round1_schema.sql` | schema-reparasjon etter portal-runde 1 |
| `20260603110000_add_postal_address_to_private_leads.sql` | adressefelt på privatleads |
| `20260603113000_backfill_private_lead_postal_addresses.sql` | backfill av privatlead-adresser |
| `20260603120000_add_lead_company_metadata.sql` | ekstra selskapsmetadata på leads |
| `20260603153000_add_contact_phone_to_leads.sql` | telefon på leads |
| `20260604100000_add_role_to_private_leads.sql` | rollefelt på privatleads |
| `20260604113000_fix_meetings_type_to_channel.sql` | justerer møtetype-/kanalmodell |
| `20260604113100_harden_sales_constraints.sql` | strammer salgsconstraints |

### 29.6 Hva databasen forteller om produktretningen
Databasen viser at Jobfit OS ikke bare er et CRM. Den kombinerer:
- **pipeline og møtehåndtering**
- **B2B-kundeleveranse**
- **privatkundeoppfølging**
- **målinger/surveys og portal**
- **økonomi, dokumenter, partnere og kommunikasjon**

Det gjør databasen til kjernen i hele operations-plattformen.

---

## 30. Deploy-/runbook for Vercel + Supabase

Denne runbooken er skrevet ut fra faktisk repo-oppsett: `package.json`, `.env.example`, `next.config.ts`, `supabase/config.toml` og de eksisterende migrasjonene.

### 30.1 Arkitektur for drift
- **Applikasjon/runtime:** Vercel-hostet Next.js-app
- **Database + auth + storage:** Supabase
- **Kodekilde:** GitHub-repo med `main` som produksjonsnær gren
- **Bygg:** standard Next.js build (`next build`)

### 30.2 Miljøvariabler som må settes
Fra `.env.example`:

| Variabel | Hvor brukes den | Type |
|---|---|---|
| `NEXT_PUBLIC_SUPABASE_URL` | Supabase URL i browser + SSR | public |
| `NEXT_PUBLIC_SUPABASE_ANON_KEY` | client/SSR auth mot Supabase | public |
| `SUPABASE_SERVICE_ROLE_KEY` | admin-klient/server-side privilegerte kall | server-only |
| `NEXT_PUBLIC_APP_URL` | produksjons-/base-URL brukt i enkelte flows med fallback til `https://jobfit-vert.vercel.app` | public/recommended |

### 30.3 Første gangs oppsett i Vercel
1. Importer GitHub-repoet i Vercel.
2. La Vercel autodetektere at dette er en **Next.js**-app.
3. Sett miljøvariablene over for minst miljøene **Production** og **Preview**.
4. Sørg for at `NEXT_PUBLIC_APP_URL` peker på riktig miljø-URL, ikke bare fallback-domenet.
5. Bekreft at build-kommandoen er standard (`next build`) og at install skjer med npm.

### 30.4 Lokal verifisering før deploy
Kjør i repo-roten:

```bash
npm ci
npm run typecheck
npm run build
npm test
```

Praktisk tolkning:
- `npm run typecheck` verifiserer TypeScript uten emit
- `npm run build` verifiserer at Next.js-ruter, SSR og bundling faktisk bygger
- `npm test` kjører den eksplisitte testflyten som i dag finnes i `package.json`

### 30.5 Forslag til branch → miljø-policy
| Branch / hendelse | Miljø | Formål |
|---|---|---|
| `main` | Production | reell produksjonsdeploy |
| feature-/arbeidsbranch | Preview | kontrollere UI, flows og regressjoner |
| lokal maskin | Local | rask iterasjon mot lokal/remote Supabase |

### 30.6 Migrasjonsrutine for Supabase
Repoet har `supabase/config.toml` og SQL-migrasjoner. Anbefalt operativ rutine:

1. **Les migrasjonen før deploy** og forstå om den er additive, destructive eller backfill.
2. **Kjør/verifiser lokalt eller i testprosjekt** først hvis endringen berører auth, sessions, portal eller salgsconstraints.
3. **Appliser migrasjoner til riktig Supabase-prosjekt** før eller i tett koordinasjon med appdeploy.
4. **Deploy appen til Vercel** etter at schemaet er kompatibelt.
5. **Verifiser kritiske flows**: innlogging, dashboard, portal, survey, admin, lead/meeting actions.

### 30.7 Praktisk produksjons-sjekkliste ved release
- [ ] `git diff`/endringsomfang forstått
- [ ] alle relevante migrasjoner identifisert
- [ ] miljøvariabler er satt i Vercel
- [ ] `npm run typecheck` grønn
- [ ] `npm run build` grønn
- [ ] Supabase-schema og appkode er kompatible
- [ ] `/auth`, `/dashboard`, `/portal` og `/survey/[token]` fungerer
- [ ] adminflaten laster for superadmin-konto

### 30.8 Rollback-/incident-tankegang
Hvis deploy gir feil:
1. **Finn først om feilen er appkode eller schema-drift**.
2. Hvis appen er feil men schema fortsatt bakoverkompatibelt: **rollback Vercel deploy**.
3. Hvis schema er feil: lag **fremover-reparerende migrasjon** eller gjenopprett fra backup etter behov.
4. Verifiser spesielt auth-gating, portalredirects og server actions som bruker service-role.

### 30.9 Viktigste operative risikoer
- tung bruk av `createAdminClient()` betyr at appen selv må være konsekvent på tilgangskontroll
- rolleinformasjon finnes i flere lag (auth metadata, `profiles.role`, superadmin-e-post)
- migrasjoner og appkode må holdes tett synkronisert, ellers kommer fallback-/repair-scenarier i spill

---

## 31. Bruker-/rolle-dokument

### 31.1 Roller som finnes i koden
| Rolle | Hvor den finnes | Praktisk betydning |
|---|---|---|
| `admin` | `profiles.role` og auth metadata | full dashboard-tilgang; kan nå adminnære flater |
| `coach` | `profiles.role` og auth metadata | dashboard-bruker uten portalrolle |
| `client` | `profiles.role` og auth metadata | sendes til portal, ikke dashboard |
| `superadmin` (implisitt) | `SUPER_ADMIN_EMAIL = lavrans.al@jobfit.no` | ekstra e-postbasert gate til `/dashboard/admin` |

### 31.2 Tilgangslogikk i praksis
#### I `src/proxy.ts`
- ikke innlogget bruker → redirect til `/auth`
- innlogget `client` → låses til `/portal`
- innlogget ikke-`client` → låses bort fra `/portal` og sendes til `/dashboard`
- `/survey/*` er offentlig
- server-action POST-er får passere for å unngå Next.js-feilrespons ved middleware-redirect

#### I `src/lib/auth.ts`
- `requireUser()` → krever bare innlogging
- `requireUserWithProfile()` → krever innlogging + `profiles`-rad
- `requirePortalUser()` → krever `profile.role === 'client'`
- `requireDashboardUser()` → blokkerer `client` fra dashboard
- `requireAdminUser()` → krever `profile.role === 'admin'`

#### I `src/app/dashboard/admin/page.tsx`
- siden krever innlogging
- deretter sjekkes `user.email === SUPER_ADMIN_EMAIL`
- vanlige admins kan dermed være `admin` i rollemodell, men likevel ikke få denne spesifikke superadmin-flaten

### 31.3 Flater per rolle
| Flate | Admin | Coach | Client |
|---|---:|---:|---:|
| `/auth` | ja, men redirectes videre når innlogget | ja, men redirectes videre når innlogget | ja, men redirectes videre når innlogget |
| `/dashboard` | ja | ja | nei |
| `/dashboard/admin` | bare superadmin-e-post | nei | nei |
| `/dashboard/companies` | ja | i praksis dashboard-rolle kan nå hvis lenket/guard ikke blokkerer | nei |
| `/dashboard/private-clients` | ja | i praksis dashboard-rolle kan nå hvis lenket/guard ikke blokkerer | nei |
| `/dashboard/coaches` | ja | dashboard-bruker | nei |
| `/dashboard/chat` | ja | ja | nei |
| `/dashboard/documents` | ja | ja | nei |
| `/portal` | nei | nei | ja |
| `/portal/progresjon` | nei | nei | ja |
| `/portal/sessions` | nei | nei | ja |
| `/portal/chat` | nei | nei | ja |
| `/portal/profile` | nei | nei | ja |
| `/survey/[token]` | offentlig | offentlig | offentlig |

### 31.4 Navigasjon som bekrefter rollebildet
Fra `src/components/ui/SidebarNav.tsx`:
- coach-navigasjon er en nedskalert dashboardvariant med fokus på **min oversikt**, **mine kunder**, **økter**, **meldinger**, **dokumenter** og **rutiner**
- admin/dashboard-navigasjon er bredere og dekker **salg**, **kunder**, **partnere**, **team**, **operasjon** og **innsikt**
- adminlenken i sidebaren vises bare når `isAdmin` er sann

Fra `src/components/ui/PortalSidebar.tsx`:
- portalbrukere får **Oversikt**, **Progresjon**, **Sessions**, **Meldinger** og **Profil**

### 31.5 Onboarding-/driftsimplikasjoner
For at en ny bruker skal fungere riktig, må minst disse tingene være synkronisert:
1. Supabase Auth-bruker finnes
2. `profiles`-rad finnes
3. `profiles.role` er korrekt
4. auth metadata role er korrekt nok til tidlig redirect
5. hvis coach: tilhørende `coaches`-rad finnes
6. hvis client: `profiles.company_id` peker til riktig `companies.id`

### 31.6 Praktisk risiko i rollemodellen
- det finnes **to operative rollelag**: auth metadata og `profiles.role`
- det finnes i tillegg en **e-postbasert superadmin-gate**
- admin repair-flaten eksisterer nettopp fordi disse lagene kan komme i ubalanse

### 31.7 Anbefalt presisering videre
Den reneste dokumenterte regelen ser ut til å være:
- **auth metadata** brukes til tidlig request-routing
- **`profiles.role`** brukes som appens operative kilde
- **superadmin-e-post** brukes bare til særlig sensitive adminfunksjoner

---

## 32. Filkart for `main`-branch på GitHub

Denne delen er generert fra:

```bash
git ls-tree -r --name-only main
```

Det betyr at tabellen viser **sporbare filer på `main`-branch**, ikke nødvendigvis lokale/urørte arbeidsfiler i denne sesjonen. I denne repo-tilstanden ga kommandoen **170 filer**.

### 32.1 Mappeoversikt
| Område | Forklaring |
|---|---|
| `Rot` | Prosjektkonfig, package scripts, lint/build-oppsett og eksempel-env. |
| `docs/` | Eksisterende intern dokumentasjon og plan-/operasjonsnotater. |
| `public/` | Statiske assets som servest direkte av Next.js. |
| `src/app/` | App Router-ruter, layouts, page entries, loading/error states og server actions. |
| `src/components/ui/` | Gjenbrukbare UI-byggesteiner for dashboard, portal og admin. |
| `src/lib/` | Auth, typer, domenehjelpere, konfig og Supabase-klienter. |
| `supabase/` | Supabase-konfig og migrasjoner som beskriver schemautviklingen. |
| `tmp*/` | Midlertidige backup-/debugfiler som fortsatt er sporet på `main`. |

### 32.2 Full filtabell for `main`
| Fil | Type | Hva den gjør |
|---|---|---|
| `.env.example` | `example` | Eksempel på nødvendige miljøvariabler for lokal/prosjektoppsett. |
| `.gitignore` | `fil` | Git-regler for hvilke lokale filer og artefakter som ikke skal spores. |
| `CLAUDE.md` | `md` | Prosjektkontekst og arbeidsnotater for AI/agenter som jobber i repoet. |
| `docs/database/current-schema.md` | `md` | Eksisterende databasekart/notat om schema og tabellbruk. |
| `docs/operations/jobfit-os-map.md` | `md` | Operativt oversiktsdokument over systemet og modulene. |
| `docs/plans/2026-05-30-next-middleware-to-proxy.md` | `md` | Plan/notat for overgang fra middleware til proxy. |
| `docs/plans/2026-05-30-oppfolgingsmotor-v1.md` | `md` | Plan/notat for oppfølgingsmotor versjon 1. |
| `eslint.config.mjs` | `mjs` | ESLint-oppsett for kodekvalitet og Next.js/TypeScript-regler. |
| `next.config.ts` | `ts` | Next.js-konfigurasjon; øker body-size-grensen for Server Actions til 8 MB. |
| `package-lock.json` | `json` | Låst npm-avhengighetstre for reproduserbare installs. |
| `package.json` | `json` | Prosjektmanifest med scripts, runtime-avhengigheter og dev-verktøy. |
| `postcss.config.mjs` | `mjs` | PostCSS-oppsett som kobler inn Tailwind CSS 4. |
| `public/file.svg` | `svg` | Standard statisk SVG-ikon levert med Next/public. |
| `public/globe.svg` | `svg` | Standard statisk SVG-ikon levert med Next/public. |
| `public/next.svg` | `svg` | Next.js-logo som statisk asset. |
| `public/vercel.svg` | `svg` | Vercel-logo som statisk asset. |
| `public/window.svg` | `svg` | Standard statisk SVG-ikon levert med Next/public. |
| `src/app/actions/chat.ts` | `ts` | Feature-nær TypeScript-modul for `actions`. |
| `src/app/actions/follow-ups.ts` | `ts` | Feature-nær TypeScript-modul for `actions`. |
| `src/app/actions/news.ts` | `ts` | Feature-nær TypeScript-modul for `actions`. |
| `src/app/actions/operations.ts` | `ts` | Feature-nær TypeScript-modul for `actions`. |
| `src/app/auth/logout/route.ts` | `ts` | Route handler/API-endepunkt for `/auth/logout`. |
| `src/app/auth/page.tsx` | `tsx` | Next.js page route for `/auth`. |
| `src/app/dashboard/DashboardCalendar.tsx` | `tsx` | Feature-komponent for routeområdet `dashboard` (DashboardCalendar). |
| `src/app/dashboard/admin/AdminPanel.tsx` | `tsx` | Feature-komponent for routeområdet `dashboard/admin` (AdminPanel). |
| `src/app/dashboard/admin/actions.ts` | `ts` | Server Actions for feature/ruteområdet `dashboard/admin`. |
| `src/app/dashboard/admin/page.tsx` | `tsx` | Next.js page route for `/dashboard/admin`. |
| `src/app/dashboard/admin/repair-email.test.ts` | `ts` | Feature-nær TypeScript-modul for `dashboard/admin`. |
| `src/app/dashboard/admin/repair-email.ts` | `ts` | Feature-nær TypeScript-modul for `dashboard/admin`. |
| `src/app/dashboard/calendar/CalendarView.tsx` | `tsx` | Feature-komponent for routeområdet `dashboard/calendar` (CalendarView). |
| `src/app/dashboard/calendar/error.tsx` | `tsx` | Error boundary/feilvisning for routeområdet `/dashboard/calendar`. |
| `src/app/dashboard/calendar/loading.tsx` | `tsx` | Loading-state for routeområdet `/dashboard/calendar`. |
| `src/app/dashboard/calendar/page.tsx` | `tsx` | Next.js page route for `/dashboard/calendar`. |
| `src/app/dashboard/cashflow/CashflowClient.tsx` | `tsx` | Feature-komponent for routeområdet `dashboard/cashflow` (CashflowClient). |
| `src/app/dashboard/cashflow/actions.ts` | `ts` | Server Actions for feature/ruteområdet `dashboard/cashflow`. |
| `src/app/dashboard/cashflow/page.tsx` | `tsx` | Next.js page route for `/dashboard/cashflow`. |
| `src/app/dashboard/chat/ChatInbox.tsx` | `tsx` | Feature-komponent for routeområdet `dashboard/chat` (ChatInbox). |
| `src/app/dashboard/chat/page.tsx` | `tsx` | Next.js page route for `/dashboard/chat`. |
| `src/app/dashboard/coaches/CoachesWorkspace.tsx` | `tsx` | Feature-komponent for routeområdet `dashboard/coaches` (CoachesWorkspace). |
| `src/app/dashboard/coaches/[id]/CoachCompensation.tsx` | `tsx` | Feature-komponent for routeområdet `dashboard/coaches/[id]` (CoachCompensation). |
| `src/app/dashboard/coaches/[id]/actions.ts` | `ts` | Server Actions for feature/ruteområdet `dashboard/coaches/[id]`. |
| `src/app/dashboard/coaches/[id]/error.tsx` | `tsx` | Error boundary/feilvisning for routeområdet `/dashboard/coaches/[id]`. |
| `src/app/dashboard/coaches/[id]/loading.tsx` | `tsx` | Loading-state for routeområdet `/dashboard/coaches/[id]`. |
| `src/app/dashboard/coaches/[id]/page.tsx` | `tsx` | Next.js page route for `/dashboard/coaches/[id]`. |
| `src/app/dashboard/coaches/error.tsx` | `tsx` | Error boundary/feilvisning for routeområdet `/dashboard/coaches`. |
| `src/app/dashboard/coaches/loading.tsx` | `tsx` | Loading-state for routeområdet `/dashboard/coaches`. |
| `src/app/dashboard/coaches/page.tsx` | `tsx` | Next.js page route for `/dashboard/coaches`. |
| `src/app/dashboard/companies/CompaniesTable.tsx` | `tsx` | Feature-komponent for routeområdet `dashboard/companies` (CompaniesTable). |
| `src/app/dashboard/companies/CompanyEditor.tsx` | `tsx` | Feature-komponent for routeområdet `dashboard/companies` (CompanyEditor). |
| `src/app/dashboard/companies/NewCompanyDrawer.tsx` | `tsx` | Feature-komponent for routeområdet `dashboard/companies` (NewCompanyDrawer). |
| `src/app/dashboard/companies/[id]/CompanyDetail.tsx` | `tsx` | Feature-komponent for routeområdet `dashboard/companies/[id]` (CompanyDetail). |
| `src/app/dashboard/companies/[id]/error.tsx` | `tsx` | Error boundary/feilvisning for routeområdet `/dashboard/companies/[id]`. |
| `src/app/dashboard/companies/[id]/loading.tsx` | `tsx` | Loading-state for routeområdet `/dashboard/companies/[id]`. |
| `src/app/dashboard/companies/[id]/measurementActions.ts` | `ts` | Feature-nær TypeScript-modul for `dashboard/companies/[id]`. |
| `src/app/dashboard/companies/[id]/page.tsx` | `tsx` | Next.js page route for `/dashboard/companies/[id]`. |
| `src/app/dashboard/companies/actions.ts` | `ts` | Server Actions for feature/ruteområdet `dashboard/companies`. |
| `src/app/dashboard/companies/error.tsx` | `tsx` | Error boundary/feilvisning for routeområdet `/dashboard/companies`. |
| `src/app/dashboard/companies/loading.tsx` | `tsx` | Loading-state for routeområdet `/dashboard/companies`. |
| `src/app/dashboard/companies/page.tsx` | `tsx` | Next.js page route for `/dashboard/companies`. |
| `src/app/dashboard/contacts/page.tsx` | `tsx` | Next.js page route for `/dashboard/contacts`. |
| `src/app/dashboard/documents/DocumentsManager.tsx` | `tsx` | Feature-komponent for routeområdet `dashboard/documents` (DocumentsManager). |
| `src/app/dashboard/documents/page.tsx` | `tsx` | Next.js page route for `/dashboard/documents`. |
| `src/app/dashboard/error.tsx` | `tsx` | Error boundary/feilvisning for routeområdet `/dashboard`. |
| `src/app/dashboard/kpi/page.tsx` | `tsx` | Next.js page route for `/dashboard/kpi`. |
| `src/app/dashboard/layout.tsx` | `tsx` | Layout-komponent som definerer shell/navigasjon for `/dashboard`. |
| `src/app/dashboard/leads/LeadsKanban.tsx` | `tsx` | Feature-komponent for routeområdet `dashboard/leads` (LeadsKanban). |
| `src/app/dashboard/leads/actions.ts` | `ts` | Server Actions for feature/ruteområdet `dashboard/leads`. |
| `src/app/dashboard/leads/bedrift/page.tsx` | `tsx` | Next.js page route for `/dashboard/leads/bedrift`. |
| `src/app/dashboard/leads/error.tsx` | `tsx` | Error boundary/feilvisning for routeområdet `/dashboard/leads`. |
| `src/app/dashboard/leads/loading.tsx` | `tsx` | Loading-state for routeområdet `/dashboard/leads`. |
| `src/app/dashboard/leads/page.tsx` | `tsx` | Next.js page route for `/dashboard/leads`. |
| `src/app/dashboard/leads/privat/PrivateLeadsList.tsx` | `tsx` | Feature-komponent for routeområdet `dashboard/leads/privat` (PrivateLeadsList). |
| `src/app/dashboard/leads/privat/page.tsx` | `tsx` | Next.js page route for `/dashboard/leads/privat`. |
| `src/app/dashboard/loading.tsx` | `tsx` | Loading-state for routeområdet `/dashboard`. |
| `src/app/dashboard/measurements/page.tsx` | `tsx` | Next.js page route for `/dashboard/measurements`. |
| `src/app/dashboard/meetings/MeetingsList.tsx` | `tsx` | Feature-komponent for routeområdet `dashboard/meetings` (MeetingsList). |
| `src/app/dashboard/meetings/actions.ts` | `ts` | Server Actions for feature/ruteområdet `dashboard/meetings`. |
| `src/app/dashboard/meetings/bedrift/page.tsx` | `tsx` | Next.js page route for `/dashboard/meetings/bedrift`. |
| `src/app/dashboard/meetings/error.tsx` | `tsx` | Error boundary/feilvisning for routeområdet `/dashboard/meetings`. |
| `src/app/dashboard/meetings/loading.tsx` | `tsx` | Loading-state for routeområdet `/dashboard/meetings`. |
| `src/app/dashboard/meetings/page.tsx` | `tsx` | Next.js page route for `/dashboard/meetings`. |
| `src/app/dashboard/meetings/privat/page.tsx` | `tsx` | Next.js page route for `/dashboard/meetings/privat`. |
| `src/app/dashboard/mine-kunder/page.tsx` | `tsx` | Next.js page route for `/dashboard/mine-kunder`. |
| `src/app/dashboard/packages/PackagesManager.tsx` | `tsx` | Feature-komponent for routeområdet `dashboard/packages` (PackagesManager). |
| `src/app/dashboard/packages/page.tsx` | `tsx` | Next.js page route for `/dashboard/packages`. |
| `src/app/dashboard/page.tsx` | `tsx` | Next.js page route for `/dashboard`. |
| `src/app/dashboard/participants/page.tsx` | `tsx` | Next.js page route for `/dashboard/participants`. |
| `src/app/dashboard/partnere/PartnersWorkspace.tsx` | `tsx` | Feature-komponent for routeområdet `dashboard/partnere` (PartnersWorkspace). |
| `src/app/dashboard/partnere/actions.ts` | `ts` | Server Actions for feature/ruteområdet `dashboard/partnere`. |
| `src/app/dashboard/partnere/error.tsx` | `tsx` | Error boundary/feilvisning for routeområdet `/dashboard/partnere`. |
| `src/app/dashboard/partnere/loading.tsx` | `tsx` | Loading-state for routeområdet `/dashboard/partnere`. |
| `src/app/dashboard/partnere/page.tsx` | `tsx` | Next.js page route for `/dashboard/partnere`. |
| `src/app/dashboard/private-clients/PrivateClientEditor.tsx` | `tsx` | Feature-komponent for routeområdet `dashboard/private-clients` (PrivateClientEditor). |
| `src/app/dashboard/private-clients/PrivateClientsList.tsx` | `tsx` | Feature-komponent for routeområdet `dashboard/private-clients` (PrivateClientsList). |
| `src/app/dashboard/private-clients/[id]/ClientSessions.tsx` | `tsx` | Feature-komponent for routeområdet `dashboard/private-clients/[id]` (ClientSessions). |
| `src/app/dashboard/private-clients/[id]/error.tsx` | `tsx` | Error boundary/feilvisning for routeområdet `/dashboard/private-clients/[id]`. |
| `src/app/dashboard/private-clients/[id]/loading.tsx` | `tsx` | Loading-state for routeområdet `/dashboard/private-clients/[id]`. |
| `src/app/dashboard/private-clients/[id]/page.tsx` | `tsx` | Next.js page route for `/dashboard/private-clients/[id]`. |
| `src/app/dashboard/private-clients/actions.ts` | `ts` | Server Actions for feature/ruteområdet `dashboard/private-clients`. |
| `src/app/dashboard/private-clients/error.tsx` | `tsx` | Error boundary/feilvisning for routeområdet `/dashboard/private-clients`. |
| `src/app/dashboard/private-clients/loading.tsx` | `tsx` | Loading-state for routeområdet `/dashboard/private-clients`. |
| `src/app/dashboard/private-clients/page.tsx` | `tsx` | Next.js page route for `/dashboard/private-clients`. |
| `src/app/dashboard/private-leads/actions.ts` | `ts` | Server Actions for feature/ruteområdet `dashboard/private-leads`. |
| `src/app/dashboard/profile/page.tsx` | `tsx` | Next.js page route for `/dashboard/profile`. |
| `src/app/dashboard/reports/page.tsx` | `tsx` | Next.js page route for `/dashboard/reports`. |
| `src/app/dashboard/rutiner/page.tsx` | `tsx` | Next.js page route for `/dashboard/rutiner`. |
| `src/app/dashboard/sessions/SessionsList.tsx` | `tsx` | Feature-komponent for routeområdet `dashboard/sessions` (SessionsList). |
| `src/app/dashboard/sessions/actions.ts` | `ts` | Server Actions for feature/ruteområdet `dashboard/sessions`. |
| `src/app/dashboard/sessions/error.tsx` | `tsx` | Error boundary/feilvisning for routeområdet `/dashboard/sessions`. |
| `src/app/dashboard/sessions/loading.tsx` | `tsx` | Loading-state for routeområdet `/dashboard/sessions`. |
| `src/app/dashboard/sessions/page.tsx` | `tsx` | Next.js page route for `/dashboard/sessions`. |
| `src/app/dashboard/tasks/page.tsx` | `tsx` | Next.js page route for `/dashboard/tasks`. |
| `src/app/favicon.ico` | `ico` | Applikasjonens favicon. |
| `src/app/globals.css` | `css` | Global styling, Tailwind-import og felles CSS for appen. |
| `src/app/layout.tsx` | `tsx` | Global app-layout/root shell for hele Next.js-applikasjonen. |
| `src/app/page.tsx` | `tsx` | Landingsside/root route; inngangspunkt for forsiden. |
| `src/app/portal/PortalOverview.tsx` | `tsx` | Feature-komponent for routeområdet `portal` (PortalOverview). |
| `src/app/portal/chat/page.tsx` | `tsx` | Next.js page route for `/portal/chat`. |
| `src/app/portal/layout.tsx` | `tsx` | Layout-komponent som definerer shell/navigasjon for `/portal`. |
| `src/app/portal/page.tsx` | `tsx` | Next.js page route for `/portal`. |
| `src/app/portal/profile/page.tsx` | `tsx` | Next.js page route for `/portal/profile`. |
| `src/app/portal/progresjon/page.tsx` | `tsx` | Next.js page route for `/portal/progresjon`. |
| `src/app/portal/sessions/page.tsx` | `tsx` | Next.js page route for `/portal/sessions`. |
| `src/app/survey/[token]/SurveyForm.tsx` | `tsx` | Feature-komponent for routeområdet `survey/[token]` (SurveyForm). |
| `src/app/survey/[token]/actions.ts` | `ts` | Server Actions for feature/ruteområdet `survey/[token]`. |
| `src/app/survey/[token]/page.tsx` | `tsx` | Next.js page route for `/survey/[token]`. |
| `src/components/ui/AdminSurface.tsx` | `tsx` | Gjenbrukbar UI-komponent (AdminSurface) brukt på tvers av dashboard, portal eller adminflater. |
| `src/components/ui/ChatWindow.tsx` | `tsx` | Gjenbrukbar UI-komponent (ChatWindow) brukt på tvers av dashboard, portal eller adminflater. |
| `src/components/ui/Drawer.tsx` | `tsx` | Gjenbrukbar UI-komponent (Drawer) brukt på tvers av dashboard, portal eller adminflater. |
| `src/components/ui/FollowUpPanel.tsx` | `tsx` | Gjenbrukbar UI-komponent (FollowUpPanel) brukt på tvers av dashboard, portal eller adminflater. |
| `src/components/ui/NewsFeed.tsx` | `tsx` | Gjenbrukbar UI-komponent (NewsFeed) brukt på tvers av dashboard, portal eller adminflater. |
| `src/components/ui/NewsPostForm.tsx` | `tsx` | Gjenbrukbar UI-komponent (NewsPostForm) brukt på tvers av dashboard, portal eller adminflater. |
| `src/components/ui/NotificationBell.tsx` | `tsx` | Gjenbrukbar UI-komponent (NotificationBell) brukt på tvers av dashboard, portal eller adminflater. |
| `src/components/ui/PageError.tsx` | `tsx` | Gjenbrukbar UI-komponent (PageError) brukt på tvers av dashboard, portal eller adminflater. |
| `src/components/ui/PageLoading.tsx` | `tsx` | Gjenbrukbar UI-komponent (PageLoading) brukt på tvers av dashboard, portal eller adminflater. |
| `src/components/ui/PortalEmptyState.tsx` | `tsx` | Gjenbrukbar UI-komponent (PortalEmptyState) brukt på tvers av dashboard, portal eller adminflater. |
| `src/components/ui/PortalSidebar.tsx` | `tsx` | Gjenbrukbar UI-komponent (PortalSidebar) brukt på tvers av dashboard, portal eller adminflater. |
| `src/components/ui/ProfileWorkspace.tsx` | `tsx` | Gjenbrukbar UI-komponent (ProfileWorkspace) brukt på tvers av dashboard, portal eller adminflater. |
| `src/components/ui/SessionEditDrawer.tsx` | `tsx` | Gjenbrukbar UI-komponent (SessionEditDrawer) brukt på tvers av dashboard, portal eller adminflater. |
| `src/components/ui/SidebarNav.tsx` | `tsx` | Gjenbrukbar UI-komponent (SidebarNav) brukt på tvers av dashboard, portal eller adminflater. |
| `src/lib/auth.ts` | `ts` | Server-side auth-helpers som krever innlogging, roller og profil. |
| `src/lib/config.ts` | `ts` | Små applikasjonskonstanter, inkludert superadmin-e-post. |
| `src/lib/domain-options.ts` | `ts` | Delte domenevalg og metadata for statuser, programtyper og kategorier. |
| `src/lib/env.ts` | `ts` | Miljøvariabelhjelpere og runtime-validering av env-oppsett. |
| `src/lib/follow-ups.ts` | `ts` | Domenehjelpere og robust fallback-logikk for oppfølgingsmotoren. |
| `src/lib/leads.ts` | `ts` | Lead-relaterte typer/konstanter/hjelpere. |
| `src/lib/meetings.ts` | `ts` | Meeting-relaterte typer/konstanter/hjelpere. |
| `src/lib/supabase/admin.ts` | `ts` | Server-only Supabase service-role-klient for administrative/database-bypassing kall. |
| `src/lib/supabase/client.ts` | `ts` | Browser-klient for Supabase i client components. |
| `src/lib/supabase/server.ts` | `ts` | SSR/server-klient for Supabase koblet til cookies/session. |
| `src/lib/types.ts` | `ts` | Typed representasjon av Supabase-databasen og sentrale appdomener. |
| `src/proxy.ts` | `ts` | Request-gate/proxy som håndterer auth-basert redirect-logikk før rutene rendres. |
| `supabase/.gitignore` | `fil` | Ignoreringsregler for lokale Supabase-artefakter. |
| `supabase/config.toml` | `toml` | Supabase CLI-prosjektkonfig for lokal dev, auth, storage og database. |
| `supabase/migrations/20260530000000_create_follow_ups.sql` | `sql` | Supabase-migrasjon som innfører eller justerer schema for: create follow ups. |
| `supabase/migrations/20260601000000_create_partners.sql` | `sql` | Supabase-migrasjon som innfører eller justerer schema for: create partners. |
| `supabase/migrations/20260601010000_portal_round1.sql` | `sql` | Supabase-migrasjon som innfører eller justerer schema for: portal round1. |
| `supabase/migrations/20260602000000_private_sales_split.sql` | `sql` | Supabase-migrasjon som innfører eller justerer schema for: private sales split. |
| `supabase/migrations/20260602100000_repair_portal_round1_schema.sql` | `sql` | Supabase-migrasjon som innfører eller justerer schema for: repair portal round1 schema. |
| `supabase/migrations/20260603110000_add_postal_address_to_private_leads.sql` | `sql` | Supabase-migrasjon som innfører eller justerer schema for: add postal address to private leads. |
| `supabase/migrations/20260603113000_backfill_private_lead_postal_addresses.sql` | `sql` | Supabase-migrasjon som innfører eller justerer schema for: backfill private lead postal addresses. |
| `supabase/migrations/20260603120000_add_lead_company_metadata.sql` | `sql` | Supabase-migrasjon som innfører eller justerer schema for: add lead company metadata. |
| `supabase/migrations/20260603153000_add_contact_phone_to_leads.sql` | `sql` | Supabase-migrasjon som innfører eller justerer schema for: add contact phone to leads. |
| `supabase/migrations/20260604100000_add_role_to_private_leads.sql` | `sql` | Supabase-migrasjon som innfører eller justerer schema for: add role to private leads. |
| `supabase/migrations/20260604113000_fix_meetings_type_to_channel.sql` | `sql` | Supabase-migrasjon som innfører eller justerer schema for: fix meetings type to channel. |
| `supabase/migrations/20260604113100_harden_sales_constraints.sql` | `sql` | Supabase-migrasjon som innfører eller justerer schema for: harden sales constraints. |
| `tmp/sigurd-auth-backup-2.json` | `json` | Midlertidig backup-/debug-JSON relatert til Sigurd-auth, fortsatt sporet på main. |
| `tmp/sigurd-auth-backup.json` | `json` | Midlertidig backup-/debug-JSON relatert til Sigurd-auth, fortsatt sporet på main. |
| `tmp_sigurd_backup.json` | `json` | Midlertidig toppnivå-backupfil relatert til Sigurd-auth, fortsatt sporet på main. |
| `tsconfig.json` | `json` | TypeScript-kompilatorinnstillinger og alias/oppløsningsregler. |
