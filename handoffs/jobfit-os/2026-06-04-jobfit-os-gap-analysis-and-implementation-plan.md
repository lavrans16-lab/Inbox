# Jobfit OS – gap-analyse og implementasjonsplan for ny AI-agent

> Basert på ønsket målarkitektur fra Lavrans + faktisk kodebase i `/root/projects/jobfit` per dagens tilstand.
> Dette dokumentet er skrevet som et **handoff-dokument til en ny AI-agent** som skal kunne gå rett på arbeid uten å gjette på mål, prioriteringer eller retning.

---

## 1. Oppdraget i én setning

**Refaktorer Jobfit OS videre som én samlet Next.js/Supabase/Vercel-monolitt, men gjør rollemodellen, domenestrukturen og B2B-datamodellen tydeligere – spesielt rundt Admin / Coach / Bedrift og Bedrift → Gruppe → Deltaker → Session → Måling.**

Viktig:
- **Ikke bygg systemet på nytt.**
- **Ikke splitt det i flere apper eller tjenester nå.**
- **Prioriter endringer som gir høy strukturell verdi med minst mulig omskriving.**

---

## 2. Produktmål og ønsket målarkitektur

Jobfit OS skal være det sentrale operativsystemet for hele Jobfit, ikke bare:
- et CRM
- et coachsystem
- en kundeportal

Systemet skal dekke disse hoveddomenene:
- Salg
- Kundeoppfølging
- Coachdrift
- B2B-leveranser
- B2C-leveranser
- Kommunikasjon
- Rapportering
- Økonomi

### Primære roller
1. **Admin**
2. **Coach**
3. **Bedrift**

### Viktigste produktprinsipp
For bedriftsportalen er **resultater** viktigere enn kalender og sessions.

Bedriften skal kunne følge utvikling over tid innen:
- Stress
- Produktivitet
- Teamfølelse
- Energi

### Viktigste ønskede B2B-struktur
- Bedrift
  - Gruppe
    - Deltakere
    - Sessions
    - Målinger

Målinger skal være del av B2B-modellen, ikke privatkundemodellen.

---

## 3. Faktisk repo- og systemtilstand nå

Dette er bekreftet i dagens kodebase.

### Stack
- Next.js App Router
- TypeScript
- React
- Supabase Auth + Postgres
- Vercel deploymodell
- Tailwind CSS

### Faktiske sentrale domener som allerede finnes i kode/database
- `profiles`
- `coaches`
- `companies`
- `leads`
- `private_leads`
- `meetings`
- `private_clients`
- `participants`
- `sessions`
- `measurement_rounds`
- `measurements`
- `conversations`
- `messages`
- `notifications`
- `follow_ups`
- `documents`
- `service_packages`
- `expenses`
- `partners`

### Faktiske arbeidsflater/ruter som allerede finnes
- `/dashboard/*` for admin/coach-operasjon
- `/portal/*` for portalbruker
- `/auth`
- `/survey/[token]`

### Faktisk authmodell nå
- Tidlig redirect/gating i `src/proxy.ts`
- Side-/serverguarder i `src/lib/auth.ts`
- Superadmin-gate via `SUPER_ADMIN_EMAIL` i `src/lib/config.ts`

### Faktisk teknisk gjeld som er verifisert
- `createAdminClient()` brukes **97 ganger** i `src/`
- kun **1 testfil** finnes i repoet: `src/app/dashboard/admin/repair-email.test.ts`
- `PrivateLead` finnes i `src/lib/types.ts`, men `private_leads` mangler i `Database.public.Tables`
- B2B-grupper er **ikke** et eksplisitt førsteklasses domene i schemaet; deltakerne har bare feltet `team: string | null`
- rollemodellen er teknisk `admin | coach | client`, mens ønsket produktmodell er `admin | coach | bedrift`

---

## 4. Konklusjon fra gap-analysen

### Det som allerede matcher målbildet godt
Følgende deler trenger ikke rives opp:

#### 4.1 Én samlet app er riktig
Dagens monolitt passer godt med ønsket om ett samlet operativsystem med flere arbeidsflater.

#### 4.2 Domenene finnes allerede i stor grad
Repoet støtter allerede nesten alle hoveddomener:
- leads
- bedrifter
- privatkunder
- coacher
- sessions
- målinger
- meldinger
- økonomi

