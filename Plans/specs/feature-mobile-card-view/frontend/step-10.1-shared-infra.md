# Step 10.1 – Mobil kártyás nézet – közös infrastruktúra

## Mit állít elő

- `src/lib/breakpoints.ts` — breakpoint konstans
- `src/hooks/useMediaQuery.ts` — generikus media query hook
- `src/hooks/useInfiniteBackendList.ts` — generikus infinite-scroll adatlekérő hook
- `src/components/common/InfiniteCardList.tsx` — generikus kártyalista UI shell
- Hozzájuk tartozó unit tesztek

---

## Motiváció

Az AG Grid táblázatos nézet (Books, Locations) mobil viewportnál rosszul használható — a Books lista elforgatva is macerás, a Locations grid elforgatva még élhető, de a cél mindkettőnél a kártyás nézet mobilon. Ez a step a Books (10.2) és Locations (10.3) kártyás implementációja alatt közös, egyszer megírt infrastruktúrát ad — nincs duplikált data-fetching vagy scroll logika a két feature között.

Asztali nézetben mindkét oldal AG Grid Infinite Row Model-t használ továbbra is, változatlanul.

---

## Breakpoint

Egy konstans query definiálja a mobil breakpointot, igazítva a Tailwind `md` töréshez: `< 768px` → mobil (kártyás nézet), `≥ 768px` → asztali (grid).

A váltás **JS-alapú**, nem CSS `display:none` — így mindig csak az aktív nézet (grid vagy kártyalista) épül fel és lekérez adatot, a másik egyáltalán nem renderel, nem duplikálódik az API hívás.

---

## `useMediaQuery` hook

- Egy media query stringet fogad, boolean-t ad vissza, hogy az aktuálisan illeszkedik-e
- A böngésző natív media query API-jára épül, viewport-változásra (pl. elforgatás, ablak átméretezés) automatikusan frissül
- Kliens-oldali komponens, SSR eset nem releváns (SPA)

Használat a lapokon: a breakpoint hook eredménye dönt, hogy `GridView` vagy `CardView` render-elődik.

---

## `useInfiniteBackendList` hook

Generikus hook — nem tudja, hogy Book vagy Location listát tölt, csak egy oldal-lekérő függvényt kap paraméterként a hívó oldaltól.

**Bemenet (koncepcionálisan):**
- Egy fetch-függvény, amit a hívó oldal ad át, és amely egy oldal adatot kér le a backendtől (a meglévő `bookApi.ts` / `locationApi.ts` függvényeire épül — **nincs új API réteg**)
- Oldalméret (alapértelmezett: 20)
- Egy **reset kulcs**: stabil, primitív érték (string vagy szám), amit a hívó oldal állít elő — tipikusan a szűrő- és sort-állapot szerializálva a Zustand refresh triggerrel összefűzve, pl. `` `${refreshTrigger}|${sort}|${JSON.stringify(filters)}` `` egy `useMemo`-ban előállítva. A hook a kulcsot referencia szerint, nem mélységi összehasonlítással figyeli — ezért **objektum vagy tömb közvetlenül nem adható át reset kulcsként**, csak a belőle előállított primitív. Ez elkerüli, hogy egy minden render-ben újra létrejövő szűrő-objektum végtelen refetch-hurkot indítson.

**Kimenet (koncepcionálisan):**
- Az addig betöltött elemek listája
- Első betöltés és "következő oldal töltése" állapot külön jelezve (más UI-t kap: teljes felületet lefedő spinner vs. lista alji spinner)
- **Hibaállapot**: `error` (van-e hiba az utolsó lekérésen — akár első betöltés, akár következő oldal), és egy `retry()` függvény, ami megismétli a legutóbb sikertelen lekérést
- Van-e még következő oldal
- Egy "tölts be többet" függvény, amit a scroll-trigger hív
- **Helyben-módosító függvények** a felhalmozott listára: `updateItem(id, updater)` és `removeItem(id)` — mutáció után a hívó oldal ezekkel frissíti a már betöltött elemek közül az érintettet, teljes lista-reset nélkül (ld. lentebb, "Mutáció utáni frissítés")

