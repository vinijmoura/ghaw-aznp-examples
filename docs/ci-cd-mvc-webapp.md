# CI/CD - .NET 10 Web App

This workflow builds a .NET 10 Minimal API web application and deploys it to an Azure Web App whenever changes are pushed to the `main` branch.

## Triggers

- **Push** to the `main` branch.
- **Manual dispatch** (`workflow_dispatch`) — can be triggered at any time from the GitHub Actions UI with no additional inputs.

## Permissions

- **`contents: read`** — allows the workflow to read repository contents (checkout code). Declared at both the top-level and the `build` job level. The `deploy` job explicitly sets permissions to `{}` (no permissions), limiting its access to only what is inherited from the GitHub token minimum.

## Jobs

### Job: `build` — CI - Build

Checks out the source code, sets up the .NET 10 SDK, restores dependencies, compiles the project in Release configuration, publishes the output, and uploads it as a build artifact for downstream jobs.

- **Runs on**: `ubuntu-latest`

#### Steps

1. **Checkout code**
   - **Action / Command**: `actions/checkout@v4`
   - **Purpose**: Clones the repository into the runner so subsequent steps can access source files.

2. **Setup .NET**
   - **Action / Command**: `actions/setup-dotnet@v4`
   - **Purpose**: Installs the .NET SDK on the runner.
   - **Key inputs / parameters**: `dotnet-version: '10.0.x'` — uses the latest patch of .NET 10.

3. **Restore dependencies**
   - **Action / Command**: Runs shell command — `dotnet restore ./src/MinimalApi/MinimalApi.csproj`
   - **Purpose**: Downloads and restores all NuGet packages required by the project.

4. **Build**
   - **Action / Command**: Runs shell command — `dotnet build ./src/MinimalApi/MinimalApi.csproj --no-restore --configuration Release`
   - **Purpose**: Compiles the project in Release mode, skipping the restore step since it was already done.

5. **Publish**
   - **Action / Command**: Runs shell command — `dotnet publish ./src/MinimalApi/MinimalApi.csproj --no-build --configuration Release --output ./publish`
   - **Purpose**: Produces a self-contained publish output in `./publish`, ready for deployment.

6. **Upload artifact**
   - **Action / Command**: `actions/upload-artifact@v4`
   - **Purpose**: Uploads the `./publish` directory as a workflow artifact so the `deploy` job can download it.
   - **Key inputs / parameters**: `name: webapp`, `path: ./publish`

### Job: `deploy` — CD - Deploy to Azure Web App

Downloads the build artifact produced by the `build` job, authenticates with Azure, and deploys the application to an Azure Web App in the `Production` environment.

- **Runs on**: `ubuntu-latest`
- **Needs**: `build`
- **Environment**: `Production`

#### Steps

1. **Download artifact**
   - **Action / Command**: `actions/download-artifact@v4`
   - **Purpose**: Retrieves the `webapp` artifact uploaded by the `build` job and places it in `./publish`.
   - **Key inputs / parameters**: `name: webapp`, `path: ./publish`

2. **Login to Azure**
   - **Action / Command**: `azure/login@v2`
   - **Purpose**: Authenticates the runner with Azure using a service principal, enabling subsequent Azure CLI and Azure Actions steps to act on Azure resources.
   - **Key inputs / parameters**: `creds: ${{ secrets.AZURE_CREDENTIALS }}`

3. **Deploy to Azure Web App**
   - **Action / Command**: `azure/webapps-deploy@v3`
   - **Purpose**: Deploys the published application package to the target Azure Web App. The step id `deploy` exposes the deployed app URL via `steps.deploy.outputs.webapp-url`, which is surfaced as the environment URL.
   - **Key inputs / parameters**: `app-name: ${{ vars.AZURE_WEBAPP_NAME }}`, `package: ./publish`

## Required Secrets and Variables

| Name | Type | Description |
|------|------|-------------|
| `AZURE_CREDENTIALS` | Secret | JSON credentials for an Azure service principal (output of `az ad sp create-for-rbac`). Used by `azure/login@v2` to authenticate the runner with Azure. |
| `AZURE_WEBAPP_NAME` | Variable | The name of the target Azure Web App resource to deploy to. |
