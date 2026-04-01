# ADR-002: Autentikáció és Jogosultságkezelés Stratégia

**Dátum:** 2026-03-28
**Státusz:** Elfogadva

## Kontextus

Az alkalmazásba be kell jelentkezni. Két jogosultsági szint szükséges: `ADMIN` és `VISITOR`.
A felhasználószám alacsony (tipikusan 2-5 fő egy háztartásban + meghívott látogatók).

## Döntés

**Saját "mini IDM": Spring Security + JWT, felhasználók a saját PostgreSQL adatbázisban.**

## Alternatívák

### A: AWS Cognito
- ✅ Managed service, nem kell implementálni
- ✅ Social login, MFA, email megerősítés készen van
- ❌ AWS free tier lejárt – Cognito User Pools fizetős lehet magasabb forgalomnál
- ❌ Túlméretezett egy 2-5 fős házi alkalmazáshoz
- ❌ Összetett konfiguráció, Spring Security Cognito integráció bonyolult

### B: Keycloak (self-hosted)
- ✅ Teljes IDM megoldás (a fejlesztő munkahelyen már ismeri)
- ❌ Külön szerver kell hozzá (EC2 – fizetős, free tier lejárt)
- ❌ Súlyos overhead egy kis alkalmazáshoz

### C: Spring Security + JWT (választott)
- ✅ Nincs extra infrastruktúra szükséges
- ✅ Ingyenes, a Spring Boot alkalmazásba integrált
- ✅ Jó tanulási lehetőség (megérteni, amit a Keycloak "elrejt")
- ✅ Elegendő a use case-hez
- ⚠️ Elfelejtett jelszó, MFA – saját implementáció kell (Fázis 2+)

## Megvalósítás Részletei

### Token Stratégia
- **Access token:** JWT, 15 perces élettartam, `Authorization: Bearer` headerben
- **Refresh token:** Opaque token, 7 napos élettartam, HttpOnly cookie-ban (XSS védett)
- **Tárolás:** Refresh tokenek hash-elve a DB-ben (users táblában vagy külön táblában)

### Jogosultságok
```
ADMIN  → minden endpoint elérhető
VISITOR → GET /api/books, GET /api/locations, GET /api/books/{id}
```

Spring Security `@PreAuthorize("hasRole('ADMIN')")` annotációkkal kezelve.

### Jelszókezelés
- BCrypt hash (Spring Security alapértelmezett)
- Minimális jelszó erősség validáció

## Amit Ez NEM Tartalmaz (Szándékosan)

- Social login (Google, Facebook) – nem tervezett
- MFA – nem tervezett
- Email megerősítés – Fázis 2
- Jelszó visszaállítás – Fázis 2
- LDAP/AD integráció – nem tervezett

## Következmények

- `users` tábla a PostgreSQL adatbázisban (lásd DB_SCHEMA.md)
- JWT secret SSM Parameter Store-ban tárolva
- A fejlesztő megérti a JWT mechanizmust, ami Keycloak-nál "black box" volt