**Viselkedés:**
- A reset kulcs változásakor a felhalmozott elemek törlődnek, és a lekérés az elejéről indul — ugyanaz a szemantika, mint a grid nézet `purgeInfiniteCache()` hívása
- Ha már fut egy lekérés, egy újabb "tölts be többet" hívás nem indít párhuzamos, dupla fetch-et (gyors scroll elleni védelem)
- **Reset ↔ folyamatban lévő oldal-lekérés versenyhelyzet:** minden fetch egy monoton növekvő **kérés-generációs számlálóval** van megjelölve; a reset kulcs változásakor a számláló nő, és csak az aktuális generációhoz tartozó válasz kerül be a state-be. Így ha a felhasználó görgetés közben (page N fetch folyamatban) szűrőt vagy sortot vált, a később megérkező, elavult page N válasz nem fűződik hozzá a friss (page 0-ról induló) listához.
- Sikertelen lekérés (network error vagy 4xx/5xx) esetén a hook nem próbálkozik automatikusan újra — `error` állapotba kerül, `isLoadingMore`/`isLoading` `false` lesz, és a hívó oldalnak fel kell ajánlania a `retry()`-t
- "Van több oldal" a backend lapozott válaszának utolsó-oldal jelzéséből származik (ugyanaz a `Page<T>` válaszszerkezet, amit az AG Grid Infinite Row Model is használ)

### Mutáció utáni frissítés

A `booksRefreshTrigger` / `locationsRefreshTrigger` (Zustand) a reset kulcs egyik összetevőjeként **létrehozás** után továbbra is teljes reset-et vált ki — új elem rendezett pozíciója a felhalmozott listában nem ismerhető előre olcsón.

**Szerkesztés és törlés után viszont nem a reset kulcs a javasolt út**, hanem a hívó oldal a mutáció sikeres válaszából (amit a modal amúgy is megkap) közvetlenül hívja a hook `updateItem` / `removeItem` függvényét. Ez elkerüli, hogy a felhasználó minden apró szerkesztés után visszaugorjon a lista tetejére és újra végig kelljen görgetnie — mobilon, ahol a scroll az egyetlen navigáció, ez érezhető regresszió lenne teljes reset esetén.

A `refreshTrigger` emiatt is inkrementálódhat szerkesztés/törlés után (pl. a Locations oldalon a Rooms panel `locationCount`-jának frissítéséhez), de ez **nem** kényszeríti ki a kártyalista reset-jét — a kártyalista a saját `updateItem`/`removeItem` hívásán keresztül frissül, a Rooms panel adatfetch-e (`GET /api/rooms/all`) pedig a refresh triggerre külön iratkozik fel.

---

## `InfiniteCardList` component

Generikus UI shell — a kártya *tartalmát* nem ismeri, azt a hívó oldal adja meg soronként (a konkrét mezőket ld. a hívó feature specekben: Books — 10.2, Locations — 10.3).

- A lista végén egy figyelt terület van, ami látótérbe kerülve kéri a következő oldalt (amíg van még betölthető és nincs már folyamatban lévő lekérés)
- Első betöltés alatt: teljes területet lefedő betöltés-jelzés
- Következő oldal töltése alatt: kompakt, lista alji betöltés-jelzés, a már megjelenő kártyák felett
- **Hiba állapotban:**
  - Ha az első betöltés hibázott (még nincs egy elem sem): teljes felületű hibaállapot, "Nem sikerült betölteni" üzenettel (i18n kulcs: `common.errorUnexpected`, konzisztens a tech-debt.md "Frontend hiba-üzenetek differenciálása" tételével) és "Újrapróbálom" gombbal, ami a hook `retry()`-ját hívja
  - Ha egy következő oldal lekérése hibázott (már vannak betöltött elemek): lista alji, kompakt hibasáv ugyanezzel az üzenettel és retry gombbal, a meglévő kártyák megmaradnak
