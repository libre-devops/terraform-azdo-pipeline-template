# Terraform Template for Azure DevOps

```yaml
---
parameters:

  - name: TERRAFORM_PATH
    type: string
    default: ""
    displayName: "What is the path to your terraform code?"

  - name: TERRAFORM_VERSION
    type: string
    default: ""
    displayName: "What version of terraform should be installed?"

  - name: TERRAFORM_DESTROY
    default: false
    type: boolean
    displayName: "Do you wish to run terraform destroy?"

  - name: TERRAFORM_PLAN_ONLY
    default: true
    type: boolean
    displayName: "Do you wish to run terraform destroy?"

  - name: TERRAFORM_STORAGE_RG_NAME
    default: ""
    type: string
    displayName: "What is the resource group name in which the storage account exists in?"

  - name: TERRAFORM_STORAGE_ACCOUNT_NAME
    default: ""
    type: string
    displayName: "What is the name of the storage account in which the state file is being stored?"

  - name: TERRAFORM_BLOB_CONTAINER_NAME
    default: ""
    type: string
    displayName: "What is the name of the blob container in which the state file is being stored?"

  - name: TERRAFORM_STORAGE_KEY
    default: ""
    type: string
    displayName: "What is the key used to access your storage account? Please note, this value is a secret"

  - name: TERRAFORM_STATE_NAME
    default: ""
    type: string
    displayName: "What name should the state file have?"

  - name: TERRAFORM_WORKSPACE_NAME
    default: ""
    type: string
    displayName: "Which workspace should be used or created?"

  - name: TERRAFORM_COMPLIANCE_PATH
    type: string
    default: ""
    displayName: "Where is your terraform-compliance policy files located?"

  - name: AZURE_TARGET_CLIENT_ID
    default: ""
    type: string
    displayName: "What is the client id of the service principle you wish to use with Terraform?"

  - name: AZURE_TARGET_CLIENT_SECRET
    default: ""
    type: string
    displayName: "What is the client of the service principle you wish to use with Terraform?  Note, this value is a secret"

  - name: AZURE_TARGET_SUBSCRIPTION_ID
    default: ""
    type: string
    displayName: "What is the subscription ID of the target subscription you are trying to deploy to?"

  - name: AZURE_TARGET_TENANT_ID
    default: ""
    type: string
    displayName: "What is the tenant ID in which the target subscription resides?"

  - name: SHORTHAND_PROJECT_NAME
    default: ""
    type: string
    displayName: "What is the shorthand name for your project?"

  - name: SHORTHAND_ENVIRONMENT_NAME
    default: ""
    type: string
    displayName: "What is the shorthand (3 character) name for environment you are deploying to?"

  - name: SHORTHAND_LOCATION_NAME
    default: ""
    type: string
    displayName: "What is the shorthand location name? E.g. uks for UK South etc"

  - name: CHECKOV_SKIP_TESTS
    default: ""
    type: string
    displayName: "What Checkov steps should be skipped, null by default, should be value like CKV_AZURE_50, CKV_AZURE_20 etc."

steps:

  - task: ms-devlabs.custom-terraform-tasks.custom-terraform-installer-task.TerraformInstaller@0
    displayName: "Install Terraform ${{ parameters.TERRAFORM_VERSION }}"
    inputs:
      terraformVersion: ${{ parameters.TERRAFORM_VERSION }}
    enabled: true

  - ${{ if and(eq(parameters.TERRAFORM_DESTROY, false), eq(parameters.TERRAFORM_PLAN_ONLY, true)) }}:

      - pwsh: |
          New-Item -Path . -Name .terraform -ItemType "Directory" -Force ; `

          terraform init `
          -backend-config="storage_account_name=${{ parameters.TERRAFORM_STORAGE_ACCOUNT_NAME }}" `
          -backend-config="container_name=${{ parameters.TERRAFORM_BLOB_CONTAINER_NAME }}" `
          -backend-config="access_key=${{ parameters.TERRAFORM_STORAGE_KEY }}" `
          -backend-config="key=${{ parameters.TERRAFORM_STATE_NAME }}" ; `

          Write-Output "${{ parameters.TERRAFORM_WORKSPACE_NAME }}" > .terraform/environment ; `

          terraform workspace new "${{ parameters.TERRAFORM_WORKSPACE_NAME }}" ; `
          terraform workspace select "${{ parameters.TERRAFORM_WORKSPACE_NAME }}" ; `

          terraform validate ; `

          terraform plan -out pipeline.plan
        displayName: Terraform Init, Validate & Plan
        workingDirectory: ${{ parameters.TERRAFORM_PATH }}
        enabled: true
        env:
          TF_VAR_short: ${{ parameters.SHORTHAND_PROJECT_NAME }}
          TF_VAR_env: ${{ parameters.SHORTHAND_ENVIRONMENT_NAME }}
          TF_VAR_loc: ${{ parameters.SHORTHAND_LOCATION_NAME }}

          TF_VAR_TERRAFORM_STORAGE_RG_NAME: ${{ parameters.TERRAFORM_STORAGE_RG_NAME }}
          TF_VAR_TERRAFORM_STORAGE_ACCOUNT_NAME: ${{ parameters.TERRAFORM_STORAGE_ACCOUNT_NAME }}
          TF_VAR_TERRAFORM_BLOB_CONTAINER_NAME: ${{ parameters.TERRAFORM_BLOB_CONTAINER_NAME }}
          TF_VAR_TERRAFORM_STORAGE_KEY: ${{ parameters.TERRAFORM_STORAGE_KEY }}

          ARM_CLIENT_ID: ${{ parameters.AZURE_TARGET_CLIENT_ID }}
          ARM_CLIENT_SECRET: ${{ parameters.AZURE_TARGET_CLIENT_SECRET }}
          ARM_SUBSCRIPTION_ID: ${{ parameters.AZURE_TARGET_SUBSCRIPTION_ID }}
          ARM_TENANT_ID: ${{ parameters.AZURE_TARGET_TENANT_ID }}

      - pwsh: |
          pip3 install terraform-compliance ; `

          terraform-compliance -p pipeline.plan -f ${{ parameters.TERRAFORM_COMPLIANCE_PATH }}
        displayName: 'Terraform-Compliance Check'
        workingDirectory: "${{ parameters.TERRAFORM_PATH }}"
        continueOnError: true
        enabled: true

      - pwsh: |
          if ($IsLinux)
          {
          brew install tfsec
          }
          elseif ($IsMacOS)
          {
            brew install tfsec
          }
          elseif ($IsWindows)
          {
            Set-ExecutionPolicy Bypass -Scope Process -Force; [System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072; Invoke-Expression ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))
            choco install tfsec -y
          }
          tfsec . --force-all-dirs
        displayName: 'TFSEC Check'
        workingDirectory: "${{ parameters.TERRAFORM_PATH }}"
        continueOnError: false
        enabled: true

      - pwsh: |
         pip3 install checkov ; `

         terraform show -json pipeline.plan > pipeline.plan.json ; `

         checkov -f pipeline.plan.json
        displayName: 'CheckOV Check'
        workingDirectory: "${{ parameters.TERRAFORM_PATH }}"
        continueOnError: false
        condition: eq('${{ parameters.CHECKOV_SKIP_TESTS }}', ' ')
        enabled: true

      - pwsh: |
         pip3 install checkov ; `

         terraform show -json pipeline.plan > pipeline.plan.json ; `

         checkov -f pipeline.plan.json --skip-check ${{ parameters.CHECKOV_SKIP_TESTS }}
        displayName: 'CheckOV Check with Skipped Tests'
        workingDirectory: "${{ parameters.TERRAFORM_PATH }}"
        continueOnError: false
        condition: not(eq('${{ parameters.CHECKOV_SKIP_TESTS }}', ' '))
        enabled: true

  - ${{ if and(eq(parameters.TERRAFORM_DESTROY, false), eq(parameters.TERRAFORM_PLAN_ONLY, false)) }}:

      - pwsh: |
          New-Item -Path . -Name .terraform -ItemType "Directory" -Force ; `

          terraform init `
          -backend-config="storage_account_name=${{ parameters.TERRAFORM_STORAGE_ACCOUNT_NAME }}" `
          -backend-config="container_name=${{ parameters.TERRAFORM_BLOB_CONTAINER_NAME }}" `
          -backend-config="access_key=${{ parameters.TERRAFORM_STORAGE_KEY }}" `
          -backend-config="key=${{ parameters.TERRAFORM_STATE_NAME }}" ; `

          Write-Output "${{ parameters.TERRAFORM_WORKSPACE_NAME }}" > .terraform/environment ; `

          terraform workspace new "${{ parameters.TERRAFORM_WORKSPACE_NAME }}" ; `
          terraform workspace select "${{ parameters.TERRAFORM_WORKSPACE_NAME }}" ; `

          terraform validate ; `

          terraform plan -out pipeline.plan
        displayName: Terraform Init, Validate & Plan
        workingDirectory: ${{ parameters.TERRAFORM_PATH }}
        enabled: true
        env:
          TF_VAR_short: ${{ parameters.SHORTHAND_PROJECT_NAME }}
          TF_VAR_env: ${{ parameters.SHORTHAND_ENVIRONMENT_NAME }}
          TF_VAR_loc: ${{ parameters.SHORTHAND_LOCATION_NAME }}

          TF_VAR_TERRAFORM_STORAGE_RG_NAME: ${{ parameters.TERRAFORM_STORAGE_RG_NAME }}
          TF_VAR_TERRAFORM_STORAGE_ACCOUNT_NAME: ${{ parameters.TERRAFORM_STORAGE_ACCOUNT_NAME }}
          TF_VAR_TERRAFORM_BLOB_CONTAINER_NAME: ${{ parameters.TERRAFORM_BLOB_CONTAINER_NAME }}
          TF_VAR_TERRAFORM_STORAGE_KEY: ${{ parameters.TERRAFORM_STORAGE_KEY }}

          ARM_CLIENT_ID: ${{ parameters.AZURE_TARGET_CLIENT_ID }}
          ARM_CLIENT_SECRET: ${{ parameters.AZURE_TARGET_CLIENT_SECRET }}
          ARM_SUBSCRIPTION_ID: ${{ parameters.AZURE_TARGET_SUBSCRIPTION_ID }}
          ARM_TENANT_ID: ${{ parameters.AZURE_TARGET_TENANT_ID }}

      - pwsh: |
         pip3 install terraform-compliance ; `

         terraform-compliance -p pipeline.plan -f ${{ parameters.TERRAFORM_COMPLIANCE_PATH }}
        displayName: 'Terraform-Compliance Check'
        workingDirectory: "${{ parameters.TERRAFORM_PATH }}"
        continueOnError: true
        enabled: true

      - pwsh: |
         if ($IsLinux)
         {
          brew install tfsec
         }
         elseif ($IsMacOS)
         {
           brew install tfsec
         }
          elseif ($IsWindows)
         {
           Set-ExecutionPolicy Bypass -Scope Process -Force; [System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072; Invoke-Expression ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))
           choco install tfsec -y
         }
         tfsec . --force-all-dirs
        displayName: 'TFSEC Check'
        workingDirectory: "${{ parameters.TERRAFORM_PATH }}"
        continueOnError: false
        enabled: true

      - pwsh: |
         pip3 install checkov ; `

         terraform show -json pipeline.plan > pipeline.plan.json ; `

         checkov -f pipeline.plan.json
        displayName: 'CheckOV Check'
        workingDirectory: "${{ parameters.TERRAFORM_PATH }}"
        continueOnError: false
        condition: eq('${{ parameters.CHECKOV_SKIP_TESTS }}', ' ')
        enabled: true

      - pwsh: |
         pip3 install checkov ; `

         terraform show -json pipeline.plan > pipeline.plan.json ; `

         checkov -f pipeline.plan.json --skip-check ${{ parameters.CHECKOV_SKIP_TESTS }}
        displayName: 'CheckOV Check with Skipped Tests'
        workingDirectory: "${{ parameters.TERRAFORM_PATH }}"
        continueOnError: false
        condition: not(eq('${{ parameters.CHECKOV_SKIP_TESTS }}', ' '))
        enabled: true

      - pwsh: |
          New-Item -Path . -Name .terraform -ItemType "Directory" -Force ; `

          terraform init `
          -backend-config="storage_account_name=${{ parameters.TERRAFORM_STORAGE_ACCOUNT_NAME }}" `
          -backend-config="container_name=${{ parameters.TERRAFORM_BLOB_CONTAINER_NAME }}" `
          -backend-config="access_key=${{ parameters.TERRAFORM_STORAGE_KEY }}" `
          -backend-config="key=${{ parameters.TERRAFORM_STATE_NAME }}" ; `

          Write-Output "${{ parameters.TERRAFORM_WORKSPACE_NAME }}" > .terraform/environment ; `

          terraform workspace new "${{ parameters.TERRAFORM_WORKSPACE_NAME }}" ; `
          terraform workspace select "${{ parameters.TERRAFORM_WORKSPACE_NAME }}" ; `

          terraform validate ; `

          terraform plan -out pipeline.plan

          terraform apply pipeline.plan
        displayName: Terraform Init, Validate, Plan & Apply
        workingDirectory: ${{ parameters.TERRAFORM_PATH }}
        enabled: true
        env:
          TF_VAR_short: ${{ parameters.SHORTHAND_PROJECT_NAME }}
          TF_VAR_env: ${{ parameters.SHORTHAND_ENVIRONMENT_NAME }}
          TF_VAR_loc: ${{ parameters.SHORTHAND_LOCATION_NAME }}

          TF_VAR_TERRAFORM_STORAGE_RG_NAME: ${{ parameters.TERRAFORM_STORAGE_RG_NAME }}
          TF_VAR_TERRAFORM_STORAGE_ACCOUNT_NAME: ${{ parameters.TERRAFORM_STORAGE_ACCOUNT_NAME }}
          TF_VAR_TERRAFORM_BLOB_CONTAINER_NAME: ${{ parameters.TERRAFORM_BLOB_CONTAINER_NAME }}
          TF_VAR_TERRAFORM_STORAGE_KEY: ${{ parameters.TERRAFORM_STORAGE_KEY }}

          ARM_CLIENT_ID: ${{ parameters.AZURE_TARGET_CLIENT_ID }}
          ARM_CLIENT_SECRET: ${{ parameters.AZURE_TARGET_CLIENT_SECRET }}
          ARM_SUBSCRIPTION_ID: ${{ parameters.AZURE_TARGET_SUBSCRIPTION_ID }}
          ARM_TENANT_ID: ${{ parameters.AZURE_TARGET_TENANT_ID }}

  - ${{ if and(eq(parameters.TERRAFORM_DESTROY, true), eq(parameters.TERRAFORM_PLAN_ONLY, false)) }}:

      - pwsh: |
          New-Item -Path . -Name .terraform -ItemType "Directory" -Force ; `

          terraform init `
          -backend-config="storage_account_name=${{ parameters.TERRAFORM_STORAGE_ACCOUNT_NAME }}" `
          -backend-config="container_name=${{ parameters.TERRAFORM_BLOB_CONTAINER_NAME }}" `
          -backend-config="access_key=${{ parameters.TERRAFORM_STORAGE_KEY }}" `
          -backend-config="key=${{ parameters.TERRAFORM_STATE_NAME }}" ; `

          Write-Output "${{ parameters.TERRAFORM_WORKSPACE_NAME }}" > .terraform/environment ; `

          terraform workspace new "${{ parameters.TERRAFORM_WORKSPACE_NAME }}" ; `
          terraform workspace select "${{ parameters.TERRAFORM_WORKSPACE_NAME }}" ; `

          terraform validate ; `

          terraform plan -destroy -out pipeline.plan
        displayName: 'Terraform Init, Validate & Plan Destroy'
        workingDirectory: "${{ parameters.TERRAFORM_PATH }}"
        continueOnError: false
        enabled: true
        env:
          TF_VAR_short: ${{ parameters.SHORTHAND_PROJECT_NAME }}
          TF_VAR_env: ${{ parameters.SHORTHAND_ENVIRONMENT_NAME }}
          TF_VAR_loc: ${{ parameters.SHORTHAND_LOCATION_NAME }}

          TF_VAR_TERRAFORM_STORAGE_RG_NAME: ${{ parameters.TERRAFORM_STORAGE_RG_NAME }}
          TF_VAR_TERRAFORM_STORAGE_ACCOUNT_NAME: ${{ parameters.TERRAFORM_STORAGE_ACCOUNT_NAME }}
          TF_VAR_TERRAFORM_BLOB_CONTAINER_NAME: ${{ parameters.TERRAFORM_BLOB_CONTAINER_NAME }}
          TF_VAR_TERRAFORM_STORAGE_KEY: ${{ parameters.TERRAFORM_STORAGE_KEY }}

          ARM_CLIENT_ID: ${{ parameters.AZURE_TARGET_CLIENT_ID }}
          ARM_CLIENT_SECRET: ${{ parameters.AZURE_TARGET_CLIENT_SECRET }}
          ARM_SUBSCRIPTION_ID: ${{ parameters.AZURE_TARGET_SUBSCRIPTION_ID }}
          ARM_TENANT_ID: ${{ parameters.AZURE_TARGET_TENANT_ID }}

      - pwsh: |
          New-Item -Path . -Name .terraform -ItemType "Directory" -Force ; `

          terraform init `
          -backend-config="storage_account_name=${{ parameters.TERRAFORM_STORAGE_ACCOUNT_NAME }}" `
          -backend-config="container_name=${{ parameters.TERRAFORM_BLOB_CONTAINER_NAME }}" `
          -backend-config="access_key=${{ parameters.TERRAFORM_STORAGE_KEY }}" `
          -backend-config="key=${{ parameters.TERRAFORM_STATE_NAME }}" ; `

          Write-Output "${{ parameters.TERRAFORM_WORKSPACE_NAME }}" > .terraform/environment ; `

          terraform workspace new "${{ parameters.TERRAFORM_WORKSPACE_NAME }}" ; `
          terraform workspace select "${{ parameters.TERRAFORM_WORKSPACE_NAME }}" ; `

          terraform validate ; `

          terraform plan -destroy -out pipeline.plan

          terraform apply pipeline.plan
        displayName: 'Terraform Init, Validate, Plan and Apply Destroy'
        workingDirectory: "${{ parameters.TERRAFORM_PATH }}"
        continueOnError: false
        enabled: true
        env:
          TF_VAR_short: ${{ parameters.SHORTHAND_PROJECT_NAME }}
          TF_VAR_env: ${{ parameters.SHORTHAND_ENVIRONMENT_NAME }}
          TF_VAR_loc: ${{ parameters.SHORTHAND_LOCATION_NAME }}

          TF_VAR_TERRAFORM_STORAGE_RG_NAME: ${{ parameters.TERRAFORM_STORAGE_RG_NAME }}
          TF_VAR_TERRAFORM_STORAGE_ACCOUNT_NAME: ${{ parameters.TERRAFORM_STORAGE_ACCOUNT_NAME }}
          TF_VAR_TERRAFORM_BLOB_CONTAINER_NAME: ${{ parameters.TERRAFORM_BLOB_CONTAINER_NAME }}
          TF_VAR_TERRAFORM_STORAGE_KEY: ${{ parameters.TERRAFORM_STORAGE_KEY }}

          ARM_CLIENT_ID: ${{ parameters.AZURE_TARGET_CLIENT_ID }}
          ARM_CLIENT_SECRET: ${{ parameters.AZURE_TARGET_CLIENT_SECRET }}
          ARM_SUBSCRIPTION_ID: ${{ parameters.AZURE_TARGET_SUBSCRIPTION_ID }}
          ARM_TENANT_ID: ${{ parameters.AZURE_TARGET_TENANT_ID }}

  - ${{ if and(eq(parameters.TERRAFORM_DESTROY, true), eq(parameters.TERRAFORM_PLAN_ONLY, true)) }}:

        - pwsh: |
            New-Item -Path . -Name .terraform -ItemType "Directory" -Force ; `

            terraform init `
            -backend-config="storage_account_name=${{ parameters.TERRAFORM_STORAGE_ACCOUNT_NAME }}" `
            -backend-config="container_name=${{ parameters.TERRAFORM_BLOB_CONTAINER_NAME }}" `
            -backend-config="access_key=${{ parameters.TERRAFORM_STORAGE_KEY }}" `
            -backend-config="key=${{ parameters.TERRAFORM_STATE_NAME }}" ; `

            Write-Output "${{ parameters.TERRAFORM_WORKSPACE_NAME }}" > .terraform/environment ; `

            terraform workspace new "${{ parameters.TERRAFORM_WORKSPACE_NAME }}" ; `
            terraform workspace select "${{ parameters.TERRAFORM_WORKSPACE_NAME }}" ; `

            terraform validate ; `

            terraform plan -destroy -out pipeline.plan
          displayName: 'Terraform Init, Validate & Plan Destroy'
          workingDirectory: "${{ parameters.TERRAFORM_PATH }}"
          continueOnError: false
          enabled: true
          env:
            TF_VAR_short: ${{ parameters.SHORTHAND_PROJECT_NAME }}
            TF_VAR_env: ${{ parameters.SHORTHAND_ENVIRONMENT_NAME }}
            TF_VAR_loc: ${{ parameters.SHORTHAND_LOCATION_NAME }}

            TF_VAR_TERRAFORM_STORAGE_RG_NAME: ${{ parameters.TERRAFORM_STORAGE_RG_NAME }}
            TF_VAR_TERRAFORM_STORAGE_ACCOUNT_NAME: ${{ parameters.TERRAFORM_STORAGE_ACCOUNT_NAME }}
            TF_VAR_TERRAFORM_BLOB_CONTAINER_NAME: ${{ parameters.TERRAFORM_BLOB_CONTAINER_NAME }}
            TF_VAR_TERRAFORM_STORAGE_KEY: ${{ parameters.TERRAFORM_STORAGE_KEY }}

            ARM_CLIENT_ID: ${{ parameters.AZURE_TARGET_CLIENT_ID }}
            ARM_CLIENT_SECRET: ${{ parameters.AZURE_TARGET_CLIENT_SECRET }}
            ARM_SUBSCRIPTION_ID: ${{ parameters.AZURE_TARGET_SUBSCRIPTION_ID }}
            ARM_TENANT_ID: ${{ parameters.AZURE_TARGET_TENANT_ID }}

```

