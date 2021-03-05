# Azure DevOps (Example) Pipeline
## Azure Policies to Enable Azure Diagnostics
## INTRODUCTION
Leverage this repository as an example to get started deploying Azure Diagnostics via Custom Azure Policies wrapped in a policy initiative ARM template.  This DevOps pipeline allows you to dynamically build the necessary custom Azure Policies to configure your PaaS resources in Azure that support Azure Diagnostics.  The pipeline can be run on a schedule (example: daily) to ensure that your resources are always configured with the latest Azure Diagnostic configurations and stay compliant.

1. Pipeline automatically pulls down the scripts from https://aka.ms/AzPolicyScripts with an optional parameter to not automatically update once initially downloaded

1. Runs an assessment against your Azure Subscription to determine what ResourceTypes are available that support Azure Diagnostics.

1. Creates the custom policies for each ResourceType and creates an ARM template Policy Initiative that can be leveraged to deploy directly to your Azure Subscription.

1. As updates are made to ResourceTypes or on first execution 

    a. New resources that support Azure Diagnostics or new categories / metrics are enabled for a ResourceType.

    b. A new ARM template is generated, compared against the baseline ARM template in your Git, promotes it to your new baseline (capturing changes in Git history)

    c. Initiates a policy refresh against your subscription.

    d. Then remediates the Azure Policy Initiative to ensure all resourceTypes are configured for Azure Diagnostics.

## CURRENT KNOWN LIMITATIONS & RESTRICTIONS

* The scripts (in their current release) and the pipeline that is leveraged to run them do not support Subscription Level Diagnostic Logs (ex: Activity Logs).

* The scripts (in their current release) do not support proxy resources off of Azure Storage Accounts (example: blobs, files, tables, queues).

* Currently this pipeline is configured to target a subscription scope (not a management group scope) though once imported it can obviously be modified to do so.

* The target sink point for Azure Diagnostics is Log Analytics for this pipeline.  Event Hub and Azure Storage Sink points are out of scope for this pipeline configuration.

* This is meant for evaluation / pre-production testing to validate that this will support your environment. Test! Test! Test!

## IMPORTING PIPELINE FROM GITHUB TO AZURE DEVOPS

The pipeline example source file is located here: https://github.com/jimgbritt/AzureDiagnosticsPipeline

1. Select the Code dropdown
1. Copy the URL to the source on GitHub to be used in a later step

1. Create a new project in your Azure DevOps instance (I called mine AzureDiagnostics) and feel free to make this a private repo as it is not necessary to publish this publicly.
1. Provide a description.
1. Set the configuration appropriate to your needs and select Create.
 
