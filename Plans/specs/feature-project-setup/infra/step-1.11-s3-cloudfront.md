# Step 1.11 – S3 + CloudFront

## Mit állít elő

- `HomelibraryStack` kiegészítve: S3 bucket + CloudFront distribution a React SPA hostinghoz

---

## S3 bucket

- Publikus hozzáférés tiltva — csak CloudFront fér hozzá (Origin Access Control)
- Statikus fájlok tárolása: `frontend/dist/` tartalma (step 1.14 deployer)

---

## CloudFront distribution

- Origin: S3 bucket (OAC-on keresztül)
- Default root object: `index.html`
- SPA routing: minden `4xx` hiba → `index.html` (React Router kezeli a kliensen)
- HTTPS: CloudFront managed certificate
- Cache: default CloudFront cache policy

---

## Stack output

A CloudFront distribution URL-je stack outputként publikálva — a step 1.10 CORS konfigurációja ezt használja.

---

## Elfogadási kritériumok

- `cdk synth` hiba nélkül lefut, a template tartalmazza az S3 és CloudFront erőforrásokat
- Deploy után a CloudFront URL-en a React app betöltődik
