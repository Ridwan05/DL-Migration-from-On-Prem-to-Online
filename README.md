# DL-Migration-from-On-Prem-to-Online

## Step 1 – Exchange On-Premise: export all information for distribution groups
### Exchange On-Premise: Export data

We’re going to export distribution groups, their settings, SMTP aliases, and members from Exchange On-Premise into three different files. Here are the PowerShell scripts you’ll need to run. (Note: If you need localization exports/imports, see my blog here on foreign characters ). By the way, some commenters have reported issues with ‘Alias‘ field (in EXO) having issues with foreign characters, and they had to go back and adjust these characters during clean-up step 2, for example change (Ä –>A)

````
#Get all groups into temp variable
$groups = Get-DistributionGroup -ResultSize Unlimited -IgnoreDefaultScope

#Export 1) ON-PREM export all distribution groups and a few settings
$groups | Select-Object RecipientTypeDetails,Name,Alias,DisplayName,PrimarySmtpAddress,@{name="SMTPDomain";expression={$_.PrimarySmtpAddress.Domain}},MemberJoinRestriction,MemberDepartRestriction,RequireSenderAuthenticationEnabled,@{Name="ManagedBy";Expression={$_.ManagedBy -join “;”}},@{name=”AcceptMessagesOnlyFrom”;expression={$_.AcceptMessagesOnlyFrom -join “;”}},@{name=”AcceptMessagesOnlyFromDLMembers”;expression={$_.AcceptMessagesOnlyFromDLMembers -join “;”}},@{name=”AcceptMessagesOnlyFromSendersOrMembers”;expression={$_.AcceptMessagesOnlyFromSendersOrMembers -join “;”}},@{name=”ModeratedBy”;expression={$_.ModeratedBy -join “;”}},@{name=”BypassModerationFromSendersOrMembers”;expression={$_.BypassModerationFromSendersOrMembers -join “;”}},@{Name="GrantSendOnBehalfTo";Expression={$_.GrantSendOnBehalfTo -join “;”}},ModerationEnabled,SendModerationNotifications,LegacyExchangeDN,@{Name="EmailAddresses";Expression={$_.EmailAddresses -join “;”}} | Export-Csv C:tempdgdistributiongroups.csv -NoTypeInformation

#Export 2) ON-PREM export distribution groups’ smtp aliases
$groups | Select-Object RecipientTypeDetails,PrimarySmtpAddress -ExpandProperty emailaddresses | select RecipientTypeDetails,PrimarySmtpAddress, @{name="TYPE";expression={$_}} | Export-Csv C:tempdgdistributiongroups-SMTPproxy.csv -NoTypeInformation

#Export 3) ON-PREM export all distribution groups and members (and member type)
$groups |% {$guid=$_.Guid;$GroupType=$_.RecipientTypeDetails;$Name=$_.Name;$SMTP=$_.PrimarySmtpAddress ;Get-DistributionGroupMember -Identity $guid.ToString() -ResultSize Unlimited | Select-Object @{name=”GroupType”;expression={$GroupType}},@{name=”Group”;expression={$name}},@{name=”GroupSMTP”;expression={$SMTP}},@{name="PrimarySMTPDomain";expression={$SMTP.Domain}},@{Label="Member";Expression={$_.Name}},@{Label="MemberSMTP";Expression={$_.PrimarySmtpAddress}},@{Label="MemberType";Expression={$_.RecipientTypeDetails}}} | Export-Csv C:tempdgdistributiongroups-and-members.csv -NoTypeInformation

````

## Step 2 – Prepare and clean data, add columns that prepend “NEW” to avoid conflicts
### Clean Files
Use exports from the previous step to prepare and clean data. I’m not a fan of manipulating data “on the fly” in PowerShell, because it’s nearly impossible roll-back quickly. I like to create CSV files that have both ‘old’ and ‘new’ data, which allows quick roll-back if necessary. It’s best to use Excel, since we can filter and use macros. When it’s time to delete data, delete cell contents rather than delete rows – this is due to limitations when using Excel ‘filters’.  After deleting data, remember to sort the columns which will remove blank rows.
1. Clean file from export 1 (Distribution Groups file, distributiongroups.csv)

