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
3. `mvn clean package -DskipTests`
4. AWS hitelesítés OIDC-vel (step 1.13 szerepköre)
5. `aws lambda update-function-code` — JAR feltöltése
6. `aws lambda publish-version` — SnapStart csak published verziókon aktív

---

## frontend-deploy.yml lépései

1. Checkout
2. Node.js setup
3. `npm ci && npm run build`
4. AWS hitelesítés OIDC-vel (step 1.13 szerepköre)
5. `aws s3 sync dist/ s3://<bucket> --delete`
6. `aws cloudfront create-invalidation --paths "/*"`

---

## GitHub Actions Secrets / Variables

| Név | Típus | Tartalom |
|-----|-------|----------|
| `AWS_ROLE_ARN` | Secret | Step 1.13-ban létrehozott IAM szerepkör ARN-je |
| `AWS_REGION` | Variable | `eu-central-1` |
| `CLOUDFRONT_DISTRIBUTION_ID` | Variable | Step 1.11 outputja |
| `S3_BUCKET_NAME` | Variable | Step 1.11 outputja |

---

## Elfogadási kritériumok

- `main`-re pusholt backend változás után a Lambda automatikusan frissül
- `main`-re pusholt frontend változás után az S3 tartalma frissül és a CloudFront cache invalidálódik
- Egyik workflow sem fut le, ha a másik könyvtárban történt a változás
