# Step 2.13 – Role-alapú menü szűrés + logout gomb

## Mit állít elő

- `src/components/layout/AppSidebar.tsx` bővítése — bejelentkezett user neve + role-alapú menü szűrés + logout gomb

---

## Sidebar bővítések

**Bejelentkezett user neve:** a sidebar tetején vagy alján megjelenik a `user.username` a Zustand store-ból.

**Role-alapú menü szűrés:** az 1.7-es stepben minden menüpont megjelenik, itt kerül bekötésre a láthatóság:

| Menüpont | Látható |
|---|---|
| Könyvek | ADMIN, VISITOR |
| Helyszínek | ADMIN |
| Kölcsönzések | ADMIN |
| Felhasználók | ADMIN |
| Saját profil | ADMIN, VISITOR |

**Logout gomb:** a sidebar alján, a `DarkModeToggle` mellett.

**Logout flow:**
1. `POST /api/auth/logout` az axiosInstance-on keresztül (Bearer token automatikusan megy a request interceptortól)
2. `clearAuth()` a Zustand store-ban
3. Redirect `/login`-ra (React Router `navigate`)

---

## Elfogadási kritériumok

**Manuálisan:**
- Bejelentkezve a user neve megjelenik a sidebarban
- ADMIN userként minden menüpont látható
- VISITOR userként csak Könyvek és Saját profil látható
- Logout gombra kattintva: logout hívás, store törlődik, redirect `/login`
- Logout után a visszagomb nem visz vissza védett oldalra (store üres → ProtectedRoute redirect)