1. Once created, select Repos from the options available (as we are going to be selecting our GitHub example repo mentioned previously.
 
1. Click Import under Import a repository.
1. Select Git from the repository type.
1. Paste the clone URL you copied to the clipboard earlier or type it in directly.
1. Select Import.
 
1. Wait for the Import Successful! screen to indicate things are done importing.

## CONFIGURATION OF PIPELINE

Once things are imported, there are a few configurations that need to be done to ensure this can work within your environment.
## SERVICE CONNECTION CONFIGURATION (SPN)

You need to create a Service Connection to your Azure Subscription that has rights to connect to your environment, create an ARM template deployment, assign a policy initiative, initiate the policy compliance evaluation and finally policy initiative remediation.

1. Select Project settings on the main page (lower left).
 
1. Service Connections / Create service connection.
 
1. Type in Azure and select Azure Resource Manager from the connection type options.
 
1. I selected and utilized Service principal (automatic) which is the recommended option.
1. Select Next to continue.
 
1. For this example (and what is currently supported with the pipeline provided) select Subscription.
1. Select a resource group.
1. Give it a name under Service connection name (and note this down as we’ll use it later).
1. Finally, select the checkbox to Grant permissions to all pipelines.
1. Save to continue.
 
1. At this stage you should have your named Service Connection available (mine is called AzureDiagsPipelineSPN).
 
## IMPORTING PIPELINE FROM GIT SOURCE (LOCAL REPO)

The next step is to import your pipeline so that you can leverage it on demand or on a schedule as needed.

1. Select Pipelines / Create Pipeline.
 
1. Select Azure Repos Git (YAML).
 
1. Select the AzureDiagnostics repo (or other name if you renamed it upon import).
 
1. Select Existing Azure Pipelines YAML file from the options available.
 
1. Select /diagnostic-policy.yml from the list on the right and click Continue.
 
## MODIFICATIONS NEEDED TO PIPELINE SOURCE

There are some modifications needed for this pipeline to work in your environment. 

### Diagnostic-Policy.Yml

The pipeline comes with two target “environments” to show the option of targeting two different subs for different purposes.  In this case Integration and PreProd are my two example environments.  
 
### Default Target Environments

In the below example, if these values were left as is, upon execution of the pipeline you’d see the option for two different environments (Integration and PreProd and Integration is the default option that shows).  You should update according to your needs (even reduce to one environment if desired)
parameters:

```
- name: environment
  displayName: 'Environment to deploy'
  type: string
  default: Integration
  values:
  - Integration
  - PreProd
```

### Download of Scripts from GitHub Repo (https://aka.ms/AzPolicyScripts) 

Scripts from https://aka.ms/AzPolicyScripts can be automatically kept up to date.  If you want to manage this on your own, set default to false otherwise, we’ll ensure you always have the latest in this pipeline configuration.
```
- name: fromGitHub
  displayName: 'Get scripts from GitHub'
  type: boolean
  default: true
```

### Target Environment Variable References

The above environments that you setup then seed the next logic set to determine what variable file(s) to leverage for execution.
variables:

```
- ${{ if eq(parameters.environment, 'Integration') }}:
  - template: diagnostic-policy-integration.variables.yml
- ${{ if eq(parameters.environment, 'PreProd') }}:
  - template: diagnostic-policy-preprod.variables.yml
```

### Create-AzDiagPolicy.PS1 script behavior

If you want to change how the policies are built, according to your target environment, you’ll want to review the script options at https://aka.ms/AzPolicyScripts and customize the below section accordingly (specifically the bolded ones below)
```
        $(ScriptPath)/Create-AzDiagPolicy.ps1 -ExportDir '$(ExportDir)' -ExportAll -ExportLA -AllRegions -ValidateJSON -ExportInitiative `
          -InitiativeDisplayName '$(InitiativeDisplayName)' -TemplateFileName '$(TemplateFileName)' `
          -SubscriptionId $context.Subscription.id
```

### Updating Git User / User Email details and commit messages

The default below should work fine but if you are interested in configuring another user that will show up on Git commits as well as an email address and the actual commit message, update the below.

```
  - task: CmdLine@2
    displayName: 'Push template to the $(Build.SourceBranchName) branch'
    condition: and(succeeded(), eq(variables.isChanged, true))
    inputs:
      script: |
        git config --global user.email ""
        git config --global user.name "AzureDiagnosticsPipeline"
        git checkout $(Build.SourceBranchName)
        git add --all
        git commit -m "Update Diagnostics Policy Initiative ARM template"
        git push
```

## Pipeline Variable File (diagnostic-policy.variables.yml)

The Diagnostic-policy.variables.yml file has the top level variables that apply to all target environments.  

### Variable details

* ScriptPath: this is the path in your Git repo where the policy generation scripts will be downloaded and sourced for execution during the pipeline.  It starts with a readme.md marker and will be populated with the scripts on initial run.
* armTemplatePath: these are templates to assist in applying role assignments as well as the actual policy assignment during pipeline execution.
* PolicyTemplatePath: this directory is where the baseline ARM template Policy Initiative will be sourced and referenced as your “currently deployed initiative”.  You will have an ARM template for each environment you target.
* InitiativeDisplayName: this is the display name that you want to have for your Policy Initiative.  You can keep the default or update according to your needs.
* ContributorRoleDefinitionId: We are leveraging the Contributor RBAC Role the rights required for the Policy Initiative to ensure it has rights to do what it needs to do remediation.  The value in this variable is the RBAC role object ID that will be leveraged.  Given we are using the SDK, we have to provide rights to the managed identity once the initiative is assigned and ID is created to ensure it has the rights it needs to do remediation on resources as needed.  See this link for more information: Remediate non-compliant resources - Azure Policy | Microsoft Docs. This variable is leveraged in the diagnostic-policy.yml file to assign the rights (once assignment is successful).

```
        $templateParameters = @{
          principalId = $policyAssignment.Outputs.principalId.value
          roleDefinitionId = '$(contributorRoleDefinitionId)'
        }

        New-AzDeployment -Name 'Assign_Role_To_Initiative_$(Build.BuildNumber)' -Location $(location) `
          -TemplateFile '$(armTemplatePath)/Microsoft.Authorization/roleAssignment.json' `
          -TemplateParameterObject $templateParameters -Verbose
```

* Policy.profileName: This value is the name that the Azure Diagnostics configuration will set.  This is the name you will see in the Azure Diagnostics blade for the resource when you review what logs and metrics have been selected.
* *LogAnalyticsName: This is your target Log Analytics workspace.  

## Variable File Updates Needed [ex: diagnostic-integration-variables.xml]

The environment specific variable file contains specific details that allow you to further cater to that environment such as prefix / suffix for leveraging that further in separating artifacts as well as the SPN that will be used for that environment.
 
* Prefix: leveraged to create a prefix on the ARM template export.
* Suffix: will be appended to the end of the ARM template export file (not used in this example).
* AzureSubscriptionEndpoint: This is your Service Connection you created at the beginning to connect to Azure.
* Location: Update this according to your target region preferred for your ARM demployment.

## SETTING UP RIGHTS

Service Connection (SPN) Rights in Subscription Target
You will initially need to setup rights for your SPN that you previously created in the Service Connection section of this how-to.  I have setup owner rights but you can certainly define according to least rights privilege.  This SPN needs to have enough rights to assign rights to the managed identity for your policy assignments, etc.

**Note**: *The SPN created appears to have the naming of subscriptionName-RepositoryName-subID.*
 
### Service Connection Rights in Azure DevOps Git Repository

In order for your Service Connection to commit to your Git the necessary files (scripts and ARM templates that are generated from your policy initiative creation) you need to provide your Service Connection Contributor Rights to your Repositories.
 
## DEPLOYMENT TESTING

To test your pipeline browse to pipelines, locate your named pipeline (in my case AzureDiagnostics), click the three dots on the right and select Run pipeline. If you have multiple environments, you’ll see options to select between them as well as the option to get scripts from GitHub (these are the scripts at https://aka.ms/AzPolicyScripts).  For those risk averse folks (not a bad idea 😊), there is an option to source these yourself if you’d rather control the update of these scripts in source as they are updated.
 
Next you can click on the job and monitor the steps as they execute
 
The entire process generally takes about 15 mins max on the first run and if no changes are picked up (no drift in policies from the last run) the execution is very efficient in just a couple of minutes.
 

## VALIDATION

To validate that everything is working, you should see a Policy Initiative Assigned within the target subscription you have defined.
 
You can click on View Definition to see what policies have been created and can drill into those to get specific information.
 
Next, if you click on the Remediation section within Azure Policy and look at Remediation Tasks you should see the operations getting fired (individual remediation for each policy within your policy initiative) and the results.
 
**Note**: *You may have to do a hard refresh of this page to update the compliance details after remediation tasks have completed.*

## Troubleshooting

### ARM Template Fails to Deploy

If you receive a message during the Deploy-ARMTemplateExport template phase of the pipeline (see below) please refer back to this section in this how-to: Service Connection (SPN) Rights in Subscription Target to ensure your Service Connection has the necessary rights to execute the pipeline.
 
### Pushing to Main Branch (Git) Fails

If you receive a message during the Push template to the main branch phase of the pipeline (see below) please refer to the section titled Service Connection Rights in Azure DevOps Git Repository of this how-to to ensure your Service Connection has the rights it needs to Git.
 
