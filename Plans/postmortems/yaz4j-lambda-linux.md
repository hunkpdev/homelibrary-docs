# Z39.50 (yaz4j) natív könyvtár betöltése AWS Lambda Linux runtime-on

**Dátum:** 2026-05-12
**Status:** Megoldva, élesben működik
**Kapcsolódik:** ADR-010 (Z39.50 client library), Feature 4 (ISBN lookup), Feature 5 (Books CRUD)
**Repó-hivatkozás:** `.github/workflows/extract-native-libs.yml`, branch `fix/yaz4j-native-library-loading`

---

## Kontextus

Az OSZK NEKTÁR ISBN-lookup a Z39.50 protokollra épül, amit a `yaz4j` Java wrapper használ a `libyaz` natív C könyvtáron keresztül. A backend Spring Boot fat JAR-ként AWS Lambda `java21` runtime-on fut (Amazon Linux 2023 alap, glibc 2.34).

Lokál Windows alatt minden ment (bundled `yaz5.dll` egyetlen `System.load()` hívással betöltve), Lambda alatt viszont az `UnsatisfiedLinkError` szivárgott — első ránézésre triviális natív-lib hiba, valójában négy egymást elfedő rétegű probléma.

---

## A hibarétegek (felülről lefelé)

### 1. réteg — Dlopen DT_NEEDED feloldás

A yaz4j `Connection` osztály static initializere a JAR resource-ból kicsomagolja a `libyaz4j.so`-t `/tmp/xxxNNNlibyaz4j.so`-ra, és `System.load()`-olja. A dlopen viszont a `libyaz4j.so` DT_NEEDED `libyaz.so.5` függőségét nem találja:

```
/tmp/xxxNNNlibyaz4j.so: libyaz.so.5: cannot open shared object file: No such file or directory
```

A két korábbi fix-kísérlet (`fix/yaz4j-native-library-loading` branch első két commit-ja) azt feltételezte, hogy a `libyaz4j.so` `RPATH=$ORIGIN`-nel van fordítva, így ha a libyaz.so.5-öt ugyanabba a temp dir-be tesszük, megtalálja. **Ez nem igaz** — a `readelf -d libyaz4j.so` output:

```
NEEDED  libyaz.so.5
NEEDED  libstdc++.so.6
NEEDED  libc.so.6
(NINCS RPATH, NINCS RUNPATH)
```

Tehát a dlopen kizárólag `LD_LIBRARY_PATH` + `/etc/ld.so.cache` + system path-okon (`/lib64`, `/usr/lib64`) keres. A `/tmp`-be tett file-t soha nem találja, akármilyen sorrendben is `System.load`-olnánk.

**Fontos részlet:** a Java `System.load()` `RTLD_LOCAL`-lal hív dlopen-t. A betöltött library szimbólumai **nem kerülnek a globális symbol table-be**, így a következő dlopen DT_NEEDED feloldása sem találja meg őket. A "System.load order" trükk technikailag képtelenség Java-ból.

### 2. réteg — Tranzitív .so zárás hiányzik AL2023 base image-en

A `libyaz.so.5` saját NEEDED listája:

```
libgnutls.so.30
libwrap.so.0      (deprecated; AL2023-on egyáltalán nincs)
libexslt.so.0
libxslt.so.1
libxml2.so.2
libc.so.6
```

Ezek többsége nincs a Lambda `java21` runtime AL2023 base image-én. Még ha a `libyaz.so.5` feloldódik is, a saját DT_NEEDED-jei nem.

### 3. réteg — Glibc-mismatch a tranzitív lib-ekben

Első körben az `extract-native-libs.yml` workflow Ubuntu 24.04 docker image-ből szedte ki a libyaz-t és a tranzitív zárást (`apt-get install libyaz-dev`). A két fő binárison (libyaz.so.5, libyaz4j.so) a `readelf -V` szerint a legmagasabb GLIBC szimbólum 2.34 — pontosan az AL2023 glibc verziója. Az analízis ebből arra a következtetésre jutott, hogy az Ubuntu 24.04 build kompatibilis lesz.

