# HomeLibrary – Architektúra

> **Utolsó frissítés:** 2026-03-28
> **Státusz:** v3 – végleges, Lambda + Neon PostgreSQL stack

## Tech Stack

### Backend

| Komponens | Technológia | Indok |
|-----------|-------------|-------|
| Language | Java 21 LTS | Meglévő tudás, LTS |
| Framework | Spring Boot 3.x | Meglévő tudás |
| Lambda adapter | `aws-serverless-java-container` | Spring Boot → Lambda bridge |
| Auth | Spring Security + JWT | Elegendő ennél a skálánál, Cognito nem szükséges |
| ORM | Spring Data JPA + Hibernate | Standard Spring stack |
| DB migráció | Flyway | Verziókövetett sémaváltozások |
| Build | Maven + maven-shade-plugin | Executable JAR Lambda deployhoz |
| Container | Docker (csak lokális dev) | Lambda-ra JAR-t deployzunk, nem image-et |

### Frontend

| Komponens | Technológia | Indok |
|-----------|-------------|-------|
| Framework | React 18 + TypeScript | Projekt követelmény, TS hosszú távon jobb |
| Build tool | Vite | Gyors, modern (CRA deprecated) |
| UI library | shadcn/ui + Tailwind CSS | Dark mode natívan, könnyű skin váltás |
| i18n | i18next | De facto standard React i18n |
| State | Zustand | Egyszerűbb mint Redux, elég ennél az appnál |
| HTTP | Axios | Standard, interceptorokkal JWT kezelés |
| Vonalkód | react-zxing | Böngészős kamera API wrapper, mobil-optimalizált |
| Routing | React Router v6 | Standard |

### Adatbázis

