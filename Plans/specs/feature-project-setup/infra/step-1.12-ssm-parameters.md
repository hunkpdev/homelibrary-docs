# Step 1.12 – SSM Parameter Store

## Mit állít elő

- `HomelibraryStack` kiegészítve: SSM paraméterek olvasása és Lambda env var-ként injektálása

---

## Paraméterek

| SSM kulcs | Lambda env var | Leírás |
|-----------|---------------|--------|
| `/homelibrary/neon-connection-string` | `SPRING_DATASOURCE_URL` | Neon PostgreSQL JDBC connection string (host + db + sslmode, user/password nélkül) |
| `/homelibrary/neon-username` | `SPRING_DATASOURCE_USERNAME` | Neon adatbázis felhasználónév |
| `/homelibrary/neon-password` | `SPRING_DATASOURCE_PASSWORD` | Neon adatbázis jelszó |
| `/homelibrary/jwt-secret` | `JWT_SECRET` | JWT aláírási kulcs |
| `/homelibrary/admin-password-hash` | `ADMIN_PASSWORD_HASH` | Admin user BCrypt hash (Liquibase seed) |

---

## Megjegyzések

- A paramétereket **manuálisan kell feltölteni** az AWS konzolban vagy AWS CLI-vel a CDK deploy előtt — a CDK stack csak olvassa őket, nem hozza létre
- Típus: **`String`** (nem `SecureString`) — lásd magyarázat lent
- A Lambda execution role-nak `ssm:GetParameter` jogosultság szükséges ezekre a kulcsokra (step 1.13)

### Miért nem SecureString?

A CDK `valueForStringParameter()` hívás CloudFormation `AWS::SSM::Parameter::Value<String>` típust generál, ami **csak `String` típusú SSM paraméterekkel működik**. A CloudFormation nem tudja `SecureString` értékeket Lambda env var-okba injektálni.

A valóban korrekt megoldás az AWS Secrets Manager lenne, de annak havi díja (~$0.40/secret/hó) miatt ezt a projektnél kizártuk.

A Lambda env var-ok at rest titkosítottak az AWS-ben — ez nem kereskedelmi projektnél elfogadható kompromisszum.

---

## Elfogadási kritériumok

- `cdk synth` hiba nélkül lefut
