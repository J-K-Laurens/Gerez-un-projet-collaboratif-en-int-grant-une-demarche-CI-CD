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
docker run -p 80:80 --name bobapp-front -d bobapp-front
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

### 🐳 Docker Compose (Recommandé)

Lancer backend + frontend ensemble :
```bash
docker-compose up
```

**Ports** :
- Backend : `http://localhost:8080`
- Frontend : `http://localhost:80` (ou `http://localhost`)

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
- `build-test` : 
  - 🔍 Lint (ESLint frontend + Checkstyle backend)
  - Maven (backend) + Node (frontend) tests
  - JaCoCo coverage reports
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

## 📊 Quality Gates

Pour garantir la qualité et la stabilité de l'application, le projet respecte les seuils minimums suivants :

| KPI | Minimum Requis | Mesure | Responsabilité |
|-----|---|--------|-----------------|
| **Coverage Global** | ≥ 80% | JaCoCo (backend) + Karma (frontend) | À chaque PR |
| **Issues Critiques** | 0 | SonarQube scan | À corriger en urgence |
| **Security Hotspots** | 0 | SonarQube scan | À corriger avant production |

**Politique** :
- ✅ Chaque PR doit maintenir ou améliorer ces métriques
- ✅ Aucune régression tolérée
- ✅ Code review obligatoire avant merge

---

## 🔧 Outils de Linting

### Frontend
```bash
npm run lint   # ESLint + TSLint (Angular default)
```

### Backend
```bash
mvn checkstyle:check   # Checkstyle for Java code style
```

Ces outils sont exécutés automatiquement dans la pipeline CI/CD avant les tests. 