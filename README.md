# socpoist-apc-ops

Tento repozitár slúži ako **centrálne miesto pre evidenciu a riadenie operácií nad OpenShift clusterom**.

Podporuje:
- upgrade OpenShift klastra
- upgrade platformových komponentov
- inštaláciu a upgrade aplikácií
- maintenance operácie

---

# 🧩 Základný koncept

Každá zmena na clustri = **jedna operácia**

Operácia je reprezentovaná jedným folderom:


# 🧭 Postup krok za krokom

#### 1. Vytvorenie Jira epicu
#### 2. Vytvorenie branch s cislom Jira epicu
#### 3. Vytvor nový folder zo šablóny:

```bash
cp -r templates/operation operations/2026-03-<name-of-operation>
```

#### 4. Vyplnenie metadát operácie
#### 5. Príprava dokumentácie plan.md, changelog.md, checklist.md
#### 6. Review PR spolu so spustením automatizácie (kontrola vygenerovaného emailu, atď)
#### 7. Merge PR
#### 8. Github actions vygeneruje email notifikáciu zákazníkovi
#### 9. Vykonanie inštalácie/upgrade
#### 10. Vygenerovať a odoslať po inštalácií Post report

# 📌 Pravidlá

1 operácia = 1 folder
1 operácia = 1 Jira ticket
metadata.yaml je povinný
changelog.md je zdroj pre klientsku komunikáciu
interný checklist sa nikdy neposiela klientovi


# 🔁 Proces operácie v cluster-operations repozitári

```mermaid
flowchart TD

A[Vytvorenie Jira Epicu] --> B[Vytvorenie branch<br/>s číslom Jira Epicu]

B --> C[Vytvorenie operácie zo šablóny]

C --> C1[cp -r templates/operation<br/>operations/2026-03-<name>]

C1 --> D[Vyplnenie metadát operácie<br/>metadata.yaml]

D --> E[Príprava dokumentácie]

E --> E1[plan.md]
E --> E2[changelog.md]
E --> E3[checklist.md]

E1 --> F[Pull Request + Review]

E2 --> F
E3 --> F

F --> G{Schválené?}

G -- Nie --> E

G -- Áno --> H[Merge PR]

H --> I[GitHub Actions<br/>vygeneruje email pre zákazníka]

I --> J[Notifikácia zákazníkovi<br/>- plán operácie]

J --> K[Vykonanie inštalácie / upgrade]

K --> L[Vytvorenie post-report.md]

L --> M[GitHub Actions<br/>vygeneruje post-report email]

M --> N[Odoslanie post-report zákazníkovi]

N --> O[Uzavretie Jira Epicu]
