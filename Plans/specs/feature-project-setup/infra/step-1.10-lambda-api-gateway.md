# Step 1.10 – Lambda + API Gateway HTTP API

## Mit állít elő

- `HomelibraryStack` kiegészítve: Lambda function + API Gateway HTTP API

---

## Lambda function

- Runtime: Java 21
- Handler: `com.homelibrary.StreamLambdaHandler`
- Memory: 512 MB
- Timeout: 30 s
- SnapStart: `PublishedVersions` — Java cold start csökkentésére (3–8 s → ~1 s)
- JAR forrása: `../backend/target/homelibrary-*.jar` (fat JAR, step 1.1)
- Environment variables:
  - `SPRING_PROFILES_ACTIVE=prod`
  - `SPRING_DATASOURCE_URL` — SSM-ből (step 1.12)
  - `JWT_SECRET` — SSM-ből (step 1.12)
  - `ADMIN_PASSWORD_HASH` — SSM-ből (step 1.12)

---

## API Gateway HTTP API

- Típus: HTTP API (nem REST API — olcsóbb, elegendő)
- Integráció: Lambda proxy
- CORS: explicit CloudFront origin (step 1.11 outputja), `withCredentials: true` miatt `*` nem megengedett
  - `allowOrigins`: CloudFront distribution URL
  - `allowMethods`: GET, POST, PUT, DELETE, OPTIONS
  - `allowHeaders`: Content-Type, Authorization
  - `allowCredentials`: true

---

## Elfogadási kritériumok

- `cdk synth` hiba nélkül lefut, a template tartalmazza a Lambda és HTTP API erőforrásokat
- Deploy után `GET <api-gateway-url>/api/health` → `200 OK`
