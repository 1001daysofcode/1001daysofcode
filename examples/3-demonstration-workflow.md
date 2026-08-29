name: Workflow 3 - Demo Functional Prototype

on:
  repository_dispatch:
    types: [demo]

jobs:
  emulationTests:
    name: Firebase Emulation Tests
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

      - name: Install Angular CLI
        run: npm install -g @angular/cli

      - name: Install dependencies
        run: npm ci

      - name: Verify dependency pins
        run: npm run verify:pins

      - name: Install functions dependencies
        run: cd functions && npm ci

      - name: Build app for emulators
        run: npx ng build

      - name: Build functions
        run: cd functions && npm run build

      - name: Install Firebase CLI
        run: npm install -g firebase-tools@15.19.1

      - name: Log emulation test stages
        run: |
          echo "[emulation] Starting alpha emulator suite"
          echo "[emulation] Stage order: health-check -> vitest emulated tests -> role registration -> cypress emulation"

      - name: Run Firebase Functions functions and hosting emulation tests
        run: npm run alpha:emulation:test:once

      - name: Log emulation test completion
        if: ${{ success() }}
        run: echo "[emulation] Alpha emulator suite completed successfully"

  alphaHostingDeployment:
    name: Firebase Deployment to Alpha Website
    needs: emulationTests
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v6

      - name: Set up Node.js
        uses: actions/setup-node@v6
        with:
          node-version: '24.18.0'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Verify dependency pins
        run: npm run verify:pins

      - name: Validate callable CORS coverage
        run: npm run validate:callable:cors

      - name: Build Alpha
        run: npx ng build --configuration alpha

      - name: Authenticate to Google Cloud
        id: auth
        uses: google-github-actions/auth@v3
        with:
          credentials_json: ${{ secrets.ALPHA_GCP_SA_KEY }}
          create_credentials_file: true
          export_environment_variables: true

      - name: Setup Cloud SDK
        uses: google-github-actions/setup-gcloud@v3
        with:
          project_id: troydcthompson-alpha

      - name: Install Firebase CLI
        run: npm install -g firebase-tools@15.19.1

      - name: Deploy Firebase Hosting
        env:
          GOOGLE_APPLICATION_CREDENTIALS: ${{ steps.auth.outputs.credentials_file_path }}
          GCLOUD_PROJECT: troydcthompson-alpha
          GOOGLE_CLOUD_PROJECT: troydcthompson-alpha
          PROJECT_ID: troydcthompson-alpha
          HOSTING_SITE_ID: troydcthompson-alpha
        run: node scripts/deploy-hosting-with-retry.mjs --project "$PROJECT_ID" --site "$HOSTING_SITE_ID" --attempts 3 --initial-delay-ms 2500 --command "firebase deploy --only hosting --project troydcthompson-alpha --non-interactive --debug"

      - name: Hosting failure diagnostics
        if: ${{ failure() }}
        env:
          GOOGLE_APPLICATION_CREDENTIALS: ${{ steps.auth.outputs.credentials_file_path }}
          GCLOUD_PROJECT: troydcthompson-alpha
          GOOGLE_CLOUD_PROJECT: troydcthompson-alpha
          PROJECT_ID: troydcthompson-alpha
          HOSTING_SITE_ID: troydcthompson-alpha
        run: node scripts/verify-hosting-site.mjs --project "$PROJECT_ID" --site "$HOSTING_SITE_ID" --attempts 1 --diagnostic-no-fail

      - name: Deploy Firebase Functions, Firestore Rules, Firestore Indexes, and Storage
        env:
          GOOGLE_APPLICATION_CREDENTIALS: ${{ steps.auth.outputs.credentials_file_path }}
          GCLOUD_PROJECT: troydcthompson-alpha
          GOOGLE_CLOUD_PROJECT: troydcthompson-alpha
        run: firebase deploy --only functions,firestore:rules,firestore:indexes,storage --project troydcthompson-alpha --non-interactive --force --debug

      - name: Ensure callable Cloud Run invoker is public
        run: |
          set -euo pipefail

          PROJECT_ID="troydcthompson-alpha"
          REGION="us-central1"
          SERVICES=(
            "bootstrapauthenticateduser"
            "createsignuproletoken"
            "redeemsignuproletoken"
          )

          for service in "${SERVICES[@]}"; do
            # Bind only if allUsers is not already an invoker. add-iam-policy-binding
            # is idempotent but each call costs ~5s of API round-trip.
            if gcloud run services get-iam-policy "${service}" \
                --region "${REGION}" \
                --project "${PROJECT_ID}" \
                --format="value(bindings.members)" \
                --flatten="bindings[].members" \
                --filter="bindings.role=roles/run.invoker AND bindings.members=allUsers" \
                | grep -q "allUsers"; then
              echo "Skip ${service}: allUsers already has roles/run.invoker"
              continue
            fi

            echo "Binding roles/run.invoker for allUsers on ${service} (${PROJECT_ID}/${REGION})"
            gcloud run services add-iam-policy-binding "${service}" \
              --region "${REGION}" \
              --project "${PROJECT_ID}" \
              --member="allUsers" \
              --role="roles/run.invoker"
          done

  troydcthompsonPresentationCypressTests:
    name: troydcthompson-alpha.web.app Presentation Cypress Tests
    needs: alphaHostingDeployment
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v6
      - name: Set up Node.js
        uses: actions/setup-node@v6
        with:
          node-version: '24.18.0'
          cache: 'npm'

      - uses: cypress-io/github-action@v7
        with:
          config: baseUrl=https://troydcthompson-alpha.web.app
          wait-on: https://troydcthompson-alpha.web.app
          wait-on-timeout: 180000
          config-file: cypress/support/configurations/presentation/troydcthompson-alpha-presentation.config.ts

  alphawebsitePresentationCypressTests:
    name: alphawebsite.com Presentation Cypress Tests
    needs: alphaHostingDeployment
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v6
      - name: Set up Node.js
        uses: actions/setup-node@v6
        with:
          node-version: '24.18.0'
          cache: 'npm'

      - uses: cypress-io/github-action@v7
        with:
          config: baseUrl=https://alphawebsite.com
          wait-on: https://alphawebsite.com
          wait-on-timeout: 180000
          config-file: cypress/support/configurations/presentation/alphawebsite-presentation.config.ts

      - name: Trigger Workflow 4 - Detect Feature Regression
        uses: peter-evans/repository-dispatch@v4
        with:
          token: ${{ secrets.REPO_ACCESS_TOKEN }}
          repository: 1001daysofcode/app
          event-type: detect
          client-payload: '{"ref":"${{ github.ref }}","sha":"${{ github.sha }}"}'