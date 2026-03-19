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
cp -r templates/operations operations/2026-03-<name-of-operation>
```

#### 4. Vyplnenie metadát operácie
#### 5. Príprava dokumentácie changelog.md, checklist.md
#### 6. Review PR spolu so spustením automatizácie (kontrola vygenerovaného emailu, atď)
#### 7. Merge PR
#### 8. Github actions vygeneruje email notifikáciu zákazníkovi
#### 9. Vykonanie inštalácie/upgrade
#### 10. Vygenerovať a odoslať po inštalácií Post report

# 📌 Pravidlá

- 1 operácia = 1 folder
- 1 operácia = 1 Jira ticket
- metadata.yaml je povinný
- changelog.md je zdroj pre klientsku komunikáciu
- interný checklist sa nikdy neposiela klientovi



# 🔁 Proces operácie v cluster-operations repozitári

```mermaid
flowchart TD

A[Vytvorenie Jira Epicu]

B[Vytvorenie branch<br/>s číslom Jira Epicu]

C[Vytvorenie operácie zo šablóny]
C1[cp -r templates/operation<br/>operations/2026-03-<name>]

D[Vyplnenie metadát operácie<br/>metadata.yaml]

E[Príprava dokumentácie]

E1[changelog.md]
E2[checklist.md]

F[Pull Request + Review<br/>+ spustenie automatizácie]

G{Schválené?}

H[Merge PR]

I[GitHub Actions<br/>vygeneruje email pre zákazníka]

J[Odoslanie notifikácie zákazníkovi]

K[Vykonanie inštalácie / upgrade]

L[Vytvorenie post-report.md]

M[GitHub Actions<br/>vygeneruje post-report email]

N[Odoslanie post-report zákazníkovi]

O[Uzavretie Jira Epicu]

A --> B --> C --> C1 --> D --> E --> E1 --> F
E --> E2 --> F

F --> G

G -- Nie --> E
G -- Áno --> H --> I --> J --> K --> L --> M --> N --> O
