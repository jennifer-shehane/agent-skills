# Cypress — Debugging Flaky Tests

## Retry-ability

Queries (`cy.get()`, `.find()`, `cy.contains()`) and the assertions chained to them retry until they pass or time out (`defaultCommandTimeout`, 4s by default). Action commands (`.click()`, `.type()`) are not retried, but they wait for the element to become actionable (visible, not covered, not animating, not disabled) before acting.

```javascript
// ✅ The query retries until .loading is gone
cy.get('.loading').should('not.exist');
cy.get('.results').should('have.length.greaterThan', 0);

// ✅ cy.get() retries until .results exists, then .click() waits for it to be actionable
cy.get('.results').click();

// ❌ .then() ends the retry chain. The queries after it retry against the $list
// it yielded, so if the list re-renders, $list is stale and the test times out.
cy.get('.results').then(($list) => {
  cy.wrap($list).find('li').first().should('contain', 'Item 1');
});

// ✅ Keep the whole chain as queries so it re-runs from the top on each retry
cy.get('.results li').first().should('contain', 'Item 1');
```

## Common Causes

1. **Race conditions with API calls**
```javascript
// ❌ Bad
cy.visit('/dashboard');
cy.get('.data-table'); // might not be loaded yet

// ✅ Good
cy.intercept('GET', '/api/data').as('getData');
cy.visit('/dashboard');
cy.wait('@getData');
cy.get('.data-table').should('exist');
```

2. **Detached from DOM**
```javascript
// ❌ Element re-rendered by React
cy.get('.item').then($el => {
  // ... page re-renders ...
  cy.wrap($el).click(); // detached!
});

// ✅ Re-query after action
cy.get('.item').first().click();
cy.get('.item').first().should('have.class', 'selected');
```

3. **Animation interference**

Cypress already waits for an element to stop animating before it acts on it (`waitForAnimations: true`). If animations still cause flakes, tune the threshold or turn CSS animations off in the app under test:

```javascript
// cypress.config.js
module.exports = defineConfig({
  e2e: {
    // Pixels an element may move between checks and still count as "not animating" (default 5)
    animationDistanceThreshold: 20,
  },
});
```

```javascript
// cypress/support/e2e.js — disable CSS transitions and animations in the app
Cypress.on('window:before:load', (win) => {
  win.addEventListener('DOMContentLoaded', () => {
    const style = win.document.createElement('style');
    style.innerHTML = '*, *::before, *::after { transition: none !important; animation: none !important; }';
    win.document.head.appendChild(style);
  });
});
```

## Debug Tools

```bash
npx cypress open              # Interactive mode with time travel
DEBUG=cypress:* npx cypress run  # Verbose logging
```

```javascript
// In test
cy.pause();  // Pause test execution
cy.debug();  // Open DevTools debugger
```
