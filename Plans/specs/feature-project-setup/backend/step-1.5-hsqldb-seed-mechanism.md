# Step 1.5 – HSQLDB seed mechanizmus

## Mit állít elő

- `src/main/java/com/homelibrary/config/LocalDataSeeder.java` — `ApplicationRunner` implementáció
- `local-data/seed.sql` — opcionális seed fájl (gitignorált)
- `.gitignore` kiegészítés: `local-data/` bejegyzés

---

## LocalDataSeeder

- `@Profile("local")` — kizárólag `local` Spring profilon aktív
- `ApplicationRunner` implementáció: alkalmazás induláskor fut le, a Liquibase után
- Ha `local-data/seed.sql` létezik, betölti és végrehajtja HSQLDB-n
- Ha a fájl nem létezik, csendesen átugorja (nem dob hibát)

---

## local-data/seed.sql

Fejlesztő által kézzel karbantartott SQL fájl — teszt könyvek, helyszínek, kölcsönzések feltöltéséhez. Nem kerül a repóba (`.gitignore`).

A projektben egy `local-data/seed.sql.example` fájl dokumentálja az elvárt formátumot — ez commitolva van, placeholder adatokkal.

---

## Elfogadási kritériumok

- `local` profilon, ha `local-data/seed.sql` nem létezik: az alkalmazás figyelmeztetés nélkül elindul
- `local` profilon, ha `local-data/seed.sql` létezik: az SQL lefut, az adatok lekérdezhetők
- `prod` profilon a `LocalDataSeeder` bean nem példányosodik
