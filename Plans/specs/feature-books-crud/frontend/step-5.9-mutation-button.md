# Step 5.9 – MutationButton komponens és DEMO role detektálás

## Mit állít elő

- `src/components/common/MutationButton.tsx` — DEMO-tudatos mutáció gomb wrapper
- `src/store/authStore.ts` módosítása: `AuthUser.role` típus bővítve `'DEMO'`-val
- Feature 3 meglévő mutáció gombjainak (mentés, törlés, megerősítés) cseréje `MutationButton`-ra

---

## `AuthUser` típus bővítése

```ts
role: 'ADMIN' | 'VISITOR' | 'DEMO'
```

A `'DEMO'` értéket a JWT `role` claim tartalmazza — a login flow (step 2.9) változatlan, csak a típus bővül.

---

## `MutationButton` komponens

**Fájl:** `src/components/common/MutationButton.tsx`

**Props:**

| Prop | Típus | Leírás |
|------|-------|--------|
| `children` | `React.ReactNode` | Gomb felirata |
| `onClick` | `() => void` | Opcionális click handler |
| `type` | `'button' \| 'submit' \| 'reset'` | Default: `'button'` |
| `variant` | shadcn Button variant | Default: `'default'`; törlés gomboknál `'destructive'` |
| `disabled` | `boolean` | Szülő által kért disabled állapot (pl. form submitting) — DEMO disabled-del OR-olódik |
| `className` | `string` | Opcionális Tailwind osztályok |

**Működés:**
- `isDemo = useAuthStore(state => state.user)?.role === 'DEMO'`
- Ha `isDemo` VAGY `disabled` → a gomb `disabled`
- Ha `isDemo` → shadcn `Tooltip`-ba csomagolva, hover-re megjelenik az üzenet: `"Demo módban ez a művelet nem elérhető"`
- A Tooltip trigger egy `<span>` wrapper a gomb körül — szükséges, mert a böngésző `disabled` gombon nem tüzel hover eseményt (`pointer-events: none`)

**Biztonsági megjegyzés:** a kliensoldali disabled UX célú; a tényleges védelmet a step 5.2-ben beállított Spring Security szerver oldali szabályok adják.

---

## Feature 3 retrofit

A Feature 3-ban (Rooms és Locations) már implementált mentés / törlés / megerősítés gombok cseréje `MutationButton`-ra. Érintett komponensek:
- `RoomFormModal` — mentés és törlés gomb
- `LocationFormModal` — mentés és törlés gomb

A csere mechanikus: `<Button onClick={...} variant="destructive">Törlés</Button>` → `<MutationButton onClick={...} variant="destructive">Törlés</MutationButton>`.

---

## Kulcs döntések

- Nincs `isDemo` boolean flag a store-ban — a `user.role === 'DEMO'` értéket a komponens lokálisan számítja
- A Tooltip üzenet egyelőre hardcoded string — Feature 8 (i18n) fogja kulcsra cserélni
- Feature 6–7 minden mutáció gombja ezt a komponenst használja; nem szükséges visszatérni erre

---

## Elfogadási kritériumok

- ADMIN tokennel bejelentkezve: `MutationButton` kattintható, tooltip nem jelenik meg
- DEMO tokennel bejelentkezve: minden `MutationButton` disabled, hoverre tooltip látható
- Feature 3 room/location törlés és mentés gombok DEMO usernek disabled állapotban vannak
- `AuthUser.role` típusban `'DEMO'` TypeScript fordítási hiba nélkül elfogadott
