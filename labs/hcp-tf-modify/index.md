# Modifying Infrastructure in HCP Terraform

---

### Overview

In this lab, you will learn how to make changes to your infrastructure managed by HCP Terraform. You will explore the workspace overview, update variables (including using per-run variables), trigger new runs, and review and approve planned changes. This process allows you to safely and collaboratively update your infrastructure using the CLI-driven workflow and the HCP Terraform UI.

---

### Context

After provisioning infrastructure with HCP Terraform, you often need to make changes—such as resizing an instance or updating configuration values. HCP Terraform provides multiple ways to set and override variables, and every change is tracked in the workspace’s run history. Understanding how to use per-run variables and review changes before applying them is essential for safe infrastructure management.

---

### Hands-On Tasks

#### 1. Explore Your Workspace

- Navigate to your workspace’s overview page in the HCP Terraform UI.
- Review the lock status, resource count, Terraform version, and time since last update.
- Examine the **Latest run** section for details about recent changes, resource modifications, and estimated cost.
- Use the sidebar to view workspace tags and settings.
- Explore the **Runs** page to see the history of all plan and apply actions, and use filters to view runs by status, operation, or source.

---

#### 2. Edit Variables and Use Per-Run Overrides

- You can update variables in the HCP Terraform UI under the **Variables** page.
- To temporarily override a variable for a single run, use the `-var` flag on the command line. For example, to change the instance type:

```sh
terraform apply -var="instance_type=t2.small"
```

- Note: Per-run variables do not persist in the workspace and are ideal for testing or temporary changes.

---

#### 3. Run and Approve Infrastructure Changes

- Run `terraform apply -var="instance_type=t2.small"` to trigger a new run in HCP Terraform.
- Review the plan output in your terminal and/or follow the run URL to the HCP Terraform UI for more details.
- The plan step summarizes proposed changes. Approve the run in the UI or terminal to apply the changes.
- After approval, Terraform will update your infrastructure and the workspace state.

---

#### 4. Observe the Effect of Changes

- After the apply completes, return to the workspace’s **Overview** page.
- Review the updated resources and outputs tables to confirm your changes.
- Use the **Runs** and **States** pages to audit the history and state of your infrastructure.

---

### Expected Results

- You have successfully made and applied changes to your infrastructure using per-run variables and the CLI-driven workflow.
- All changes are tracked and auditable in the HCP Terraform UI.
- You understand how to review, approve, and verify infrastructure updates.

---

### Reflection & Challenge

- **Reflection:** How does using per-run variables and the HCP Terraform UI improve the safety and auditability of infrastructure changes?
- **Challenge:**
  - Try making a different change (such as updating another variable or resource) and observe how the run history and state files are updated.

---

### Using the VCS-Driven Workflow in HCP Terraform

#### Overview

The VCS-driven workflow in HCP Terraform enables teams to manage infrastructure as code by connecting Terraform workspaces directly to version control repositories such as GitHub. This workflow makes your repository the single source of truth for infrastructure configuration, allowing changes to be tracked, reviewed, and automatically applied through pull requests and merges. By integrating with VCS, HCP Terraform streamlines collaboration and enforces best practices for infrastructure management.

---

#### 1. Update Configuration for VCS Workflow

- Open your `main.tf` file and comment out the `cloud` block, as it is not required for the VCS-driven workflow.

Example:

```hcl
terraform {
/*
  cloud {
    organization = "policy-as-code-training"
    workspaces {
      name = "tf-vault-qa-{your-initials}"
    }
  }
*/
}
```

- Add and commit your changes:

```sh
git add .
git commit -m "Remove cloud block"
```

- Push your configuration to your personal repository:

```sh
git push
```

---

#### 3. Enable VCS Integration in HCP Terraform

- In the HCP Terraform UI, go to your workspace's **Settings** and select **Version Control**.
- Click **Connect to version control**, select **Version Control Workflow** and choose your VCS provider (e.g., GitHub.com)
- Authorize HCP Terraform to access your repository and select the repository you created.
- Confirm the connection and enable automatic speculative plans (checkbox below Pull Requests), which preview infrastructure changes for every pull request.

---

#### 4. Make an Explicit Change to Trigger a Plan

To test your HCP Terraform integration, make a simple, valid change to your configuration. For this lab, you will update the value of the `environment` tag in all tag blocks in `main.tf` from `"dev"` to `"development"`.

1. Open `main.tf` in your code editor.
2. Find all occurrences of:
   ```hcl
   environment = "dev"
   ```
3. Change them to:
   ```hcl
   environment = "development"
   ```
4. Save the file.
5. Commit and push your change to the main branch:
   ```sh
   git add main.tf
   git commit -m "Change environment tag from dev to development"
   git push
   ```
6. This push will automatically trigger a Terraform plan in your HCP Terraform workspace. You can monitor the run and its results in the HCP Terraform UI.
7. Once you've thoroughly evaluated the proposed changes, apply your changes via the HCP terraform UI.

---

#### Summary

- You set up a VCS-driven workflow by connecting a GitHub repository to your HCP Terraform workspace.
- You updated your configuration, enabled VCS integration, and triggered speculative plans with pull requests.
- You merged changes and observed how HCP Terraform automatically applied infrastructure updates.

---

#### Reflection & Challenge

- **Reflection:** How does the VCS-driven workflow improve collaboration and change management for infrastructure as code?
- **Challenge:**
  - Try making a change in a new branch, opening a pull request, and reviewing the speculative plan before merging.
  - Experiment with configuring your workspace to trigger runs on changes to specific paths or tags in your repository.

---