#### 4.3 Portal/dashbord-separasjon finnes allerede
Det finnes allerede en naturlig oppdeling mellom:
- intern operativ flate (`/dashboard`)
- ekstern portalflate (`/portal`)

#### 4.4 Målinger er allerede delvis modellert
Det finnes allerede:
- `participants`
- `measurement_rounds`
- `measurements`
- survey-token-flyt via `/survey/[token]`

Det betyr at “resultatmotoren” ikke må oppfinnes, bare strammes opp og løftes tydeligere inn i B2B-kjernen.

---

## 5. Viktigste gap mellom ønsket struktur og dagens løsning

## Gap A – Rollen `client` er for vag
### Dagens tilstand
- `UserRole = 'admin' | 'coach' | 'client'`
- portalguarder og proxy bruker `client`

### Måltilstand
- Admin
- Coach
- Bedrift

### Problem
Ordet `client` er for uklart fordi det kan bety:
- bedriftsbruker
- deltaker
- hvilken som helst ekstern bruker

Men ønsket produktmodell peker tydeligst mot **bedriftsportalbruker**.

### Anbefalt retning
- behold eksisterende teknisk rolle midlertidig hvis det minimerer omskriving
- men dokumenter og implementer gradvis at dagens `client` i praksis betyr **bedriftsbruker/company client**
- samle dette i én eksplisitt auth/identity helper

### Verdi
Høy verdi, lav til moderat innsats.

---

## Gap B – B2B-grupper mangler som førsteklasses domene
### Dagens tilstand
- `participants` har `team: string | null`
- sessions, participants og measurements er primært company-bundet
- ingen tydelig `company_groups`-/`groups`-tabell i dagens repo/migrasjoner

### Måltilstand
- Bedrift
  - Gruppe
    - Deltakere
    - Sessions
    - Målinger

### Problem
Dagens modell er for flat til å støtte:
- flere team/grupper i én bedrift
- gruppespesifikke sessions
- gruppespesifikke målerunder/resultater
- tydelig rapportering per gruppe

### Anbefalt retning
Innfør eksplisitt gruppedomene i databasen og koden.

### Verdi
Svært høy verdi, moderat innsats.

---

## Gap C – Service-role brukes for bredt
### Dagens tilstand
- `createAdminClient()` brukes 97 ganger i `src/`
- mye lesing/skriving skjer med service-role i stedet for brukerbundet SSR-klient

### Problem
Tilgangskontroll hviler for tungt på appkode, ikke tydelig nok på brukerbinding og databasegrenser.

### Anbefalt retning
- behold admin client for eksplisitt privilegerte admin-/repair-operasjoner
- flytt gradvis brukerbundne reads/writes til SSR/user-scoped klient der det er naturlig

### Verdi
Svært høy verdi, men bør gjøres gradvis.

---

## Gap D – Forretningslogikk er for spredt
### Dagens tilstand
Logikk ligger i en blanding av:
- `src/app/**/page.tsx`
- `src/app/**/actions.ts`
- `src/app/actions/*.ts`
- `src/lib/*.ts`
- UI-komponenter

### Problem
Systemet er allerede stort nok til at denne spredningen vil gjøre videre arbeid tregere og mer feilutsatt.

### Anbefalt retning
Flytt logikken gradvis mot en tydelig domenestruktur under `src/lib/domain/*`.

### Verdi
Høy verdi, moderat innsats.

---

## Gap E – Type-/schema-drift finnes allerede
### Dagens tilstand
- `PrivateLead` er definert
- `private_leads` brukes i appen
- `private_leads` mangler i `Database.public.Tables`

### Problem
Appens TypeScript-kildesannhet er ikke helt pålitelig.

### Anbefalt retning
- rett mismatch umiddelbart
- vurder senere å generere database-typene fra Supabase-schema

### Verdi
Rask høy-verdi-fiks.

---

## Gap F – Testdekningen er for tynn
### Dagens tilstand
Det finnes bare én eksplisitt testfil i repoet.

### Problem
Dette passer dårlig med raske pushes til `main`, særlig for auth, redirects og rolle-/portalregler.

### Anbefalt retning
Bygg tester rundt:
- proxy redirect-matrisen
- auth/rolle-guarder
- admin repair-flyter
- lead/meeting-overganger
- portalbinding til bedrift

### Verdi
Svært høy verdi.

---

## 6. Hva den nye AI-agenten IKKE skal gjøre

