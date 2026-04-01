# ADR-001: Adatbázis Technológia Választás

**Dátum:** 2026-03-28
**Státusz:** Elfogadva

## Kontextus

Az alkalmazáshoz perzisztens adattárolás szükséges. Az AWS free tier 12 hónapos
időszaka (2025. január – 2026. január) lejárt, tehát az RDS nem elérhető ingyenesen.
Az AWS "always free" adatbázis opciók: csak DynamoDB (NoSQL).

## Döntés

**PostgreSQL 15, Neon SaaS free tier-en futtatva.**

## Alternatívák

### A: AWS DynamoDB (always free)
- ✅ 100% AWS, always free (25GB)
- ✅ Nincs külső függőség
- ❌ NoSQL – a HomeLibrary adatmodellje természetesen relációs
- ❌ JPA/Hibernate nem használható, AWS SDK szükséges helyette
- ❌ JOIN-ok, foreign key-ek, tranzakciók nehézkesek
- ❌ Meglévő SQL/JPA tudás nem hasznosítható

### B: PostgreSQL Neon SaaS-on (választott)
- ✅ Valódi PostgreSQL – azonos driver, JPA, Flyway, SQL
- ✅ Ingyenes tier: 0.5GB, perzisztens
- ✅ Meglévő tudás teljes mértékben hasznosítható
- ✅ Migrációs útvonal: connection string csere → RDS (ha szükséges)
- ⚠️ Külső függőség (nem AWS) – de a fejlesztési célhoz elfogadható

### C: H2 in-memory (lokálisan)
- ❌ Nem perzisztens, production-ra alkalmatlan

## Miért Relációs és Nem NoSQL?

A HomeLibrary adatmodellje tisztán relációs struktúrát igényel:

```
books → locations     (N:1 – könyv egy helyen van)
books → loans         (1:N – egy könyvnek több kölcsönzése lehet)
books → descriptions  (1:N – több nyelvű leírás)
loans → users         (N:1 – kölcsönző felhasználó)
```

Ezek JOIN-okat, foreign key integritást és tranzakciókat igényelnek –
pontosan arra való a relációs adatbázis.

## Következmények

- A Spring Boot alkalmazásban standard JPA/Hibernate konfiguráció
- Flyway migrációk kezelik a sémaváltozásokat
- Neon connection string SSM Parameter Store-ban tárolva
- SSL kapcsolat kötelező (Neon alapértelmezés)
- Auto-pause inaktivitás után: az első kérés ~500ms késéssel érkezhet
