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

## Cache-stratégia (2026-08, elavult bundle bug alapján felvéve)

A hash nélküli és a hash-elt fájlok cache-stratégiája szándékosan ellentétes. Az `index.html` az egyetlen belépési pont, aminek a neve nem árulja el a tartalmát — ezért ez az egyetlen fájl, aminek sosem szabad hálózat nélkül kiszolgálódnia (`no-cache`). Minden más fájl neve tartalmi hash-t tartalmaz, tehát a név maga a verzió: ugyanazon a néven a tartalom sosem változik, így az egy éves `immutable` TTL helyes, és gyorsabb is, mint a `Cache-Control` nélküli heurisztikus viselkedés.

A `Cache-Control` beállítása **feltöltéskor** történik (`aws s3 cp`/`sync --cache-control`, ld. step 1.14), nem CloudFront `ResponseHeadersPolicy`-val — ez egyszerűbb és a CDK stackhez nem kell hozzányúlni. Ha később biztonsági header-ek (CSP, HSTS) is kellenek, érdemes újragondolni, hol a header-ek helye.

---

## Elfogadási kritériumok

- `cdk synth` hiba nélkül lefut, a template tartalmazza az S3 és CloudFront erőforrásokat