- **PostgreSQL 15** – [Neon](https://neon.tech) free tier SaaS-on futtatva
- Free tier: 0.5GB tárhely, **perzisztens** (auto-pause inaktivitás után, de adat megmarad)
- Pontosan standard PostgreSQL: azonos JDBC driver, JPA, Flyway – semmi vendor-specifikus
- Migrációs útvonal: connection string csere → RDS PostgreSQL (ha és amikor szükséges)

### Fejlesztői toolchain

| Tool | Mire | Megjegyzés |
|------|------|------------|
| IntelliJ IDEA Ultimate | IDE | Spring Boot, AWS Toolkit plugin |
| Claude Code CLI | AI-asszisztált fejlesztés | Terminálból, a projekt mappájából indítva |
| Claude Code plugin | IntelliJ integráció | Diff néző, fájl kontextus megosztás |
| SonarLint plugin | Real-time statikus analízis | SonarQube Cloud-hoz kötve |
| Docker Desktop | Lokális stack | PostgreSQL + backend + frontend |
| Git + GitHub | Verziókezelés | Mono-repo (kód), külön repo (docs) |
| SonarQube Cloud | CI statikus analízis | GitHub Actions-ből, public repo, ingyenes |
| GitHub Actions | CI/CD | OIDC alapú AWS hitelesítés |

---

## AWS Infrastruktúra (free tier kompatibilis)

```
Internet
    │
    ▼
CloudFront (CDN + HTTPS)              ← always free: 1TB transfer/hó
    │                     │
    ▼                     ▼
S3 Bucket            API Gateway      ← always free: 1M req/hó
(React SPA)          (HTTP API)         S3: ~$0.02/GB (fillér)
                          │
                          ▼
                    Lambda Function    ← always free: 1M req/hó, 400K GB-sec
                    (Spring Boot JAR     SnapStart: hidegindítás ~200-600ms
                     Java 21)
                          │
               ┌──────────┴──────────┐
               ▼                     ▼
         Neon PostgreSQL        SSM Parameter Store   ← ingyenes
         (külső SaaS,           (Neon conn string,
          ingyenes)              JWT secret,
                                 Google Books API key)
```

### AWS Szolgáltatások és költségek

| Szolgáltatás | Mire | Költség |
|-------------|------|---------|
| **Lambda** | Spring Boot backend | ✅ Always free (1M req/hó, 400K GB-sec) |
| **API Gateway HTTP API** | Lambda URL + routing | ✅ Always free (1M req/hó) |
| **S3** | React SPA statikus hosting | ~$0.02/GB – fillér |
| **CloudFront** | CDN, HTTPS | ✅ Always free (1TB transfer/hó) |
| **SSM Parameter Store** | Secretek tárolása | ✅ Standard tier ingyenes |
| **CloudWatch** | Logok, monitoring | ✅ Always free (alap szint) |

> 💡 **Becsült havi költség: ~$0** (S3 tárhelyen kívül, ami fillér nagyságrendű)

### Lambda SnapStart

A Spring Boot JVM hidegindítása Lambda-n SnapStart nélkül 3-8 másodperc lenne.
A SnapStart az inicializált JVM memória-snapshotját menti el, és abból indítja a következő kérést.
Eredmény: ~200-600ms hidegindítás Java 21-en is. Bekapcsolása egyetlen Lambda konfiguráció.

### Miért nem Secrets Manager?

A Secrets Manager $0.40/secret/hó – 3 secret esetén $1.2/hó.
Az SSM Parameter Store standard tier ingyenes, és a mi use case-ünkre (connection string,
JWT secret, API key tárolás) teljesen megfelelő. Lásd: ADR-004.

---

## CI/CD Pipeline

```
GitHub Push → main branch
        │
        ▼
GitHub Actions
  ├── Backend workflow:
  │     1. mvn test                          (unit tesztek)
  │     2. mvn package                       (shade plugin → fat JAR)
  │     3. SonarQube Cloud scan              (statikus analízis)
  │     4. JAR feltöltés S3-ra
  │     5. aws lambda update-function-code
  │     6. Lambda SnapStart: új verzió publish
  │
  └── Frontend workflow:
        1. npm ci
        2. npm run build                     (Vite → dist/)
        3. SonarQube Cloud scan              (ESLint alapú)
        4. aws s3 sync ./dist → S3
        5. CloudFront invalidation
```

> **AWS hitelesítés GitHub Actions-ből: OIDC** (nem tárolt access key!)
> A GitHub Actions ideiglenes tokent kap az IAM-tól – ez a biztonságos, modern megoldás.
> Nincs AWS_ACCESS_KEY_ID és AWS_SECRET_ACCESS_KEY a GitHub secretekben.

---

## Kód Repo Struktúra (Mono-repo)

```
homelibrary/                        ← GitHub repo: homelibrary
├── backend/                        ← Spring Boot projekt
│   ├── src/
│   │   ├── main/java/
│   │   └── test/java/
│   ├── Dockerfile                  ← csak lokális dev-hez
│   └── pom.xml
├── frontend/                       ← React projekt
│   ├── src/
│   ├── public/
│   ├── vite.config.ts
│   └── package.json
├── infra/                          ← AWS CDK (Java)
│   ├── src/
│   └── pom.xml
├── .github/
│   └── workflows/
│       ├── backend-deploy.yml
│       └── frontend-deploy.yml
├── docker-compose.yml              ← lokális fejlesztéshez
└── README.md
```

---

## Dokumentáció Repo Struktúra

```
homelibrary-docs/                   ← GitHub repo: homelibrary-docs
├── PROJECT.md
├── ARCHITECTURE.md
├── DB_SCHEMA.md
├── API_DESIGN.md
└── ADR/
    ├── 001-database-choice.md
    ├── 002-auth-strategy.md
    ├── 003-isbn-api.md
    └── 004-aws-compute-choice.md
```

> A dokumentáció szándékosan külön repóban van a kódtól.
> A fejlesztő menet közben nem szerkeszti – a tervek rögzítik az architektúrális döntéseket.
> Claude Code-ban a docs repo tartalmát kontextusként lehet átadni: `@docs/ARCHITECTURE.md`

---

## Lokális Fejlesztési Környezet

```yaml
# docker-compose.yml – a teljes stack lokálisan
services:
  postgres:     # lokális PostgreSQL (nem Neon) – gyors, offline fejlesztés
  backend:      # Spring Boot hagyományos Tomcat módban (nem Lambda adapter)
  frontend:     # Vite dev server (HMR – Hot Module Replacement)
```

**Miért Tomcat lokálisan?**
A Spring Boot Lambda módban fut production-ban, de lokálisan a hagyományos embedded Tomcat
gyorsabb fejlesztési ciklust ad. Lambda-specifikus viselkedés teszteléséhez: AWS SAM CLI
(`sam local start-api`).

**IntelliJ IDEA Ultimate-ből:**
- Spring Boot run config – direktben futtatható, Docker nélkül
- Database plugin – lokális PostgreSQL direktben böngészhető
- AWS Toolkit plugin – Lambda lokális debug, CloudWatch logok
- Claude Code plugin – diff néző, fájl kontextus megosztás az integrált terminálban

---

## Biztonsági Megfontolások

- JWT access token: 15 perces élettartam
- Refresh token: HttpOnly cookie (XSS ellen védett)
- CORS: csak a CloudFront domain engedélyezett
- SSM Parameter Store: SOHA nem kerül secret kódba vagy `.env` fájlba plain textként
- Neon PostgreSQL: SSL kapcsolat kötelező (alapértelmezett)
- GitHub Actions → AWS: OIDC, nem tárolt access key
- S3 (SPA): public read, de csak a `dist/` tartalmára
- GitHub repo: **public** (SonarQube Cloud free tier feltétele)
  - Secretek soha nem kerülnek a kódba (SSM-ben vannak)
  - `.gitignore` gondosan karbantartva

---

## ISBN Lookup Stratégia

1. **OpenLibrary API** (elsődleges) – ingyenes, API kulcs nélkül
   `GET https://openlibrary.org/api/books?bibkeys=ISBN:{isbn}&format=json&jscmd=data`
2. **Google Books API** (fallback) – ingyenes, API key kell (SSM-ben tárolva), 1000 req/nap
   `GET https://www.googleapis.com/books/v1/volumes?q=isbn:{isbn}`
3. **Manuális bevitel** – ha egyik API sem talál eredményt

---

## AI Fordítás (Fázis 3 – döntés nyitott)

- **Jelölt A:** Google Gemini API (ingyenes tier, bőkezű kvóta)
- **Jelölt B:** DeepL free tier (500k karakter/hó, fordításra specializált)
- Mindkettőt leteszteljük Fázis 3-ban, akkor döntünk
- A backend hívja (API key SSM-ben, nem a frontend)
- Cache-elés: lefordított szöveg DB-ben (`book_descriptions` tábla, `AI_TRANSLATED` source)
- Fallback: ha az API nem elérhető, az eredeti nyelven jelenik meg a leírás

---

## Nyitott Döntések

- [ ] AWS CDK stack nyelve: Java (egységes a backenddel) vagy TypeScript (CDK-ban elterjedtebb)?
- [ ] AI fordítás: Gemini vs DeepL (Fázis 3-ban, tesztelés után döntünk)