## Example Call of Template

```yaml
---
name: $(Build.DefinitionName)-$(date:yyyyMMdd)$(rev:.r)

trigger: none

# parameters are typed with defaults so they are correctly populated, you will get a choice in the GUI to edit these, but you should keep all changes as code.
parameters:

  - name: SHORTHAND_ENVIRONMENT_NAME
    default: dev
    displayName: "What is the shorthand name for this environment?"
    type: string
    values:
      - dev
      - poc
      - mvp
      - tst
      - uat
      - ppd
      - prd

  - name: SHORTHAND_PROJECT_NAME
    type: string
    default: "ldo"
    displayName: "Shorthand Project e.g. lbdo for libredevops"

  - name: SHORTHAND_LOCATION_NAME
    type: string
    default: "euw"
    displayName: "3 character location name, e.g., uks, ukw, euw"

  - name: TERRAFORM_PATH
    type: string
    default: "$(Build.SourcesDirectory)/azure-pipelines-module-development-build/terraform"
    displayName: "What is the path to your terraform code?"

  - name: TERRAFORM_VERSION
    type: string
    default: "1.1.7"
    displayName: "Which version of Terraform should be installed?"

  - name: VARIABLE_GROUP_NAME
    type: string
    default: "svp-kv-ldo-euw-dev-mgt-01"
    displayName: "Enter the variable group which contains your authentication information"

# This variable sets up a condition in the template, if set to true, it will run terraform plan -destroy instead of the normal plan
  - name: TERRAFORM_DESTROY
    default: false
    displayName: "Check box to run a Destroy"
    type: boolean
  
  - name: TERRAFORM_PLAN_ONLY
    default: true
    displayName: "Check box to run plan ONLY and never run apply"
    type: boolean

  - name: CHECKOV_SKIP_TESTS
    type: string
    default: ' '
    displayName: "CheckOV tests to skip if comment skips don't work.  All checks run if parameter is empty, empty by default"

# Declare variable group to pass variables to parameters, in this case, a libre-devops keyvault which is using a service principle for authentication
variables:
  - group: ${{ parameters.VARIABLE_GROUP_NAME }}

# Sets what repos need cloned, for example, a library repo for modules and a poly-repo for target code
resources:
  repositories:

  - repository: azure-naming-convention
    type: github
    endpoint: github_service_connection
    name: libre-devops/azure-naming-convention
    ref: main

  - repository: azure-pipelines-module-development-build
    type: github
    endpoint: github_service_connection
    name: libre-devops/azure-pipelines-module-development-build
    ref: dev

# You may wish to use a separate or self-hosted agent per job, by default, all jobs will inherit stage agent
pool:
  name: Azure Pipelines
  vmImage: ubuntu-latest

# Sets stage so that multiple stages can be used if needed, as it stands, only 1 stage is expected and is thus passed as a parameter
stages:
  - stage: "${{ parameters.SHORTHAND_ENVIRONMENT_NAME }}"
    displayName: "${{ parameters.SHORTHAND_ENVIRONMENT_NAME }} Stage"
    jobs:
      - job: Terraform_Build
        workspace:
          clean: all
        displayName: Terraform Build
        steps:

          # Declare the repos needed from the resources list
          - checkout: self
          - checkout: azure-naming-convention

          # Remotely fetch pipeline template, in this case, I am using one in my development repo.
          - template: /templates/terraform-cicd-template.yml@azure-pipelines-module-development-build
            parameters:
              SHORTHAND_PROJECT_NAME: ${{ parameters.SHORTHAND_PROJECT_NAME }} # Parameters entered in YAML
              SHORTHAND_ENVIRONMENT_NAME: ${{ parameters.SHORTHAND_ENVIRONMENT_NAME }}
              SHORTHAND_LOCATION_NAME: ${{ parameters.SHORTHAND_LOCATION_NAME }}
              TERRAFORM_PATH: ${{ parameters.TERRAFORM_PATH }}
              TERRAFORM_VERSION: ${{ parameters.TERRAFORM_VERSION }}
              TERRAFORM_DESTROY: ${{ parameters.TERRAFORM_DESTROY }}
              TERRAFORM_PLAN_ONLY: ${{ parameters.TERRAFORM_PLAN_ONLY }}
              TERRAFORM_STORAGE_RG_NAME: $(SpokeSaRgName) # Key Vault variable
              TERRAFORM_STORAGE_ACCOUNT_NAME: $(SpokeSaName)
              TERRAFORM_BLOB_CONTAINER_NAME: $(SpokeSaBlobContainerName)
              TERRAFORM_STORAGE_KEY: $(SpokeSaPrimaryKey)
              TERRAFORM_STATE_NAME: "${{ parameters.SHORTHAND_PROJECT_NAME }}-${{ parameters.SHORTHAND_LOCATION_NAME }}.terraform.tfstate"
              TERRAFORM_WORKSPACE_NAME: $(System.StageName)
              TERRAFORM_COMPLIANCE_PATH: "$(Build.SourcesDirectory)/azure-naming-convention/az-terraform-compliance-policy"
              AZURE_TARGET_CLIENT_ID: $(SpokeSvpClientId)
              AZURE_TARGET_CLIENT_SECRET: $(SpokeSvpClientSecret)
              AZURE_TARGET_TENANT_ID: $(SpokeTenantId)
              AZURE_TARGET_SUBSCRIPTION_ID: $(SpokeSubID)
              CHECKOV_SKIP_TESTS: ${{ parameters.CHECKOV_SKIP_TESTS }}

```

