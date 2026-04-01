# ADR-004: AWS Compute és Infrastruktúra Stratégia

**Dátum:** 2026-03-28
**Státusz:** Elfogadva

## Kontextus

Az alkalmazást AWS-en kell hosztolni, kizárólag ingyenes megoldásokkal.
Az AWS 12 hónapos free tier (EC2, RDS, ECS Fargate) 2026. január 5-én lejárt.
A fejlesztő célja AWS tapasztalat szerzése álláskereséshez.

## Döntés

**Lambda (Spring Boot JAR + SnapStart) + API Gateway + S3 + CloudFront + SSM Parameter Store**

## Elvetett Alternatívák

### EC2 + RDS
- ❌ Free tier lejárt
- ❌ Folyamatos költség (~$15-30/hó t3.micro esetén)

### ECS Fargate
- ❌ Free tier lejárt
- ❌ Folyamatos költség (~$15-30/hó)
- ✅ Az OpenShift tapasztalat jól jött volna, de nem free

### Lambda + DynamoDB (100% AWS)
- ✅ Teljesen free
- ❌ NoSQL – az adatmodell relációs (lásd ADR-001)
- ❌ JPA/Hibernate nem használható

### Render / Fly.io / Railway (külső platform)
- ✅ Ingyenes tier van
- ❌ Nem AWS – nincs AWS tapasztalat szerzés

## Választott Megoldás Részletei

### Lambda + SnapStart
A Spring Boot JAR Lambda-n fut `aws-serverless-java-container` adapter segítségével.
**Hidegindítási probléma megoldása:** Lambda SnapStart – az inicializált JVM
memória-snapshotját menti el, ~200-600ms hidegindítást eredményez Java 21-en.

### API Gateway (HTTP API típus)
HTTP API típus választva (REST API helyett) – olcsóbb és elegendő a use case-hez.

### S3 + CloudFront
A React SPA statikus fájlokként S3-on, CloudFront CDN-en keresztül kiszolgálva.
Konténer **nem szükséges** – a React build eredménye statikus fájlok halmaza.

### SSM Parameter Store (Secrets Manager helyett)
A Secrets Manager $0.40/secret/hó lenne. Az SSM Parameter Store standard tier ingyenes
és teljesen megfelelő a mi use case-ünkre.

Tárolt paraméterek:
- `/homelibrary/neon-connection-string`
- `/homelibrary/jwt-secret`
- `/homelibrary/google-books-api-key`

### GitHub Actions OIDC (access key helyett)
A GitHub Actions OIDC tokent kap az AWS IAM-tól – nem kell AWS_ACCESS_KEY_ID és
AWS_SECRET_ACCESS_KEY secreteket tárolni a GitHub-on. Ez a biztonságos, modern megoldás.

## AWS Tudás amit ez a Stack Ad

Ez az architektúra a következő AWS szolgáltatások megismerését teszi lehetővé:
- **Lambda** – serverless compute, SnapStart, verziók, alias-ok
- **API Gateway** – HTTP API, routing, CORS, throttling
- **S3** – bucket policy, statikus website hosting
- **CloudFront** – distribution, origin, cache behavior, invalidation
- **SSM Parameter Store** – SecureString, IAM jogosultságok
- **IAM** – role-ok, policy-k, OIDC identity provider
- **CloudWatch** – log groups, metric filters, alarmok
- **CDK** – infrastruktúra kódként (Java vagy TypeScript)

## Becsült Havi Költség

| Szolgáltatás | Limit | Várható forgalom | Költség |
|-------------|-------|-----------------|---------|
| Lambda | 1M req, 400K GB-sec | ~15.000 req/hó | $0 |
| API Gateway | 1M req/hó | ~15.000 req/hó | $0 |
| S3 | - | ~15MB tárolt | ~$0.001 |
| CloudFront | 1TB transfer/hó | Elhanyagolható | $0 |
| SSM | Ingyenes | 3 paraméter | $0 |
| **Összesen** | | | **~$0** |

## Következmények

- A Spring Boot kódba bekerül az `aws-serverless-java-container` adapter
- Lokálisan Docker Compose-zal fut (hagyományos Tomcat mód) – gyorsabb fejlesztési ciklus
- Lambda viselkedés tesztelése: AWS SAM CLI (`sam local start-api`)
- Deployment: JAR alapú (nem Docker image) – eltér az OpenShift tapasztalattól, de egyszerűbb
- A GitHub repo **public** kell legyen (SonarQube Cloud free tier feltétele)
