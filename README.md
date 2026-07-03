trigger:
  branches:
    include:
      - main

pool:
  vmImage: ubuntu-latest

variables:
  TF_VERSION: "1.8.5"

stages:

# ==========================
# CI STAGE
# ==========================

- stage: CI
  displayName: "Terraform Validation"

  jobs:

  - job: Terraform_Check
    displayName: "Terraform Validation Job"

    steps:

    - checkout: self

    # Install Terraform
    - task: TerraformInstaller@1
      inputs:
        terraformVersion: $(TF_VERSION)

    # Terraform Init
    - script: |
        terraform init
      displayName: Terraform Init

    # Terraform Format
    - script: |
        terraform fmt -check
      displayName: Terraform Format Check

    # Terraform Validate
    - script: |
        terraform validate
      displayName: Terraform Validate

    # Install TFLint
    - script: |
        curl -s https://raw.githubusercontent.com/terraform-linters/tflint/master/install_linux.sh | bash
      displayName: Install TFLint

    # Run TFLint
    - script: |
        tflint
      displayName: Run TFLint

    # Install tfsec
    - script: |
        curl -s https://raw.githubusercontent.com/aquasecurity/tfsec/master/scripts/install_linux.sh | bash
      displayName: Install tfsec

    # Run tfsec
    - script: |
        tfsec .
      displayName: Security Scan

    # Install TruffleHog
    - script: |
        curl -sSfL https://raw.githubusercontent.com/trufflesecurity/trufflehog/main/scripts/install.sh | sh
      displayName: Install TruffleHog

    # Secret Scan
    - script: |
        trufflehog filesystem .
      displayName: Secret Scan

    # Terraform Plan
    - script: |
        terraform plan -out=tfplan
      displayName: Terraform Plan

    # Publish Artifact
    - task: PublishPipelineArtifact@1
      inputs:
        targetPath: tfplan
        artifact: terraformPlan

# ==========================
# CD STAGE
# ==========================

- stage: CD

  displayName: "Terraform Deployment"

  dependsOn: CI

  jobs:

  - deployment: DeployTerraform

    environment: Production

    strategy:

      runOnce:

        deploy:

          steps:

          - checkout: self

          - task: DownloadPipelineArtifact@2
            inputs:
              artifact: terraformPlan
              path: $(Pipeline.Workspace)

          - task: TerraformInstaller@1
            inputs:
              terraformVersion: $(TF_VERSION)

          - script: |
              terraform init
            displayName: Terraform Init

          - script: |
              terraform apply -auto-approve $(Pipeline.Workspace)/tfplan
            displayName: Terraform Apply