# Use the main copy

Want to use the copy held here rather than copying it locally?  Try something like this

```yaml
---
name: $(Build.DefinitionName)-$(date:yyyyMMdd)$(rev:.r)

trigger: none

# parameters are typed with defaults so they are correctly populated, you will get a choice in the GUI to edit these, but you should keep all changes as code.
parameters:

  - name: SHORTHAND_ENVIRONMENT_NAME
    default: dev
    displayName: "What is the shorthand name for this environment?"
    type: string
    values:
      - dev
      - poc
      - mvp
      - tst
      - uat
      - ppd
      - prd

  - name: SHORTHAND_PROJECT_NAME
    type: string
    default: "ldo"
    displayName: "Shorthand Project e.g. lbdo for libredevops"

  - name: SHORTHAND_LOCATION_NAME
    type: string
    default: "euw"
    displayName: "3 character location name, e.g., uks, ukw, euw"

  - name: TERRAFORM_PATH
    type: string
    default: "$(Build.SourcesDirectory)/azure-pipelines-module-development-build/terraform"
    displayName: "What is the path to your terraform code?"

  - name: TERRAFORM_VERSION
    type: string
    default: "1.1.7"
    displayName: "Which version of Terraform should be installed?"

  - name: VARIABLE_GROUP_NAME
    type: string
    default: "svp-kv-ldo-euw-dev-mgt-01"
    displayName: "Enter the variable group which contains your authentication information"

# This variable sets up a condition in the template, if set to true, it will run terraform plan -destroy instead of the normal plan
  - name: TERRAFORM_DESTROY
    default: false
    displayName: "Check box to run a Destroy"
    type: boolean
  
  - name: TERRAFORM_PLAN_ONLY
    default: true
    displayName: "Check box to run plan ONLY and never run apply"
    type: boolean

  - name: CHECKOV_SKIP_TESTS
    type: string
    default: ' '
    displayName: "CheckOV tests to skip if comment skips don't work.  All checks run if parameter is empty, empty by default"

# Declare variable group to pass variables to parameters, in this case, a libre-devops keyvault which is using a service principle for authentication
variables:
  - group: ${{ parameters.VARIABLE_GROUP_NAME }}

# Sets what repos need cloned, for example, a library repo for modules and a poly-repo for target code
resources:
  repositories:

  - repository: azure-naming-convention
    type: github
    endpoint: github_service_connection
    name: libre-devops/azure-naming-convention
    ref: main

  - repository: azure-pipelines-module-development-build
    type: github
    endpoint: github_service_connection
    name: libre-devops/azure-pipelines-module-development-build
    ref: dev

  - repository: terraform-azdo-pipeline-template
    type: github
    endpoint: github_service_connection
    name: libre-devops/terraform-azdo-pipeline-template
    ref: main

# You may wish to use a separate or self-hosted agent per job, by default, all jobs will inherit stage agent
pool:
  name: Azure Pipelines
  vmImage: ubuntu-latest

# Sets stage so that multiple stages can be used if needed, as it stands, only 1 stage is expected and is thus passed as a parameter
stages:
  - stage: "${{ parameters.SHORTHAND_ENVIRONMENT_NAME }}"
    displayName: "${{ parameters.SHORTHAND_ENVIRONMENT_NAME }} Stage"
    jobs:
      - job: Terraform_Build
        workspace:
          clean: all
        displayName: Terraform Build
        steps:

          # Declare the repos needed from the resources list
          - checkout: self
          - checkout: azure-naming-convention

          # Remotely fetch pipeline template, in this case, I am using one in my development repo.
          - template: /.azurepipelines/.templates/terraform-cicd-template.yml@terraform-azdo-pipeline-template
            parameters:
              SHORTHAND_PROJECT_NAME: ${{ parameters.SHORTHAND_PROJECT_NAME }} # Parameters entered in YAML
              SHORTHAND_ENVIRONMENT_NAME: ${{ parameters.SHORTHAND_ENVIRONMENT_NAME }}
              SHORTHAND_LOCATION_NAME: ${{ parameters.SHORTHAND_LOCATION_NAME }}
              TERRAFORM_PATH: ${{ parameters.TERRAFORM_PATH }}
              TERRAFORM_VERSION: ${{ parameters.TERRAFORM_VERSION }}
              TERRAFORM_DESTROY: ${{ parameters.TERRAFORM_DESTROY }}
              TERRAFORM_PLAN_ONLY: ${{ parameters.TERRAFORM_PLAN_ONLY }}
              TERRAFORM_STORAGE_RG_NAME: $(SpokeSaRgName) # Key Vault variable
              TERRAFORM_STORAGE_ACCOUNT_NAME: $(SpokeSaName)
              TERRAFORM_BLOB_CONTAINER_NAME: $(SpokeSaBlobContainerName)
              TERRAFORM_STORAGE_KEY: $(SpokeSaPrimaryKey)
              TERRAFORM_STATE_NAME: "${{ parameters.SHORTHAND_PROJECT_NAME }}-${{ parameters.SHORTHAND_LOCATION_NAME }}.terraform.tfstate"
              TERRAFORM_WORKSPACE_NAME: $(System.StageName)
              TERRAFORM_COMPLIANCE_PATH: "$(Build.SourcesDirectory)/azure-naming-convention/az-terraform-compliance-policy"
              AZURE_TARGET_CLIENT_ID: $(SpokeSvpClientId)
              AZURE_TARGET_CLIENT_SECRET: $(SpokeSvpClientSecret)
              AZURE_TARGET_TENANT_ID: $(SpokeTenantId)
              AZURE_TARGET_SUBSCRIPTION_ID: $(SpokeSubID)
              CHECKOV_SKIP_TESTS: ${{ parameters.CHECKOV_SKIP_TESTS }}
```