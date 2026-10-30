# Cypress — Advanced Implementation Playbook

## §1 — Production Configuration

```typescript
// cypress.config.ts
import { defineConfig } from 'cypress';

export default defineConfig({
  e2e: {
    baseUrl: process.env.CYPRESS_BASE_URL || 'http://localhost:3000',
    specPattern: 'cypress/e2e/**/*.cy.{js,ts}',
    supportFile: 'cypress/support/e2e.ts',
    viewportWidth: 1280,
    viewportHeight: 720,
    video: true,
    screenshotOnRunFailure: true,
    retries: { runMode: 2, openMode: 0 },
    defaultCommandTimeout: 10000,
    requestTimeout: 15000,
    responseTimeout: 30000,
    setupNodeEvents(on, config) {
      on('before:browser:launch', (browser, launchOptions) => {
        if (browser.name === 'chrome' && browser.isHeadless) {
          launchOptions.args.push('--disable-gpu');
        }
        return launchOptions;
      });
      // Load environment config. Merge `expose` key by key so values set
      // elsewhere (CLI, other plugins) aren't wiped out.
      const envName = config.expose.envName || 'staging';
      const envConfig = require(`./cypress/config/${envName}.json`);
      config.baseUrl = envConfig.baseUrl;
      config.expose = { ...config.expose, ...envConfig.expose };
      return config;
    },
  },
  // Public, non-secret values: read in specs with Cypress.expose('apiUrl').
  // Secrets (tokens, passwords) go in `env` and are read with cy.env().
  expose: {
    apiUrl: 'http://localhost:3001/api',
  },
  component: {
    devServer: { framework: 'react', bundler: 'vite' },
  },
});
```

```json
// cypress/config/staging.json
{
  "baseUrl": "https://staging.example.com",
  "expose": { "apiUrl": "https://staging-api.example.com" }
}
// cypress/config/production.json
{
  "baseUrl": "https://www.example.com",
  "expose": { "apiUrl": "https://api.example.com" }
}
// Run: npx cypress run --expose envName=production
```

## §2 — Auth with cy.session()

```typescript
// cypress/support/commands.ts
Cypress.Commands.add('login', (email: string, password: string) => {
  cy.session([email, password], () => {
    cy.visit('/login');
    cy.get('[data-testid="email"]').type(email);
    cy.get('[data-testid="password"]').type(password);
    cy.get('[data-testid="submit"]').click();
    cy.url().should('include', '/dashboard');
  }, {
    validate() {
      cy.getCookie('session').should('exist');
    },
  });
});

// Fast API login (for non-login tests)
Cypress.Commands.add('apiLogin', (email: string, password: string) => {
  cy.session(['api', email], () => {
    cy.request('POST', '/api/auth/login', { email, password }).then(resp => {
      window.localStorage.setItem('token', resp.body.token);
    });
  });
});
```

## §3 — Page Object / App Actions Pattern

```typescript
// cypress/pages/LoginPage.ts
export class LoginPage {
  visit() {
    cy.visit('/login');
    return this;
  }

  typeEmail(email: string) {
    cy.getByTestId('email').clear().type(email);
    return this;
  }

  typePassword(password: string) {
    cy.getByTestId('password').clear().type(password);
    return this;
  }

  submit() {
    cy.getByTestId('submit').click();
    return this;
  }

  loginAs(email: string, password: string) {
    this.typeEmail(email).typePassword(password).submit();
    return this;
  }

  assertError(message: string) {
    cy.getByTestId('error-message').should('contain', message);
    return this;
  }
}

// cypress/pages/index.ts — barrel export
export { LoginPage } from './LoginPage';
export { DashboardPage } from './DashboardPage';
export { ProductsPage } from './ProductsPage';

// Usage in test
import { LoginPage } from '../pages';

describe('Login', () => {
  const loginPage = new LoginPage();

  it('logs in with valid credentials', () => {
    loginPage.visit().loginAs('user@test.com', 'password123');
    cy.url().should('include', '/dashboard');
  });

  it('shows error for invalid credentials', () => {
    loginPage.visit().loginAs('bad@test.com', 'wrong');
    loginPage.assertError('Invalid credentials');
  });
});
```

