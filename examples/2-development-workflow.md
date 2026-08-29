name: Workflow 2 - Develop Quality Code
on:
  repository_dispatch:
    types: [develop]
jobs:
  vitestUnitTests:
    name: Vitest Unit Tests
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v6

      - name: Set up Node.js
        uses: actions/setup-node@v6
        with:
          node-version: '24.18.0'
          cache: 'npm'
      - run: npm ci
      - name: Verify dependency pins
        run: npm run verify:pins
      - name: Report unit tests
        run: npm run development:test:once

  lintErrorsReport:
    name: Lint Errors Report
    needs: vitestUnitTests
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v6

      - name: Set up Node.js
        uses: actions/setup-node@v6
        with:
          node-version: '24.18.0'
          cache: 'npm'
      - run: npm ci
      - name: Verify dependency pins
        run: npm run verify:pins
      - name: Run linting results
        run: npm run lint

  baseIntegrationTests:
    name: Base Integration Tests
    needs: lintErrorsReport
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - name: Set up Node.js
        uses: actions/setup-node@v6
        with:
          node-version: '24.18.0'
          cache: 'npm'
      - name: Verify dependency pins
        run: npm ci && npm run verify:pins
      - uses: cypress-io/github-action@v7
        with:
          install: npm ci
          build:  npm install -g @angular/cli
          start:  npm start
          config: baseUrl=http://localhost:4200
          wait-on: 'http://localhost:4200'
          config-file: cypress/support/configurations/integration/base-integration.config.ts
          wait-on-timeout: 180
        continue-on-error: false

      - run: echo ${{ github.event.client_payload.sha }}
      - name: Trigger Workflow 3 - Demo Functional Prototype
        uses: peter-evans/repository-dispatch@v4
        with:
         token: ${{ secrets.REPO_ACCESS_TOKEN }}
         repository: 1001daysofcode/app
         event-type: demo
         client-payload: '{"ref": "${{ github.ref }}", "sha": "${{ github.sha }}"}'