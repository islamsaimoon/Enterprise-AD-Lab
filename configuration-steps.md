# Configuration Steps for Enterprise AD Lab

This document provides detailed instructions to reproduce the **Enterprise Active Directory Infrastructure Lab**.  Each section includes prerequisites and the exact steps required to configure your virtual machines, install roles, create organizational units, apply group policies, configure file shares and folder redirection, and join a client to the domain.

## 1. Prepare the Servers

1. **Install Windows Server 2022** on two virtual machines for domain controllers (DC1 and DC2) and on a third VM for the file server.  Install Windows 10 on a separate client VM.  
2. **Rename each server** (e.g., DC1, DC2, FileServer) and configure **static IP addresses**.  Assign IPs within the same subnet and configure the preferred DNS server on DC1 to its own address; set DC2’s preferred DNS to DC1.  Static IPs ensure consistent connectivity, a prerequisite for Active Directory【932202218295430†L178-L193】.  
3. **Update the servers** via Windows Update and install the latest drivers to avoid issues during role installation.

## 2. Install AD DS and Promote DC1

1. Log on to **DC1** with administrative privileges.  
2. Open **Server Manager** and select **Manage → Add Roles and Features**.  In the wizard, choose **Role‑based or feature‑based installation**, select the local server, and tick **Active Directory Domain Services**.  Accept the required features and click **Install**【932202218295430†L233-L279】.  
3. After the role installation completes, a yellow warning appears.  Click **Promote this server to a domain controller**.  
4. In the **Deployment Configuration** page, select **Add a new forest** and enter a root domain name (e.g., `corp.example.com`).  
5. On **Domain Controller Options**, leave default selections (Domain Controller, DNS and Global Catalog) and specify a **Directory Services Restore Mode (DSRM)** password.  
6. Ignore any DNS delegation warnings, keep default paths for the AD database, SYSVOL and logs, review selections and run the **prerequisites check**.  If all checks pass, click **Install**.  The server will reboot automatically, completing the promotion【932202218295430†L331-L432】.

### Verify DC1

- Sign in using the domain credentials (e.g., `CORP\Administrator`).  
- Open **Active Directory Users and Computers** to confirm the new domain exists.  
- Open **DNS Manager** to see that the forward lookup zone for the new domain has been created and that DC1 is listed as a nameserver.

## 3. Install DNS and DHCP Roles

1. On **DC1**, open **Server Manager** and add the **DNS Server** and **DHCP Server** roles.  The DNS role is installed automatically with AD DS; you still need to install and configure DHCP.  
2. In the **Add Roles and Features** wizard, select **DHCP Server** and confirm installation【625699008318383†L438-L480】.  After installation, click the notification flag and complete post‑deployment configuration.  
3. Launch **DHCP Manager** via *Tools → DHCP*.  Right‑click the server name and select **Authorize** to authorize it in Active Directory.  
4. Expand **IPv4** and right‑click **New Scope**.  Use the wizard to create a scope: specify a **scope name**, **start and end IP addresses**, **subnet mask** and any **excluded ranges**【625699008318383†L576-L621】.  
5. Configure **DHCP Options**: set the **default gateway**, **DNS domain name** and **DNS server IP** (use DC1’s IP)【625699008318383†L601-L621】.  Activate the scope and refresh; you should see the new scope running【625699008318383†L630-L649】.

## 4. Add DC2 as an Additional Domain Controller

1. Join **DC2** to the domain by setting its primary DNS to DC1’s IP and using **System → About → Join domain**.  After the reboot, log on with domain credentials.  
2. Install the **AD DS** role on DC2 using the Add Roles and Features wizard, as you did for DC1.  
3. After installation, select **Promote this server to a domain controller**.  Choose **Add a domain controller to an existing domain**, enter domain admin credentials, and specify a **DSRM password**【280511281745717†L169-L232】.  
4. Select a replication partner (choose DC1 if available), accept default paths, review selections and run the prerequisites check.  Then click **Install**.  DC2 will reboot and become a replica domain controller.  
5. After reboot, verify that DC2 appears under the **Domain Controllers** OU and that replication is healthy by running `repadmin /showrepl`【280511281745717†L276-L307】.  
6. Adjust DNS settings so that DC1 and DC2 point to each other for Preferred/Alternate DNS to improve fault tolerance【280511281745717†L236-L255】.

## 5. Create Organizational Units (OUs)

Organizational units allow you to group users, computers and other objects for administration.  

1. On either domain controller, open **Active Directory Users and Computers**.  Expand the domain and right‑click the domain name.  Select **New → Organizational Unit**【791934392047309†L45-L56】.  
2. Name the OU according to your department (e.g., **HR**, **IT**, **Finance**) and ensure that **Protect container from accidental deletion** is ticked.  
3. Repeat for each department.  Later, you can create sub‑OUs for teams if needed.  
4. Move user and computer accounts into the appropriate OU by dragging them or using the **Move** option.

