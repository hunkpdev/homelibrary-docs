# Step 5.5.1 – Mobil kártyás nézet – közös infrastruktúra

## Mit állít elő

- `src/lib/breakpoints.ts` — breakpoint konstans
- `src/hooks/useMediaQuery.ts` — generikus media query hook
- `src/hooks/useInfiniteBackendList.ts` — generikus infinite-scroll adatlekérő hook
- `src/components/common/InfiniteCardList.tsx` — generikus kártyalista UI shell
- Hozzájuk tartozó unit tesztek

---

## Motiváció

Az AG Grid táblázatos nézet (Books, Locations) mobil viewportnál rosszul használható — a Books lista elforgatva is macerás, a Locations grid elforgatva még élhető, de a cél mindkettőnél a kártyás nézet mobilon. Ez a step a Books (5.5.2) és Locations (5.5.3) kártyás implementációja alatt közös, egyszer megírt infrastruktúrát ad — nincs duplikált data-fetching vagy scroll logika a két feature között.

Asztali nézetben mindkét oldal AG Grid Infinite Row Model-t használ továbbra is, változatlanul.

---

## Breakpoint

Egy konstans query definiálja a mobil breakpointot, igazítva a Tailwind `md` töréshez (768px alatt mobil).

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
- Egy "reset kulcs" — bármi, ami változáskor újratöltést jelent: szűrő állapot, sort, vagy a Zustand refresh trigger (`booksRefreshTrigger` / `locationsRefreshTrigger`)

**Kimenet (koncepcionálisan):**
- Az addig betöltött elemek listája
- Első betöltés és "következő oldal töltése" állapot külön jelezve (más UI-t kap: teljes felületet lefedő spinner vs. lista alji spinner)
- Van-e még következő oldal
- Egy "tölts be többet" függvény, amit a scroll-trigger hív

**Viselkedés:**
- A "reset kulcs" változásakor a felhalmozott elemek törlődnek, és a lekérés az elejéről indul — ugyanaz a szemantika, mint a grid nézet `purgeInfiniteCache()` hívása
- Ha már fut egy lekérés, egy újabb "tölts be többet" hívás nem indít párhuzamos, dupla fetch-et (gyors scroll elleni védelem)
- "Van több oldal" a backend lapozott válaszának utolsó-oldal jelzéséből származik (ugyanaz a `Page<T>` válaszszerkezet, amit az AG Grid Infinite Row Model is használ)

---

## `InfiniteCardList` component

Generikus UI shell — a kártya *tartalmát* nem ismeri, azt a hívó oldal adja meg soronként.

- A lista végén egy figyelt terület van, ami látótérbe kerülve kéri a következő oldalt (amíg van még betölthető és nincs már folyamatban lévő lekérés)
- Első betöltés alatt: teljes területet lefedő betöltés-jelzés
- Következő oldal töltése alatt: kompakt, lista alji betöltés-jelzés, a már megjelenő kártyák felett
- Üres eredmény esetén (nincs elem és nincs folyamatban lekérés): a hívó oldal által megadott, i18n-elt "nincs találat" üzenet
- A kártya-layoutot (Books: cím/szerző/leírás/badge-ek; Locations: név/leírás/room badge/bookCount badge) a hívó oldal adja, ez a komponens csak a lista-mechanikát (scroll, betöltés-állapotok, empty state) biztosítja

---

## Kulcs döntések

- **Nincs TanStack Query bevezetve** — a jelenlegi axios + Zustand refresh trigger minta konzisztenciáját tartjuk meg; a hook kézzel implementálja az akkumulációt és reset logikát
- Breakpoint JS hookkal döntve, nem CSS-sel, hogy csak egy nézet fetch-eljen egyszerre
- Alapértelmezett oldalméret kártyás nézetben: **20** — nem kell egyezzen az asztali grid oldalméretével
- A grid nézet (AG Grid Infinite Row Model) ezzel a steppel nem módosul
- A meglévő API hívó függvények (`bookApi.ts`, `locationApi.ts`) újrahasználva, nem duplikálva — a hook csak orchestrálja a meglévő hívásokat

---

## Tech-debt / jövőbeli megfontolás

- Ha a listázó feature-ök száma nő (Loans, Users hasonló mintával), érdemes újra megnézni a TanStack Query bevezetését a kézzel írt cache/reset logika helyett

---

## Elfogadási kritériumok

**Unit tesztek** (Vitest + React Testing Library, `matchMedia` és axios mockolva):
- `useMediaQuery`: kezdeti érték a query state-jét tükrözi; viewport-változásra frissül
- `useInfiniteBackendList`: "tölts be többet" hívásra a következő oldal hozzáfűződik a listához, nem duplikál
- `useInfiniteBackendList`: ha nincs több oldal, "tölts be többet" nem indít újabb fetch-et
- `useInfiniteBackendList`: a reset kulcs változásakor a lista törlődik, majd az elejéről tölt újra
- `useInfiniteBackendList`: gyors, egymást követő "tölts be többet" hívások nem indítanak dupla fetch-et, amíg az előző fut
- `InfiniteCardList`: nulla elem és nincs folyamatban lekérés esetén az üres-állapot üzenet jelenik meg
- `InfiniteCardList`: a lista-vég figyelt terület látótérbe kerülésekor a "tölts be többet" meghívódik

**Manuálisan:**
- Viewport 767px alá szűkítésekor a hívó oldal (jövőbeli 5.5.2/5.5.3) kártyás nézetre vált, 768px felett visszavált gridre, oldal-újratöltés nélkül
