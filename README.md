# Setting Up AD DS on Windows Server Core

## Objective

Install and configure a new Active Directory Domain Services (AD DS {Active Directory Domain Services}) forest on a Windows Server Core machine, and create an Organizational Unit (OU {Organizational Unit}) to organize resources.

## Prerequisites Overview

Before starting, ensure the server meets the following requirements:

- Windows Server Core installed (e.g., Windows Server 2016 or later).
- Administrator credentials available.
- Network connectivity established.
- Sufficient disk space (minimum 32 GB recommended for AD DS).
- Server is not joined to another domain (must be a standalone server or in a workgroup).

## Steps

### 1\. Prerequisites

- **Check Server Status**:
    
    - Verify the server is running Windows Server Core and not joined to an existing domain:
        
        ```powershell
        systeminfo
        ```
        
        - **Parameter Descriptions for `systeminfo`**:
            - No parameters needed; displays system information, including OS version and domain/workgroup status.
    - Ensure the server has the latest updates:
        
        ```powershell
        Install-WindowsFeature -Name Windows-Update -IncludeManagementTools
        sconfig
        ```
        
        - **Parameter Descriptions for `Install-WindowsFeature`**:
            - `-Name Windows-Update`: Installs the Windows Update feature for managing updates.
            - `-IncludeManagementTools`: Includes tools for managing updates.
        - **Note on `sconfig`**: Launches the Server Configuration tool to install updates interactively (select option 6 for updates).
- **Set Static IP**:
    
    - Use `Get-NetAdapter` to find the network interface index:
        
        ```powershell
        Get-NetAdapter
        ```
        
        - **Parameter Descriptions for `Get-NetAdapter`**:
            - No parameters needed; lists network adapters and their indices.
    - Configure a static IP address and DNS settings to ensure the server is network-ready and can function as a DNS {Domain Name System} server post-promotion:
        
        ```powershell
        New-NetIPAddress -InterfaceIndex 4 -IPAddress 192.168.1.10 -PrefixLength 24 -DefaultGateway 192.168.1.1
        Set-DnsClientServerAddress -InterfaceIndex 4 -ServerAddresses "127.0.0.1"
        ```
        
        - **Parameter Descriptions for `New-NetIPAddress`**:
            - `-InterfaceIndex 4`: Specifies the index of the network adapter (obtained from `Get-NetAdapter`).
            - `-IPAddress 192.168.1.10`: Sets the static IP address for the server.
            - `-PrefixLength 24`: Defines the subnet mask (e.g., 255.255.255.0 for /24).
            - `-DefaultGateway 192.168.1.1`: Specifies the gateway for routing network traffic.
        - **Parameter Descriptions for `Set-DnsClientServerAddress`**:
            - `-InterfaceIndex 4`: Matches the adapter index used for the IP configuration.
            - `-ServerAddresses "127.0.0.1"`: Sets the DNS server to the local machine’s loopback address, as it will become a DNS server after AD DS promotion.
- **Set Computer Name**:
    
    - Rename the server to a descriptive name (e.g., for a Primary Domain Controller) and restart:
        
        ```powershell
        Rename-Computer -NewName "PH-DC01" -Restart
        ```
        
        - **Parameter Descriptions for `Rename-Computer`**:
            - `-NewName "PH-DC01"`: Specifies the new hostname (e.g., “PH-DC01” for Primary Domain Controller 01).
            - `-Restart`: Forces a reboot to apply the name change.

### 2\. Install AD DS Role

- Install the AD DS {Active Directory Domain Services} role and management tools to prepare the server for promotion:
    
    ```powershell
    Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools
    ```
    
    - **Parameter Descriptions for `Install-WindowsFeature`**:
        - `-Name AD-Domain-Services`: Specifies the AD DS role to install.
        - `-IncludeManagementTools`: Installs additional management tools (e.g., PowerShell cmdlets like `Get-ADDomain`).

### 3\. Promote to Domain Controller

- Create a secure Directory Services Restore Mode (DSRM {Directory Services Restore Mode}) password:
    
    ```powershell
    $safeModePassword = Read-Host -AsSecureString
    ```
    
    - **Parameter Descriptions for `Read-Host`**:
        - `-AsSecureString`: Prompts for input and stores it as a secure string, suitable for sensitive data like the DSRM password used for recovery operations.