**A tévedés:** csak a két fő binárist néztem, a tranzitív zárást nem. A `libgnutls.so.30`, `libxml2.so.2`, `libxslt.so.1` és társaik Ubuntu 24.04 csomagolásában **GLIBC_2.35–2.38 szimbólumokat is használnak** — ezek AL2023 glibc 2.34-en `version GLIBC_2.X not found` hibával nem töltődnek.

Ubuntu 20.04-re visszalépés (glibc 2.31, "alulról" passzolna) ötletként felmerült, de elvetve: 2025 áprilisában EOL, és az infrát felfelé húzzuk, nem lefelé.

### 4. réteg (a tényleges blocker) — yaz4j upstream pom.xml swig-bug

Az AL2023 source-build átállás (yaz és yaz4j source-ból az `amazonlinux:2023` docker image-en) után a swig parancs `exit 1`-gyel megdöglött a yaz4j `generate-sources` Maven fázisban, konkrét hibaüzenet nélkül (`mvn -q` elnyelte).

A `yaz4j` upstream `pom.xml` Unix-profile-jában a swig hívás:

```xml
<exec failonerror="true" executable="${swig}">
    <arg value="-Isrc/main/native" />
    <arg value="${yaz.cflags}" />   <!-- HIBÁS: value, line helyett -->
    <arg value="-I/usr/include" />
    ...
</exec>
```

Az ant `<arg value="...">` az egész stringet **egyetlen argumentumként** adja át a swig-nek. A `<arg line="...">` az, ami szóközök mentén több argumentumra bontja.

A `yaz.pc.in` `Cflags:` sora autoconf-substituted:

```
Cflags: -I${includedir} @YAZ_CONFIG_CFLAGS@
```

`@YAZ_CONFIG_CFLAGS@` config után pl.: `-DYAZ_HAVE_XML2=1 -DYAZ_HAVE_XSLT=1 -I/usr/include/libxml2`. Tehát `pkg-config --cflags yaz` Unix-on **többtagú output**-ot ad. Ez a `<arg value>` miatt **egy stringként** érkezik a swig-hez, ami nem tudja értelmezni a `-I` flag-ként → `exit 1`.

A pom.xml `compile-zoom-extra` execution-ja a `<arg line="${yaz.cflags}" />` (helyesen) használja — tehát a build-system készítője tudott a különbségről, csak a swig hívásnál figyelmen kívül hagyta. Windows profile-ban `<yaz.cflags>` egytagú string (`-I${yaz.path}\include`), ott `value` és `line` ekvivalensen viselkedik — ezért a bug Windows-on nem jelenik meg, és upstream-en mindeddig észrevétlen maradt.

---

## A megoldás (négy független rész)

### 1. Bundle a fat JAR-ba

A tranzitív .so-k Maven resources-on át kerülnek a fat JAR-ba:

- `backend/src/main/resources/native/Linux/amd64/libyaz4j.so` — a yaz4j Connection static init keresi a classpath-en, ő maga csomagolja ki `/tmp`-re.
- `backend/src/main/resources/lib/libyaz.so.5` + tranzitív zárás (libgnutls, libxml2, libxslt, libexslt és azok dep-jei).

A shade plugin a fat JAR-ba ágyazza, a Lambda Java 21 runtime `/var/task/`-ba csomagolja ki.

### 2. `LD_LIBRARY_PATH=/var/task/lib` Lambda env var

Egyszeri kézi beállítás (`aws lambda update-function-configuration`-nal a meglévő env varokat is megőrizve). A dlopen ezt nézi DT_NEEDED feloldásakor, így a `/var/task/lib/`-ben bundle-elt tranzitív zárás láthatóvá válik.

### 3. Source-build `amazonlinux:2023` docker image-en

Az `extract-native-libs.yml` workflow yaz-t és yaz4j-t source-ból fordít AL2023 image-en. Így minden bináris és tranzitív lib **natívan AL2023 toolchain-nel és glibc 2.34-re** épül — garantáltan kompatibilis. A `make install` után a header-ek és a `yaz.pc` a standard `--prefix` alatt helyükre kerülnek.

A yaz `--disable-tcp-wrappers` configure-flag-gel a libwrap függőséget teljesen kihagyjuk (a libwrap deprecated, AL2023-on amúgy sincs csomagolva).