- Üres eredmény esetén (nincs elem, nincs folyamatban lekérés, nincs hiba): a hívó oldal által megadott, i18n-elt "nincs találat" üzenet
- A kártya-layoutot a hívó oldal adja, ez a komponens csak a lista-mechanikát (scroll, betöltés-állapotok, hibaállapotok, empty state) biztosítja

---

## Kulcs döntések

- **Nincs TanStack Query bevezetve** — a jelenlegi axios + Zustand refresh trigger minta konzisztenciáját tartjuk meg; a hook kézzel implementálja az akkumulációt és reset logikát
- Breakpoint JS hookkal döntve, nem CSS-sel, hogy csak egy nézet fetch-eljen egyszerre
- Alapértelmezett oldalméret kártyás nézetben: **20** — nem kell egyezzen az asztali grid oldalméretével
- A grid nézet (AG Grid Infinite Row Model) ezzel a steppel nem módosul
- A meglévő API hívó függvények (`bookApi.ts`, `locationApi.ts`) újrahasználva, nem duplikálva — a hook csak orchestrálja a meglévő hívásokat
- Reset kulcs kizárólag stabil primitív (nem objektum-referencia) — a hívó oldal felelőssége a szerializálás
- Szerkesztés/törlés mutáció után helyben-frissítés (`updateItem`/`removeItem`), nem teljes reset — csak létrehozás után reset

---

## Tech-debt / jövőbeli megfontolás

- Ha a listázó feature-ök száma nő (Loans, Users hasonló mintával), érdemes újra megnézni a TanStack Query bevezetését a kézzel írt cache/reset logika helyett

---

## Elfogadási kritériumok

**Unit tesztek** (Vitest + React Testing Library, `matchMedia`, `IntersectionObserver` és axios mockolva — az `IntersectionObserver` jsdom-ban alapból nincs implementálva, `vi.stubGlobal`-lal mockolandó):
- `useMediaQuery`: kezdeti érték a query state-jét tükrözi; viewport-változásra frissül
- `useInfiniteBackendList`: "tölts be többet" hívásra a következő oldal hozzáfűződik a listához, nem duplikál
- `useInfiniteBackendList`: ha nincs több oldal, "tölts be többet" nem indít újabb fetch-et
- `useInfiniteBackendList`: a reset kulcs változásakor a lista törlődik, majd az elejéről tölt újra
- `useInfiniteBackendList`: változatlan reset kulcs melletti újrarender nem indít fetch-et
- `useInfiniteBackendList`: gyors, egymást követő "tölts be többet" hívások nem indítanak dupla fetch-et, amíg az előző fut
- `useInfiniteBackendList`: fetch hiba után a lista nem próbálkozik automatikusan újra; `error` állapotba kerül
- `useInfiniteBackendList`: `retry()` hívása megismétli a legutóbb sikertelen lekérést
- `useInfiniteBackendList`: folyamatban lévő oldal-lekérés közbeni reset esetén a régi (elavult generációjú) válasz nem jelenik meg a listában
- `useInfiniteBackendList`: `updateItem` a megadott id-jű elemet cseréli a felhalmozott listában, a többi elem és a scroll-pozíció változatlan
- `useInfiniteBackendList`: `removeItem` eltávolítja a megadott id-jű elemet a felhalmozott listából
- `InfiniteCardList`: nulla elem, nincs hiba és nincs folyamatban lekérés esetén az üres-állapot üzenet jelenik meg
- `InfiniteCardList`: a lista-vég figyelt terület látótérbe kerülésekor a "tölts be többet" meghívódik
- `InfiniteCardList`: első betöltés hibája esetén teljes felületű hibaállapot jelenik meg, retry gombbal
- `InfiniteCardList`: következő oldal hibája esetén lista alji hibasáv jelenik meg, a meglévő kártyák megmaradnak

**Manuálisan:**
- Viewport `< 768px` szélességnél a hívó oldal (10.2/10.3) kártyás nézetre vált, `≥ 768px`-nél visszavált gridre, oldal-újratöltés nélkül
