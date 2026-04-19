# Tech debt & jövőbeli teendők

Olyan feladatok, amelyek nem tartoznak aktív feature-höz, de határidőre vagy opportunisztikusan elvégzendők.

---

## Nyitott

### GitHub Actions — Node.js 24 migráció

**Határidő:** 2026-06-02 (forced Node.js 24 default)

**Érintett fájlok:**
- `.github/workflows/backend-deploy.yml`
- `.github/workflows/frontend-deploy.yml`

**Teendő:** `actions/checkout`, `actions/setup-java`, `dorny/test-reporter` action-ök Node.js 24-kompatibilis verziókra frissítendők. A jelenlegi verziók Node.js 20-at használnak, amely 2026-06-02 után deprecated lesz a GitHub Actions futtatókörnyezetben.

**Forrás:** Step 1.15 implementáció közben azonosítva.

---

## Lezárt

*(még üres)*
