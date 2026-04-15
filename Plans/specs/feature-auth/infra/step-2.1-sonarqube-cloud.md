# Step 2.1 – SonarQube Cloud integráció

## Mit állít elő

- `.github/workflows/backend-deploy.yml` kiegészítése SonarQube scan + Quality Gate job-bal
- `.github/workflows/frontend-deploy.yml` kiegészítése SonarQube scan + Quality Gate job-bal

---

## Előfeltételek (kézzel, az implementáció előtt)

**Secret** (Settings → Secrets and variables → Actions → **Secrets**) — csak az érzékeny adat:

| Secret neve | Honnan | Megjegyzés |
|---|---|---|
| `SONAR_TOKEN` | SonarQube Cloud projekt Settings → Security | Project Analysis Token (nem User Token) |

**Repository variable** (Settings → Secrets and variables → Actions → **Variables**) — nem érzékeny, a workflow logban látható és debugolható:

| Variable neve | Érték példa | Megjegyzés |
|---|---|---|
| `SONAR_ORGANIZATION` | `hunkpdev` | SonarQube Cloud szervezet neve |
| `SONAR_PROJECT_KEY_BACKEND` | `hunkpdev_homelibrary-backend` | Backend SonarQube Cloud projekt kulcsa |
| `SONAR_PROJECT_KEY_FRONTEND` | `hunkpdev_homelibrary-frontend` | Frontend SonarQube Cloud projekt kulcsa |

A backend és frontend **külön SonarQube Cloud projektet** alkotnak — így a QG-k egymástól függetlenek, és az analízisek nem írják felül egymást. A SonarQube Cloud-on mindkét projektet létre kell hozni manuálisan az implementáció előtt.

---

## Mindkét workflow trigger kiegészítése

A meglévő `push` trigger mellé `workflow_dispatch` trigger is kerül mindkét workflow-ban — ez lehetővé teszi a kézi indítást a GitHub Actions UI-ról (Actions tab → Run workflow), commit nélkül. Az AC validálásához is ez az ajánlott módszer.

---

## Backend workflow kiegészítés

A `backend-deploy.yml`-be egy új **`sonar` job** kerül be, **a meglévő deploy job elé**. A deploy job-on `needs: sonar` függőség kerül.

### `sonar` job lépései

1. **Checkout** — `fetch-depth: 0` (shallow clone kikapcsolva — SonarQube blame analízishez szükséges)
2. **Java 21 setup** (Temurin)
3. **Maven build + scan + Quality Gate wait** — egyetlen Maven parancsban:
   `mvn clean verify sonar:sonar -Dsonar.qualitygate.wait=true`
   - A `verify` fázis lefuttatja a unit teszteket; Surefire XML és JaCoCo report a Sonar plugin számára elérhetők lesznek
   - A `sonar.qualitygate.wait=true` property hatására a Maven goal addig blokkolja a futást, amíg a SonarQube Cloud vissza nem jelzi a Quality Gate eredményét — ha failed, a step elbukik, és a deploy job nem indul el
   - A szükséges Sonar property-k (`sonar.projectKey`, `sonar.organization`, `sonar.host.url`) Maven property-ként adandók át (`-D` flag-ek)

### Env változók a `sonar` job-ban

| Változó | Érték | Miért |
|---|---|---|
| `GITHUB_TOKEN` | `${{ secrets.GITHUB_TOKEN }}` | A Maven Sonar plugin explicit env var-t vár a GitHub API hívásokhoz (commit status visszajelzés) |
| `SONAR_TOKEN` | `${{ secrets.SONAR_TOKEN }}` | SonarQube Cloud hitelesítés |

A `sonar.projectKey` és `sonar.organization` Maven property-k értéke: `${{ vars.SONAR_PROJECT_KEY_BACKEND }}` és `${{ vars.SONAR_ORGANIZATION }}`.

---

## Frontend workflow kiegészítés

A `frontend-deploy.yml`-be szintén egy új **`sonar` job** kerül be, a deploy job elé, `needs: sonar` függőséggel.

### `sonar` job lépései

1. **Checkout** — `fetch-depth: 0`
2. **Node.js setup**
3. **`npm ci`**
4. **SonarQube scan** — `sonarqube-scan-action` (SonarSource official action), `args`-ban a szükséges property-k (`sonar.projectKey`, `sonar.organization`)
5. **Quality Gate check** — `sonarqube-quality-gate-check` (SonarSource official action) — ha a QG failed, ez a step elbukik, és a deploy job nem indul el

### Env változók a `sonar` job-ban

| Változó | Érték | Miért |
|---|---|---|
| `SONAR_TOKEN` | `${{ secrets.SONAR_TOKEN }}` | SonarQube Cloud hitelesítés |

A `sonar.projectKey` és `sonar.organization` action args értéke: `${{ vars.SONAR_PROJECT_KEY_FRONTEND }}` és `${{ vars.SONAR_ORGANIZATION }}`.

> **Megjegyzés:** `GITHUB_TOKEN`-t itt nem kell explicit deklarálni — a `sonarqube-scan-action` automatikusan felveszi a GitHub Actions környezetéből. Ez aszimmetria a backend Maven plugin megközelítéssel szemben, ahol explicit átadás szükséges.

---

## Coverage kizárások (`pom.xml`)

A default Quality Gate **minimum 80% coverage** elvárással fut. A coverage scope kizárólag a `service/`, `controller/` és `util/` package-ekre vonatkozik — minden más ki van zárva. A kizárások a `pom.xml` `<properties>` blokkjában, `sonar.coverage.exclusions` property-vel kerülnek be:

| Kizárt pattern | Miért |
|---|---|
| `**/entity/**` | JPA entitások — nem üzleti logika |
| `**/dto/**` | Adathordozó POJO-k |
| `**/model/**` | Cross-domain enum-ok |
| `**/config/**` | Spring konfigurációs osztályok |
| `**/repository/**` | Spring Data interfészek — nincs implementáció |
| `**/exception/**` | Egyedi kivételek — POJO-k, nem üzleti logika |

---

## Coverage kizárások (frontend — `sonar-project.properties`)

A frontend coverage kizárások a `frontend/sonar-project.properties` fájlban kerülnek be, `sonar.coverage.exclusions` property-vel:

| Kizárt pattern | Miért |
|---|---|
| `src/components/ui/**` | shadcn/ui generált komponensek — nem saját kód |
| `src/lib/**` | shadcn/ui utility (`cn()`) — nem saját kód |
| `src/api/model/**` | API request/response típusdefiníciók — pure TypeScript interfészek, nincs logika |
| `src/main.tsx` | App bootstrap entry point — nincs üzleti logika |
| `src/i18n/**` | i18n konfiguráció |

---

## Elfogadási kritériumok

- `main`-re pusholt backend változás után:
  - `sonar` job lefut, SonarQube Cloud dashboardon megjelenik az analízis
  - Ha a Quality Gate passed → a deploy job elindul
  - Ha a Quality Gate failed → a deploy job nem indul el, a workflow piros
- `main`-re pusholt frontend változás után — ugyanígy
- `SONAR_TOKEN` secret és `SONAR_ORGANIZATION`, `SONAR_PROJECT_KEY_BACKEND`, `SONAR_PROJECT_KEY_FRONTEND` repository variable-k beállítva (manuális előfeltétel, a workflow-ból nem ellenőrizhető)
- A backend és frontend analízis a SonarQube Cloud-on két külön projektként jelenik meg
- A meglévő deploy job-ok logikája nem változik (csak a `needs: sonar` függőség kerül rájuk)