Create “NEW” values. Insert columns after the following (Name, Alias, DisplayName, PrimarySmtpAddress), and prefix column header with “NEW” by using following formula. Then copy the formula down through data, so that all data is prefixed with “NEW”. You should now have the following columns (Name, NEWName, Alias, NEWAlias, DisplayName, NEWDisplayName, PrimarySmtpAddress, NEWPrimarySmtpAddress)  
Use the excel formula _"="NEW"&B1"_  

Clean up any attribute that has a full path for a user account, most notably “ManagedBy“, “AcceptMessagesOnlyFrom”, “AcceptMessagesOnlyFromDLMembers”, and “AcceptMessagesOnlyFromSendersOrMembers” columns. Leave the semicolons in place and do NOT add quotes even though DisplayName values are being used (which contain spaces). You can use “Find and Replace”, CRTL+H to complete this task. You should note, if there are blank values altogether, you might want to specify a group admin, otherwise whoever creates the new groups in powershell will become the owner by default. This can be important if a group requires approval to add/remove members. (e.g.: contoso.local/User Accounts/USA/FTEmployees/Ryan Jackson; contoso.local/User Accounts/JPN/FTEmployees/Dave Rowe —should become–> Ryan Jackson;Dave Rowe)

Note: if you want to exclude mail-enabled security groups, filter columns, and in “RecipientTypeDetails” select rows with “MailUniversalSecurityGroup” and hit delete key.

Save the csv file as distributiongroups_modified.csv
1. Clean file from export 2 (SMTP Proxy/ALIAS file, distributiongroups-SMTPproxy.csv)

Let’s remove everything except alternate smtp and x500, this includes removing Primary SMTP address. We’ll need to add a few columns and use macros to help us find what we’re looking for.

First let’s add a few columns.

Highlight Column C (TYPE), right-click and copy, then paste into Column E (skip column D). Rename Column E header to “FULLADDRESS”. Go back and highlight Column C, then select ‘Data’ tab in Excel ribbon and select ‘Text to Columns’ button.  
Select ‘Delimited’, click Next, uncheck everything except ‘Other:’ checkbox and insert colon “:”, then click ‘Finish’. Afterwards, in Column D, give the header name “ALIAS”  
We’ll add one more column to help us identify uppercase SMTP. Insert a blank column after Column C, and give header name “PRIMARYCHECK”. The following formula is case sensitive and will help us identify primary SMTP and not smtp – copy the formula in Column D (PRIMARYCHECK) down through all data.  
Find primary SMTP using _"=IF(ISNUMBER(FIND("SMTP",C2)),"Primary", "Alternate")"_  

Now that we have all of our columns, we can now filter data and delete what we do not need. To filter data (see filter screenshot above).

• First let’s delete primary SMTP. In Column D (PRIMARYCHECK) select “Primary”, then highlight the data and hit delete key. Now view all results in filter, and sort to remove blank rows.
• Second let’s delete everything EXCEPT smtp and x500 (e.g. x400, EUM). While data is filtered, in Column C (TYPE), uncheck smtp, x500, and blanks – so that everything else is selected – then highlight the data and hit delete key. Now view all results in filter, and sort to remove blank rows.

As long as there is only lowercase “smtp” and “x500” in Column C (TYPE), you are good to go. Now if you scroll to bottom you’ll see all the blank rows. You should highlight these rows from left side, right-click then select delete. Otherwise these rows will error when running scripts.

Save the csv file as distributiongroups-SMTPproxy_modified.csv  
3. Clean file from export 3 (Distribution Groups and Members file, distributiongroups-and-members.csv)  
This section will fix nested-groups since they are members. In the export you can verify if any nested-groups exist. If no nested-groups exist, then just copy previous values in new columns.

Create “NEW” values. Insert columns after the following (Group, GroupSMTP), and prefix column header with “NEW” by using following formula. Then copy the formula down through data, so that all data is prefixed with “NEW”. You should now have the following columns (Group, NEWGroup, GroupSMTP, NEWGroupSMTP)  
_="NEW"&B1_  

