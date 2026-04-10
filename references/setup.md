# Referencia de Setup — Playwright + Test Agents

## Prerrequisitos

- Node.js >= 18
- npm o yarn
- Una aplicación web corriendo (URL local o de staging)
- Credenciales de usuario de test (si la app está detrás de auth)

---

## Captura: Vídeo, Trace y Screenshots (LEE ESTO PRIMERO)

Antes de configurar nada, entiende qué captura Playwright en cada test run. Esta skill fuerza estos defaults — no los desactives.

### Tres artefactos, tres propósitos

| Artefacto | Archivo | Qué te da | Default en esta skill |
|---------|------|-------------------|----------------------|
| **Vídeo** | `video.webm` | Grabación continua del browser. Mira lo que el usuario "vio". | `on-first-retry` (solo en fallos) |
| **Trace** | `trace.zip` | Timeline interactivo: cada acción + snapshot del DOM + network + console log. **La herramienta principal de debugging.** | `on-first-retry` |
| **Screenshot** | `test-failed-1.png` | Imagen única en el momento del fallo. Fácil de incrustar en reports. | `only-on-failure` |

### Por qué los traces le ganan al vídeo plano

`expect-cli` y herramientas similares graban vídeo. Playwright graba vídeo **y** trace. El trace es dramáticamente más útil porque:

- **Click en cualquier paso del timeline → ves el DOM en ese momento exacto.** Inspeccionas elementos como estaban renderizados, no como están ahora.
- **El network log está marcado en tiempo contra las acciones.** Ves exactamente qué request se disparó en qué click.
- **El console log + errores está inline.** No tienes que rebuscar en DevTools después.
- **El trace es portable.** Un solo archivo `.zip`. Mándalo por Slack, adjúntalo a un bug report, súbelo como artefacto de CI. Cualquiera con `npx playwright show-trace trace.zip` puede reproducir tu sesión de debugging.

### Abrir un trace

```bash
# Test fallido más reciente
npx playwright show-trace test-results/*/trace.zip

# Archivo de trace específico
npx playwright show-trace path/to/trace.zip

# O abre el report HTML y haz click en cualquier test fallido → tab Trace
npx playwright show-report
```

El trace viewer se abre en un browser. Usa el timeline de abajo — cada acción es un paso clickeable.

### Subida de artefactos en CI

El workflow de GitHub Actions en `ci-cd.md` ya sube `test-results/` y `playwright-report/` como artefactos en cada run. Después de un fallo en CI, descarga el artefacto, descomprímelo y ejecuta `show-trace` localmente — misma experiencia de debugging que si hubieras corrido el test en tu máquina. Las sesiones futuras de Claude también pueden descargar estos artefactos y razonar sobre los fallos desde otra conversación.

### Cuándo sobreescribir los defaults

- **Debugging local de un test flaky:** pon temporalmente `trace: 'on'` y `video: 'on'` para capturar cada run, incluyendo los que pasan.
- **CI para suites de alto tráfico:** mantén los defaults — capturar solo fallos ahorra storage.
- **Nunca desactives el trace.** Es el artefacto más valioso que esta skill produce.

---

## Paso 1: Instala Playwright

```bash
# Inicializa Playwright en tu proyecto
npm init playwright@latest

# Esto va a:
# - Instalar @playwright/test
# - Crear playwright.config.ts
# - Crear un directorio tests/ con un ejemplo
# - Opcionalmente instalar los browsers
```

Cuando te pregunte:
- Lenguaje: **TypeScript**
- Carpeta de tests: **tests**
- GitHub Actions: **Yes** (si quieres CI)
- Instalar browsers: **Yes**

## Paso 2: Inicializa los Agentes

```bash
# Para Claude Code (recomendado para esta skill)
npx playwright init-agents --loop=claude

# Para VS Code Copilot
npx playwright init-agents --loop=vscode

# Para OpenCode
npx playwright init-agents --loop=opencode
```

Esto crea la carpeta `.github/` con las definiciones de los agentes (instrucciones + herramientas MCP). **Regenéralo cada vez que actualices Playwright.**

## Paso 3: Instala los Browsers

```bash
# Instala solo Chromium (más rápido, normalmente suficiente)
npx playwright install --with-deps chromium

# O instala todos los browsers
npx playwright install --with-deps
```

