---
layout: post
title:  Azure AD Access Packages to the resque!
author: Rik Groenewoud
tags: microsoft azure azuread aad identitymanagement
---

## The challenge

For one of our customers we had the task to enroll end users into multiple Azure AD Groups in one go. We already had setup SSO via Azure AD and via the "App Roles" feature in Azure AD App Registration these AD groups connected to roles in the customer's application. Since we are talking about several hundreds of accounts and around 60+ different AD Groups we aimed to **automate** this process as much as possible. We initially tried to do this by writing a custom Powershell script. But because of the complex requirements and a lot of unique combinations of AD groups this attempt turned into a mess quite fast.

Luckily my good colleague [Hans Bakker](https://xpirit.com/team/hans-bakker/) came with the bright idea to use an Azure native solution for this use case: **Access Packages**.

## What are Azure AD Access Packages?

To describe this pretty comprehensive feature in a few words is not easy. To my mind access packages can be used to centralize multiple resources into one entity in which Azure AD users can be enrolled. These users can be added by administrators but also self-enrollment is possible with or without approval steps. These enrollments can have a set life time or can set to indefinitely. So it is as service created to manage and govern identities, internal and external in a centralized way. Furthermore, enrollment using the Graph API is possible. This makes it possible to automate the enrollment of users into these access packages. This is exactly what we needed!

## How did we leverage Access Packages?

To use access packages an Azure AD Premium P1 or P2 license is required. You can find more information about the different Azure AD Premium licenses [here](https://azure.microsoft.com/en-us/pricing/details/active-directory/). We decided to setup the actual packages themselves using the Azure Portal. Although the amount of groups grew overtime, it was no recurring task worth automating. See [this](https://learn.microsoft.com/en-us/azure/active-directory/governance/entitlement-management-access-package-create) documentation on how to do this. Each access package contains several AD Groups so when a user is enrolled into the access package he/she is automatically added to all the groups.

We did automate the enrollment of users into these access packages. This was done by using the Azure AD Graph API and Powershell. We created CSV files with user details per action group and the below powershell script to enroll them into the groups. By running an Azure pipeline the script gets executed. Using parameters we can easily run the script for as many CSV files as we want.

I know I am not the cleanest Powershell coder but I hope you can get the idea. The script is pretty straight forward. It imports the CSV file, authenticates to the Graph API, loops through the users and checks if the user already exists in Azure AD. If not, it will invite the user and wait for the user to become visible in Azure AD. After that it will enroll the user into the access package.

```powershell
#Import CSV file with users
$users = Import-Csv -Delimiter ";" -Path $path

#Authentication to Graph API
$appid =  [CLIENTID]
$tenantid = [TENANTID]
$secret = $secret
$body = @{
    Grant_Type    = "client_credentials"
    Scope         = "https://graph.microsoft.com/.default"
    Client_Id     = $appid
    Client_Secret = $secret
}

$connection = Invoke-RestMethod `
    -Uri https://login.microsoftonline.com/$tenantid/oauth2/v2.0/token `
    -Method POST `
    -Body $body

$token = $connection.access_token

Connect-MgGraph -AccessToken $token

Select-MgProfile -Name "beta"

#Loop through users
foreach ($user in $users) {
    $AADuser = Get-MgUser -Search "Mail:$($user.Email.Trim())" -ConsistencyLevel eventual

    if ($null -eq $AADuser) {
        #Invite user if not yet exists
        New-MgInvitation -InvitedUserEmailAddress "$($user.Email.Trim())" -InviteRedirectUrl "[URL]" -SendInvitationMessage:$true
        #Wait for user to become visible
        $counter = 0
        do {
            $AADuser = Get-MgUser -Search "Mail:$($user.Email.Trim())" -ConsistencyLevel eventual
            Write-Host "Try to find user $($user.Email.Trim()) attempt $($counter = $counter + 1) $counter"
        }
        while ($null -eq $AADuser)
        $accessPackage = Get-MgEntitlementManagementAccessPackage -DisplayNameEq $user.AccessPackage -ExpandProperty "accessPackageAssignmentPolicies"
        $policy = $accessPackage.AccessPackageAssignmentPolicies[0]
        New-MgEntitlementManagementAccessPackageAssignmentRequest -AccessPackageId $accessPackage.Id -AssignmentPolicyId $policy.Id -TargetId $AADuser.Id
        Write-Host "NEW User $($user.Displayname) is added to Access Package"
    }
    else {
        #Check if user already is added to Package
        $accesspackage = Get-MgEntitlementManagementAccessPackage -DisplayNameEq $user.AccessPackage -ExpandProperty "accessPackageAssignmentPolicies"
        $assignments = Get-MgEntitlementManagementAccessPackageAssignment -AccessPackageId $accesspackage.Id -ExpandProperty target -All -ErrorAction Stop | Where-Object { $_.AssignmentState -eq 'Delivered' }

        if ($null -eq $assignments) {
            $policy = $accessPackage.AccessPackageAssignmentPolicies[0]
            New-MgEntitlementManagementAccessPackageAssignmentRequest -AccessPackageId $accessPackage.Id -AssignmentPolicyId $policy.Id -TargetId $AADuser.Id
            Write-Host "User $($user.Displayname) is added to Access Package"
        }
        else {
            $check = Compare-Object -DifferenceObject $assignments.Target.ObjectId -ReferenceObject $AADuser.Id -ExcludeDifferent
            if ($null -eq $check) {
                $policy = $accessPackage.AccessPackageAssignmentPolicies[0]
                New-MgEntitlementManagementAccessPackageAssignmentRequest -AccessPackageId $accessPackage.Id -AssignmentPolicyId $policy.Id -TargetId $AADuser.Id
                Write-Host "User $($user.Displayname) is added to Access Package"
            }
            else {
                Write-Host "User $($user.Displayname) is already member of this Access Package"
            }
        }
    }
}

```

The pipeline looks like this:

```yaml

pool: "ERTMS-AzDoAgents-Ubuntu"

parameters:
  - name: paths
    type: object
    default:
      - users.csv
      - users2.csv
      - etc

    displayName: 'Pick the team CSV file you want to update'

variables:
  - group: variableGroup

stages:
  - stage: RunOnboardingScript
    displayName: Run script to add Users to Access Package
    jobs:
      - job: RunPSScript
        steps:
         - ${{ each path in parameters.paths }}:
            - task: AzurePowerShell@5
              displayName: 'Validate CSV delimiter and headers for ${{ path }}'
              inputs:
                azureSubscription: 'SubscriptionName'
                ScriptType: 'FilePath'
                ScriptPath: '$(Build.Repository.LocalPath)/Scripts/validateCSV.ps1'
                ScriptArguments:
                  -path '${{ path }}'
                azurePowerShellVersion: 'LatestVersion'
                pwsh: true

            - task: AzurePowerShell@5
              displayName: 'Run script for: ${{ path }}'
              inputs:
                azureSubscription: 'SubscriptionName'
                ScriptType: 'FilePath'
                ScriptPath: '$(Build.Repository.LocalPath)/Scripts/AddUsersToAccessPackage.ps1'
                ScriptArguments:
                  -path '${{ path }}' `
                  -secret '$(userOnboardingSecret)'
                azurePowerShellVersion: 'LatestVersion'
                pwsh: true

```

There is one additional task where I validate the CSV file. This is to make sure the CSV file is in the correct format.

To remove users from the access packages, I came up with an addition to the script shown below. Simply remove users from the CSV file and run the pipeline again for the desired CSV files. The script will check if the user is still in the CSV and if not, it will remove the user from the access package.

```powershell

## Remove users from Access Package
#Get latest assignments
$assignments = Get-MgEntitlementManagementAccessPackageAssignment -AccessPackageId $accesspackage.Id -ExpandProperty target -ErrorAction Stop | Where-Object { $_.ExpiredDateTime -eq $null }

$userid = $null
foreach ($user in $users) {
    $userid += @((Get-MgUser -Search "Mail:$($user.Email.Trim())" -ConsistencyLevel eventual).Id)
}

Write-Host "Assigned IDs: $($assignments.TargetId)"
Write-Host "CSV User IDs: $userid"

#Check if there are assignments other than users from CSV
$check = Compare-Object -DifferenceObject $assignments2.TargetId -ReferenceObject $userid

Write-Host "Differences in IDs: $($check.InputObject)"

if ($null -ne $check2) {
    foreach ($assignment in $check.InputObject) {
        #Get AssignmentId for user that has to be removed
        $assignmentId = ($assignments | Where-Object { $_.TargetId -eq $assignment }).Id
        New-MgEntitlementManagementAccessPackageAssignmentRequest -AccessPackageAssignmentId $assignmentId -RequestType "AdminRemove"
        Write-Host "Removed $assignment"
    }
}

else {
    Write-Host 'No users removed'
}
```

## Conclusion

We are very happy with the results of this project. It provides for a flexible and scalable solution that is easy to maintain. We can easily add new access packages and add users to them. Further, we can easily remove users from access packages.

I hope this blog post will help you to get started with Access Packages, the Microsoft Graph API and the Microsoft Graph PowerShell module. If you have any questions or comments, please let me know in the comments below.

If you want to learn more about access packages, I also wrote an article in the latest edition of the [XPRT Magazine](https://hubs.ly/Q01Pxdxx0) on this topic.

<script src="https://giscus.app/client.js"
        data-repo="RikGr/cloudwoud"
        data-repo-id="R_kgDOHLlC9w"
        data-category="Announcements"
        data-category-id="DIC_kwDOHLlC984CO_2O"
        data-mapping="pathname"
        data-reactions-enabled="0"
        data-emit-metadata="0"
        data-input-position="bottom"
        data-theme="light"
        data-lang="en"
        crossorigin="anonymous"
        async>
</script>