### 4. yaz4j pom.xml swig-patch a klón után

```bash
sed -i 's|<arg value="${yaz.cflags}" />|<arg line="${yaz.cflags}" />|g' pom.xml
```

Workflow-szintű patch, nem upstream PR-rel (a workflow minden futáskor frissen klónozza a yaz4j master-t, így a sed mindig fut). Globális csere — a Windows profile-ban is biztonságos, mert ott `<yaz.cflags>` egytagú, és `line` egytagú inputon ekvivalensen viselkedik a `value`-val.

### Validációs lépés

A workflow `Validate native lib closure against Lambda AL2023 runtime` step-je a commit előtt lefuttatja `ldd`-vel a `public.ecr.aws/lambda/java:21` image-en a bundle-ölt `lib/` mappát. Ha bármely tranzitív `.so` `not found` állapotba kerül, a workflow piros, és a hibás zárás nem kerül be a repóba.

---

## Általános tanulságok

A jövőbeli hasonló helyzetekre érdemes lehet elővenni:

1. **Java `System.load()` RTLD_LOCAL-lal hív dlopen-t.** A betöltött library szimbólumai NEM kerülnek a globális symbol table-be, így a következő dlopen DT_NEEDED-feloldása sem találja meg őket. JNI-világban a függő .so-k feloldása **LD_LIBRARY_PATH** dolga, nem Java oldali `System.load` order-é.

2. **A dynamic linker DT_NEEDED feloldás csak az RPATH/RUNPATH/LD_LIBRARY_PATH/system-defaults path-okat nézi.** Akármilyen file ott van valahol a filesystem-en, ha nincs ezeken az útvonalakon, soha nem találja meg.

3. **Glibc forward-compatibility nincs.** Újabb glibc-vel épített binárisok régebbi glibc-n szimbólumhiányos hibával nem futnak. Lambda runtime alatt a base image glibc verzióját kell megnézni és arra kell épülni. **Legtisztább**: a runtime base image-én magában fordítani.

4. **Autotools `--prefix` + `make install` után minden a helyén van.** Ha "header not found" hibára futunk a build-ben, az első kérdés ne az legyen hogy a header miért nincs ott (jó eséllyel ott van), hanem hogy a `-I` flag-ek jól-e jutnak el a feldolgozó eszközhöz.

5. **Ant `<arg value>` vs `<arg line>`:** `value` egy argumentumként ad át, `line` szóköz-mentén szétbont. Multi-token stringek (pkg-config output, CFLAGS) átadásakor kritikus a különbség.

6. **A diagnosztikai output elnyelése (`mvn -q`, `make -s` stb.) félrediagnosztizálást okozhat.** Amikor egy build-tool `exit 1`-et ad konkrét hibaüzenet nélkül, az első lépés a verbose mode visszahelyezése — a konkrét stderr legalább annyit szokott segíteni, mint amennyit "spórolnánk" vele.

7. **Workaround-rétegek halmozódása veszélyes.** Minden workaround azért készül, hogy egy nem-pontosan-értett problémát megkerüljön. Ha a valódi gyökér-ok rejtve marad, a workaround-ok csak rakódnak egymásra, és a setup egyre fragilisabb lesz. Tünet: rövid időn belül több hasonló-de-más fix-commit megy egymás után. Ekkor érdemes lépni egyet hátra, és a tényleges hibát egzakt diagnosztikával megkeresni, nem újabb kerülő úttal megpróbálkozni.

---

## Hivatkozások

- ADR-010 — Z39.50 client library választás (yaz4j Lambda-on)
- `fix/yaz4j-native-library-loading` branch — a hibakeresés és a megoldás commit-jai
- `.github/workflows/extract-native-libs.yml` — a végleges native-lib pipeline
- OSZK NEKTÁR Z39.50 endpoint: `tagetes2.oszk.hu:1616` (plain TCP, nincs TLS)
- yaz upstream: https://github.com/indexdata/yaz
- yaz4j upstream: https://github.com/indexdata/yaz4j (a swig argumentum-átadás bug `master`-en jelen van; workflow-szintű sed-patch a megoldás)