## Paso 4: Configura playwright.config.ts

```typescript
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './tests',
  
  /* Tiempo máximo por test */
  timeout: 30_000,
  
  /* Timeout de aserciones */
  expect: { timeout: 5_000 },
  
  /* Ejecutar tests en paralelo */
  fullyParallel: true,
  
  /* Hacer fallar el build en CI si quedó algún test.only en el código */
  forbidOnly: !!process.env.CI,
  
  /* Reintentar tests fallidos */
  retries: process.env.CI ? 2 : 0,
  
  /* Workers paralelos */
  workers: process.env.CI ? 1 : undefined,
  
  /* Reporter */
  reporter: [
    ['html', { open: 'never' }],
    ['json', { outputFile: 'test-results/results.json' }],
    ['list'],
  ],

  /* Settings compartidos para todos los proyectos */
  use: {
    baseURL: process.env.BASE_URL || 'http://localhost:3000',
    
    /* Capturar trace en fallos */
    trace: 'on-first-retry',
    
    /* Screenshot en fallos */
    screenshot: 'only-on-failure',
    
    /* Vídeo en fallos */
    video: 'on-first-retry',
    
    /* Defaults sensatos */
    actionTimeout: 10_000,
    navigationTimeout: 15_000,
  },

  projects: [
    /* Setup project para autenticación */
    {
      name: 'setup',
      testMatch: /.*\.setup\.ts/,
    },
    
    {
      name: 'chromium',
      use: { 
        ...devices['Desktop Chrome'],
        storageState: 'playwright/.auth/user.json',
      },
      dependencies: ['setup'],
    },

    /* Descomenta para multi-browser */
    // {
    //   name: 'firefox',
    //   use: { ...devices['Desktop Firefox'] },
    //   dependencies: ['setup'],
    // },
    
    /* Viewport mobile */
    // {
    //   name: 'mobile-chrome',
    //   use: { ...devices['Pixel 5'] },
    //   dependencies: ['setup'],
    // },
  ],

  /* Ejecutar el dev server local antes de empezar los tests */
  // webServer: {
  //   command: 'npm run dev',
  //   url: 'http://localhost:3000',
  //   reuseExistingServer: !process.env.CI,
  // },
});
```

## Paso 5: Setup de Autenticación

Crea `tests/auth.setup.ts` para apps que requieran login:

```typescript
import { test as setup, expect } from '@playwright/test';

const authFile = 'playwright/.auth/user.json';

setup('authenticate', async ({ page }) => {
  // Navega al login
  await page.goto('/login');
  
  // Rellena credenciales
  await page.getByLabel('Email').fill(process.env.TEST_USER_EMAIL || 'test@example.com');
  await page.getByLabel('Password').fill(process.env.TEST_USER_PASSWORD || 'testpassword');
  
  // Submit
  await page.getByRole('button', { name: 'Log in' }).click();
  
  // Espera a que la auth complete
  await page.waitForURL('/dashboard');
  
  // Guarda el estado autenticado
  await page.context().storageState({ path: authFile });
});
```

### Auth multi-rol (recomendado para cualquier app con permisos)

La mayoría de apps reales tienen al menos 2-3 roles (admin, user, guest). Configura un archivo de auth por rol y parametriza los tests:

```typescript
// tests/auth.setup.ts
import { test as setup } from '@playwright/test';

const roles = [
  { name: 'admin',        email: 'admin@test.com',        password: 'test1234' },
  { name: 'profesional',  email: 'profesional@test.com',  password: 'test1234' },
  { name: 'superadmin',   email: 'superadmin@test.com',   password: 'test1234' },
];

for (const role of roles) {
  setup(`authenticate as ${role.name}`, async ({ page }) => {
    await page.goto('/login');
    await page.getByLabel('Email').fill(role.email);
    await page.getByLabel('Password').fill(role.password);
    await page.getByRole('button', { name: /log in|iniciar sesión/i }).click();
    await page.waitForURL(/\/dashboard|\/superadmin/);
    await page.context().storageState({ path: `playwright/.auth/${role.name}.json` });
  });
}
```

Después en `playwright.config.ts`, define un proyecto por rol:

