# Step 1.9 – CDK projekt setup

## Mit állít elő

- `infra/` könyvtár a projekt gyökerében
- TypeScript CDK projekt alapstruktúrával
- Egyetlen stack: `HomelibraryStack`

---

## Könyvtárstruktúra

```
infra/
├── bin/
│   └── homelibrary.ts        ← CDK app entry point
├── lib/
│   └── homelibrary-stack.ts  ← HomelibraryStack (egyelőre üres)
├── cdk.json
├── tsconfig.json
└── package.json
```

---

## Konfiguráció

- CDK app: egyetlen stack (`HomelibraryStack`)
- Region: `eu-central-1` (Frankfurt) — env var-ból: `CDK_DEFAULT_REGION`
- Account: `CDK_DEFAULT_ACCOUNT`

---

## Elfogadási kritériumok

- `cdk synth` hiba nélkül lefut, üres CloudFormation template generálódik
- `npm run build` TypeScript fordítás hiba nélkül
