# Step 1.15 – Lambda deployment S3 bucket

## Mit állít elő

- `HomelibraryStack` kiegészítve: S3 bucket deployment artifaktoknak
- Lambda execution role és GitHub Actions OIDC role kiegészítve S3 jogosultságokkal
- Új SSM paraméter: deployment bucket neve
- `backend-deploy.yml` frissítve: S3-on keresztüli JAR feltöltés

---

## Háttér

A backend fat JAR mérete meghaladja a Lambda direkt upload 70 MB-os korlátját (`--zip-file`). Az AWS S3-on keresztüli kódfrissítés (`--s3-bucket` + `--s3-key`) nem rendelkezik méretkorláttal — ehhez dedikált S3 bucket szükséges.

---

## CDK — deployment S3 bucket

Új `BlockPublicAccess.BLOCK_ALL` S3 bucket a `HomelibraryStack`-ben:

- Név: `homelibrary-deployments-<account>-<region>` (CDK auto-generated, ütközésmentes)
- Versioning: kikapcsolva — nem szükséges, Lambda mindig a legfrissebb JAR-t tölti be
- Lifecycle policy: objektumok törlése **7 nap** után — régi JAR-ok automatikus cleanup-ja
- Encryption: S3-managed (SSE-S3, default) — elfogadható, nem érzékeny adat

---

## IAM jogosultság kiegészítések

### Lambda execution role (step 1.10)

| Művelet | Erőforrás | Mire |
|---------|-----------|------|
| `s3:GetObject` | deployment bucket | Lambda kódfrissítés során a JAR letöltése az S3-ról |

### GitHub Actions OIDC role (step 1.13)

| Művelet | Erőforrás | Mire |
|---------|-----------|------|
| `s3:PutObject` | deployment bucket | JAR feltöltése CI/CD workflow-ból |

---

## SSM paraméter

| SSM kulcs | GitHub Variable | Leírás |
|-----------|-----------------|--------|
| `/homelibrary/deployment-bucket-name` | `DEPLOYMENT_BUCKET_NAME` | Deployment S3 bucket neve — CDK deploy után egyszer, kézzel felvinni a repo Settings > Variables oldalon |

A CDK stack a bucket nevét SSM-be írja (`StringParameter.fromStringParameterName` helyett `new StringParameter` — ez a stack hozza létre, nem olvassa).

---

## backend-deploy.yml frissítése

A meglévő `aws lambda update-function-code --zip-file` lépés helyett két lépés:

**1. JAR feltöltése S3-ra:**
```
aws s3 cp target/homelibrary-*.jar \
  s3://${{ vars.DEPLOYMENT_BUCKET_NAME }}/backend/homelibrary-${{ github.sha }}.jar
```

**2. Lambda kódfrissítés S3-ról:**
```
aws lambda update-function-code \
  --function-name HomelibraryFunction \
  --s3-bucket ${{ vars.DEPLOYMENT_BUCKET_NAME }} \
  --s3-key backend/homelibrary-${{ github.sha }}.jar
```

A `${{ github.sha }}` alapú S3 key biztosítja, hogy minden deploy egyedi objektumot hoz létre — a lifecycle policy 7 nap után törli a régieket.

---

## Elfogadási kritériumok

- `cdk synth` hiba nélkül lefut, a template tartalmazza az S3 bucketet és a frissített IAM policy-kat
- `DEPLOYMENT_BUCKET_NAME` repository variable beállítva (manuális előfeltétel a workflow futtatása előtt)
- `main`-re pusholt backend változás után a JAR megjelenik az S3 bucketben és a Lambda frissül
- `GET <api-gateway-url>/api/health` → `200 OK` az új deploy után
