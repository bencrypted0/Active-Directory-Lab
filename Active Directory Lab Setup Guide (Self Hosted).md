# How to Build an Active Directory Homelab: A Complete PowerShell-Driven Guide

### Set up a Windows Server 2022 domain controller, connect a Windows Pro client, and automate management using PowerShell.

---

Active Directory (AD) is the backbone of enterprise identity and access management. For system administrators, security analysts, and penetration testers alike, a solid understanding of how AD functions—and how to manage it efficiently—is a critical career skill. 

While you can read documentation all day, nothing beats hands-on experience. 

In this guide, we will walk through setting up a self-hosted Active Directory homelab from scratch. We will deploy a Windows Server 2022 virtual machine on Oracle VirtualBox, promote it to a Domain Controller, connect a Windows 10/11 Pro client machine, and then leverage PowerShell to automate daily administrative tasks like user onboarding, system auditing, password resets, and bulk CSV ingestion.

---

## 1. Lab Architecture & Design

To keep resources light and accessible on any typical home PC, we will build a two-machine homelab. One VM will act as the Domain Controller (DC), hosting both the Active Directory Domain Services (AD DS) and Domain Name System (DNS) roles. The second VM will act as a client workstation.

```
Your PC (Host)
└── VirtualBox (Using NAT Network: 10.0.2.0/24)
    ├── WinSrv2022-DC  (IP: 10.0.2.15 | 4GB RAM | DC + DNS | Domain: corp.example.com)
    └── WinPro-Client  (IP: 10.0.2.20 [DHCP/Static] | 4GB RAM | Member Workstation)
```

### Lab Specifications
*   **Virtualization Platform:** Oracle VirtualBox
*   **Domain Controller (DC):** 1 × Windows Server 2022 (Desktop Experience)
    *   *Resources:* 4 GB RAM, 2 CPU cores, 60 GB dynamically allocated disk space
    *   *Domain Namespace:* `corp.example.com`
*   **Workstation Client:** 1 × Windows 10 or Windows 11 Pro
    *   *Resources:* 4 GB RAM, 2 CPU cores, 50 GB dynamically allocated disk space
*   **Network configuration:** VirtualBox **NAT Network** (Allows both VMs to communicate with each other while maintaining internet access)

> **Note on Domains:** The domain namespace `corp.example.com` is used purely as an internal AD namespace. It will not conflict with public DNS resolutions, and your external DNS services remain untouched.

---

## 2. Deploying the Virtual Machine (Domain Controller)

