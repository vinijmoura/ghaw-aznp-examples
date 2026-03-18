# Build and Deploy .NET 10 Minimal API

This workflow builds a .NET 10 Minimal API application, runs its tests, and then sequentially deploys the published artifact to a Staging environment followed by a Production environment on Azure App Service.

## Triggers

- **Push** to the `main` branch — runs automatically whenever commits are pushed to `main`.
- **Manual dispatch** (`workflow_dispatch`) — can be triggered on demand from the GitHub Actions UI on any branch.

## Permissions

| Permission | Scope | Purpose |
|---|---|---|
| `contents: read` | Workflow-level | Allows the workflow to read repository contents (code checkout). |

> The `staging` and `production` jobs override this with `permissions: {}` (no permissions), as they only operate on pre-built artifacts and external Azure services.

## Jobs

### Job: `build` — Build

Restores NuGet dependencies, compiles the project in Release configuration, runs tests, publishes the application, and uploads the output as a reusable artifact.

- **Runs on**: `ubuntu-latest`

#### Steps

1. **Checkout code**
   - **Action / Command**: `actions/checkout@v4`
   - **Purpose**: Clones the repository into the runner so subsequent steps can access the source code.

2. **Setup .NET**
   - **Action / Command**: `actions/setup-dotnet@v4`
   - **Purpose**: Installs the .NET SDK on the runner so `dotnet` CLI commands are available.
   - **Key inputs / parameters**: `dotnet-version: '10.0.x'` — uses the latest patch of .NET 10.

3. **Restore dependencies**
   - **Action / Command**: Runs shell command — `dotnet restore ./src/MinimalApi/MinimalApi.csproj`
   - **Purpose**: Downloads all NuGet packages required by the project before building.

4. **Build**
   - **Action / Command**: Runs shell command — `dotnet build ./src/MinimalApi/MinimalApi.csproj --no-restore --configuration Release`
   - **Purpose**: Compiles the project in Release mode. `--no-restore` skips a redundant restore since the previous step already did it.

5. **Test**
   - **Action / Command**: Runs shell command — `dotnet test ./src/ --no-build --configuration Release --verbosity normal`
   - **Purpose**: Executes all test projects found under `./src/` to validate the build. `--no-build` reuses the binaries already compiled in the Build step.

6. **Publish**
   - **Action / Command**: Runs shell command — `dotnet publish ./src/MinimalApi/MinimalApi.csproj --no-build --configuration Release --output ./publish`
   - **Purpose**: Packages the compiled application into the `./publish` directory, ready for deployment.

7. **Upload artifact**
   - **Action / Command**: `actions/upload-artifact@v4`
   - **Purpose**: Stores the `./publish` directory as a named workflow artifact (`webapp`) so subsequent jobs can download and deploy it without rebuilding.
   - **Key inputs / parameters**: `name: webapp`, `path: ./publish`

---

### Job: `staging` — Deploy to Staging

Downloads the build artifact and deploys it to the `staging` slot of an Azure Web App in the `Staging` GitHub environment (which may include required reviewers or protection rules).

- **Runs on**: `ubuntu-latest`
- **Needs**: `build`
- **Environment**: `Staging`

#### Steps

1. **Download artifact**
   - **Action / Command**: `actions/download-artifact@v4`
   - **Purpose**: Retrieves the `webapp` artifact produced by the `build` job into `./publish`.
   - **Key inputs / parameters**: `name: webapp`, `path: ./publish`

2. **Login to Azure**
   - **Action / Command**: `azure/login@v2`
   - **Purpose**: Authenticates the runner against Azure using service principal credentials, enabling subsequent Azure CLI / SDK calls.
   - **Key inputs / parameters**: `creds: ${{ secrets.AZURE_CREDENTIALS }}`

3. **Deploy to Azure Web App - Staging slot**
   - **Action / Command**: `azure/webapps-deploy@v3`
   - **Purpose**: Deploys the published package to the `staging` slot of the Azure Web App. The step output `webapp-url` is used to populate the environment URL shown in the GitHub UI.
   - **Key inputs / parameters**: `app-name: ${{ vars.AZURE_WEBAPP_NAME }}`, `slot-name: staging`, `package: ./publish`

---

### Job: `production` — Deploy to Production

Downloads the same build artifact and deploys it to the default (production) slot of the Azure Web App in the `Production` GitHub environment.

- **Runs on**: `ubuntu-latest`
- **Needs**: `staging`
- **Environment**: `Production`

#### Steps

1. **Download artifact**
   - **Action / Command**: `actions/download-artifact@v4`
   - **Purpose**: Retrieves the `webapp` artifact produced by the `build` job into `./publish`.
   - **Key inputs / parameters**: `name: webapp`, `path: ./publish`

2. **Login to Azure**
   - **Action / Command**: `azure/login@v2`
   - **Purpose**: Authenticates the runner against Azure using service principal credentials.
   - **Key inputs / parameters**: `creds: ${{ secrets.AZURE_CREDENTIALS }}`

3. **Deploy to Azure Web App - Production slot**
   - **Action / Command**: `azure/webapps-deploy@v3`
   - **Purpose**: Deploys the published package to the production (default) slot of the Azure Web App. The step output `webapp-url` is used to populate the environment URL shown in the GitHub UI.
   - **Key inputs / parameters**: `app-name: ${{ vars.AZURE_WEBAPP_NAME }}`, `package: ./publish`

## Required Secrets and Variables

| Name | Type | Description |
|---|---|---|
| `AZURE_CREDENTIALS` | Secret | JSON object containing the Azure service principal credentials (client ID, client secret, tenant ID, subscription ID) used by `azure/login` to authenticate against Azure. |
| `AZURE_WEBAPP_NAME` | Variable | The name of the Azure App Service resource to deploy to. |