```typescript
projects: [
  { name: 'setup', testMatch: /.*\.setup\.ts/ },
  {
    name: 'admin',
    use: { ...devices['Desktop Chrome'], storageState: 'playwright/.auth/admin.json' },
    dependencies: ['setup'],
    testMatch: /.*\.admin\.spec\.ts/,
  },
  {
    name: 'profesional',
    use: { ...devices['Desktop Chrome'], storageState: 'playwright/.auth/profesional.json' },
    dependencies: ['setup'],
    testMatch: /.*\.profesional\.spec\.ts/,
  },
  // ... etc
],
```

### Estrategias de auth — elige la que coincida con tu stack

Tres patrones cubren ~95% de las apps reales. Usa locators semánticos (`getByLabel`, `getByRole`) para que el mismo código funcione independientemente del framework.

#### Patrón A — Cookie de sesión + CSRF (apps server-rendered)

Aplica a la mayoría de frameworks server-rendered: Laravel, Rails, Django, Symfony, ASP.NET MVC, Phoenix, Express + sessions, etc. El login envía un formulario, el servidor pone una cookie de sesión, las requests siguientes la llevan automáticamente.

```typescript
setup('authenticate (session + CSRF)', async ({ page }) => {
  // Empieza siempre limpio — los CSRF tokens stale son la causa #1 de auth flaky
  await page.context().clearCookies();

  await page.goto('/login');

  // Los locators semánticos funcionan en distintos i18n y variantes de framework
  await page.getByLabel(/email|user|correo/i).fill(process.env.TEST_USER_EMAIL!);
  await page.getByLabel(/password|contraseña|mot de passe/i).fill(process.env.TEST_USER_PASSWORD!);
  await page.getByRole('button', { name: /log in|sign in|iniciar sesión|entrar/i }).click();

  // Espera el redirect post-login — ajusta el regex a tu app
  await page.waitForURL(/\/dashboard|\/home|\/$/);

  // Sanity check: debería existir alguna cookie relacionada con sesión
  const cookies = await page.context().cookies();
  const hasSession = cookies.some(c =>
    /session|auth|jwt|sid/i.test(c.name) || /xsrf|csrf/i.test(c.name)
  );
  if (!hasSession) {
    throw new Error('Login appeared to succeed but no session cookie was set');
  }

  await page.context().storageState({ path: 'playwright/.auth/user.json' });
});
```

#### Patrón B — Token (SPA + REST API)

Aplica a SPAs de React/Vue/Angular que guardan un JWT o token de API en `localStorage` después del login. Más rápido que la auth por UI — llama directamente a la API.

```typescript
setup('authenticate (API token)', async ({ request, page }) => {
  const response = await request.post('/api/auth/login', {
    data: {
      email: process.env.TEST_USER_EMAIL,
      password: process.env.TEST_USER_PASSWORD,
    },
  });
  if (!response.ok()) throw new Error(`Login failed: ${response.status()}`);

  const { token } = await response.json();

  await page.goto('/');
  await page.evaluate(t => localStorage.setItem('auth_token', t), token);

  await page.context().storageState({ path: 'playwright/.auth/user.json' });
});
```

#### Patrón C — OAuth / SSO (proveedor de identidad de terceros)

Aplica a apps que delegan auth a Google, Microsoft, Auth0, Okta, etc. Tres opciones ordenadas por fiabilidad:

1. **Mejor:** salta el flujo de OAuth completamente con un endpoint de login solo-para-test que emite una sesión para un usuario de test. Añade `/test-login?user=qa@test.com` protegido por `APP_ENV === 'testing'`.
2. **OK:** usa el modo de test del proveedor si lo tiene (Auth0 lo tiene).
3. **Último recurso:** automatiza la UI de login del proveedor. Frágil — los proveedores cambian su HTML a menudo.

### Fallos comunes de auth (framework-agnósticos)

