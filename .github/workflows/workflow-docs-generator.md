---
description: |
  This workflow generates or updates documentation for GitHub Actions workflow YML files.
  Whenever a YML file is changed in .github/workflows, this workflow creates or updates a corresponding markdown documentation file in the docs folder, describing step by step all the steps inside each YML.

on:
  push:
    branches: [main]
    paths:
      - '.github/workflows/*.yml'
  workflow_dispatch:

permissions:
  contents: read
  pull-requests: read
  issues: read

checkout:
  fetch-depth: 2

tools:
  bash: true
  edit:
  github:
    mode: remote
    toolsets: [default]

safe-outputs:
  create-pull-request:
    title-prefix: "[docs] "
    labels: [documentation]
    draft: false
    if-no-changes: ignore
---

# Workflow Documentation Generator

Generate or update markdown documentation in the `docs/` folder for every GitHub Actions workflow YML file that was added or changed in this push.

## Steps

1. **Find the workflow YML files to document**:
   - For a `push` event: run `git diff --name-only HEAD^ HEAD -- '.github/workflows/*.yml'` to list changed files. Filter out any file whose name ends in `.lock.yml` — document only plain `.yml` files. If the git diff command fails (e.g., first commit), fall back to `find .github/workflows -maxdepth 1 -name '*.yml' ! -name '*.lock.yml' | sort`.
   - For a `workflow_dispatch` event: run `find .github/workflows -maxdepth 1 -name '*.yml' ! -name '*.lock.yml' | sort` to list every workflow YML file.

2. **For each identified YML file** (e.g., `.github/workflows/build-and-deploy.yml`):
   a. Read its content with `cat <filepath>`.
   b. Carefully analyse the YAML structure: workflow name, triggers (`on:`), permissions, jobs, and every step inside each job. Create a flow based on memrmaid diagram that exemplify YAML structure.
   c. Write a documentation file to `docs/<filename-without-extension>.md` (e.g., `docs/build-and-deploy.md`) using the edit tool.

3. **Documentation format** — each `docs/<name>.md` file must follow this structure exactly:

   ```
   # <Workflow Name>

   <One or two sentences describing the overall purpose of this workflow.>

   ## Triggers

   <Describe all events that activate this workflow, including branch filters, path filters, schedule expressions, or manual dispatch. Be specific — list branch names, paths, and cron expressions where present.>

   ## Permissions

   <List every permission declared in the `permissions:` block and explain what it allows the workflow to do.>

   ## Jobs

   ### Job: `<job-id>` — <job `name:` value>

   <Short description of what this job accomplishes.>

   - **Runs on**: `<runner>`
   - **Needs**: `<dependent job ids>` *(omit line if no dependencies)*
   - **Environment**: `<environment name>` *(omit line if not set)*

   #### Steps

   1. **<step name or inferred name>**
      - **Action / Command**: `<uses: action@version>` or "Runs shell command"
      - **Purpose**: <What does this step do? Why is it needed?>
      - **Key inputs / parameters**: <List important `with:`, `run:`, or `env:` values — omit trivial or self-explanatory ones>

   *(Repeat for every step in the job, in order)*

   *(Repeat the "### Job" section for every job in the workflow, in order)*

   ## Required Secrets and Variables

   <List every `secrets.*` and `vars.*` reference found in the file. Describe what each secret or variable is expected to contain. If there are none, write "None.">
   ```

4. **Ensure the `docs/` directory exists** by running `mkdir -p docs` before writing any file.

5. **Create a pull request** that includes all new or updated documentation files so the changes can be reviewed and merged.
