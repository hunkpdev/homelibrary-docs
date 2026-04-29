# Step 4.3 – OszkNektarClient

## Mit állít elő

- `OszkNektarClient` Spring `@Component`
- `IsbnLookupResult` Java record
- `IsbnSource` enum (`OSZK`, `MANUAL`)
- `src/main/resources/native/linux-x86_64/libyaz.so.5` (becommitelendő)
- `src/main/resources/native/win32-x86_64/yaz5.dll` (lokális fejlesztéshez)

---

## Maven függőségek

- `yaz4j` — Z39.50 Java client (BSD-3-Clause)
- `marc4j` — MARC21 parser (Apache 2.0)
- `org.scijava:native-lib-loader` — natív bináris automatikus extract és betöltés

## Natív bináris megszerzése (egyszeri, GitHub Actions workflow)

A `.so` fájlt nem lokálisan kell előállítani — egy egyszeri `workflow_dispatch` workflow végzi el:

1. Felhúzza a `public.ecr.aws/lambda/java:21` image-et (GitHub Actions Linux runner)
2. `dnf install -y yaz` → kinyeri a `libyaz.so.5` fájlt
3. Commitolja `src/main/resources/native/linux-x86_64/libyaz.so.5` útvonalra

A workflow fájl helye: `.github/workflows/extract-native-libs.yml`, trigger: `workflow_dispatch` (manuális, egyszer futtatandó). Utána a fájl a repóban van, CI-ban mindig elérhető.

Ha tranzitív `.so` függőség hiányzik Lambda-n: CloudWatch logban `dlopen failed: libxxx.so.N not found` → a workflow bővítendő az adott `.so` fájl kinyerésével is.

## Z39.50 kapcsolat

| Paraméter | Érték |
|---|---|
| Host | `tagetes2.oszk.hu` |
| Port | `1616` |
| Adatbázis | `B1` |
| ISBN keresés attribútum | `@attr 1=7 {ISBN}` (Bib-1) |
| Rekord szintaxis | MARC21 (USmarc) |

Kapcsolat init költséges (~3-4 sec) — a `Connection` objektum bean szinten cachelendő (ne kérésenként nyissa meg).

> Az OSZK hivatalos elérhetőségi ablaka 03:30–23:00, de empirikus teszt alapján azon kívül is válaszol. Időablak-ellenőrzés nincs implementálva — a meglévő hibakezelés (connection timeout → `Optional.empty()`) fedezi a kiesési eseteket.

## MARC21 → IsbnLookupResult mapping

| `IsbnLookupResult` mező | MARC tag |
|---|---|
| `isbn` | 020 $a (első érték, kötőjelek nélkül) |
| `title` | 245 $a (záró ` /` levágva) |
| `subtitle` | 245 $b |
| `authors` | 100 $a + $j összefűzve; 700 $a + $j hozzáadva (társszerzők) |
| `publisher` | 260 $b (záró `,` levágva) |
| `publishYear` | 260 $c (numerikus rész kinyerve) |
| `pageCount` | 300 $a (numerikus rész kinyerve) |
| `language` | 041 $a (ISO 639-2, 3 betűs) |
| `source` | `IsbnSource.OSZK` |

## `hasMinimumFields()` definíció

`title` és legalább egy `author` kötelező. Ha az OSZK csonka rekordot ad (hiányzik title vagy author): `Optional.empty()` → service rétegben `found: false`.

## Hibakezelés

| Helyzet | Kezelés |
|---|---|
| 0 találat | `Optional.empty()` |
| `show` hívás hit count ellenőrzés nélkül | hit count > 0 ellenőrzés `present` előtt |
| Connection timeout | `log.warn`, `Optional.empty()` |
| MARC parsing hiba | `log.warn`, `Optional.empty()` |
| OSZK nem elérhető (bármikor) | `log.warn`, `Optional.empty()` |

## Elfogadási kritériumok

- Érvényes ISBN-re MARC21 rekord visszaadása és helyes leképezés
- Csonka rekordnál (title vagy author hiányzik) `Optional.empty()`
- 0 találat → `Optional.empty()`
- Unit teszt: MARC21 leképezés, időablak check, csonka rekord kezelés