Create “NEW” values for nested groups only, and use previous values for individual members. Copy the entire column from “MemberSMTP” and insert as new column right next to it, then rename column header to “NEWMemberSMTP”. You should now have the following columns (MemberSMTP, NEWMemberSMTP). Now filter data (see previous screenshot) and go to “MemberType” column and select the following values (MailUniversalDistributionGroup, DynamicDistributionGroup, MailUniversalSecurityGroup) and unselect the rest. Now you should only see nested groups in “NEWMemberSMTP” column. Replace the value with the following formula (depending on where first cell is, modify formula to that cell), and copy formula to rest of cells that are displayed. This ensures the nested groups are updated with “NEW”.  
_"NEW"&H27_  
Note: if you excluded mail-enabled security groups from distributiongroups_modified.csv, you might consider also removing from this file too. Otherwise you’ll see errors when trying to add members to groups that don’t exist.  Filter columns, and in “GroupType” select rows with “MailUniversalSecurityGroup” and hit delete key.

Save the csv file as distributiongroups-and-members_modified.csv

## Step 3 – Exchange Online: create “NEW” distribution groups, hide from GAL, and add members
### Exchange Online: Create Groups
Let’s create the new distribution groups (and security groups if included) and hide from GAL in Exchange Online. We’ll use one of the files we cleaned up earlier (distributiongroups_modified.csv). Take note, if a group did not have a previous owner (ManagedBy), then whoever creates the distribution group in PowerShell will be the owner by default.

````
Import-Csv C:tempdgdistributiongroups_modified.csv | ForEach-Object{
    $RecipientTypeDetails=$_.RecipientTypeDetails
    $Name = $($_.NEWName -replace 's','')[0..63] -join "" # remove spaces first, then truncate to first 64 characters
    $Alias = $($_.NEWAlias -replace 's','')[0..63] -join "" # remove spaces first, then truncate to first 64 characters
    $DisplayName=$_.NEWDisplayName
    $smtp=$_.NEWPrimarySmtpAddress
    $RequireSenderAuthenticationEnabled=[System.Convert]::ToBoolean($_.RequireSenderAuthenticationEnabled)
    $join=$_.MemberJoinRestriction
    $depart=$_.MemberDepartRestriction
    $ManagedBy=$_.ManagedBy -split ';'
    $AcceptMessagesOnlyFrom=$_.AcceptMessagesOnlyFrom -split ';'
    $AcceptMessagesOnlyFromDLMembers=$_.AcceptMessagesOnlyFromDLMembers -split ';'
    $AcceptMessagesOnlyFromSendersOrMembers=$_.AcceptMessagesOnlyFromSendersOrMembers -split ';'
    
    Write-Output ""
    Write-Output "working on Group: $Name"
    Write-Output ""

    if ($RecipientTypeDetails -eq "MailUniversalSecurityGroup")
        {
        if ($ManagedBy)
            {
            New-DistributionGroup -Type security -Name $Name -Alias $Alias -DisplayName $DisplayName -PrimarySmtpAddress $smtp -RequireSenderAuthenticationEnabled $RequireSenderAuthenticationEnabled -MemberJoinRestriction $join -MemberDepartRestriction $depart -ManagedBy $ManagedBy
            Start-Sleep -s 10
            Set-DistributionGroup -Identity $Name -HiddenFromAddressListsEnabled $true
            }
            Else
            {
            New-DistributionGroup -Type security -Name $Name -Alias $Alias -DisplayName $DisplayName -PrimarySmtpAddress $smtp -RequireSenderAuthenticationEnabled $RequireSenderAuthenticationEnabled -MemberJoinRestriction $join -MemberDepartRestriction $depart 
            Start-Sleep -s 10
            Set-DistributionGroup -Identity $Name -HiddenFromAddressListsEnabled $true
            }
        }

    if ($RecipientTypeDetails -eq "MailUniversalDistributionGroup")
        {
        if ($ManagedBy)
            {
            New-DistributionGroup -Name $Name -Alias $Alias -DisplayName $DisplayName -PrimarySmtpAddress $smtp -RequireSenderAuthenticationEnabled $RequireSenderAuthenticationEnabled -MemberJoinRestriction $join -MemberDepartRestriction $depart -ManagedBy $ManagedBy
            Start-Sleep -s 10
            Set-DistributionGroup -Identity $Name -HiddenFromAddressListsEnabled $true
            }
            Else
            {
            New-DistributionGroup -Name $Name -Alias $Alias -DisplayName $DisplayName -PrimarySmtpAddress $smtp -RequireSenderAuthenticationEnabled $RequireSenderAuthenticationEnabled -MemberJoinRestriction $join -MemberDepartRestriction $depart 
            Start-Sleep -s 10
            Set-DistributionGroup -Identity $Name -HiddenFromAddressListsEnabled $true
            }
        }

    if ($RecipientTypeDetails -eq "RoomList")
        {
        if ($ManagedBy)
            {
            New-DistributionGroup -RoomList -Name $Name -Alias $Alias -DisplayName $DisplayName -PrimarySmtpAddress $smtp -RequireSenderAuthenticationEnabled $RequireSenderAuthenticationEnabled -MemberJoinRestriction $join -MemberDepartRestriction $depart -ManagedBy $ManagedBy
            Start-Sleep -s 10
            Set-DistributionGroup -Identity $Name -HiddenFromAddressListsEnabled $true
            }
            Else
            {
            New-DistributionGroup -RoomList -Name $Name -Alias $Alias -DisplayName $DisplayName -PrimarySmtpAddress $smtp -RequireSenderAuthenticationEnabled $RequireSenderAuthenticationEnabled -MemberJoinRestriction $join -MemberDepartRestriction $depart 
            Start-Sleep -s 10
            Set-DistributionGroup -Identity $Name -HiddenFromAddressListsEnabled $true
            }
        }


    if ($AcceptMessagesOnlyFrom) {Set-DistributionGroup -Identity $Name -AcceptMessagesOnlyFrom $AcceptMessagesOnlyFrom}
    if ($AcceptMessagesOnlyFromDLMembers) {Set-DistributionGroup -Identity $Name -AcceptMessagesOnlyFromDLMembers $AcceptMessagesOnlyFromDLMembers}
    if ($AcceptMessagesOnlyFromSendersOrMembers) {Set-DistributionGroup -Identity $Name -AcceptMessagesOnlyFromSendersOrMembers $AcceptMessagesOnlyFromSendersOrMembers}
  }