Den nye agenten skal **ikke**:
- splitte systemet i mikroservices
- lage egen backend-app før det er nødvendig
- redesigne alt rundt et generisk CRM-mønster
- erstatte Next.js + Supabase + Vercel
- gjøre en stor rewrite uten tydelig verdi
- prioritere kosmetisk kodeopprydding over rollemodell, B2B-grupper og tilgangskontroll

---

## 7. Anbefalt målarkitektur etter refaktor

## 7.1 Applikasjonsnivå
Behold:
- én Next.js-app
- én Supabase-database
- én deploykjede via Vercel

## 7.2 Rollemodell
Målbildet bør dokumenteres slik:
- **admin** = full systemtilgang
- **coach** = egne kunder, egne bedrifter, egne sessions, egne meldinger
- **bedrift/company client** = kun egen bedrift og egne resultater

Praktisk overgang:
- eksisterende `client` kan beholdes teknisk først
- men må eksplisitt behandles som bedriftsportalrolle i dokumentasjon og auth-hjelpere

## 7.3 Domenestruktur i kode
Ny kode bør gradvis organiseres rundt:
- `src/lib/domain/auth/*`
- `src/lib/domain/leads/*`
- `src/lib/domain/companies/*`
- `src/lib/domain/company-groups/*`
- `src/lib/domain/private-clients/*`
- `src/lib/domain/coaches/*`
- `src/lib/domain/sessions/*`
- `src/lib/domain/measurements/*`
- `src/lib/domain/messaging/*`
- `src/lib/domain/finance/*`
- `src/lib/domain/admin/*`

Pages og server actions bør bli tynnere og kalle disse domenetjenestene.

## 7.4 B2B-datamodell
Målretning:
- Company
  - CompanyGroup
    - Participants
    - Sessions
    - MeasurementRounds
    - Measurements / aggregater

Dette kan innføres gradvis uten full rewrite.

---

## 8. Konkret implementasjonsplan

Dette er den anbefalte rekkefølgen for en ny AI-agent.

## Fase 1 – Høy verdi, lav omskriving
Mål: rydde kildesannhet og tilgangsgrunnlag først.

### 8.1 Fiks type-/schema-drift
**Oppgaver**
- legg `private_leads` inn i `Database.public.Tables` i `src/lib/types.ts`
- verifiser alle steder som bruker `private_leads`
- kjør `npm run typecheck`
- kjør `npm run build`

**Akseptkriterium**
- `private_leads` finnes i typed DB-strukturen
- ingen nye typefeil introdusert

---

### 8.2 Samle auth-/rollehjelpere
**Oppgaver**
- behold `src/lib/auth.ts` som entrypoint, men rydd logikken bak den
- innfør en delt helper, f.eks. `src/lib/domain/auth/identity.ts` eller lignende
- lag eksplisitte helpers som:
  - `getCurrentUserWithProfile()`
  - `getEffectiveRole()`
  - `isSuperAdmin()`
  - `syncAuthMetadataFromProfile()`

**Målregel**
- `src/proxy.ts` bruker auth metadata kun for tidlig redirect
- operativ appautorisasjon bruker `profiles.role`
- superadmin-sjekk går via én helper, ikke spredte e-postsammenligninger

**Akseptkriterium**
- auth/rollebeslutninger er samlet og gjenbrukes fra ett sted

---

### 8.3 Fjern duplisert identity-/profile-sync
**Særlig kandidater**
- `src/app/actions/operations.ts`
- `src/app/dashboard/admin/actions.ts`

**Oppgaver**
- identifiser duplisert logikk for user/profile/metadata-sync
- erstatt med én felles funksjon

**Akseptkriterium**
- profil-/role-/metadata-sync skjer gjennom ett delt kall

---

### 8.4 Bygg tester rundt auth og redirect
**Lag først tester for**
- ikke-innlogget → `/auth`
- `client` på dashboard → `/portal`
- ikke-`client` på portal → `/dashboard`
- `/survey/*` er offentlig
- admin-/coach-/portal-guards oppfører seg riktig

**Akseptkriterium**
- redirect- og rollelogikk er testet, ikke bare manuelt sjekket

---

## Fase 2 – Modellér B2B-grupper riktig
Mål: gjøre databasen og koden mer tro mot ønsket leveransemodell.

