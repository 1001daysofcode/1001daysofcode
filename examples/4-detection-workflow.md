name: Workflow 4 - Detect Feature Regression

on:
  repository_dispatch:
    types: [detect]

jobs:
  betaHostingDeployment:
    name: Firebase Deployment to Beta
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

      - name: Build Beta
        run: npx ng build --configuration beta

      - name: Authenticate to Google Cloud
        id: auth
        uses: google-github-actions/auth@v3
        with:
          credentials_json: ${{ secrets.BETA_GCP_SA_KEY }}
          create_credentials_file: true
          export_environment_variables: true

      - name: Setup Cloud SDK
        uses: google-github-actions/setup-gcloud@v3
        with:
          project_id: troydcthompson-beta

      - name: Install Firebase CLI
        run: npm install -g firebase-tools@15.19.1

      - name: Deploy Firebase Hosting
        env:
          GOOGLE_APPLICATION_CREDENTIALS: ${{ steps.auth.outputs.credentials_file_path }}
          GCLOUD_PROJECT: troydcthompson-beta
          GOOGLE_CLOUD_PROJECT: troydcthompson-beta
          PROJECT_ID: troydcthompson-beta
          HOSTING_SITE_ID: troydcthompson-beta
        run: node scripts/deploy-hosting-with-retry.mjs --project "$PROJECT_ID" --site "$HOSTING_SITE_ID" --attempts 3 --initial-delay-ms 2500 --command "firebase deploy --only hosting --project troydcthompson-beta --non-interactive --debug"

      - name: Hosting failure diagnostics
        if: ${{ failure() }}
        env:
          GOOGLE_APPLICATION_CREDENTIALS: ${{ steps.auth.outputs.credentials_file_path }}
          GCLOUD_PROJECT: troydcthompson-beta
          GOOGLE_CLOUD_PROJECT: troydcthompson-beta
          PROJECT_ID: troydcthompson-beta
          HOSTING_SITE_ID: troydcthompson-beta
        run: node scripts/verify-hosting-site.mjs --project "$PROJECT_ID" --site "$HOSTING_SITE_ID" --attempts 1 --diagnostic-no-fail

      - name: Deploy Firebase Functions, Firestore Rules, Firestore Indexes, and Storage
        env:
          GOOGLE_APPLICATION_CREDENTIALS: ${{ steps.auth.outputs.credentials_file_path }}
          GCLOUD_PROJECT: troydcthompson-beta
          GOOGLE_CLOUD_PROJECT: troydcthompson-beta
        run: firebase deploy --only functions,firestore:rules,firestore:indexes,storage --project troydcthompson-beta --non-interactive --force --debug

      - name: Ensure callable Cloud Run invoker is public
        run: |
          set -euo pipefail

          PROJECT_ID="troydcthompson-beta"
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
  daysofcodeBetaRegressionCypressTests:
    name: 1000DaysOfCode.com Regression Cypress Tests
    needs: betaHostingDeployment
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
          config: baseUrl=https://1000DaysOfCode.com
          wait-on: https://1000DaysOfCode.com
          wait-on-timeout: 180000
          config-file: cypress/support/configurations/regression/1000daysofcode-regression.config.ts

  troydcthompsonBetaRegressionCypressTests:
    name: TroyDCThompson-beta.web.app Regression Cypress Tests
    needs: betaHostingDeployment
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
          config: baseUrl=https://troydcthompson-beta.web.app
          wait-on: https://troydcthompson-beta.web.app
          wait-on-timeout: 180000
          config-file: cypress/support/configurations/regression/troydcthompson-beta-regression.config.ts

      - name: Trigger Workflow 5 - Deploy Production Application
        uses: peter-evans/repository-dispatch@v4
        with:
          token: ${{ secrets.REPO_ACCESS_TOKEN }}
          repository: 1001daysofcode/app
          event-type: deploy
          client-payload: '{"ref":"${{ github.ref }}","sha":"${{ github.sha }}"}'