# ADR-010: Z39.50 Java Client Library és Natív Bináris Kezelés

**Dátum:** 2026-04-29  
**Státusz:** Elfogadva

## Kontextus

Az ADR-003 döntése alapján az OSZK NEKTÁR Z39.50 protokollon érhető el. Java oldalon Z39.50 client library szükséges, amely JNI-n keresztül natív binárist igényel. A döntést két megkötés befolyásolja:

- A meglévő deploy flow (maven-shade-plugin → fat jar → S3 → Lambda) változatlan kell maradjon — Lambda layer bevezetése nem cél.
- A projekt licence PolyForm Personal Use 1.0.0 (ADR-011) — az alkalmazás licencére hatást gyakorló library nem fogadható el.

## Döntés

**YAZ4J (BSD-3-Clause) + `libyaz.so.5` natív bináris fat jar-ba csomagolva, `native-lib-loader` library-val betöltve.**

## Miért YAZ4J és nem JZKit2?

JZKit2 (AGPL v3) megköveteli, hogy a teljes alkalmazás AGPL-lé váljon — inkompatibilis a PolyForm Personal Use 1.0.0 licenccel. YAZ4J BSD-3-Clause licencelésű: semmilyen extra megkötés nincs a saját kódra; egyetlen kötelezettség a YAZ copyright szöveg mellékelése (`THIRD_PARTY_LICENSES.md`). A YAZ upstream aktívan karbantartott (Index Data, 2026-os commitek).

## Natív bináris csomagolás

A `libyaz.so.5` (Linux) és `yaz5.dll` (Windows) bekerül a fat jar-ba:

```
src/main/resources/native/linux-x86_64/libyaz.so.5
src/main/resources/native/win32-x86_64/yaz5.dll
```

Runtime betöltés: `native-lib-loader` library (`org.scijava`) — automatikusan extractálja `/tmp`-be és betölti. Alternatíva: kézi `System.load()` `/tmp` extract után.

## Miért nem Lambda layer?

A meglévő shade+S3 deploy flow változatlan kell maradjon. Lambda layer tanulása és bevezetése jelenleg nem cél.

## Linux natív bináris megszerzése (egyszeri, becommitelendő)

```bash
docker run --rm -v $(pwd)/output:/output public.ecr.aws/lambda/java:21 \
  sh -c "dnf install -y yaz && cp /usr/lib64/libyaz.so.5 /output/"
```

## Tranzitív JNI függőségek

`ldd libyaz.so.5` tranzitív `.so` függőségei (`libxml2.so.2`, `libgnutls.so.30`, `libz.so.1` stb.) az AWS Lambda AL2023 alap image-ben általában megtalálhatók. Ha hiányzó `.so` van, CloudWatch logban `dlopen failed: libxxx.so.N not found` jelenik meg — akkor azt is be kell csomagolni.

## Következmények

- Maven függőségek: `yaz4j`, `native-lib-loader`, `marc4j`
- `src/main/resources/native/` mappa becommitelve (~5–10 MB)
- YAZ BSD-3-Clause copyright szöveg: `THIRD_PARTY_LICENSES.md`-be kerül
- Lokális fejlesztés (Windows): YAZ Windows installer vagy `yaz5.dll` PATH-ba téve
- Lambda cold start: első híváskor native lib extract `/tmp`-be (néhány ms overhead)
