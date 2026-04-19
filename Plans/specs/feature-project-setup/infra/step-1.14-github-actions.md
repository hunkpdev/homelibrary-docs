# Step 1.14 – GitHub Actions workflow-ok

## Mit állít elő

- `.github/workflows/backend-deploy.yml` — backend deploy workflow
- `.github/workflows/frontend-deploy.yml` — frontend deploy workflow

---

## Triggerek

Mindkét workflow csak `main` branch-re való push esetén fut — és csak akkor, ha az érintett könyvtárban változás történt:

| Workflow | Trigger path |
|----------|-------------|
| backend-deploy | `backend/**` |
| frontend-deploy | `frontend/**` |

---

## backend-deploy.yml lépései

1. Checkout
2. Java 21 setup (Temurin)
3. `mvn clean package`
4. AWS hitelesítés OIDC-vel (step 1.13 szerepköre)
5. `aws lambda update-function-code` — JAR feltöltése
6. `aws lambda publish-version` — SnapStart csak published verziókon aktív

---

## frontend-deploy.yml lépései

1. Checkout
2. Node.js setup
3. `npm ci`
4. Build frontend — `npm run build`, env blokkal:
   ```yaml
   - name: Build frontend
     run: npm run build
     env:
       VITE_API_BASE_URL: ${{ vars.API_GATEWAY_URL }}
   ```
5. AWS hitelesítés OIDC-vel (step 1.13 szerepköre)
6. `aws s3 sync dist/ s3://<bucket> --delete`
7. `aws cloudfront create-invalidation --paths "/*"`

---

## GitHub Actions Secrets / Variables

| Név | Típus | Tartalom |
|-----|-------|----------|
| `AWS_ROLE_ARN` | Secret | Step 1.13-ban létrehozott IAM szerepkör ARN-je |
| `AWS_REGION` | Variable | `eu-central-1` |
| `CLOUDFRONT_DISTRIBUTION_ID` | Variable | Step 1.11 outputja |
| `S3_BUCKET_NAME` | Variable | Step 1.11 outputja |
| `API_GATEWAY_URL` | Variable | CDK deploy után egyszer, kézzel: repo Settings > Variables |

---

> **`VITE_API_BASE_URL` és `API_GATEWAY_URL`:** A `VITE_API_BASE_URL` Vite build-time változó — prod-on nincs Vite proxy, az axios-nak build időben tudnia kell az API Gateway URL-t. A backend és frontend deploy külön workflow, ezért a CDK stack outputja nem érhető el a frontend workflow-ban. Az API Gateway URL ritkán változik, ezért egyszer kézzel beállított repository variable (`vars.API_GATEWAY_URL`) a megfelelő megközelítés. A változót a CDK deploy lefutása után kell egyszer felvinni a repo Settings > Variables oldalon.

> **Megjegyzés:** SonarQube Cloud integráció és test reporting (scan + Quality Gate + human-readable riport mindkét workflow-ba) a Feature 2 step 2.1-ben kerül be — az első érdemi kód (Auth) előtt.

---

## Elfogadási kritériumok

- `main`-re pusholt backend változás után a Lambda automatikusan frissül
- `main`-re pusholt frontend változás után az S3 tartalma frissül és a CloudFront cache invalidálódik
- Egyik workflow sem fut le, ha a másik könyvtárban történt a változás

**Smoke test (az összes infra step és az első deploy után):**
- `GET <api-gateway-url>/api/health` → `200 OK`
- A CloudFront URL-en a React app betöltődik
- A Lambda environment variables között megjelennek az SSM-ből injektált értékek