- Promote the server to a domain controller, creating a new AD forest:
    
    ```powershell
    Install-ADDSForest `
        -DomainName "ad.mycompany.local" `
        -DomainNetbiosName "MYCOMPANY" `
        -InstallDns:$true `
        -DomainMode Win2016 `
        -ForestMode Win2016 `
        -SafeModeAdministratorPassword $safeModePassword `
        -NoRebootOnCompletion:$false `
        -Force:$true
    ```
    
    - **Parameter Descriptions for `Install-ADDSForest`**:
        - `-DomainName "ad.mycompany.local"`: Sets the fully qualified domain name (FQDN) for the new AD domain (e.g., a subdomain like `ad.mycompany.local`).
        - `-DomainNetbiosName "MYCOMPANY"`: Defines the NetBIOS name, a shorter legacy name for compatibility (e.g., `MYCOMPANY`).
        - `-InstallDns:$true`: Installs and configures the DNS server role, essential for AD DS functionality.
        - `-DomainMode Win2016`: Sets the domain functional level to Windows Server 2016, enabling modern AD features.
        - `-ForestMode Win2016`: Sets the forest functional level to Windows Server 2016, ensuring compatibility across the forest.
        - `-SafeModeAdministratorPassword`: Uses the secure DSRM password created earlier for recovery operations.
        - `-NoRebootOnCompletion:$false`: Ensures the server reboots automatically after promotion (default behavior).
        - `-Force:$true`: Suppresses confirmation prompts, streamlining the process.
- Confirm the operation (type `Y` if prompted), and the server will configure and restart.
    

### 4\. Post-Installation Verification

- Log in with the new domain admin account (e.g., `MYCOMPANY\Administrator`).
- Verify:
    - **SYSVOL/NETLOGON Shares**: Confirm critical AD shares exist:
        
        ```powershell
        Get-SmbShare
        ```
        
        - **Parameter Descriptions for `Get-SmbShare`**:
            - No parameters needed; retrieves all SMB {Server Message Block} shares, expecting `SYSVOL` and `NETLOGON` for a healthy domain controller.
    - **Domain Info**: Check domain configuration details:
        
        ```powershell
        Get-ADDomain
        ```
        
        - **Parameter Descriptions for `Get-ADDomain`**:
            - No parameters needed; retrieves properties of the current domain (e.g., `ad.mycompany.local`).
    - **DNS Zones**: Verify AD-related DNS zones:
        
        ```powershell
        Get-DnsServerZone
        ```
        
        - **Parameter Descriptions for `Get-DnsServerZone`**:
            - No parameters needed; lists DNS zones, expecting `_msdcs.ad.mycompany.local` and `ad.mycompany.local` for AD DNS integration.
    - **Check AD DS Health**: Verify the domain controller’s health:
        
        ```powershell
        dcdiag
        ```
        
        - **Parameter Descriptions for `dcdiag`**:
            - No parameters needed; runs diagnostic tests on the domain controller to verify AD DS functionality (e.g., replication, services).

### 5\. Create an Organizational Unit (OU)

