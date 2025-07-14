





```yaml
# Generic Azure Data Factory CI/CD Pipeline
# Replace the placeholders with your specific values:
# - YOUR_SUBSCRIPTION_ID: Your Azure subscription ID
# - YOUR_PROJECT_NAME: Your project name (used in resource naming)
# - YOUR_SERVICE_CONNECTION: Your Azure DevOps service connection name
# - YOUR_RESOURCE_GROUP: Your resource group name
# - YOUR_ADF_NAME: Your Azure Data Factory name
# - YOUR_LOCATION: Your Azure region (e.g., 'east us', 'west europe')
# - YOUR_POOL_NAME: Your Azure DevOps agent pool name
# - YOUR_ENVIRONMENT_NAMES: Your environment names in Azure DevOps

trigger:
  branches:
    include:
      - development
      - uat
      - master

pool:
  name: YOUR_POOL_NAME  # Replace with your agent pool name (e.g., 'Azure Pipelines', 'Default', etc.)

variables:
  system.debug: true
  # Format current date/time as YYYYMMDDHHmm
  timestamp: $[format('{0:yyyyMMddHHmm}', pipeline.startTime)]
  deploymentName: 'ADFDeploy-$(Build.BuildId)-$(timestamp)'
  
  # Project-specific variables - Replace these with your values
  subscriptionId: 'YOUR_SUBSCRIPTION_ID'  # Replace with your subscription ID
  projectName: 'YOUR_PROJECT_NAME'        # Replace with your project name
  location: 'YOUR_LOCATION'               # Replace with your Azure region

stages:
  # -------------------------------------------------------------------------------------------
  # Commit, Build and Publish
  # -------------------------------------------------------------------------------------------
  - stage: 'Build'
    displayName: 'Build'

    jobs:
      - job: 'build_job'
        displayName: 'Build Stage'

        steps:

          - checkout: self
            displayName: 'Check out Branch'

          - script: echo $(Build.SourceBranch) $(Build.Repository.LocalPath)
            displayName: 'Build Source Branch'

          - task: UseNode@1
            displayName: 'Install Node.js'
            inputs:
             version: '18.x'

          - script: |
             npm install
            workingDirectory: '$(Build.SourcesDirectory)'
            displayName: 'npm install'
            # Installs Node and the npm packages saved in your package.json file in the build

          - task: Npm@1
            inputs:
             command: 'custom'
             workingDir: '$(Build.Repository.LocalPath)' # Replace with the package.json folder
             customCommand: 'run build validate $(Build.Repository.LocalPath) /subscriptions/$(subscriptionId)/resourceGroups/$(projectName)-dev-rg/providers/Microsoft.DataFactory/factories/$(projectName)-dev-adf'
            displayName: 'Validate ADF'

          - task: Npm@1
            inputs:
              command: 'custom'
              workingDir: '$(Build.Repository.LocalPath)' # Replace with the package.json folder
              customCommand: 'run build export $(Build.Repository.LocalPath) /subscriptions/$(subscriptionId)/resourceGroups/$(projectName)-dev-rg/providers/Microsoft.DataFactory/factories/$(projectName)-dev-adf "ArmTemplate"'
            displayName: 'Validate and Generate ARM Template'

          - task: PublishPipelineArtifact@1
            inputs:
             targetPath: '$(Build.Repository.LocalPath)/ArmTemplate' # Replace with the package.json folder
             artifact: 'adfartifact'
             publishLocation: 'pipeline'

  # -------------------------------------------------------------------------------------------
  # Deploy to Development Environment
  # -------------------------------------------------------------------------------------------
  - stage: Deploy_Dev_ADF
    displayName: 'Deploy to Development'
    dependsOn: Build
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/development'))
    jobs:
      - deployment: DeployADF
        environment: Deploy_Dev_ADF  # Replace with your environment name
        strategy:
          runOnce:
            deploy:
              steps:
                - task: DownloadPipelineArtifact@2
                  inputs:
                    artifact: 'adfartifact'
                    path: '$(Build.ArtifactStagingDirectory)'

                - task: AzureResourceGroupDeployment@2
                  displayName: 'ADF Development Deployment'
                  inputs:
                     azureSubscription: 'YOUR_SERVICE_CONNECTION_DEV'  # Replace with your dev service connection
                     action: 'Create Or Update Resource Group'
                     resourceGroupName: '$(projectName)-dev-rg'       # Replace with your dev resource group
                     location: '$(location)'
                     templateLocation: 'Linked artifact'
                     csmFile: '$(Build.ArtifactStagingDirectory)/ARMTemplateForFactory.json'
                     csmParametersFile: '$(Build.ArtifactStagingDirectory)/ARMTemplateParametersForFactory.json'
                     overrideParameters: '-factoryName $(projectName)-dev-adf'  # Replace with your dev ADF name
                     deploymentName: '$(deploymentName)'
                     deploymentMode: 'Incremental'

                - task: AzureCLI@2
                  condition: always()
                  displayName: 'Show Failed Deployment Operations'
                  inputs:
                    azureSubscription: 'YOUR_SERVICE_CONNECTION_DEV'   # Replace with your dev service connection
                    scriptType: bash
                    scriptLocation: inlineScript
                    inlineScript: |
                      az rest --method get --url "https://management.azure.com/subscriptions/$(subscriptionId)/resourceGroups/$(projectName)-dev-rg/providers/Microsoft.Resources/deployments/$(deploymentName)/operations?api-version=2021-04-01" \
                      --query "value[?properties.provisioningState=='Failed'].{Id:id, Resource:properties.targetResource.resourceType, Status:properties.provisioningState, Timestamp:properties.timestamp}" \
                      --output table

  # -------------------------------------------------------------------------------------------
  # Deploy to UAT Environment
  # -------------------------------------------------------------------------------------------
  - stage: Deploy_UAT_ADF
    displayName: 'Deploy to UAT'
    dependsOn: Build
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/uat'))
    jobs:
      - deployment: DeployADF
        environment: Deploy_UAT_ADF  # Replace with your UAT environment name
        strategy:
          runOnce:
            deploy:
              steps:
                - task: DownloadPipelineArtifact@2
                  inputs:
                    artifact: 'adfartifact'
                    path: '$(Build.ArtifactStagingDirectory)'

                - task: AzureResourceGroupDeployment@2
                  displayName: 'ADF UAT Deployment'
                  inputs:
                     azureSubscription: 'YOUR_SERVICE_CONNECTION_UAT'  # Replace with your UAT service connection
                     action: 'Create Or Update Resource Group'
                     resourceGroupName: '$(projectName)-uat-rg'       # Replace with your UAT resource group
                     location: '$(location)'
                     templateLocation: 'Linked artifact'
                     csmFile: '$(Build.ArtifactStagingDirectory)/ARMTemplateForFactory.json'
                     csmParametersFile: '$(Build.ArtifactStagingDirectory)/ARMTemplateParametersForFactory.json'
                     overrideParameters: '-factoryName $(projectName)-uat-adf'  # Replace with your UAT ADF name
                     deploymentName: '$(deploymentName)'
                     deploymentMode: 'Incremental'

                - task: AzureCLI@2
                  condition: always()
                  displayName: 'Show Failed Deployment Operations'
                  inputs:
                    azureSubscription: 'YOUR_SERVICE_CONNECTION_UAT'   # Replace with your UAT service connection
                    scriptType: bash
                    scriptLocation: inlineScript
                    inlineScript: |
                      az rest --method get --url "https://management.azure.com/subscriptions/$(subscriptionId)/resourceGroups/$(projectName)-uat-rg/providers/Microsoft.Resources/deployments/$(deploymentName)/operations?api-version=2021-04-01" \
                      --query "value[?properties.provisioningState=='Failed'].{Id:id, Resource:properties.targetResource.resourceType, Status:properties.provisioningState, Timestamp:properties.timestamp}" \
                      --output table
            
  # -------------------------------------------------------------------------------------------
  # Deploy to Production Environment
  # -------------------------------------------------------------------------------------------
  - stage: Deploy_Prod_ADF
    displayName: 'Deploy to Production'
    dependsOn: Build
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/master'))  # Fixed condition for master branch
    jobs:
      - deployment: DeployADF
        environment: Deploy_Prod_ADF  # Replace with your production environment name
        strategy:
          runOnce:
            deploy:
              steps:
                # Optional: Change Management Integration Task
                # Remove or replace this task if you don't use similar change management
                # - task: ChangeManagementTask@1  # Replace with your change management task if applicable
                #   inputs:
                #     enableChangeManagementIntegration: true
                #     changeTemplateService: 'YOUR_CHANGE_TEMPLATE_SERVICE'
                #     standardChangeId: 'YOUR_STANDARD_CHANGE_TEMPLATE'

                - task: DownloadPipelineArtifact@2
                  inputs:
                    artifact: 'adfartifact'
                    path: '$(Build.ArtifactStagingDirectory)'

                - task: AzureResourceGroupDeployment@2
                  displayName: 'ADF Production Deployment'
                  inputs:
                     azureSubscription: 'YOUR_SERVICE_CONNECTION_PROD'  # Replace with your production service connection
                     action: 'Create Or Update Resource Group'
                     resourceGroupName: '$(projectName)-prod-rg'       # Replace with your production resource group
                     location: '$(location)'
                     templateLocation: 'Linked artifact'
                     csmFile: '$(Build.ArtifactStagingDirectory)/ARMTemplateForFactory.json'
                     csmParametersFile: '$(Build.ArtifactStagingDirectory)/ARMTemplateParametersForFactory.json'
                     overrideParameters: '-factoryName $(projectName)-prod-adf'  # Replace with your production ADF name
                     deploymentName: '$(deploymentName)'
                     deploymentMode: 'Incremental'

                - task: AzureCLI@2
                  condition: always()
                  displayName: 'Show Failed Deployment Operations'
                  inputs:
                    azureSubscription: 'YOUR_SERVICE_CONNECTION_PROD'   # Replace with your production service connection
                    scriptType: bash
                    scriptLocation: inlineScript
                    inlineScript: |
                      az rest --method get --url "https://management.azure.com/subscriptions/$(subscriptionId)/resourceGroups/$(projectName)-prod-rg/providers/Microsoft.Resources/deployments/$(deploymentName)/operations?api-version=2021-04-01" \
                      --query "value[?properties.provisioningState=='Failed'].{Id:id, Resource:properties.targetResource.resourceType, Status:properties.provisioningState, Timestamp:properties.timestamp}" \
                      --output table

# -------------------------------------------------------------------------------------------
# Additional Configuration Notes:
# -------------------------------------------------------------------------------------------
# 1. Service Connections: Create Azure Resource Manager service connections in Azure DevOps
#    for each environment (dev, uat, prod) with appropriate permissions
#
# 2. Environments: Create environments in Azure DevOps under Pipelines > Environments
#    - Deploy_Dev_ADF
#    - Deploy_UAT_ADF  
#    - Deploy_Prod_ADF
#
# 3. Variable Groups (Optional): Consider using variable groups for environment-specific values
#
# 4. Branch Protection: Set up branch protection rules for master/main branch
#
# 5. Resource Naming Convention: 
#    - Resource Group: {projectName}-{environment}-rg
#    - Data Factory: {projectName}-{environment}-adf
#
# 6. Required Extensions:
#    - Azure Resource Group Deployment task
#    - Azure CLI task
#    - Node.js tool installer
#
# 7. Repository Structure:
#    - Ensure package.json exists in repository root
#    - ADF ARM templates will be generated in /ArmTemplate folder
#
# 8. Permissions Required:
#    - Contributor access to Azure subscription/resource groups
#    - Data Factory Contributor role for ADF deployments