## 6. Create and Link Group Policy Objects (GPOs)

Group Policy allows centralized configuration of user and computer settings.  

1. Open **Group Policy Management** from Server Manager’s Tools menu.  
2. Right‑click **Group Policy Objects** and choose **New** to create a new GPO.  Give it a descriptive name (e.g., `HR-SecurityPolicy`).  
3. Select the GPO in the list and, under **Security Filtering**, add or remove users, groups or computers that the policy should apply to【126904375430747†L71-L97】.  
4. Right‑click the GPO and choose **Edit**.  Use the **Group Policy Management Editor** to configure settings under **Computer Configuration** or **User Configuration**.  For example, you can enforce password policies, disable Control Panel access, or deploy software.  
5. To apply the GPO, locate the target OU in Group Policy Management, right‑click it and select **Link an existing GPO**.  Choose the GPO you created and click **OK**【126904375430747†L71-L97】.  
6. To test the policy, run `gpupdate /force` on a client machine and `gpresult /r` to view applied GPOs.

## 7. Configure Folder Redirection (Optional)

Folder redirection stores user data on a network share so it is available on any domain computer and simplifies backups.  

1. On the file server, create a root folder (e.g., `\FileServer\UserDocs`) and configure NTFS permissions so that **Administrators**, **System** and **Authenticated Users** have modify access【330814559545438†screenshot】.  Share the folder and grant **Authenticated Users** and **Domain Admins** **Full Control**【330814559545438†screenshot】.  
2. In **Group Policy Management**, create a new GPO (e.g., `Folder-Redirection`) and link it to the OU containing user accounts.  
3. Edit the GPO: under **User Configuration → Policies → Windows Settings → Folder Redirection**, right‑click **Documents** and select **Properties**.  Set the redirection option to **Basic – redirect everyone’s folder to the same location**.  Enter the UNC path to the shared folder created earlier【330814559545438†screenshot】.  
4. On the **Settings** tab, you may choose to uncheck **Grant the user exclusive rights to Documents** to simplify access for administrators【330814559545438†screenshot】.  Click **OK** to apply the policy.  Users will be prompted to log off/log on to apply redirection.

## 8. Configure the File Server

Creating departmental shares allows controlled access to data.  

1. Decide on the folder structure for your departments (e.g., `E:\Shares\HR`, `E:\Shares\IT`, `E:\Shares\Finance`).  Create these folders on the file server.  
2. **Share each folder**: right‑click the folder, choose **Properties → Sharing → Advanced Sharing** and check **Share this folder**【83614535428773†L575-L627】.  Click **Permissions**, remove `Everyone` and add domain groups (e.g., `HR_Group`) granting them appropriate share permissions (e.g., Read/Write).  Add **Domain Administrators** with Full Control【83614535428773†L615-L627】.  
3. **Set NTFS permissions**: go to the **Security** tab, click **Edit** and add the same groups.  Grant the necessary NTFS permissions (Full Control for HR_Group, Read/Execute for others)【83614535428773†L631-L671】.  Click **OK** to save.  
4. Test access from a client to ensure that users in the correct groups can create, modify and delete files while others cannot.

## 9. Join the Windows 10 Client to the Domain

1. On the Windows 10 VM, open **Settings → System → About → Join a domain** (alternatively, use **System Properties → Computer Name**).  Enter the domain name (e.g., `corp.example.com`) and provide credentials of a domain user that has rights to join computers to the domain.  
2. After the reboot, log on using a domain account.  Verify that you can access the shared folders and that group policies (including folder redirection) are applied.  Use `gpresult /r` to see the list of applied policies.

## 10. Troubleshooting Tips

- **Cannot join domain** – Ensure the client’s DNS server points to DC1 or DC2 and that time settings are in sync.  Try pinging the domain name to confirm DNS resolution.  
- **DHCP not handing out addresses** – Verify that the scope is active and authorized.  Check that no conflicting DHCP servers exist.  
- **GPOs not applying** – Use `gpupdate /force` on clients and check the GPO’s Security Filtering and link order.  Ensure the GPO is linked to the correct OU and that inheritance isn’t blocked.  
- **Folder redirection errors** – Confirm share and NTFS permissions are correct and that the UNC path is accessible from clients.  Examine the event viewer for folder redirection warnings.

By following the above steps, you will build a complete multi‑tier Active Directory environment with DHCP, DNS, OUs, GPOs, folder redirection and a file server, similar to what enterprises use in production environments.
