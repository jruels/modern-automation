# Publishing and Versioning Terraform Modules in HCP Terraform

---

### Overview

In this lab, you will create a new Terraform module repository with the correct naming convention, add a basic module, and publish it to the HCP Terraform private registry. You will then manage module versions and observe how versioning and documentation are handled in the registry. This process enables you to share reusable infrastructure code securely and efficiently within your organization.

---

### Context

Publishing modules to the HCP Terraform private registry allows teams to reuse, version, and document infrastructure code. Each module version is tracked, and consumers can select specific versions for use in their workspaces. Understanding the publishing and versioning workflow is essential for maintaining reliable and collaborative infrastructure practices.

---

### Hands-On Tasks

#### 1. Create a New Module Repository

- Choose a provider and module name. For this lab, we'll use AWS and create a simple S3 bucket module.
- Name your repository using the convention: `terraform-aws-s3-bucket-{your-initials}`.
- **First, create a new repository in your VCS provider (e.g., GitHub) named `terraform-aws-s3-bucket-{your-initials}`.**
- On your local machine, clone the repository:
  ```sh
  git clone https://github.com/your-username/terraform-aws-s3-bucket-{your-initials}.git
  cd terraform-aws-s3-bucket-{your-initials}
  ```
Create a new folder for the module code `tf-publish-module`. Enter the directory
- Clone the class repository: 
    ```
    cd tf-publish-module
    git clone https://github.com/jruels/modern-automation
    cd modern-automation/hcp-tf-publish-module
    ```
- Copy the provided files (`main.tf`, `variables.tf`, `outputs.tf`, `README.md`) from the `hcp-tf-publish-module` lab folder into your new repository directory.

- Review the files and make any necessary adjustments (such as updating the README usage example with your organization name).

---

#### 2. Push the Module to a VCS Provider

- Commit and push your changes:
  ```sh
  git add .
  git commit -m "Initial commit: basic S3 bucket module"
  git push -u origin main
  ```

---

#### 3. Create an initial tag version

- Create and push a semantic version tag (e.g., `v1.0.0`) to your repository:
  ```sh
  git tag v1.0.0
  git push origin v1.0.0
  ```
- Update the README, and push a new tag (e.g., `v1.1.0`).

#### 4. Publish the Module to HCP Terraform

- In the HCP Terraform UI, navigate to your organization’s **Registry** section.
- Click **Publish** and select **Module**.
- Choose your VCS provider and select the `terraform-aws-s3-bucket-{your-initials}` repository.
- Follow the prompts to configure publishing:
    - Module name: `s3-bucket-{your-initials}`
    - Provider: `aws`
- Select **Publish Module**.

---

#### 5. Consume a Specific Module Version

- In your `learn-terraform-variables` repository (which you forked and cloned in the **hcp-tf-setup** lab and have been using in the **hcp-tf-modify** lab), add a module block to your configuration to consume the published S3 bucket module. Copy the block from the **Usage Instructions** on the right side of the Registry module page. An example is below (replace `bucket_name` with a unique string): 
  ```hcl
  module "s3_bucket" {
    source  = "app.terraform.io/policy-as-code-training/s3-bucket-{your-initials}/aws"
    version = "1.0.0"
    bucket_name = "my-bucket"
  }
  ```
- Save your changes, then commit and push them to your `learn-terraform-variables` repository:
  ```sh
  git add .
  git commit -m "Add s3_bucket module from private registry"
  git push
  ```
- This push will trigger a new run in your HCP Terraform workspace associated with the `learn-terraform-variables` repo. Monitor the run in the HCP Terraform UI, review the plan, and apply the changes to provision the S3 bucket using your published module.

---

### Expected Results

- You have created a new module repository, published it to the HCP Terraform private registry, and released at least two versions.
- The registry displays version history, documentation, and usage instructions for your module.
- You can reference and use specific module versions in your Terraform configurations.

---

### Reflection & Challenge

- **Reflection:** How does versioning modules in the registry improve reliability and collaboration for your team?
- **Challenge:**
  - Deprecate an older module version by updating documentation or using the registry UI.
  - Try publishing a breaking change as a new major version (e.g., `v2.0.0`) and observe how consumers can select which version to use.

---

Reference: [Publishing Private Modules in HCP Terraform](https://developer.hashicorp.com/terraform/cloud-docs/registry/publish-modules) 