- **Objective**: Create an OU {Organizational Unit} (e.g., for the IT department) to organize users, computers, and apply Group Policy Objects (GPOs {Group Policy Objects}).
- **Steps**:
    1.  **Create the OU**:
        
        ```powershell
        New-ADOrganizationalUnit -Name "IT" -Path "DC=ad,DC=mycompany,DC=local" -ProtectedFromAccidentalDeletion $true -Description "IT Department OU"
        ```
        
        - **Parameter Descriptions for `New-ADOrganizationalUnit`**:
            - `-Name "IT"`: Specifies the name of the OU (e.g., `IT` for the IT department).
            - `-Path "DC=ad,DC=mycompany,DC=local"`: Defines the distinguished name of the parent container (e.g., domain root).
            - `-ProtectedFromAccidentalDeletion $true`: Prevents accidental deletion, enhancing OU security.
            - `-Description "IT Department OU"`: Adds a description for clarity in management tools.
    2.  **Verify the OU**:
        
        ```powershell
        Get-ADOrganizationalUnit -Filter 'Name -eq "IT"' | Select-Object Name, DistinguishedName
        ```
        
        - **Parameter Descriptions for `Get-ADOrganizationalUnit`**:
            - `-Filter 'Name -eq "IT"'`: Filters for the OU named `IT`.
            - `| Select-Object Name, DistinguishedName`: Displays the OU’s name and distinguished name (e.g., `OU=IT,DC=ad,DC=mycompany,DC=local`).
    3.  **Add Objects (Optional)**:
        
        - Create a user in the OU:
            
            ```powershell
            New-ADUser -Name "John Doe" -SamAccountName "jdoe" -Path "OU=IT,DC=ad,DC=mycompany,DC=local" -AccountPassword (ConvertTo-SecureString "P@ssw0rd123!" -AsPlainText -Force) -Enabled $true -UserPrincipalName "jdoe@ad.mycompany.local"
            ```
            
            - **Parameter Descriptions for `New-ADUser`**:
                - `-Name "John Doe"`: Sets the full name of the user.
                - `-SamAccountName "jdoe"`: Specifies the logon name (pre-Windows 2000 format).
                - `-Path "OU=IT,DC=ad,DC=mycompany,DC=local"`: Places the user in the `IT` OU.
                - `-AccountPassword (ConvertTo-SecureString "P@ssw0rd123!" -AsPlainText -Force)`: Sets a secure password.
                - `-Enabled $true`: Enables the user account immediately.
                - `-UserPrincipalName "jdoe@ad.mycompany.local"`: Sets the UPN {User Principal Name} for modern logon (e.g., email-style login).
        - Move existing objects:
            
            ```powershell
            Move-ADObject -Identity "CN=John Doe,CN=Users,DC=ad,DC=mycompany,DC=local" -TargetPath "OU=IT,DC=ad,DC=mycompany,DC=local"
            ```
            
            - **Parameter Descriptions for `Move-ADObject`**:
                - `-Identity "CN=John Doe,CN=Users,DC=ad,DC=mycompany,DC=local"`: Specifies the distinguished name of the object to move.
                - `-TargetPath "OU=IT,DC=ad,DC=mycompany,DC=local"`: Defines the destination OU.
    4.  **Link a GPO (Optional)**:
        
        - Create a GPO:
            
            ```powershell
            New-GPO -Name "IT Security Policy" -Comment "Security settings for IT department"
            ```
            
            - **Parameter Descriptions for `New-GPO`**:
                - `-Name "IT Security Policy"`: Specifies the name of the new GPO.
                - `-Comment "Security settings for IT department"`: Adds a description for the GPO.
        - Link the GPO to the OU:
            
            ```powershell
            New-GPLink -Name "IT Security Policy" -Target "OU=IT,DC=ad,DC=mycompany,DC=local" -LinkEnabled Yes
            ```
            
            - **Parameter Descriptions for `New-GPLink`**:
                - `-Name "IT Security Policy"`: Specifies the GPO to link.
                - `-Target "OU=IT,DC=ad,DC=mycompany,DC=local"`: Links the GPO to the `IT` OU.
                - `-LinkEnabled Yes`: Ensures the GPO link is active.
        - Configure settings (e.g., minimum password length):
            
            ```powershell
            Set-GPRegistryValue -Name "IT Security Policy" -Key "HKLM\Software\Policies\Microsoft\Windows\System" -ValueName "MinimumPasswordLength" -Type DWord -Value 12
            ```
            
            - **Parameter Descriptions for `Set-GPRegistryValue`**:
                - `-Name "IT Security Policy"`: Specifies the GPO to modify.
                - `-Key "HKLM\Software\Policies\Microsoft\Windows\System"`: Defines the registry path for the setting.
                - `-ValueName "MinimumPasswordLength"`: Specifies the registry value to set.
                - `-Type DWord`: Sets the registry value type to a 32-bit integer.
                - `-Value 12`: Sets the minimum password length to 12 characters.
        - Force GPO update on clients:
            
            ```powershell
            gpupdate /force
            ```
            
            - **Parameter Descriptions for `gpupdate`**:
                - `/force`: Forces immediate application of all GPOs, ignoring cached policies.
    5.  **Delegate Control (Optional)**:
        
        - Grant permissions to manage the OU (e.g., for `ITAdmins` group):
            
            ```powershell
            $itAdminGroup = "CN=ITAdmins,OU=IT,DC=ad,DC=mycompany,DC=local"
            $acl = Get-Acl -Path "AD:OU=IT,DC=ad,DC=mycompany,DC=local"
            $sid = (Get-ADGroup -Identity "ITAdmins").SID
            $rule = New-Object System.Security.AccessControl.ActiveDirectoryAccessRule($sid, "GenericAll", "Allow", "All")
            $acl.AddAccessRule($rule)
            Set-Acl -Path "AD:OU=IT,DC=ad,DC=mycompany,DC=local" -AclObject $acl
            ```
            
            - **Parameter Descriptions for `Get-Acl`, `New-Object`, `Set-Acl`**:
                - `$itAdminGroup = "CN=ITAdmins,OU=IT,DC=ad,DC=mycompany,DC=local"`: Defines the distinguished name of the group (e.g., `ITAdmins`).
                - `Get-Acl -Path "AD:OU=IT,DC=ad,DC=mycompany,DC=local"`: Retrieves the ACL {Access Control List} for the `IT` OU.
                - `$sid = (Get-ADGroup -Identity "ITAdmins").SID`: Gets the SID {Security Identifier} of the `ITAdmins` group.
                - `New-Object System.Security.AccessControl.ActiveDirectoryAccessRule($sid, "GenericAll", "Allow", "All")`: Creates a rule granting full control to the group for all objects in the OU.
                - `Set-Acl -Path "AD:OU=IT,DC=ad,DC=mycompany,DC=local" -AclObject $acl`: Applies the updated ACL to the OU.
    6.  **Troubleshooting**:
        
        - Check replication status:
            
            ```powershell
            repadmin /replsummary
            ```
            
            - **Parameter Descriptions for `repadmin /replsummary`**:
                - No parameters needed; summarizes replication status across domain controllers to ensure OU creation is replicated.
        - View directory service errors:
            
            ```powershell
            Get-EventLog -LogName "Directory Service" -EntryType Error
            ```
            
            - **Parameter Descriptions for `Get-EventLog`**:
                - `-LogName "Directory Service"`: Specifies the event log to query.
                - `-EntryType Error`: Filters for error events to diagnose issues.