| Síntoma | Causa | Fix |
|---------|-------|-----|
| `waitForURL` da timeout | El login falló en silencio — credenciales mal | Revisa los valores del `.env`; intenta loguearte manualmente primero |
| `419` / `403` a mitad de test | El CSRF token expiró o la sesión se reusó entre runs | `clearCookies()` al inicio de `auth.setup.ts` |
| `401 Unauthenticated` después de que el auth setup pase | El path de `storageState` no coincide entre setup y la config del proyecto | Usa exactamente el mismo string de path en ambos sitios |
| Auth funciona en local, falla en CI | `BASE_URL` o dominio de cookie distinto | Pon la env `BASE_URL` en CI a la URL de la app desplegada |
| La sesión expira a mitad de test | Test largo excedió el lifetime de sesión del servidor | Acorta el test, o extiende el lifetime de sesión en el entorno de test |
| Pasa/falla flaky cada otra run | Dos workers paralelos pegando a la misma cuenta de usuario | Dale a cada worker su propio usuario de test, o ejecuta los tests dependientes de auth con `workers: 1` |

### Auth multi-tenant (basada en subdominio o path)

Si tu app es multi-tenant, el usuario de test debe pertenecer a un tenant específico. O bien:

- Pre-seedea tenants de test y usa un archivo de auth por tenant, O
- Usa un solo tenant para tests y aísla los datos de test con un prefijo `test_`.

Documenta tu elección en el seeder de test para que los devs futuros (y Claude) sepan a qué tenant están pegando.

## Paso 6: Fixtures Custom

Crea `tests/fixtures.ts`:

```typescript
import { test as base, expect } from '@playwright/test';

// Extiende el test base con fixtures custom
export const test = base.extend<{
  // Añade fixtures custom aquí
  authenticatedPage: any;
}>({
  // Ejemplo: una página que ya está en la app
  authenticatedPage: async ({ page }, use) => {
    await page.goto('/');
    await use(page);
  },
});

export { expect };
```

## Paso 7: Seed Test

Crea `tests/seed.spec.ts`:

```typescript
import { test, expect } from './fixtures';

test('seed', async ({ page }) => {
  // Navega a la app
  await page.goto('/');
  
  // Verifica que la app cargó correctamente
  await expect(page).toHaveTitle(/.+/);
  
  // Este seed test sirve como:
  // 1. Bootstrap para los agentes Planner/Generator
  // 2. Ejemplo de estructura y convenciones de test
  // 3. Validación del entorno
});
```

## Paso 8: Variables de Entorno

Crea `.env` para testing local:

```bash
# .env
BASE_URL=http://localhost:3000
TEST_USER_EMAIL=qa@example.com
TEST_USER_PASSWORD=secure_test_password

# Para testing de API
API_BASE_URL=http://localhost:3000/api
API_KEY=test_api_key
```

Añade al `.gitignore`:
```
playwright/.auth/
test-results/
playwright-report/
.env
```

## Paso 9: Verifica el Setup

```bash
# Ejecuta el seed test para verificar que todo funciona
npx playwright test seed.spec.ts

# Abre el report HTML
npx playwright show-report

# Mira los traces para debugging
npx playwright show-trace test-results/*/trace.zip
```

## Paso 10: Directorio de Specs

```bash
mkdir -p specs
```

Aquí es donde el agente Planner guardará los planes de test como archivos Markdown.

---

## Estructura del Proyecto Después del Setup

```
project/
├── .github/                      # Definiciones de agentes (auto-generadas)
│   ├── playwright-planner.md
│   ├── playwright-generator.md
│   └── playwright-healer.md
├── specs/                        # Planes de test (output del Planner)
├── tests/
│   ├── fixtures.ts               # Fixtures custom
│   ├── seed.spec.ts              # Seed test
│   └── auth.setup.ts             # Setup de auth
├── playwright/
│   └── .auth/                    # Estado de auth guardado
├── playwright.config.ts
├── .env
└── package.json
```

---

## Comandos Útiles

```bash
# Ejecutar todos los tests
npx playwright test

# Ejecutar un archivo de test específico
npx playwright test tests/checkout/place-order.spec.ts

# Ejecutar tests que coincidan con un patrón
npx playwright test --grep "place order"

# Ejecutar en modo headed (ver el browser)
npx playwright test --headed

# Ejecutar en modo debug
npx playwright test --debug

# Ejecutar en modo UI (interactivo)
npx playwright test --ui

# Generar test vía codegen
npx playwright codegen http://localhost:3000

# Mostrar el último report
npx playwright show-report

# Ver un trace
npx playwright show-trace path/to/trace.zip
```
