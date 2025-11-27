# 1. SonarQube Integration

## Run SonarQube Locally (Docker)

```bash
docker run -d --name sonarqube -p 9000:9000 sonarqube:lts
```

## Login to SonarQube

* URL: [http://localhost:9000](http://localhost:9000)
* Default credentials: `admin / admin`

## Generate Project & Token

1. Create new project
2. Generate token
3. Store token as CI/CD secret (`SONAR_TOKEN`)

## Example `sonar-project.properties`

```properties
sonar.projectKey=web
sonar.projectName=web
sonar.sources=.
```

## Run Scan Manually

```bash
npm run sonar
```

---

# 1.1. SonarQube in GitHub Actions

```yaml
ame: CI

on:
  push:
    branches: ["main"]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 18

      - name: Install dependencies
        run: npm install

      - name: Run tests
        run: npm test --if-present

      - name: Run SonarQube Scanner
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
        run: |
          npm install -g sonar-scanner
          sonar-scanner
```

---

# 2. CI/CD

## 2.1 Required GitHub Secrets

These secrets must be added in GitHub under:
**Repository → Settings → Secrets and variables → Actions → Repository secrets**

### Repository Secrets Required

| Name               | Description                                                       |
| ------------------ | ----------------------------------------------------------------- |
| `ACR_LOGIN_SERVER` | Your ACR login server (example: `myprivateregistry15.azurecr.io`) |
| `ACR_USERNAME`     | ACR username (from `az acr credential show`)                      |
| `ACR_PASSWORD`     | ACR password (from `az acr credential show`)                      |
| `SONAR_HOST_URL`   | SonarQube server URL (example: `http://<public-ip>:9000`)         |
| `SONAR_TOKEN`      | Token generated in SonarQube for CI authentication                |

✅ These are **repository-level secrets**, not environment secrets.

---

```
ACR_LOGIN_SERVER
ACR_USERNAME
ACR_PASSWORD
SONAR_HOST_URL
SONAR_TOKEN
```

---

## 2.2 Jenkins Pipeline (Web Service)

### 🔐 Configure Azure Credentials in Jenkins

To enable AKS and ACR access, create a Service Principal credential in Jenkins.

#### **Jenkins UI Path:**

> **Manage Jenkins → Credentials → System → Global → Add Credentials**

#### **Credential Type:**

✅ **Azure Service Principal**

#### **Fields to Fill (replace with your values):**

| Field             | Value                      |
| ----------------- | -------------------------- |
| Subscription ID   | `<YOUR_SUBSCRIPTION_ID>`   |
| Client ID         | `<YOUR_CLIENT_ID>`         |
| Client Secret     | `<YOUR_CLIENT_SECRET>`     |
| Tenant ID         | `<YOUR_TENANT_ID>`         |
| Azure Environment | `Azure`                    |
| ID                | `AZURE_CREDENTIALS`        |
| Description       | `Verify Service Principal` |

⚠️ Do **NOT** store real credentials in the README or commit history.

✅ After saving, Jenkins Pipeline can reference it:

```groovy
withCredentials([azureServicePrincipal('AZURE_CREDENTIALS')]) {
  sh '''
    az login --service-principal \
      -u $AZURE_CLIENT_ID \
      -p $AZURE_CLIENT_SECRET \
      --tenant $AZURE_TENANT_ID
  '''
}
```

---
