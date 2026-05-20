############################################################
# LOG FUNCTION
############################################################
function fnlogg {
    $rb = "MODIFY DISTRIBUTION LIST"
    $logpre = $rb + " - " + (Get-Date).ToString("MMM-yyyy-dd:hhmmss") + " Info : "
    return $logpre
}

############################################################
# MODULE CHECK
############################################################
if (-not (Get-Module -ListAvailable -Name ExchangeOnlineManagement)) {
    Install-Module ExchangeOnlineManagement -Force -AllowClobber -Scope AllUsers
}

Import-Module ExchangeOnlineManagement

############################################################
# INPUTS
############################################################

$DLIdentity = "DL-Test@qagarrettmotion.com"

# Requested operations coming from workflow
$requiredservices = "DLName,Owner,AddMember,RemoveMember"

# Values
$NewDLName      = "DL-Test1"
$NewOwner       = "Roopali.Kanni@qagarrettmotion.com"
$AddMembers     = "Manju.MohanKumar@qagarrettmotion.com"
$RemoveMembers  = "Roopali.Kanni@qagarrettmotion.com"

$svc_account = "<ACC-NAME>"
$Pass        = "<PASSWORD>"

############################################################
# PREP
############################################################

$services = $requiredservices.Split(",") | ForEach-Object { $_.Trim() }

$SecPassword = ConvertTo-SecureString $Pass -AsPlainText -Force
$Credential  = New-Object System.Management.Automation.PSCredential ($svc_account, $SecPassword)

############################################################
# CONNECT
############################################################

try {
    Connect-ExchangeOnline -Credential $Credential -ShowBanner:$false
    $logg = $(fnlogg) + "Connected to Exchange Online`r`n"
}
catch {
    $logg = $(fnlogg) + "Connection failed : $($_.Exception.Message)`r`n"
    return $logg
}

############################################################
# VALIDATE DL
############################################################

$dl = Get-DistributionGroup -Identity $DLIdentity -ErrorAction SilentlyContinue

if (-not $dl) {

    $logg += $(fnlogg) + "DL not found [$DLIdentity]`r`n"
    Disconnect-ExchangeOnline -Confirm:$false
    return $logg
}

############################################################
# UPDATE DL NAME
############################################################

if ($services -contains "DLName") {

    try {

        Set-DistributionGroup -Identity $DLIdentity -DisplayName $NewDLName
        $logg += $(fnlogg) + "Success : DL Name updated to [$NewDLName]`r`n"

    }
    catch {
        $logg += $(fnlogg) + "Failed : DL Name update - $($_.Exception.Message)`r`n"
    }
}

############################################################
# UPDATE OWNER
############################################################

if ($services -contains "Owner") {

    try {

        Set-DistributionGroup -Identity $DLIdentity -ManagedBy $NewOwner
        $logg += $(fnlogg) + "Success : Owner updated [$NewOwner]`r`n"

    }
    catch {
        $logg += $(fnlogg) + "Failed : Owner update - $($_.Exception.Message)`r`n"
    }
}

############################################################
# ADD MEMBERS
############################################################

if ($services -contains "AddMember" -and $AddMembers) {

    $memberList = $AddMembers -split ","

    foreach ($member in $memberList) {

        try {

            $exists = Get-DistributionGroupMember -Identity $DLIdentity -ResultSize Unlimited |
                      Where-Object { $_.PrimarySmtpAddress -eq $member }

            if (-not $exists) {

                Add-DistributionGroupMember -Identity $DLIdentity -Member $member -ErrorAction Stop
                $logg += $(fnlogg) + "Success : Member added [$member]`r`n"

            }
            else {

                $logg += $(fnlogg) + "Info : Member already exists [$member]`r`n"

            }

        }
        catch {
            $logg += $(fnlogg) + "Failed : Add member [$member] - $($_.Exception.Message)`r`n"
        }
    }
}

############################################################
# REMOVE MEMBERS
############################################################

if ($services -contains "RemoveMember" -and $RemoveMembers) {

    $removeList = $RemoveMembers -split ","

    foreach ($member in $removeList) {

        try {

            Remove-DistributionGroupMember -Identity $DLIdentity -Member $member -Confirm:$false -ErrorAction Stop
            $logg += $(fnlogg) + "Success : Member removed [$member]`r`n"

        }
        catch {
            $logg += $(fnlogg) + "Failed : Remove member [$member] - $($_.Exception.Message)`r`n"
        }
    }
}

############################################################
# DISCONNECT
############################################################

Disconnect-ExchangeOnline -Confirm:$false

$logg
$loggstr = $logg -join "`r`n"
#converted string!!!