### 8.5 Innfør eksplisitt gruppedomene
**Anbefalt ny tabell**
For eksempel:
- `company_groups`
  - `id`
  - `company_id`
  - `name`
  - `status`
  - `program_type` eller tilsvarende hvis nyttig
  - `assigned_coach_id` (valgfritt hvis gruppespesifikk coach trengs)
  - `start_date`
  - `end_date`
  - `created_at`

**Oppgaver**
- lag migrasjon for `company_groups`
- backfill/overgangsstrategi fra `participants.team` til gruppedomene
- vurder om `participants.team` skal beholdes midlertidig, migreres, eller erstattes

**Akseptkriterium**
- B2B-grupper er et eksplisitt objekt i schemaet

---

### 8.6 Knytt deltakere, sessions og målerunder til grupper
**Oppgaver**
- vurder å legge til `group_id` på `participants`
- vurder å legge til `group_id` på `sessions` for B2B-sessions
- vurder å legge til `group_id` på `measurement_rounds`
- sørg for at rapport-/portalgrunnlaget kan aggregeres per gruppe

**Akseptkriterium**
- datastrukturen støtter: Bedrift → Gruppe → Deltakere → Sessions → Målinger

---

## Fase 3 – Gjør bedriftsportalen “resultater først”
Mål: gjøre produktflaten mer tro mot Jobfits faktiske verdi.

### 8.7 Prioriter resultater i portalens informasjonsarkitektur
**Mål for portalrekkefølge**
1. Resultater / utvikling
2. Kommende aktiviteter / sessions
3. Grupper og deltakere
4. Rapporter
5. Meldinger
6. Dokumenter / avtale

**Oppgaver**
- gå gjennom `/portal/page.tsx` og portalnavigasjon
- vurder om nåværende oversikt må omprioriteres
- gjør måledata mer sentral enn kalender-/listevisning

**Akseptkriterium**
- portalens primære verdi for bedriften er tydelig resultatoppfølging

---

### 8.8 Tydeliggjør målingsdomenet som B2B-kjerne
**Oppgaver**
- dokumenter og refaktorer målingsflyten som B2B-spesifikk
- unngå å trekke privatkundemodellen inn i målingsmodellen
- sørg for tydelig aggregert fremstilling av:
  - stress
  - produktivitet/effektivitet
  - teamfølelse / psykologisk trygghet
  - energi

**Akseptkriterium**
- målingssystemet fremstår som et bevisst B2B-resultatdomene, ikke en sidefunksjon

---

## Fase 4 – Reduser service-role-avhengighet og samle domenelogikk
Mål: bedre langsiktig vedlikehold uten rewrite.

### 8.9 Flytt brukerbundne reads/writes bort fra admin client der det er forsvarlig
**Første kandidater**
- profilvisning for innlogget bruker
- portaldata som gjelder brukerens egen bedrift
- egne notifikasjoner/meldinger der tilgang er brukerbundet

**Behold admin client for**
- admin repair
- cross-user operasjoner
- brukeradministrasjon
- eksplisitt privilegerte server-side operasjoner

**Akseptkriterium**
- `createAdminClient()` brukes mer bevisst og mindre som default

---

### 8.10 Flytt logikk til tydelige domenemoduler
**Oppgaver**
- start med domener som har størst spredning eller høyest verdi:
  - auth
  - companies
  - measurements
  - sessions
  - admin
- gjør pages/actions tynnere

**Akseptkriterium**
- sentral forretningslogikk ligger i domenemoduler, ikke tilfeldig spredt

---

## Fase 5 – Driftssikkerhet og quality gates
Mål: passe bedre med raske pushes til `main`.

### 8.11 Utvid testdekningen
Minste prioriterte testområder:
- auth/proxy redirect-logikk
- admin repair
- leads/meetings-statusoverganger
- portal company binding
- målingsmapping og gruppekoblinger

### 8.12 Innfør enkel quality gate
Kjør minst:
```bash
npm ci
npm run typecheck
npm run build
npm test
```

Før arbeid regnes som ferdig skal minst `typecheck` og `build` være grønne.

---

## 9. Konkret anbefalt arbeidsrekkefølge for agenten

Hvis den nye agenten skal jobbe effektivt, bør den gjøre dette i rekkefølge:

1. **Les og forstå auth-/rollekjeden**
   - `src/proxy.ts`
   - `src/lib/auth.ts`
   - `src/lib/config.ts`
   - `src/app/dashboard/admin/actions.ts`
   - `src/app/actions/operations.ts`