````

Exchange Online: Add Members to Groups
After we’ve created the distribution groups, we can now add members. We’ll use the file (distributiongroups-and-members_modified.csv) to complete this task.  
````

Import-Csv C:tempdgdistributiongroups-and-members_modified.csv | ForEach-Object{
$RecipientTypeDetails=$_.GroupType
$GroupSMTP=$_.NEWGroupSMTP
$MemberSMTP=$_.NEWMemberSMTP

    if ($RecipientTypeDetails -eq "MailUniversalSecurityGroup")
        {
        Add-DistributionGroupMember -Identity $GroupSMTP -Member $MemberSMTP -BypassSecurityGroupManagerCheck
        }
    
    if ($RecipientTypeDetails -eq "MailUniversalDistributionGroup")
        {
        Add-DistributionGroupMember -Identity $GroupSMTP -Member $MemberSMTP
        }

    if ($RecipientTypeDetails -eq "RoomList")
        {
        Add-DistributionGroupMember -Identity $GroupSMTP -Member $MemberSMTP
        }

}

````
### Cutover
## Step 4 – Exchange On-Premise: delete distribution groups, and force sync
### Exchange On-Premise: Delete groups
You’ll run the following script on your Exchange server that is On-Premise. Although we’ve taken precautions to minimize impact, it’s best to do this (and remaining) steps at off-peak hours (like Friday night). We’ll use the file (distributiongroups_modified.csv) to complete this task.

````
//Delete groups
$OLDDG = Import-Csv C:tempdgdistributiongroups_modified.csv
$OLDDG | % {Remove-DistributionGroup -Identity $_.PrimarySmtpAddress -Confirm:$false}

