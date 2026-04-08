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

Ebben a stepben minden menüpont megjelenik — role-based szűrés nincs még. A láthatóság szerepkör szerinti szabályozása a Feature 2 (step 2.13) stepjében kerül bekötésre.

- Könyvek
- Helyszínek
- Kölcsönzések
- Felhasználók
- Saját profil

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
