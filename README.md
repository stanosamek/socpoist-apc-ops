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
#### 8. Vykonanie inštalácie/upgrade
#### 9. Vygenerovať a odoslať po inštalácií Post report



# 📌 Pravidlá

1 operácia = 1 folder
1 operácia = 1 Jira ticket
metadata.yaml je povinný
changelog.md je zdroj pre klientsku komunikáciu
interný checklist sa nikdy neposiela klientovi


# 🔁 Proces operácie v cluster-operations repozitári

```mermaid
flowchart TD

A[Vytvorenie novej operácie<br/>templates → operations/] --> B[Vyplnenie metadata.yaml]

B --> C[Príprava dokumentácie<br/>plan.md / changelog.md / checklist.md]

C --> D[Vytvorenie / prepojenie Jira ticketu]

D --> E[Pull Request review]

E --> F{Schválené?}

F -- Nie --> C

F -- Áno --> G[Vykonanie operácie<br/>manuálne alebo CI/CD]

G --> H[Aktualizácia checklist.md]

H --> I[Vyplnenie post-report.md]

I --> J[GitHub Actions<br/>generovanie emailu]

J --> K[Klientská komunikácia<br/>email plan / report]

K --> L[Uzavretie Jira ticketu]

L --> M[Archivácia operácie]
