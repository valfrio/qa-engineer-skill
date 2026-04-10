# Integración CI/CD — Workflow de QA Post-Deploy

## Visión general

Esta referencia cubre cómo integrar los Playwright QA agents en tu pipeline CI/CD para que se ejecuten automáticamente después de cada deploy.

> **Nota:** CI/CD es **opcional**. La mayoría de equipos ejecutan los tests en local contra el servidor desplegado, sin pipeline. Ver la sección "Ejecutar tests en local contra un servidor remoto" en el `README.md` raíz. Esta referencia es para los equipos que sí quieran un pipeline.

---

## Workflow de GitHub Actions

### Básico: Ejecutar Tests Después del Deploy

```yaml
# .github/workflows/qa-post-deploy.yml
name: QA — Post-Deploy E2E Tests

on:
  # Disparar después de que el workflow de deploy complete
  workflow_run:
    workflows: ["Deploy"]
    types: [completed]
    branches: [main, staging]
  
  # O disparar manualmente
  workflow_dispatch:
    inputs:
      base_url:
        description: 'URL contra la que testear'
        required: false
        default: 'https://staging.tuapp.com'

env:
  BASE_URL: ${{ github.event.inputs.base_url || 'https://staging.tuapp.com' }}

jobs:
  e2e-tests:
    name: E2E Tests
    runs-on: ubuntu-latest
    timeout-minutes: 30
    
    # Solo ejecutar si el deploy fue exitoso
    if: ${{ github.event.workflow_run.conclusion == 'success' || github.event_name == 'workflow_dispatch' }}
    
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'
      
      - name: Instalar dependencias
        run: npm ci
      
      - name: Instalar Playwright Browsers
        run: npx playwright install --with-deps chromium
      
      - name: Ejecutar Playwright Tests
        run: npx playwright test
        env:
          BASE_URL: ${{ env.BASE_URL }}
          TEST_USER_EMAIL: ${{ secrets.TEST_USER_EMAIL }}
          TEST_USER_PASSWORD: ${{ secrets.TEST_USER_PASSWORD }}
      
      - name: Subir Test Report
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: playwright-report-${{ github.run_id }}
          path: playwright-report/
          retention-days: 30
      
      - name: Subir Test Results
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: test-results-${{ github.run_id }}
          path: test-results/
          retention-days: 14
```

### Avanzado: Loop Planner + Generator + Healer en CI

```yaml
# .github/workflows/qa-agents-loop.yml
name: QA — AI Agent Testing Loop

on:
  workflow_dispatch:
    inputs:
      feature_area:
        description: 'Área de feature a testear (ej: checkout, auth, dashboard)'
        required: true
      base_url:
        description: 'URL contra la que testear'
        required: false
        default: 'https://staging.tuapp.com'

jobs:
  agent-qa:
    name: AI Agent QA Loop
    runs-on: ubuntu-latest
    timeout-minutes: 60
    
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'
      
      - name: Instalar dependencias
        run: |
          npm ci
          npx playwright install --with-deps chromium
      
      # Ejecutar tests existentes primero (regresión)
      - name: Ejecutar Suite de Regresión
        run: npx playwright test
        env:
          BASE_URL: ${{ github.event.inputs.base_url }}
        continue-on-error: true
      
      # Generar tests nuevos con agentes si existen specs
      - name: Comprobar specs
        id: check-specs
        run: |
          if [ -d "specs" ] && [ "$(ls -A specs/ 2>/dev/null)" ]; then
            echo "has_specs=true" >> $GITHUB_OUTPUT
          else
            echo "has_specs=false" >> $GITHUB_OUTPUT
          fi
      
      # Ejecutar la suite completa de tests
      - name: Ejecutar Suite E2E Completa
        run: npx playwright test --reporter=json,html
        env:
          BASE_URL: ${{ github.event.inputs.base_url }}
      
      - name: Subir Reports
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: qa-agent-report-${{ github.run_id }}
          path: |
            playwright-report/
            test-results/
            specs/
          retention-days: 30
```

---

## Script Post-Deploy (Local / Staging)

Script para ejecutar el loop completo de QA en local o contra staging:

```bash
#!/bin/bash
# scripts/run-qa.sh

set -e

BASE_URL=${1:-"http://localhost:3000"}
FEATURE=${2:-"all"}

echo "🧪 Ejecutando QA contra: $BASE_URL"
echo "📋 Feature scope: $FEATURE"

# 1. Ejecutar suite de regresión
echo "━━━ Fase 1: Suite de Regresión ━━━"
BASE_URL=$BASE_URL npx playwright test --reporter=list 2>&1 || true

# 2. Comprobar specs nuevos/actualizados
echo "━━━ Fase 2: Comprobar Specs ━━━"
if [ -d "specs" ] && [ "$(ls -A specs/)" ]; then
  echo "Specs encontrados, generando tests..."
  # Los agentes serán invocados por Claude Code
else
  echo "No hay specs. Ejecuta el agente Planner primero."
fi

# 3. Ejecutar suite completa con traces
echo "━━━ Fase 3: Suite E2E Completa ━━━"
BASE_URL=$BASE_URL npx playwright test \
  --reporter=html,json \
  --trace=on \
  2>&1 || true

# 4. Generar report
echo "━━━ Fase 4: Report ━━━"
echo "📊 Report: playwright-report/index.html"
echo "🔍 Traces: test-results/"

# 5. Resumen
TOTAL=$(cat test-results/results.json 2>/dev/null | jq '.stats.expected + .stats.unexpected + .stats.skipped' || echo "0")
PASSED=$(cat test-results/results.json 2>/dev/null | jq '.stats.expected' || echo "0")
FAILED=$(cat test-results/results.json 2>/dev/null | jq '.stats.unexpected' || echo "0")

echo ""
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo "Total: $TOTAL | ✅ $PASSED | ❌ $FAILED"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━"
```

