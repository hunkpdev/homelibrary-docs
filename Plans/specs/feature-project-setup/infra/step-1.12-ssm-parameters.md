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
- Típus: `SecureString` (titkosított)
- A Lambda execution role-nak `ssm:GetParameter` jogosultság szükséges ezekre a kulcsokra (step 1.13)

---

## Elfogadási kritériumok

- `cdk synth` hiba nélkül lefut
- Deploy után a Lambda environment variables között megjelennek a fenti értékek