### 6\. Additional Best Practices

- **Backup AD DS**:
    - Create a system state backup to protect AD DS data:
        
        ```powershell
        wbadmin start systemstatebackup -backupTarget:D:
        ```
        
        - **Parameter Descriptions for `wbadmin start systemstatebackup`**:
            - `-backupTarget:D:`: Specifies the storage location for the backup (replace `D:` with an available drive).
- **Monitor AD DS Health**:
    - Regularly check event logs and run diagnostics:
        
        ```powershell
        Get-EventLog -LogName "Directory Service" | Select-Object -First 10
        ```
        
        - **Parameter Descriptions for `Get-EventLog`**:
            - `-LogName "Directory Service"`: Targets the AD DS event log.
            - `| Select-Object -First 10`: Limits output to the 10 most recent entries for review.
- **Secure the Domain Controller**:
    - Enable the Windows Firewall and configure rules to allow only necessary traffic (e.g., DNS, LDAP {Lightweight Directory Access Protocol}):
        
        ```powershell
        Set-NetFirewallProfile -Profile Domain -Enabled True
        ```
        
        - **Parameter Descriptions for `Set-NetFirewallProfile`**:
            - `-Profile Domain`: Targets the domain network profile.
            - `-Enabled True`: Enables the firewall for the domain profile.
- **Document Configuration**:
    - Export AD DS configuration for reference:
        
        ```powershell
        Get-ADDomain | Export-Clixml -Path "C:\AD_Domain_Config.xml"
        ```
        
        - **Parameter Descriptions for `Export-Clixml`**:
            - `-Path "C:\AD_Domain_Config.xml"`: Specifies the file path to save the XML configuration.

## Outcome

The server is a functional AD DS {Active Directory Domain Services} Domain Controller with an `IT` OU {Organizational Unit} for organizing resources and applying GPOs {Group Policy Objects}. Manage remotely using RSAT {Remote Server Administration Tools} from a Windows 10/11 client or via PowerShell.