## §4 — Network Interception Patterns

```typescript
// Mock API response
cy.intercept('GET', '/api/products', { fixture: 'products.json' }).as('getProducts');
cy.visit('/products');
cy.wait('@getProducts');
cy.get('[data-testid="product-card"]').should('have.length', 3);

// Error simulation
cy.intercept('GET', '/api/products', {
  statusCode: 500,
  body: { error: 'Server error' },
}).as('serverError');
cy.visit('/products');
cy.wait('@serverError');
cy.getByTestId('error-banner').should('contain', 'Something went wrong');

// Modify real response
cy.intercept('GET', '/api/products', (req) => {
  req.continue((res) => {
    res.body.push({ id: 999, name: 'Injected Product' });
    res.send();
  });
});

// Wait for real API and validate
cy.intercept('POST', '/api/orders').as('createOrder');
cy.getByTestId('checkout').click();
cy.wait('@createOrder').its('response.statusCode').should('eq', 201);
cy.wait('@createOrder').its('response.body.orderId').should('exist');

// Delay response (test loading states)
cy.intercept('GET', '/api/search*', (req) => {
  req.reply({ delay: 2000, fixture: 'search-results.json' });
});
cy.get('.loading-spinner').should('be.visible');
cy.get('.loading-spinner').should('not.exist');
```

## §5 — Component Testing

```typescript
// cypress/component/Button.cy.tsx
import Button from '../../src/components/Button';

describe('<Button />', () => {
  it('renders with text', () => {
    cy.mount(<Button label="Click me" />);
    cy.get('button').should('have.text', 'Click me');
  });

  it('calls onClick when clicked', () => {
    const onClick = cy.stub().as('onClick');
    cy.mount(<Button label="Submit" onClick={onClick} />);
    cy.get('button').click();
    cy.get('@onClick').should('have.been.calledOnce');
  });

  it('renders disabled state', () => {
    cy.mount(<Button label="Save" disabled />);
    cy.get('button').should('be.disabled');
  });

  it('applies variant styles', () => {
    cy.mount(<Button label="Danger" variant="danger" />);
    cy.get('button').should('have.class', 'btn-danger');
  });
});
```

## §6 — Custom TypeScript Commands

```typescript
// cypress/support/commands.ts
declare global {
  namespace Cypress {
    interface Chainable {
      login(email: string, password: string): Chainable<void>;
      apiLogin(email: string, password: string): Chainable<void>;
      getByTestId(id: string): Chainable<JQuery<HTMLElement>>;
      dragTo(target: string): Chainable<void>;
      shouldBeWithinViewport(): Chainable<void>;
    }
  }
}

Cypress.Commands.add('getByTestId', (id: string) => {
  return cy.get(`[data-testid="${id}"]`);
});

Cypress.Commands.add('dragTo', { prevSubject: 'element' }, (subject, target) => {
  cy.wrap(subject).trigger('dragstart');
  cy.get(target).trigger('drop');
  cy.wrap(subject).trigger('dragend');
});

Cypress.Commands.add('shouldBeWithinViewport', { prevSubject: true }, (subject) => {
  const rect = subject[0].getBoundingClientRect();
  expect(rect.top).to.be.greaterThan(0);
  expect(rect.left).to.be.greaterThan(0);
  expect(rect.bottom).to.be.lessThan(Cypress.config('viewportHeight'));
  expect(rect.right).to.be.lessThan(Cypress.config('viewportWidth'));
  return cy.wrap(subject);
});
```

## §7 — Database Reset & Seeding

```typescript
// cypress/support/e2e.ts
beforeEach(() => {
  cy.request('POST', '/api/test/reset');      // reset DB
  cy.request('POST', '/api/test/seed');        // seed test data
});

// Or via Cypress task (Node.js level)
// cypress.config.ts
setupNodeEvents(on) {
  on('task', {
    async resetDB() {
      // `migrate reset --force` drops every table in DATABASE_URL. Refuse to
      // run unless it points at a database whose name marks it as a test DB.
      const dbName = new URL(process.env.DATABASE_URL).pathname;
      if (!/test/i.test(dbName)) {
        throw new Error(`Refusing to reset non-test database: ${dbName}`);
      }
      const { execSync } = require('child_process');
      execSync('npx prisma migrate reset --force');
      return null;
    },
    async seedDB(fixture: string) {
      const data = require(`./cypress/fixtures/${fixture}`);
      // insert into DB...
      return null;
    },
  });
}

// Usage: cy.task('resetDB');
```