### Step 1: Download the Windows Server 2022 Evaluation ISO
Microsoft offers a free 180-day evaluation copy of Windows Server 2022. You can download the ISO directly from the [Microsoft Evaluation Center](https://www.microsoft.com/en-us/evalcenter/evaluate-windows-server-2022). Select the **ISO download** option. No licensing key is required for the evaluation period.

### Step 2: Configure the DC VM in VirtualBox
Open VirtualBox, click **New**, and configure the virtual machine settings as follows:

| Setting | Value |
| :--- | :--- |
| **Name** | `WinSrv2022-DC` |
| **Type** | Microsoft Windows |
| **Version** | Windows 2022 (64-bit) |
| **Base Memory** | 4096 MB (4 GB minimum recommended) |
| **Processors** | 2 CPUs |
| **Hard Disk** | Create a Virtual Hard Disk (60 GB, dynamically allocated) |
| **Network** | NAT Network (Configure a network named `AD-LabNetwork` with subnet `10.0.2.0/24`) |

![](Active%20Directory%20Lab%20Setup%20Guide%20(Self%20Hosted).md_Attachments/Active%20Directory%20Lab%20Setup%20Guide%20(Self%20Hosted)-20260717165211602.png)

After creation, navigate to **Settings → Storage**, select the empty optical drive, and mount the downloaded Windows Server 2022 ISO file.

### Step 3: Install the Operating System
1. Start the virtual machine.
2. Select your language and time settings, and click **Install Now**.
3. Choose **Windows Server 2022 Standard Evaluation (Desktop Experience)**. This option installs the standard graphical user interface (GUI).
4. Select **Custom: Install Windows only (advanced)** and click Next to install on the virtual disk.
5. Once the installation completes, the system will prompt you to set a password for the local Administrator account (e.g., `Admin@2024!`).

---

## 3. Configuring and Promoting the Domain Controller

With the base operating system installed, we will configure the VM networking, host naming, and promote it to a Domain Controller. We will use PowerShell to keep our configurations precise and repeatable.

### Step 1: Set a Static IP Address
A Domain Controller must have a reliable, static IP address because it hosts critical network services like DNS. Open **PowerShell as Administrator** inside the VM and execute:

```powershell
# Retrieve the network adapter name (typically "Ethernet")
Get-NetAdapter

# Configure the static IP address and default gateway
New-NetIPAddress `
    -InterfaceAlias "Ethernet" `
    -IPAddress "10.0.2.15" `
    -PrefixLength 24 `
    -DefaultGateway "10.0.2.1"

# Point DNS client settings to the VM's own IP address
Set-DnsClientServerAddress `
    -InterfaceAlias "Ethernet" `
    -ServerAddresses "10.0.2.15"
```

### Step 2: Rename the Server
Rename the server to something that fits standard corporate naming conventions (like `DC01`) and reboot:

```powershell
Rename-Computer -NewName "DC01" -Restart
```

### Step 3: Install the Active Directory Domain Services (AD DS) Role
Once the server restarts, log back in and run the following command in PowerShell to install the AD DS role along with the required management utilities:

```powershell
Install-WindowsFeature `
    -Name AD-Domain-Services `
    -IncludeManagementTools
```

### Step 4: Promote the Server to a Forest Root
Now, promote this server to a new Domain Controller inside a brand new Active Directory forest (`corp.example.com`):

```powershell
Import-Module ADDSDeployment

Install-ADDSForest `
    -DomainName "corp.example.com" `
    -DomainNetbiosName "CORP" `
    -ForestMode "WinThreshold" `
    -DomainMode "WinThreshold" `
    -InstallDns:$true `
    -SafeModeAdministratorPassword (ConvertTo-SecureString "DSRM@2024!" -AsPlainText -Force) `
    -Force:$true
```

> **Directory Services Restore Mode (DSRM):** The DSRM password is a critical recovery credential used to log in to the DC in offline mode for database repair.

The server will automatically reboot to complete the promotion process.

---

## 4. Post-Deployment Configuration & DNS Forwarders

Once the VM restarts, log in using the newly created domain credentials: `CORP\Administrator`.

Because the Domain Controller is configured to point to itself for DNS resolution, it cannot resolve external domain names by default. To restore internet connectivity inside the VM for web browsing and updates, configure **DNS Forwarders** to forward unknown queries to external public DNS servers:

```powershell
# Add Google DNS as forwarders
Add-DnsServerForwarder -IPAddress "8.8.8.8"
Add-DnsServerForwarder -IPAddress "8.8.4.4"

# Verify the forwarders configuration
Get-DnsServerForwarder
```

![](Active%20Directory%20Lab%20Setup%20Guide%20(Self%20Hosted).md_Attachments/Active%20Directory%20Lab%20Setup%20Guide%20(Self%20Hosted)-20260717175533191.png)

---

## 5. Joining a Windows Pro Client to the Domain

Now that our Domain Controller is live, we will join our second VM—a Windows 10/11 Pro workstation—to the `corp.example.com` domain.

### Step 1: Align VirtualBox Network Adapter Settings
To let the two machines communicate, they must reside on the same VirtualBox Network.
1. Open the VirtualBox settings for **both** the DC VM (`WinSrv2022-DC`) and the Client VM (`WinPro-Client`).
2. Go to **Network**.
3. Under **Attached to**, select **NAT Network**.
4. Select the name of the network you created (e.g., `NatNetwork`).

![](Active%20Directory%20Lab%20Setup%20Guide%20(Self%20Hosted).md_Attachments/Active%20Directory%20Lab%20Setup%20Guide%20(Self%20Hosted)-20260717175713351.png)

### Step 2: Configure Client DNS Settings
For the client workstation to locate the Active Directory Domain Services, it must query the Domain Controller's DNS server.
1. Log in to the Windows Pro Client VM.
2. Open **Run** (`Win + R`), type `ncpa.cpl`, and hit Enter to open Network Connections.
3. Right-click your active network adapter and choose **Properties**.
4. Select **Internet Protocol Version 4 (TCP/IPv4)** and click **Properties**.
5. Keep "Obtain an IP address automatically" (or assign a static IP inside the subnet, e.g., `10.0.2.20`).
6. Under DNS Settings, select **Use the following DNS server addresses** and configure:
   *   **Preferred DNS Server:** `10.0.2.15` (The static IP of your Domain Controller)
   *   **Alternate DNS Server:** Leave empty or set to `8.8.8.8`.

![](Active%20Directory%20Lab%20Setup%20Guide%20(Self%20Hosted).md_Attachments/Active%20Directory%20Lab%20Setup%20Guide%20(Self%20Hosted)-20260717180006191.png)

7. Open Command Prompt and test domain resolution:
   ```cmd
   nslookup corp.example.com
   ```
   *Expected Output: It should resolve to the Domain Controller IP `10.0.2.15`.*

### Step 3: Perform the Domain Join
1. Open **Settings** on the Windows Pro Client.
2. Navigate to **System → About**, scroll down, and click **Advanced system settings** (on the right pane or bottom).
3. Switch to the **Computer Name** tab and click **Change.**
4. Change the computer name to a structured name (e.g., `DESKTOP-01`).
5. Select **Domain** under "Member of" and enter: `corp.example.com`.
6. Click **OK**.

![](Active%20Directory%20Lab%20Setup%20Guide%20(Self%20Hosted).md_Attachments/Active%20Directory%20Lab%20Setup%20Guide%20(Self%20Hosted)-20260717180211991.png)

7. A login credential prompt will appear. Enter the domain credentials:
   *   **Username:** `CORP\Administrator`
   *   **Password:** `Admin@2024!` (or your configured admin password)
8. Upon successful validation, a pop-up dialog will read *"Welcome to the corp.example.com domain."*
9. Click **OK** and restart the workstation.

![](Active%20Directory%20Lab%20Setup%20Guide%20(Self%20Hosted).md_Attachments/Active%20Directory%20Lab%20Setup%20Guide%20(Self%20Hosted)-20260717180306781.png)

### Step 4: Login with Domain Credentials
After rebooting, select **Other User** on the login screen. You can log in using `CORP\Administrator` or any domain account created in the steps below.

> **Understanding Computer vs. User Accounts:** 
> You do **not** manually set a password for the computer account `DESKTOP-01` itself. 
> * **Computer Accounts** have machine passwords managed automatically in the background by Windows and Active Directory (rotating every 30 days) to secure the channel between the workstation and the DC.
> * **User Authentication:** To log in, you use **Domain User Accounts** (like `CORP\Administrator` or users created in the verification section) whose credentials are managed on the Domain Controller. 
> * **Local Accounts:** If you ever need to log in to the original local account on `DESKTOP-01`, select **Other User** and enter `.\LocalUsername` as the username, using the local password you set up when installing Windows on the client VM.

---

## 6. Administrative Automation using PowerShell

Manual administration via graphical utilities (like Active Directory Users and Computers) is slow and prone to human error. In production environments, administrators rely on scripting. 

Let's explore how to query and manage Active Directory using PowerShell, and write modular scripts to automate daily operational tasks.

### Basic AD Queries
First, import the Active Directory module (automatically installed on Domain Controllers):

```powershell
Import-Module ActiveDirectory
```

Here are the essential cmdlets you will use to search the directory:

```powershell
# Query all users in the domain
Get-ADUser -Filter *

# Find a specific user by their username (sAMAccountName)
Get-ADUser -Identity "jsmith"

# Query users with specific extended attributes
Get-ADUser -Filter * -Properties Department, Title

# Find locked out domain accounts
Search-ADAccount -LockedOut

# Find users with expired passwords
Search-ADAccount -PasswordExpired

# List all computers joined to the domain
Get-ADComputer -Filter *

# Get high-level domain information
Get-ADDomain
```

---

### Script 1: Standardized User Onboarding
This script automates user creation. It constructs a standardized username format (first initial + last name), places the user into a specific Organizational Unit (OU), sets a temporary password, forces a password change at first logon, and assigns them to their respective department group.

name it as `New-UserOnboarding.ps1`:

```powershell
param (
    [Parameter(Mandatory=$true)] [string]$FirstName,
    [Parameter(Mandatory=$true)] [string]$LastName,
    [Parameter(Mandatory=$true)] [string]$Department,
    [Parameter(Mandatory=$true)] [string]$Title,
    [string]$OUPath = "OU=Users,DC=corp,DC=example,DC=com"
)

# Generate username: John Smith -> jsmith
$Username    = ($FirstName.Substring(0,1) + $LastName).ToLower()
$DisplayName = "$FirstName $LastName"
$TempPass    = ConvertTo-SecureString "Welcome@2024!" -AsPlainText -Force

# Create the user account in AD
New-ADUser `
    -GivenName              $FirstName `
    -Surname                $LastName `
    -Name                   $DisplayName `
    -SamAccountName         $Username `
    -UserPrincipalName      "$Username@corp.example.com" `
    -Department             $Department `
    -Title                  $Title `
    -Path                   $OUPath `
    -AccountPassword        $TempPass `
    -ChangePasswordAtLogon  $true `
    -Enabled                $true

# Automatically add the user to a matching department group if it exists
$Group = "$Department-Team"
if (Get-ADGroup -Filter { Name -eq $Group } -ErrorAction SilentlyContinue) {
    Add-ADGroupMember -Identity $Group -Members $Username
}

Write-Host "Successfully created account: $Username ($DisplayName) in $OUPath" -ForegroundColor Green
```

**Usage:**
```powershell
.\New-UserOnboarding.ps1 -FirstName "John" -LastName "Smith" -Department "IT" -Title "Security Analyst"
```

---

### Script 2: Computer Asset Onboarding & Auditing
When a new computer object joins the domain, it is placed in the default `Computers` container by default. This script relocates the system to the correct target OU based on its asset type (Server or Workstation) and stamps it with an audit description.

name it as `New-SystemOnboarding.ps1`:

```powershell
param (
    [Parameter(Mandatory=$true)] [string]$ComputerName,
    [Parameter(Mandatory=$true)] [string]$ComputerType, # "Workstation" or "Server"
    [string]$AssignedUser = ""
)

# Determine destination OU
$TargetOU = if ($ComputerType -eq "Server") {
    "OU=Servers,DC=corp,DC=example,DC=com"
} else {
    "OU=Workstations,DC=corp,DC=example,DC=com"
}

# Verify the computer asset exists in the domain
$Computer = Get-ADComputer -Identity $ComputerName -ErrorAction Stop

# Relocate the object to its structured OU
Move-ADObject -Identity $Computer.DistinguishedName -TargetPath $TargetOU
Write-Host "Moved computer $ComputerName to $TargetOU" -ForegroundColor Cyan

# Write audit information to the description property
$Desc = "Onboarded: $(Get-Date -Format 'yyyy-MM-dd')"
if ($AssignedUser) { $Desc += " | Assigned User: $AssignedUser" }
Set-ADComputer -Identity $ComputerName -Description $Desc

Write-Host "System $ComputerName successfully onboarded." -ForegroundColor Green
```

**Usage:**
```powershell
.\New-SystemOnboarding.ps1 -ComputerName "DESKTOP-01" -ComputerType "Workstation" -AssignedUser "jsmith"
```

---

### Script 3: Helpdesk Password Reset Utility
Password resets are the most frequent request for IT helpdesks. This script resets a user's password, unlocks their account if locked out due to multiple failed login attempts, and forces them to choose a new password at their next login.

name it as `Reset-UserPassword.ps1`:

```powershell
param (
    [Parameter(Mandatory=$true)] [string]$Username,
    [string]$NewPassword = "Welcome@Reset1!"
)

# Convert plain-text password to a SecureString in memory
$SecurePass = ConvertTo-SecureString $NewPassword -AsPlainText -Force

# Reset the password directly
Set-ADAccountPassword `
    -Identity $Username `
    -NewPassword $SecurePass `
    -Reset

# Force password change at next authentication
Set-ADUser -Identity $Username -ChangePasswordAtLogon $true

# Unlock account
Unlock-ADAccount -Identity $Username

# Output verification information
$UserStatus = Get-ADUser -Identity $Username -Properties LockedOut, PasswordLastSet
Write-Host "Reset account: $Username | Locked status: $($UserStatus.LockedOut) | Last Password Change: $($UserStatus.PasswordLastSet)" -ForegroundColor Green
```

**Usage:**
```powershell
.\Reset-UserPassword.ps1 -Username "jsmith"
```

---

### Script 4: User Account Modifications & Offboarding
This script handles day-to-day changes, such as job title modifications, department transfers, OU migrations, and disabling accounts for offboarding.

name it as `Update-UserAccount.ps1`:

```powershell
param (
    [Parameter(Mandatory=$true)] [string]$Username,
    [string]$NewTitle      = "",
    [string]$NewDepartment = "",
    [string]$NewOU         = "",
    [bool]$Disable         = $false
)

# Update attributes
if ($NewTitle)      { Set-ADUser -Identity $Username -Title $NewTitle }
if ($NewDepartment) { Set-ADUser -Identity $Username -Department $NewDepartment }

# Move user to a new OU if specified
if ($NewOU) {
    $DN = (Get-ADUser -Identity $Username).DistinguishedName
    Move-ADObject -Identity $DN -TargetPath $NewOU
    Write-Host "Moved $Username to $NewOU" -ForegroundColor Cyan
}

# Disable user account if requested
if ($Disable) {
    Disable-ADAccount -Identity $Username
    Write-Host "Account disabled: $Username" -ForegroundColor Yellow
}

Write-Host "Updated user account settings for $Username." -ForegroundColor Green
```

**Usage:**
```powershell
# Update Title
.\Update-UserAccount.ps1 -Username "jsmith" -NewTitle "Senior Security Analyst"

# Move to a new OU
.\Update-UserAccount.ps1 -Username "jsmith" -NewOU "OU=HR,DC=corp,DC=example,DC=com"

# Disable account
.\Update-UserAccount.ps1 -Username "jsmith" -Disable $true
```

---

### Script 5: Bulk User Ingestion from CSV
In enterprise deployments—such as office expansions, new school terms, or acquisitions—administrators must create hundreds of accounts at once. This script reads details from a CSV file, skips any duplicates to prevent errors, and imports users in bulk.

First, create a sample CSV file named `users.csv`:

```csv
FirstName,LastName,Username,Department,Title,OU
John,Smith,jsmith,IT,Analyst,"OU=IT,DC=corp,DC=example,DC=com"
Jane,Doe,jdoe,HR,Manager,"OU=HR,DC=corp,DC=example,DC=com"
Bob,Johnson,bjohnson,Finance,Accountant,"OU=Finance,DC=corp,DC=example,DC=com"
Alice,Williams,awilliams,IT,Engineer,"OU=IT,DC=corp,DC=example,DC=com"
Charlie,Brown,cbrown,HR,Recruiter,"OU=HR,DC=corp,DC=example,DC=com"
```

> **Pro Tip:** By wrapping field values containing commas (like the distinguished name `"OU=IT,DC=corp,DC=example,DC=com"`) in double quotes, standard CSV parsers—including PowerShell's `Import-Csv`—will read it correctly as a single string field.

name it as `Import-BulkUsers.ps1`:

```powershell
param (
    [Parameter(Mandatory=$true)] [string]$CsvPath,
    [string]$DefaultPassword = "Welcome@2024!"
)

if (-not (Test-Path $CsvPath)) {
    Write-Error "CSV file not found at: $CsvPath"
    exit 1
}

$Users      = Import-Csv -Path $CsvPath
$SecurePass = ConvertTo-SecureString $DefaultPassword -AsPlainText -Force
$Created = $Failed = $Skipped = 0

foreach ($User in $Users) {
    # Check if user already exists
    if (Get-ADUser -Filter { SamAccountName -eq $User.Username } -ErrorAction SilentlyContinue) {
        Write-Warning "Skipping $($User.Username) — account already exists."
        $Skipped++
        continue
    }

    try {
        New-ADUser `
            -GivenName             $User.FirstName `
            -Surname               $User.LastName `
            -Name                  "$($User.FirstName) $($User.LastName)" `
            -SamAccountName        $User.Username `
            -UserPrincipalName     "$($User.Username)@corp.example.com" `
            -Department            $User.Department `
            -Title                 $User.Title `
            -Path                  $User.OU `
            -AccountPassword       $SecurePass `
            -ChangePasswordAtLogon $true `
            -Enabled               $true

        Write-Host "Created user: $($User.Username)" -ForegroundColor Green
        $Created++
    } catch {
        Write-Host "Failed to create $($User.Username): $($_.Exception.Message)" -ForegroundColor Red
        $Failed++
    }
}

Write-Host "`nBulk Import Summary: Created: $Created | Skipped: $Skipped | Failed: $Failed" -ForegroundColor Cyan
```

**Usage:**
```powershell
.\Import-BulkUsers.ps1 -CsvPath "C:\Scripts\users.csv"
```

---

## 7. Hands-On Verification & Initial Setup

To verify the lab is working as designed and confirm the scripts are executing properly, we will run through a quick end-to-end verification.

### 1. Active Directory Domain Diagnostics
Run the command-line domain controller diagnostic tool to scan DNS configuration, connectivity, and netlogon health:

```powershell
# Run basic health diagnostics
dcdiag /test:dns
dcdiag /test:netlogons
```

### 2. Verify Active Directory Web Services & Domain Controllers
Verify that crucial domain processes are running as expected:

```powershell
# Check services status
Get-Service ADWS, DNS, KDC, Netlogon | Select-Object Name, Status
```
All listed services should display `Running`.

### 3. Verify DNS SRV Records
Domain Controllers advertise themselves to clients on a network using DNS SRV records. If these records are missing or incorrect, domain computers cannot log in or resolve DNS queries:

```powershell
# Query local SRV records for LDAP and Kerberos services
Resolve-DnsName -Name "_ldap._tcp.corp.example.com" -Type SRV
Resolve-DnsName -Name "_kerberos._tcp.corp.example.com" -Type SRV
```
The commands should return the host name (`DC01.corp.example.com`) and its static IP address (`10.0.2.15`).

### 4. Setting Up the Directory Structure
Before onboarded users can be placed into their respective organizational containers, create the organizational structure in the domain root:

```powershell
$DN = "DC=corp,DC=example,DC=com"

# Create core Organizational Units (OUs)
New-ADOrganizationalUnit -Name "IT" -Path $DN -ProtectedFromAccidentalDeletion $true
New-ADOrganizationalUnit -Name "HR" -Path $DN -ProtectedFromAccidentalDeletion $true
New-ADOrganizationalUnit -Name "Servers" -Path $DN -ProtectedFromAccidentalDeletion $true
New-ADOrganizationalUnit -Name "Workstations" -Path $DN -ProtectedFromAccidentalDeletion $true

# Confirm the OUs were created
Get-ADOrganizationalUnit -Filter * | Select-Object Name, DistinguishedName
```

### 5. Running Onboarding Verification
To check the scripts, execute this script block to verify user onboarding and security group mapping. This script sets up standard department-level distribution/security groups first, then runs user creations:

```powershell
# Create team security groups
New-ADGroup -Name "IT-Team" -GroupScope Global -GroupCategory Security -Path "OU=IT,DC=corp,DC=example,DC=com"
New-ADGroup -Name "HR-Team" -GroupScope Global -GroupCategory Security -Path "OU=HR,DC=corp,DC=example,DC=com"

# Deploy standard test accounts
$UsersList = @(
    @{ First="Alice";   Last="Adams";   UN="aadams";   Dept="IT";  Title="Security Analyst"; OU="OU=IT,DC=corp,DC=example,DC=com" },
    @{ First="Bob";     Last="Baker";   UN="bbaker";   Dept="IT";  Title="Network Engineer"; OU="OU=IT,DC=corp,DC=example,DC=com" },
    @{ First="Carol";   Last="Chen";    UN="cchen";    Dept="HR";  Title="HR Manager";       OU="OU=HR,DC=corp,DC=example,DC=com" },
    @{ First="David";   Last="Davis";   UN="ddavis";   Dept="HR";  Title="Recruiter";        OU="OU=HR,DC=corp,DC=example,DC=com" },
    @{ First="Emma";    Last="Evans";   UN="eevans";   Dept="IT";  Title="Systems Admin";    OU="OU=IT,DC=corp,DC=example,DC=com" }
)

$Pass = ConvertTo-SecureString "Welcome@2024!" -AsPlainText -Force

foreach ($U in $UsersList) {
    New-ADUser `
        -GivenName             $U.First `
        -Surname               $U.Last `
        -Name                  "$($U.First) $($U.Last)" `
        -SamAccountName        $U.UN `
        -UserPrincipalName     "$($U.UN)@corp.example.com" `
        -Department            $U.Dept `
        -Title                 $U.Title `
        -Path                  $U.OU `
        -AccountPassword       $Pass `
        -ChangePasswordAtLogon $true `
        -Enabled               $true

    # Add to team group if exists
    $TeamGroup = "$($U.Dept)-Team"
    if (Get-ADGroup -Filter { Name -eq $TeamGroup } -ErrorAction SilentlyContinue) {
        Add-ADGroupMember -Identity $TeamGroup -Members $U.UN
    }

    Write-Host "Successfully provisioned testing user: $($U.UN)" -ForegroundColor Green
}

# Verify user properties and group assignments
Get-ADUser -Filter * -Properties Department |
    Select-Object Name, SamAccountName, Department, DistinguishedName |
    Format-Table -AutoSize
```

![](Active%20Directory%20Lab%20Setup%20Guide%20(Self%20Hosted).md_Attachments/Active%20Directory%20Lab%20Setup%20Guide%20(Self%20Hosted)-20260717181444868.png)

---

## 8. Next Steps: Expanding Your Homelab

Congratulations! You now have a fully operational Active Directory homelab running on a Windows Server 2022 Domain Controller and a Windows Pro client workstation. By leveraging PowerShell, you've configured a robust base, set up automation for user provisioning and computer assets, and successfully validated the environment.

With this foundation, here are a few ways you can expand your homelab environment:
1.  **Experiment with Group Policies (GPO):** Create Group Policy Objects to enforce security settings, restrict unauthorized actions, or script startup processes on client VMs.
2.  **Learn Security Auditing & Pentesting:** Turn on Advanced Auditing policies to monitor event logs for anomalies (such as brute-force logons or unauthorized credential queries) or simulate active directory security assessments in a safe, fully legal homelab environment.
3.  **Deploy a Second Domain Controller:** Add another Windows Server VM and promote it as a backup DC to explore replication topology and high-availability operations.