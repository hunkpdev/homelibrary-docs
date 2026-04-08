# Step 1.4 – Lambda handler bekötés + health check endpoint

## Mit állít elő

- `src/main/java/com/homelibrary/StreamLambdaHandler.java` — AWS Lambda entry point
- `src/main/java/com/homelibrary/controller/HealthController.java` — `GET /api/health` endpoint

---

## StreamLambdaHandler

Az `aws-serverless-java-container-springboot4` adapter `SpringBootProxyHandlerBuilder`-én alapul, `asyncInit()` móddal (Lambda cold start timeout elkerülése).

Lambda konfigurációban a handler értéke: `com.homelibrary.StreamLambdaHandler`

---

## GET /api/health

- Útvonal: `/api/health`
- Autentikáció: nem szükséges (publikus endpoint)
- Válasz: `200 OK`, body: `{"status": "UP"}`

> A `/api/health` publikusként való konfigurálása a Feature 2 (Auth) 2.4-es stepjében történik meg a security filter chain felállításakor. Addig `local` profilon elérhető, mert a security még nincs bekötve.

---

## Elfogadási kritériumok

- `local` profilon `GET http://localhost:8080/api/health` → `200 OK`, body: `{"status":"UP"}`
- A fat JAR tartalmazza a `StreamLambdaHandler` osztályt (`jar tf target/*.jar | grep StreamLambdaHandler`)
