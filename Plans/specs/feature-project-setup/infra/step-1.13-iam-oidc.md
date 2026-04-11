# Step 1.13 – IAM + OIDC

## Mit állít elő

- `HomelibraryStack` kiegészítve: GitHub Actions OIDC provider + IAM szerepkör

---

## GitHub Actions OIDC

Jelszó/access key nélküli autentikáció GitHub Actions és AWS között — a GitHub Actions JWT tokennel hitelesíti magát az AWS-nél.

- OIDC provider: `token.actions.githubusercontent.com`
- Trust policy: csak a `hunkpdev/homelibrary` repó `main` branch-éről induló workflow-k vehetik fel a szerepkört

---

## IAM szerepkör jogosultságai (minimális)

| Művelet | Erőforrás | Mire |
|---------|-----------|------|
| `lambda:UpdateFunctionCode` | `HomelibraryFunction` | Backend JAR deploy |
| `lambda:PublishVersion` | `HomelibraryFunction` | SnapStart — új verzió publikálása |
| `lambda:UpdateAlias` | `HomelibraryFunction` | SnapStart — alias frissítése az új verzióra |
| `s3:PutObject`, `s3:DeleteObject`, `s3:ListBucket` | Frontend S3 bucket | React dist feltöltése |
| `cloudfront:CreateInvalidation` | CloudFront distribution | Cache invalidálás deploy után |
| `ssm:GetParameter` | `/homelibrary/*` | Lambda SSM olvasás futásidőben |

---

## Elfogadási kritériumok

- `cdk synth` hiba nélkül lefut
- Deploy után a GitHub Actions workflow felveheti a szerepkört és végrehajtja a deploy lépéseket (step 1.14-ben verifikálható)
