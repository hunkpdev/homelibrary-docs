# ADR-010: Z39.50 Java Client Library és Natív Bináris Kezelés

**Dátum:** 2026-04-29  
**Státusz:** Elfogadva

## Kontextus

Az ADR-003 döntése alapján az OSZK NEKTÁR Z39.50 protokollon érhető el. Java oldalon Z39.50 client library szükséges, amely JNI-n keresztül natív binárist igényel. A döntést két megkötés befolyásolja:

- A meglévő deploy flow (maven-shade-plugin → fat jar → S3 → Lambda) változatlan kell maradjon — Lambda layer bevezetése nem cél.
- A projekt licence PolyForm Personal Use 1.0.0 (ADR-011) — az alkalmazás licencére hatást gyakorló library nem fogadható el.

## Döntés

**YAZ4J (BSD-3-Clause) + `libyaz.so.5` natív bináris fat jar-ba csomagolva, kézi `System.load()`-dal betöltve.**

## Miért YAZ4J és nem JZKit2?

JZKit2 (AGPL v3) megköveteli, hogy a teljes alkalmazás AGPL-lé váljon — inkompatibilis a PolyForm Personal Use 1.0.0 licenccel. YAZ4J BSD-3-Clause licencelésű: semmilyen extra megkötés nincs a saját kódra; egyetlen kötelezettség a YAZ copyright szöveg mellékelése (`THIRD_PARTY_LICENSES.md`). A YAZ upstream aktívan karbantartott (Index Data, 2026-os commitek).

## Natív bináris csomagolás

A `libyaz.so.5` (Linux) és `yaz5.dll` (Windows) bekerül a fat jar-ba:

```
src/main/resources/native/linux-x86_64/libyaz.so.5
src/main/resources/native/win32-x86_64/yaz5.dll
```

Runtime betöltés: az `OszkNektarClient.loadNativeLibrary()` `getResourceAsStream` + `Files.createTempFile` + `System.load` mintával extracteli a `libyaz.so.5`-öt `/tmp`-be. A `libyaz4j.so` JNI bridge-et a yaz4j JAR `org.yaz4j.LoadLib` osztálya kezeli.

## Miért nem Lambda layer?

A meglévő shade+S3 deploy flow változatlan kell maradjon. Lambda layer tanulása és bevezetése jelenleg nem cél.

## Linux natív bináris megszerzése (egyszeri, becommitelendő)

A `libyaz.so.5` és `libyaz4j.so` fájlokat az `.github/workflows/extract-native-libs.yml` workflow állítja elő Ubuntu-alapú Docker containerből (yaz4j forrásból fordítva):

```bash
docker run --rm -v $(pwd)/output:/output ubuntu:20.04 bash -c "
  apt-get update -qq &&
  apt-get install -y build-essential libyaz-dev default-jdk-headless maven swig git pkg-config &&
  git clone https://github.com/indexdata/yaz4j.git &&
  cd yaz4j && mvn package -DskipTests -q &&
  cp /usr/lib/x86_64-linux-gnu/libyaz.so.5 /output/linux-x86_64/libyaz.so.5
"
```

> **Megjegyzés:** Ubuntu 20.04 support 2025 áprilisában megszűnt (EOL). Ubuntu 24.04-re frissítés javasolt.

## Tranzitív JNI függőségek

`ldd libyaz.so.5` tranzitív `.so` függőségei (`libxml2.so.2`, `libgnutls.so.30`, `libz.so.1` stb.) az AWS Lambda AL2023 alap image-ben általában megtalálhatók. Ha hiányzó `.so` van, CloudWatch logban `dlopen failed: libxxx.so.N not found` jelenik meg — akkor azt is be kell csomagolni.

## Következmények

- Maven függőségek: `yaz4j`, `marc4j`
- `src/main/resources/native/` mappa becommitelve (~5–10 MB)
- YAZ BSD-3-Clause copyright szöveg: `THIRD_PARTY_LICENSES.md`-be kerül
- Lokális fejlesztés (Windows): YAZ Windows installer vagy `yaz5.dll` PATH-ba téve
- Lambda cold start: első híváskor native lib extract `/tmp`-be (néhány ms overhead)