````
### AAD Connect / AADSync: Force synchronization
In order to speed things up, you’ll want to force delta syncs (a few) on the  AADConnect / AADSync server. This will ensure the old distribution groups (On-Premise) are no longer visible in Exchange Online. You can do this directly on the AADConnect / AADSync server with miisclient.exe, local PowerShell, or use remote PowerShell from a machine on the same network. You must be an administrator on the server and in AADConnect / AADSync local admin group (ADSyncAdmins / FIMSyncAdmins). Make sure to insert your AADConnect / AADSync server name in PowerShell.

````
//add connect
#LOCAL on AAD Connect (as of 3/29/2016)
Import-Module ADSync
Start-ADSyncSyncCycle -PolicyType Delta

#REMOTE into AAD Connect (as of 3/29/2016)
Invoke-Command -ComputerName AD-CONNECT-SERVER {Start-ADSyncSyncCycle -PolicyType Delta}

#Old AAD Connect or AADSync
Get-ScheduledTask -TaskName "Azure AD Sync Scheduler" -CimSession AADSYNC-SERVER-NAME-HERE | Start-ScheduledTask

````
Note: For larger environments, AADConnect has a protection mechanism that prevents synchronization when over 500 object-deletions are detected. This will prevent you from deleting the groups if you have over 500. To bypass this protection mechanism, run the following commands on AADConnect.  
````
//disable
Disable-ADSyncExportDeletionThreshold

//enable
Enable-ADSyncExportDeletionThreshold
````
## Step 5 – Exchange Online: rename distribution groups (remove “NEW”), unhide, and add SMTP aliases
### Exchange Online: Rename distribution groups and unhide
After you’ve validated the old distribution groups are no longer visible in Exchange Online, we can now unhide the new ones and remove “NEW” from the names. We’ll use the file (distributiongroups_modified.csv) to complete this task.

````
$RENAMEDG = Import-Csv C:tempdgdistributiongroups_modified.csv
$RENAMEDG | ForEach-Object {
    $NEWName = $($_.NEWName -replace 's','')[0..63] -join "" # remove spaces first, then truncate to first 64 characters
    $Name=$_.Name
    $Alias=$_.Alias
    $DisplayName=$_.DisplayName
    $PrimarySmtpAddress=$_.PrimarySmtpAddress
    
    Write-Output ""
    Write-Output "working on Group: $Name"
    Write-Output ""

    Set-DistributionGroup -Identity $NEWName -Name $Name -Alias $Alias -DisplayName $DisplayName -PrimarySmtpAddress $PrimarySmtpAddress -HiddenFromAddressListsEnabled $false}

````

### Exchange Online: Remove NEWPrimarySmtpAddress from -EmailAddresses for all Groups  
Since the previous step just moves “NEWPrimarySmtpAddress” into an alternate smtp alias, we can now remove it. We’ll use the file (distributiongroups_modified.csv) to complete this task.
````
// remove NEWPrimarySmtpAddress
$RemoveNEWGrouptSMTP = Import-Csv C:tempdgdistributiongroups_modified.csv
$RemoveNEWGrouptSMTP | % {Set-DistributionGroup -Identity $_.PrimarySmtpAddress -EmailAddresses @{remove=$_.NEWPrimarySmtpAddress}}

````
### Exchange Online: Add Aliases and LegacyExchangeDN
Last thing to do is add the SMTP, X500, and LegacyExchangeDN aliases in Exchange Online.

````
#add aliases
$ALIASES = Import-Csv C:tempdgdistributiongroups-SMTPproxy_modified.csv
$ALIASES | % {Set-DistributionGroup -Identity $_.PrimarySmtpAddress -EmailAddresses @{Add=$_.FULLADDRESS}}

#add LegacyExchangeDN as x500
Import-Csv C:tempdgdistributiongroups_modified.csv | ForEach-Object{
$smtp=$_.PrimarySmtpAddress
$LegacyExchangeDN="x500:"+$_.LegacyExchangeDN
Set-DistributionGroup $smtp -EmailAddresses @{Add=$LegacyExchangeDN}
}

````

# END