## §8 — Time & Clock Control

```typescript
cy.clock(new Date('2024-12-25T00:00:00').getTime());
cy.visit('/promotions');
cy.get('.holiday-banner').should('be.visible');

cy.clock();
cy.getByTestId('start-timer').click();
cy.tick(60000); // advance 1 minute
cy.getByTestId('timer').should('contain', '1:00');
```

## §9 — File Operations

```typescript
// Upload
cy.get('input[type=file]').selectFile('cypress/fixtures/test.pdf');
cy.get('input[type=file]').selectFile('cypress/fixtures/test.pdf', { action: 'drag-drop' });

// Multiple files
cy.get('input[type=file]').selectFile(['file1.pdf', 'file2.pdf']);

// Download verification
cy.getByTestId('download-btn').click();
cy.readFile('cypress/downloads/report.pdf').should('exist');
```

## §10 — iframe & Shadow DOM

```typescript
// iframe access
cy.get('iframe#payment').its('0.contentDocument.body').should('not.be.empty')
  .then(cy.wrap)
  .find('#card-number')
  .type('4111111111111111');

// Shadow DOM
cy.get('my-web-component')
  .shadow()
  .find('.inner-button')
  .click();
```

## §11 — Accessibility Testing

```typescript
// Install: npm install -D cypress-axe axe-core
// cypress/support/e2e.ts: import 'cypress-axe';
// cypress.config.ts, inside setupNodeEvents (used by the last test below):
//   on('task', { log(message) { console.log(message); return null; } });

describe('Accessibility', () => {
  beforeEach(() => {
    cy.visit('/');
    cy.injectAxe();
  });

  it('homepage has no a11y violations', () => {
    cy.checkA11y(null, {
      runOnly: { type: 'tag', values: ['wcag2a', 'wcag2aa'] },
    });
  });

  it('form elements are accessible', () => {
    cy.checkA11y('#contact-form', {
      rules: { 'color-contrast': { enabled: true } },
    });
  });

  it('logs violations without failing', () => {
    // The 4th argument (skipFailures) keeps violations from failing the test
    cy.checkA11y(null, null, (violations) => {
      cy.task('log', `${violations.length} accessibility violation(s)`);
      violations.forEach(v => cy.task('log', `  - ${v.id}: ${v.description}`));
    }, true);
  });
});
```

## §12 — Visual Regression (with Percy or cypress-image-snapshot)

### Percy

```bash
npm install -D @percy/cypress
```

```typescript
// cypress/support/e2e.ts
import '@percy/cypress';
// Put PERCY_* options under `expose` in the config, not `env`.
```

```typescript
// cypress/e2e/visual.cy.ts
it('homepage visual snapshot', () => {
  cy.visit('/');
  cy.percySnapshot('Homepage');
});
```

### @simonsmith/cypress-image-snapshot

Use `@simonsmith/cypress-image-snapshot`, the maintained fork. The original `cypress-image-snapshot` package is abandoned.

```bash
npm install -D @simonsmith/cypress-image-snapshot
```

```typescript
// cypress.config.ts
import { defineConfig } from 'cypress';
import { addMatchImageSnapshotPlugin } from '@simonsmith/cypress-image-snapshot/plugin';

export default defineConfig({
  e2e: {
    setupNodeEvents(on) {
      addMatchImageSnapshotPlugin(on);
    },
  },
});
```

```typescript
// cypress/support/e2e.ts
import { addMatchImageSnapshotCommand } from '@simonsmith/cypress-image-snapshot/command';

addMatchImageSnapshotCommand();
```

```json
// tsconfig.json — types cy.matchImageSnapshot() in specs; needed when the tsconfig doesn't include the support file
{
  "compilerOptions": {
    "types": ["cypress", "@simonsmith/cypress-image-snapshot/types"]
  }
}
```