2. **Fiks type mismatch først**
   - `src/lib/types.ts`
   - private lead-referanser

3. **Skriv tester før større auth-refaktor**
   - redirect-matrise
   - rolleguarder

4. **Refaktorer identity-/role-sync til ett sted**

5. **Design og implementer `company_groups`-migrasjon**
   - med minst mulig brudd for eksisterende data

6. **Koble B2B-målinger tydeligere til grupper**

7. **Juster portalen mot “resultater først”**

8. **Begynn gradvis å flytte logikk til `src/lib/domain/*`**

9. **Reduser admin-client-bruk der det er trygt**

10. **Verifiser alltid med reell kjøring**
    - `npm run typecheck`
    - `npm run build`
    - relevante tester

---

## 10. Fil- og kodeområder den nye agenten bør starte i

### Auth / tilgang
- `src/proxy.ts`
- `src/lib/auth.ts`
- `src/lib/config.ts`
- `src/lib/supabase/server.ts`
- `src/lib/supabase/admin.ts`

### Brukeradmin / identity-sync
- `src/app/dashboard/admin/actions.ts`
- `src/app/actions/operations.ts`
- `src/app/dashboard/admin/page.tsx`

### Typer / database
- `src/lib/types.ts`
- `supabase/migrations/*.sql`
- `docs/database/current-schema.md`

### B2B / målinger / deltakere
- `src/app/dashboard/companies/[id]/measurementActions.ts`
- `src/app/dashboard/companies/[id]/page.tsx`
- `src/app/dashboard/participants/page.tsx`
- `src/app/portal/page.tsx`
- `src/app/portal/PortalOverview.tsx`
- `src/app/survey/[token]/*`

### Portal / IA
- `src/app/portal/layout.tsx`
- `src/components/ui/PortalSidebar.tsx`
- `src/app/portal/page.tsx`

---

## 11. Beslutninger som bør behandles som låste inntil videre

Den nye agenten bør anta følgende som føringer, ikke åpne spørsmål:

1. **Behold Next.js + Supabase + Vercel**
2. **Behold monolitten**
3. **Admin / Coach / Bedrift er den riktige produktmodellen**
4. **Bedriftsportalen skal prioritere resultater**
5. **B2B-grupper er et viktig manglende domene som bør innføres**
6. **Målinger hører til B2B-leveransen, ikke privatkunde-leveransen**
7. **Aggressive pushes er akseptabelt, men må støttes av bedre testing og type-/build-verifisering**

---

## 12. Definisjon av “ferdig” for første refaktorrunde

Første runde bør regnes som vellykket når disse punktene er oppnådd:

- `private_leads`-typefeilen er rettet
- auth-/rollelogikken er mer samlet og mindre duplisert
- redirect- og rollematrisen er testet
- en konkret plan/migrasjon for B2B-grupper finnes eller er implementert
- portalretningen er tydelig dokumentert som “resultater først”
- `createAdminClient()` brukes mer bevisst i nye/refaktorerte områder
- `npm run typecheck` og `npm run build` går grønt etter endringene

---

## 13. Kort instruks til agenten

Hvis du er den nye AI-agenten som leser dette:

1. **Ikke redesign hele systemet.**
2. **Start med rollemodell, auth-opprydding og typefix.**
3. **Deretter innfør B2B-grupper som eksplisitt domene.**
4. **Prioriter målings- og resultatmodellen som kjernen i bedriftsportalen.**
5. **Flytt logikk gradvis til domenemoduler, ikke via stor rewrite.**
6. **Verifiser alt med ekte `typecheck`, `build` og tester før du sier du er ferdig.**

---

## 14. Anbefalt første konkrete arbeidsoppgave til agenten

Start med denne pakken:

### Oppgavepakke 1
- rett `private_leads` i `src/lib/types.ts`
- lag/flytt felles auth-/identity-helper
- reduser duplisert profile/metadata-sync
- legg til tester for proxy-/rollelogikk
- kjør `npm run typecheck` og `npm run build`

### Oppgavepakke 2
- design migrasjon for `company_groups`
- planlegg overgang fra `participants.team` til eksplisitt gruppedomene
- identifiser hvilke eksisterende visninger/actions som må oppdateres først

Dette er den mest verdifulle og minst destruktive måten å starte refaktoren på.
