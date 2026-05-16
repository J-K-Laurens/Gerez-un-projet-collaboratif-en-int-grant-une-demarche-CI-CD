# BobApp

## 🚀 Démarrage rapide

### Front-end

#### Local
```bash
cd front
npm install
npm run start
```

#### Docker
```bash
cd front
docker build -t bobapp-front .
docker run -p 8080:8080 --name bobapp-front -d bobapp-front
```

### Back-end

#### Local
```bash
cd back
mvn clean install
mvn spring-boot:run
```

#### Tests
```bash
mvn clean install
```

#### Docker
```bash
cd back
docker build -t bobapp-back .
docker run -p 8080:8080 --name bobapp-back -d bobapp-back
```

---

## 🔄 CI/CD Workflows

### 📋 Vue d'ensemble

Le projet utilise **3 workflows GitHub Actions** pour automatiser build, test, qualité et déploiement :

| Workflow | Déclencheur | Rôle |
|----------|-----------|------|
| **PR Validation** | Pull Request sur `main` | Valide le code avant merge |
| **Common Build** | Réutilisable (utilisée par autres) | Build & tests (backend + frontend) + SonarQube |
| **CI/CD Deploy** | Push sur `main` | Build Docker, pousse vers DockerHub, déploie |

### 📝 Fichiers YAML

#### 1️⃣ `.github/workflows/pr-validation.yml`
**Quand ?** Chaque PR ouverte/modifiée/réouverte sur `main`  
**Quoi ?** Appelle le workflow `common-build.yml`  
**Résultat** Rapport de qualité SonarQube visible dans la PR

#### 2️⃣ `.github/workflows/common-build.yml` (Réutilisable)
**Quand ?** Appelée par `pr-validation.yml` et `ci-cd-deploy.yml`  
**Jobs parallèles** :
- `build-test` : Maven (backend) + Node (frontend) + JaCoCo + tests
- `code-quality` : SonarQube scan complet avec uploads d'artefacts

#### 3️⃣ `.github/workflows/ci-cd-deploy.yml`
**Quand ?** Chaque push sur `main` (après merge d'une PR)  
**Jobs** :
- Appelle `common-build.yml` d'abord
- `build-docker` : Crée les images Docker pour backend et frontend
- `push-docker` : Pousse vers DockerHub **avec retry automatique**

### 🔁 Retry Automatique

**Où ?** Dans `push-docker`, étape "Load and push ${{ matrix.service }} image"

```yaml
uses: nick-invision/retry@v2
with:
  timeout_minutes: 6
  max_attempts: 3
```

**Pourquoi ?** 
- Gère les limites de débit de DockerHub (rate limits)
- Réessaye automatiquement en cas de problème réseau temporaire
- Jusqu'à **3 tentatives** pour réussir le push

### 🔀 Matrix Strategy

**Concept** : Exécute le même job avec des variables différentes en parallèle

**Exemple** (`build-docker` job) :
```yaml
strategy:
  matrix:
    include:
      - service: backend    # matrix.service = "backend"
        context: ./back     # matrix.context = "./back"
      - service: frontend   # matrix.service = "frontend"
        context: ./front    # matrix.context = "./front"
```

**Résultat** : 
- 2 jobs parallèles au lieu de 1
- Chacun construit sa propre image Docker
- Temps total ÷2 vs exécution séquentielle

**Dans `push-docker`** :
```yaml
matrix:
  service: [backend, frontend]
```
Crée aussi 2 jobs parallèles pour pousser chaque image

---

## 📊 Qualité du Code & KPI

### 🔍 Qu'est-ce que le Coverage (Couverture de tests) ?

Imaginez une maison avec 100 portes. Le **coverage** mesure le pourcentage de portes que vous avez testées pour vérifier qu'elles fonctionnent correctement. Actuellement, BobApp a une couverture globale de **30,2%** : seulement 30% du code a été testé automatiquement. 

- **Frontend** : 38% (assez bon) ✅
- **Backend** : 0% (aucun test automatisé) ❌

Le backend est critiquement dépourvu de tests. Cela signifie que chaque nouvelle fonctionnalité risque de créer des bugs invisibles. C'est comme construire les étages d'une maison sans vérifier que les fondations tiennent !

### 🚨 Issues Critiques & Blockers

Actuellement, le projet compte **1 issue CRITICAL** identifiée dans `JokeService.java` (problème de réutilisation de variable aléatoire). Aucun **Blocker** (problème arrêtant le déploiement) n'est détecté.

**L'issue CRITICAL** : C'est comme une fuite d'eau au premier étage — elle ne paralyse pas tout (donc pas de Blocker), mais elle dégade la qualité et peut causer des problèmes à long terme. Il faut la corriger avant de considérer l'application comme stable en production.

Les **6 issues Medium** et **5 issues Low** sont des améliorations à faire progressivement, comme repeindre ou petit travaux de maintenance.

### 🎯 Top 3 KPI Recommandés

| KPI | Objectif | Bénéfice Client |
|-----|----------|-----------------|
| **Coverage Global** | Passer de 30% à **80%** | Réduire les risques de bugs cachés, garantir la stabilité avant chaque déploiement |
| **Issues Critiques** | Réduire de 1 à **0** | Assurer que l'application n'a pas de défauts graves pouvant impacter les utilisateurs |
| **Sécurité** | Réduire les 2 Security Hotspots à **0** | Protéger les données utilisateurs et respecter les standards de sécurité |

**Pourquoi ces 3 KPI ?** Ils couvrent les trois piliers : **robustesse** (coverage), **stabilité** (zéro critique), et **sécurité** (hotspots). Atteindre ces objectifs = une application fiable en production. 