```typescript
// cypress/e2e/visual.cy.ts
it('component visual test', () => {
  cy.visit('/components');
  cy.getByTestId('product-card').first().matchImageSnapshot('product-card', {
    failureThreshold: 0.01,
    failureThresholdType: 'percent',
  });
});
```

The first run saves a baseline image to `cypress/snapshots/`. Commit it; later runs fail when the element differs from it by more than the threshold.

## §13 — CI/CD Integration

```yaml
name: Cypress Tests
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        browser: [chrome, firefox, edge]
    steps:
      - uses: actions/checkout@v4
      - uses: cypress-io/github-action@v7
        with:
          browser: ${{ matrix.browser }}
          start: npm start
          wait-on: 'http://localhost:3000'
          wait-on-timeout: 60
          record: true
          # One group per browser so the three recorded runs stay separate in Cypress Cloud
          group: ${{ matrix.browser }}
        env:
          CYPRESS_RECORD_KEY: ${{ secrets.CYPRESS_RECORD_KEY }}
      - uses: actions/upload-artifact@v4
        if: failure()
        with:
          name: cypress-${{ matrix.browser }}
          path: |
            cypress/screenshots
            cypress/videos

  # Parallel with Cypress Cloud
  parallel:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        containers: [1, 2, 3, 4]
    steps:
      - uses: actions/checkout@v4
      - uses: cypress-io/github-action@v7
        with:
          start: npm start
          wait-on: 'http://localhost:3000'
          record: true
          parallel: true
          group: 'CI Parallel'
        env:
          CYPRESS_RECORD_KEY: ${{ secrets.CYPRESS_RECORD_KEY }}
```

## §14 — Debugging Quick-Reference

| Problem | Cause | Fix |
|---------|-------|-----|
| `cy.get()` timeout | Element not in DOM yet | Use `should('be.visible')` assertion, check selectors |
| Detached from DOM | Element re-rendered mid-chain | Break chain, re-query after action |
| Cross-origin error | Visiting different superdomain | Use `cy.origin()` for cross-origin |
| `cy.intercept` not matching | Wrong URL pattern or method | Use DevTools Network tab to verify exact URL |
| File upload fails | Hidden input, custom upload widget | Use `selectFile({ action: 'drag-drop' })` on the drop zone; `force: true` only for an input the app hides on purpose |
| Flaky `cy.wait(ms)` | Hardcoded wait | Replace with `cy.wait('@alias')` or assertion |
| Session not restored | Validate fn failing | Check `cy.session` validate callback |
| `cy.click()` on hidden element | Element covered or `display:none` | Do what a user would: close the overlay, open the menu, or wait for the element to appear. `{ force: true }` skips actionability checks and can hide a real bug |
| Tests interfere with each other | Shared state/cookies | Use `cy.session()` and `beforeEach` reset |
| Component test mount fails | Missing providers | Wrap in context providers during `cy.mount` |
| Iframe not accessible | Cypress doesn't natively enter iframes | Use `.its('0.contentDocument.body')` pattern |

## §15 — Best Practices Checklist

- ✅ NEVER use `cy.wait(ms)` — use `cy.wait('@alias')` or assertions
- ✅ Login via API for non-login tests: `cy.request()` + `cy.session()`
- ✅ Use `data-testid` for selectors, avoid brittle CSS paths
- ✅ Use `cy.intercept()` to mock external APIs
- ✅ Use `retries: { runMode: 2 }` in CI, and track which tests only pass on retry; retries hide flakiness, they don't fix it
- ✅ Use `cy.session()` for auth caching across tests
- ✅ Keep shared or bulky test data in fixtures; short inline values are fine when they make the test easier to read
- ✅ Use `cy.clock()`/`cy.tick()` for time-dependent tests
- ✅ Avoid testing third-party sites/services
- ✅ Keep tests independent — no shared state
- ✅ Use environment configs for multi-env testing
- ✅ Use component testing for isolated UI unit tests
- ✅ Use `cypress-axe` for accessibility audits in pipeline
- ✅ Use TypeScript for custom commands with proper declarations
- ✅ Structure: `e2e/`, `component/`, `pages/`, `fixtures/`, `support/`
