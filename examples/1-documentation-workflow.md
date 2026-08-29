name: Workflow 1 - Acceptance Criteria Documentation
on:
  push:
    branches:
      - master
jobs:
  angularBuildCompilation:
    name: ng build compilation
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v6

      - name: Set up Node.js
        uses: actions/setup-node@v6
        with:
          node-version: '24.18.0'
          cache: 'npm'
      - name: Install Angular CLI
        run: npm install -g @angular/cli
      - name: Verify dependency pins
        run: npm run verify:pins
      - run: npm ci
      - name: build angular project
        run: ng build

  firestoreRulesMochaTests:
    name: Firestore Security Rules Mocha Unit Tests
    needs: angularBuildCompilation
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v6

      - name: Set up Node.js
        uses: actions/setup-node@v6
        with:
          node-version: '24.18.0'
          cache: 'npm'
      - name: Set up Java 21
        uses: actions/setup-java@v5
        with:
          distribution: temurin
          java-version: '21'
      - name: Verify dependency pins
        run: npm run verify:pins
      - run: npm ci
      - run: npm install -g firebase-tools@15.19.1
      - run: npm run database:security

      - name: Trigger Workflow 2 - Develop Quality Code
        uses: peter-evans/repository-dispatch@v4
        with:
         token: ${{ secrets.REPO_ACCESS_TOKEN }}
         repository: 1001daysofcode/app
         event-type: develop
         client-payload: '{"ref": "${{ github.ref }}", "sha": "${{ github.sha }}"}'