Hazlo ejecutable: `chmod +x scripts/run-qa.sh`

Uso:
```bash
# Local
./scripts/run-qa.sh http://localhost:3000

# Staging
./scripts/run-qa.sh https://staging.tuapp.com

# Feature específica
./scripts/run-qa.sh https://staging.tuapp.com checkout
```

---

## Workflow: Después de una Implementación Nueva

Cuando se implementa una feature nueva, este es el workflow orquestado por Claude:

### Paso 1: Generar Spec (Planner)

Prompt al agente Planner:
```
Use the Playwright Planner agent to explore [feature] at [URL]
and generate a comprehensive test plan covering:
- Happy paths
- Invalid data and validation
- Edge cases and boundary conditions  
- Error states
- Concurrent access
Save the plan to specs/[feature].md
```

### Paso 2: Generar Tests (Generator)

Prompt al agente Generator:
```
Use the Playwright Generator agent to create tests from 
specs/[feature].md. Use tests/seed.spec.ts as the seed test.
Save generated tests to tests/[feature]/
```

### Paso 3: Run & Heal

```bash
# Ejecuta los tests nuevos
npx playwright test tests/[feature]/

# Si hay fallos, invoca al Healer:
# "Use the Playwright Healer agent to fix failing tests in tests/[feature]/"
```

### Paso 4: Añadir a la Suite de Regresión

Una vez que los tests pasen, pasan a formar parte de la suite de regresión permanente que corre en cada deploy.

---

## Configuración de Docker (para CI)

```dockerfile
# Dockerfile.qa
FROM mcr.microsoft.com/playwright:v1.52.0-noble

WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .

CMD ["npx", "playwright", "test"]
```

```yaml
# docker-compose.qa.yml
services:
  qa:
    build:
      context: .
      dockerfile: Dockerfile.qa
    environment:
      - BASE_URL=http://app:3000
      - CI=true
    depends_on:
      - app
    volumes:
      - ./playwright-report:/app/playwright-report
      - ./test-results:/app/test-results
```

---

## Notificaciones (Opcional)

### Notificación a Slack en Fallo

Añade al workflow de GitHub Actions:

```yaml
- name: Notificar a Slack en Fallo
  if: failure()
  uses: 8398a7/action-slack@v3
  with:
    status: failure
    text: |
      🔴 QA Tests Fallidos tras el deploy
      Branch: ${{ github.ref_name }}
      Report: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}
    webhook_url: ${{ secrets.SLACK_WEBHOOK }}
```

---

## Gestión de Entornos

### Estrategia de Datos de Test

```typescript
// tests/helpers/test-data.ts

export function generateTestUser(overrides = {}) {
  const id = Date.now();
  return {
    name: `QA Test User ${id}`,
    email: `qa-test-${id}@test.com`,
    phone: `+1555${String(id).slice(-7)}`,
    ...overrides,
  };
}

export function generateTestOrder(overrides = {}) {
  const id = Date.now();
  return {
    customerEmail: `qa-test-${id}@test.com`,
    items: [{ sku: 'TEST-SKU-001', quantity: 1, price: 19.99 }],
    shippingAddress: '123 Test St, Test City, 00000',
    ...overrides,
  };
}

export function getFutureDate(days: number): string {
  const date = new Date();
  date.setDate(date.getDate() + days);
  return date.toISOString().split('T')[0];
}
```

### Cleanup Después de Tests

```typescript
// tests/global-teardown.ts
import { request } from '@playwright/test';

export default async function globalTeardown() {
  const api = await request.newContext({
    baseURL: process.env.BASE_URL,
  });
  
  // Limpiar datos de test
  await api.delete('/api/test/cleanup', {
    headers: { Authorization: `Bearer ${process.env.API_KEY}` },
  });
  
  await api.dispose();
}
```

Añade a `playwright.config.ts`:
```typescript
export default defineConfig({
  globalTeardown: './tests/global-teardown.ts',
  // ...
});
```

---

## Monitorizar la Salud de los Tests

### Detectar Tests Flaky

```typescript
// playwright.config.ts
export default defineConfig({
  retries: process.env.CI ? 2 : 0,
  reporter: [
    ['html'],
    ['json', { outputFile: 'test-results/results.json' }],
    // Detectar tests flaky a lo largo del tiempo
    ['list', { printSteps: true }],
  ],
});
```

### Integración con Dashboard

Exporta los resultados de tests como JSON y aliméntalos a tu dashboard de monitorización. Métricas clave:
- Pass rate por área de feature
- Tasa de tests flaky (pasaron en retry)
- Tendencias de tiempo de ejecución de los tests
- Huecos de cobertura (features sin testear)
