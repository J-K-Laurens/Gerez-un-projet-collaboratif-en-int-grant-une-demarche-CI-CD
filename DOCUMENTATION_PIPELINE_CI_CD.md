# Documentation - Pipeline CI/CD BobApp

**Date:** Mai 2026  
**Version:** 1.0  
**Auteur:** Documentation Technique

---

## Table des matières

1. [Introduction](#introduction)
2. [Architecture de la Pipeline CI/CD](#architecture-de-la-pipeline-cicd)
3. [Étapes du Workflow Détaillées](#étapes-du-workflow-détaillées)
4. [KPIs et Seuils de Qualité](#kpis-et-seuils-de-qualité)
5. [Métriques Actuelles](#métriques-actuelles)
6. [Analyse et Retours Utilisateurs](#analyse-et-retours-utilisateurs)
7. [Problèmes Identifiés et Recommandations](#problèmes-identifiés-et-recommandations)

---

## Introduction

La pipeline CI/CD de BobApp automatise les processus de compilation, test, analyse de qualité et déploiement. Elle s'exécute sur chaque push vers la branche `main` et sur chaque pull request pour garantir la qualité du code et la fiabilité de l'application.

**Pile technologique:**
- **Backend:** Java 17 + Spring Boot + Maven
- **Frontend:** Angular 14 + Node 18 + npm
- **Infrastructure:** Docker + DockerHub
- **Analyse de qualité:** SonarQube Cloud
- **Couverture de tests:** JaCoCo (Java) + Karma/Istanbul (TypeScript)

---

## Architecture de la Pipeline CI/CD

### Vue d'ensemble

```
Trigger (push sur main ou PR)
    ↓
┌─────────────────────────────────────┐
│   Common Build & Quality            │
├─────────────────────────────────────┤
│ • Build & Test Backend              │
│ • Build & Test Frontend             │
│ • Analyse SonarQube                 │
└─────────────────────────────────────┘
    ↓
┌─────────────────────────────────────┐
│   Build Docker Images               │
├─────────────────────────────────────┤
│ • Backend image                     │
│ • Frontend image                    │
└─────────────────────────────────────┘
    ↓
┌─────────────────────────────────────┐
│   Push to DockerHub                 │
├─────────────────────────────────────┤
│ • Authentification DockerHub        │
│ • Push images avec tags             │
└─────────────────────────────────────┘
```

---

## Étapes du Workflow Détaillées

### 1️⃣ **Build & Test - Backend et Frontend**

**Objectif:** Compiler et tester tous les composants de l'application

**Détails techniques:**

| Étape | Commande | Durée estimée | Sortie |
|-------|----------|---------------|--------|
| Setup JDK 17 | `actions/setup-java@v3` | 20-30s | JVM configurée |
| Setup Node 18 | `actions/setup-node@v3` | 15-20s | npm configuré |
| Build & Test Backend | `mvn clean verify -f back/pom.xml` | 2-3 min | `back/target/classes/` |
| Build & Test Frontend | `npm ci && npm run build && npm test -- --code-coverage` | 2-3 min | `front/coverage/lcov.info` |

**Résultats générés:**
- ✅ Classes compilées: `back/target/classes/`
- ✅ Couverture JaCoCo: `back/target/site/jacoco/jacoco.xml`
- ✅ Couverture TypeScript: `front/coverage/lcov.info`
- ✅ Rapports de test: `back/target/surefire-reports/`, `front/coverage/`

**Upload des artefacts** pour réutilisation dans l'étape suivante (durée: 1 jour)

---

### 2️⃣ **Analyse de Qualité - SonarQube**

**Objectif:** Analyser le code source et évaluer la qualité, les vulnérabilités et la couverture

**Dépendance:** Attend la fin de "Build & Test"

**Processus:**

| Phase | Action | Entrées | Sorties |
|-------|--------|---------|---------|
| **Téléchargement** | Récupération des artefacts uploadés | `build-artifacts` | `back/target/`, `front/coverage/` |
| **Scanning** | Analyse SonarQube Cloud | Sources Java/TypeScript, binaires compilés | Rapports SonarQube |
| **Reporting** | Génération des métriques | Résultats du scan | Dashboard SonarQube Cloud |

**Paramètres SonarQube:**
```properties
-Dsonar.organization=j-k-laurens
-Dsonar.projectKey=J-K-Laurens_Gerez-un-projet-...
-Dsonar.host.url=https://sonarcloud.io
-Dsonar.java.binaries=back/target/classes
-Dsonar.java.sources=back/src/main/java
-Dsonar.coverage.jacoco.xmlReportPaths=back/target/site/jacoco/jacoco.xml
-Dsonar.javascript.lcov.reportPaths=front/coverage/lcov.info
```

---

### 3️⃣ **Build Docker Images**

**Objectif:** Créer les images Docker pour backend et frontend

**Dépendance:** Attend la fin de "Code Quality"

**Processus par service (Backend & Frontend):**

1. **Métadonnées Docker:** Génération des tags (branch, semver, SHA, latest)
2. **Build Image:** Construction avec Docker Buildx
3. **Export:** Sauvegarde de l'image en TAR
4. **Upload:** Artefacts téléchargeables pour inspection (durée: 1 jour)

**Images générées:**
- `$DOCKERHUB_USERNAME/bobapp-backend:latest`
- `$DOCKERHUB_USERNAME/bobapp-frontend:latest`
- Versions taguées par commit (SHA)

---

### 4️⃣ **Push Docker to DockerHub**

**Objectif:** Déployer les images Docker sur le registre public DockerHub

**Dépendance:** Attend la fin de "Build Docker Images"

**Processus par service:**

1. **Validation:** Vérification des credentials (DOCKERHUB_USERNAME, DOCKERHUB_TOKEN)
2. **Login:** Authentification auprès de DockerHub
3. **Load & Push:** Chargement de l'image TAR et push avec retry (max 3 tentatives)

**Résultat:**
- ✅ Images disponibles sur `docker.io/$USERNAME/bobapp-backend`
- ✅ Images disponibles sur `docker.io/$USERNAME/bobapp-frontend`

---

## KPIs et Seuils de Qualité

### KPI #1: Code Coverage Minimum ✅

**Définition:** Pourcentage minimum de lignes de code exécutées par les tests

| Métrique | Seuil | Outil | Récupération |
|----------|-------|-------|--------------|
| **Code Coverage Global** | **≥ 80%** | JaCoCo (backend) + Karma (frontend) | SonarQube Dashboard |
| Backend Coverage | ≥ 80% | JaCoCo | `back/target/site/jacoco/jacoco.xml` |
| Frontend Coverage | ≥ 75% | Istanbul/Karma | `front/coverage/lcov.info` |

**Action en cas de non-respect:**
- ⚠️ **Avertissement:** Coverage entre 70-80%
- ❌ **Blocage:** Coverage < 70% → Pull Request rejetée

**Justification:** Une couverture minimale garantit une meilleure détection des bugs et facilite la maintenance.

---

### KPI #2: Pas de Bloqueurs de Sécurité / Vulnérabilités Critiques

**Définition:** Nombre maximal de problèmes critiques détectés par SonarQube

| Métrique | Seuil | Outil | Récupération |
|----------|-------|-------|--------------|
| **Issues Bloquantes (BLOCKER)** | **0** | SonarQube Cloud | SonarQube Dashboard |
| Issues Critiques (CRITICAL) | ≤ 2 | SonarQube Cloud | SonarQube Dashboard |
| Code Smells | ≤ 50 | SonarQube Cloud | SonarQube Dashboard |

**Catégories SonarQube:**
- 🔴 **BLOCKER:** Défauts critiques (injections SQL, XSS, sécurité)
- 🟠 **CRITICAL:** Problèmes graves (null pointer, débordements)
- 🟡 **MAJOR:** Problèmes importants (complexité, duplication)
- 🔵 **MINOR:** Problèmes mineurs (style, conventions)

**Action en cas de non-respect:**
- ✅ **BLOCKER = 0:** Pipeline réussie
- ❌ **BLOCKER ≥ 1:** Pull Request rejetée
- ⚠️ **CRITICAL ≥ 3:** Alerte, mais pipeline continue

---

### KPI #3: Taux de Succès des Builds

**Définition:** Pourcentage de pipelines qui terminent avec succès

| Métrique | Cible | Seuil d'alerte |
|----------|-------|----------------|
| **Build Success Rate** | **≥ 95%** | < 90% |
| Temps moyen de build | ≤ 10 min | > 15 min |

---

## Métriques Actuelles

[À REMPLIR] Exécution de la pipeline et capture des métriques SonarQube

### État des Exécutions Récentes

```
┌─────────────────────────────────────────────────────────────┐
│ [À REMPLIR - CAPTURE D'ÉCRAN]                              │
│                                                              │
│ GitHub Actions: Historique des workflow runs               │
│ - Nombre de runs: [___]                                    │
│ - Taux de succès: [___]%                                   │
│ - Temps moyen: [___ min]                                   │
│                                                              │
│ Placer ici une capture d'écran de GitHub Actions           │
└─────────────────────────────────────────────────────────────┘
```

### Métriques SonarQube Cloud

```
┌─────────────────────────────────────────────────────────────┐
│ [À REMPLIR - CAPTURE D'ÉCRAN]                              │
│                                                              │
│ SonarQube Cloud Dashboard:                                  │
│ Projet: J-K-Laurens_Gerez-un-projet-collaboratif-...      │
│                                                              │
│ • Code Coverage Global: [_____]%                           │
│   - Backend (Java): [_____]%                               │
│   - Frontend (TypeScript): [_____]%                        │
│                                                              │
│ • Issues:                                                   │
│   - BLOCKER: [_]                                           │
│   - CRITICAL: [_]                                          │
│   - MAJOR: [_]                                             │
│   - MINOR: [_]                                             │
│                                                              │
│ • Code Smells: [__]                                        │
│ • Duplications: [_]%                                       │
│ • Complexité Cyclomatique Moyenne: [_]                     │
│                                                              │
│ Placer ici une capture d'écran du dashboard SonarQube     │
└─────────────────────────────────────────────────────────────┘
```

### Couverture de Tests Détaillée

| Composant | Coverage | Ligne | Tests | Status |
|-----------|----------|-------|-------|--------|
| Backend Services | [____]% | [____] | [__] | [À REMPLIR] |
| Backend Controllers | [____]% | [____] | [__] | [À REMPLIR] |
| Frontend Components | [____]% | [____] | [__] | [À REMPLIR] |
| Frontend Services | [____]% | [____] | [__] | [À REMPLIR] |

---

## Analyse et Retours Utilisateurs

### Notes et Avis des Développeurs

[À REMPLIR] Retours pertinents des utilisateurs/développeurs

```
📝 Notes:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

[À REMPLIR] Ajouter ici les retours des développeurs sur:
- Efficacité de la pipeline
- Temps d'exécution
- Qualité des rapports
- Problèmes rencontrés
- Améliorations suggérées

Exemple de format:
📌 "Le build est très rapide (8-10 min), ce qui est excellent"
⚠️ "Les tests du frontend sont parfois flaky"
💡 "On aimerait plus de détails sur les code smells"

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

### Tendances des Métriques

```
┌─────────────────────────────────────────────────────────────┐
│ [À REMPLIR - GRAPHIQUE/TABLEAU]                            │
│                                                              │
│ Evolution du Code Coverage sur les 10 derniers jours:      │
│ Date       │ Backend │ Frontend │ Status                   │
│ ─────────────────────────────────────────────────────      │
│ 28 avril   │ [____]% │ [____]%  │ [____]                  │
│ 29 avril   │ [____]% │ [____]%  │ [____]                  │
│ 30 avril   │ [____]% │ [____]%  │ [____]                  │
│ 01 mai     │ [____]% │ [____]%  │ [____]                  │
│ ...        │ ...     │ ...      │ ...                      │
│                                                              │
│ Tendance: [STABLE / CROISSANTE / DÉCROISSANTE]             │
└─────────────────────────────────────────────────────────────┘
```

### Analyse des Défaillances

```
┌─────────────────────────────────────────────────────────────┐
│ [À REMPLIR]                                                │
│                                                              │
│ Pipeline Failures:                                          │
│ - Nombre total: [_]                                        │
│ - Taux: [_]%                                               │
│                                                              │
│ Causes principales:                                         │
│ 1. [À REMPLIR - Cause/Erreur]  → [Nb] fois               │
│ 2. [À REMPLIR - Cause/Erreur]  → [Nb] fois               │
│ 3. [À REMPLIR - Cause/Erreur]  → [Nb] fois               │
│                                                              │
│ Solutions appliquées:                                       │
│ ✓ [À REMPLIR]                                             │
│ ✓ [À REMPLIR]                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Problèmes Identifiés et Recommandations

### 🔴 Problèmes Critiques

#### Problème 1: Absence de Rapports de Couverture

**Statut:** [À REMPLIR - RÉSOLU / EN COURS / NON RÉSOLU]

**Description:** SonarQube ne reçoit pas les fichiers de couverture JaCoCo et des rapports TypeScript.

**Cause Probable:** 
- Artefacts de build non trouvés lors du scan SonarQube
- Chemins incorrects pour les binaires Java compilés

**Impact:** 
- Coverage non mesuré pour le KPI #1
- Impossible d'évaluer la qualité du test

**Solution Recommandée:**
1. ✅ **APPLIQUÉE** - Ajouter les paramètres explicites au scan SonarQube:
   ```yaml
   -Dsonar.java.binaries=back/target/classes
   -Dsonar.coverage.jacoco.xmlReportPaths=back/target/site/jacoco/jacoco.xml
   ```
2. Vérifier que les artefacts sont téléchargés correctement dans le workflow
3. Ajouter une étape de vérification des fichiers avant le scan

**Date de résolution:** [À REMPLIR]

---

### 🟠 Problèmes Majeurs

#### Problème 2: [À REMPLIR - Problème identifié]

**Statut:** [À REMPLIR - RÉSOLU / EN COURS / NON RÉSOLU]

**Description:** [À REMPLIR]

**Cause:** [À REMPLIR]

**Impact:** [À REMPLIR]

**Solution:** [À REMPLIR]

---

### 🟡 Améliorations Suggérées

#### Recommandation 1: Augmenter la Couverture de Tests

**Priorité:** HAUTE

**Description:** 
Le code coverage doit être augmenté progressivement vers 85% minimum pour améliorer la qualité.

**Plan d'action:**
- [ ] Identifier les classes/fonctions non testées
- [ ] Ajouter des tests unitaires pour les chemins critiques
- [ ] Effectuer des reviews de couverture à chaque PR
- [ ] Mettre à jour le seuil KPI progressivement

---

#### Recommandation 2: Optimiser le Temps d'Exécution

**Priorité:** MOYENNE

**Description:**
La pipeline dure actuellement ~10-15 minutes. L'objectif est de la réduire à <10 minutes.

**Optimisations possibles:**
- [ ] Paralléliser les scans backend et frontend
- [ ] Mettre en cache les dépendances Maven/npm plus efficacement
- [ ] Réduire les timeouts de Docker
- [ ] Utiliser Docker layer caching

---

#### Recommandation 3: Mise en Place d'un Gateway de Qualité

**Priorité:** MOYENNE

**Description:**
Implémenter des "quality gates" automatiques qui bloquent les PR non conformes aux KPIs.

**Actions:**
- [ ] Configurer SonarQube quality gates
- [ ] Ajouter des vérifications de branche
- [ ] Notifier les développeurs en temps réel
- [ ] Dashboard de suivi des KPIs

---

### Prochaines Étapes

1. **Court terme (1-2 semaines):**
   - Corriger le problème #1 (Rapports de couverture) ✅
   - Ajouter des tests manquants
   - Valider que tous les KPIs sont mesurables

2. **Moyen terme (1 mois):**
   - Optimiser le temps d'exécution
   - Mettre en place le quality gate automatique
   - Former l'équipe aux bonnes pratiques

3. **Long terme (2-3 mois):**
   - Atteindre 85% de code coverage
   - 0 bloqueurs de sécurité
   - Temps de build < 8 minutes

---

## Conclusion

La pipeline CI/CD de BobApp est bien structurée avec des étapes claires de build, test, analyse et déploiement. Les KPIs établis garantissent une qualité de code minimale et une sécurité satisfaisante. 

**Prochaines actions prioritaires:**
1. Corriger le scan SonarQube pour recevoir les rapports de couverture
2. Augmenter la couverture de tests
3. Optimiser les temps d'exécution
4. Mettre en place l'automatisation des quality gates

---

**Document rédigé:** Mai 2026  
**Dernière mise à jour:** [À REMPLIR]  
**Responsable:** [À REMPLIR - Nom/Équipe]
