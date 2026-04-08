# Step 1.7 – Alap layout

## Mit állít elő

- `src/components/layout/AppLayout.tsx` — fő layout komponens
- `src/components/layout/AppSidebar.tsx` — sidebar navigáció
- `src/components/layout/DarkModeToggle.tsx` — dark mode kapcsoló
- `src/App.tsx` frissítés — layout bekötése a routerbe

---

## Layout struktúra

shadcn/ui `Sidebar` komponensre épül.

```
┌─────────────────────────────────────────┐
│  AppSidebar  │  Content area            │
│              │                          │
│  - navigáció │  <Outlet />              │
│  - dark mode │  (React Router)          │
│    toggle    │                          │
└─────────────────────────────────────────┘
```

Az `AppLayout` a React Router `<Outlet />`-ot rendereli a tartalom területen — minden oldal ebbe töltődik be.

---

## Sidebar tartalom

Ebben a stepben placeholder navigációs elemek — a tényleges menüpontok a feature implementációk során kerülnek be.
A sidebar csak a bejelentkezett felhasználó szerepköréhez tartozó menüpontokat jelenítse meg (Zustand auth store alapján, Feature 2 után):

| Menüpont | Látható |
|----------|---------|
| Könyvek | ADMIN, VISITOR |
| Helyszínek | ADMIN, VISITOR |
| Kölcsönzések | ADMIN |
| Felhasználók | ADMIN |
| Saját profil | ADMIN, VISITOR |

---

## Dark mode

- `shadcn/ui` beépített dark mode támogatása (`ThemeProvider`)
- Állapot: `localStorage`-ban perzisztálva
- `DarkModeToggle`: ikon gomb a sidebar alján

---

## Elfogadási kritériumok

- Az alkalmazás megnyitásakor a sidebar és a content area látható
- Dark mode toggle működik, az állapot oldal újratöltés után is megmarad
- Reszponzív: mobilon a sidebar összecsukható (shadcn/ui `SidebarTrigger`)
