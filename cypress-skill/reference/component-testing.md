# Cypress — Component Testing

The `mount` functions for React and Vue ship inside the `cypress` package (`cypress/react`, `cypress/vue`). Don't install `@cypress/react` or `@cypress/vue`; those standalone packages exist only for projects stuck on old framework versions.

## React Setup

```bash
npm install cypress --save-dev
```

```javascript
// cypress.config.js
const { defineConfig } = require('cypress');

module.exports = defineConfig({
  component: {
    devServer: {
      framework: 'react',
      bundler: 'vite', // or 'webpack'
    },
  },
});
```

Register `cy.mount` once in the component support file so every spec can use it:

```javascript
// cypress/support/component.js
import { mount } from 'cypress/react';

Cypress.Commands.add('mount', mount);
```

## React Component Test

```javascript
import Button from './Button';

describe('Button', () => {
  it('renders with text', () => {
    cy.mount(<Button label="Click Me" />);
    cy.get('button').should('contain', 'Click Me');
  });

  it('calls onClick handler', () => {
    const onClick = cy.stub().as('click');
    cy.mount(<Button label="Click" onClick={onClick} />);
    cy.get('button').click();
    cy.get('@click').should('have.been.calledOnce');
  });
});
```

## Vue Component Test

```javascript
// cypress/support/component.js
import { mount } from 'cypress/vue';

Cypress.Commands.add('mount', mount);
```

```javascript
import Counter from './Counter.vue';

describe('Counter', () => {
  it('increments count', () => {
    cy.mount(Counter);
    cy.get('[data-cy="count"]').should('contain', '0');
    cy.get('[data-cy="increment"]').click();
    cy.get('[data-cy="count"]').should('contain', '1');
  });
});
```

## Run Component Tests

```bash
npx cypress run --component
npx cypress open --component  # interactive
```
