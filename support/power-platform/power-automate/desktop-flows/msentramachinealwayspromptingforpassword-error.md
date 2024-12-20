---
title: Unattended Desktop Flow Run Fails with MSEntraMachineAlwaysPromptingForPassword
description: Solves an error that occurs when you run an unattended desktop flow in Microsoft Power Automate for desktop.
ms.author: moelaabo
ms.reviewer: guco, alarnaud
ms.custom: sap:Desktop flows\Unattended flow runtime errors
ms.date: 12/09/2024
---
# An unattended desktop flow run fails with the MSEntraMachineAlwaysPromptingForPassword error

This article provides a resolution for an error that occurs when you run an unattended desktop flow in Microsoft Power Automate for desktop.

## Symptoms

Your unattended desktop flow run fails with the "MSEntraMachineAlwaysPromptingForPassword" error code (formerly "AADMachineAlwaysPromptingForPassword").

```jsonc
{
    "error":{
        "code": "MSEntraMachineAlwaysPromptingForPassword",
        "message": "Could not create unattended session with these credentials."  
    }    
}
```

:::image type="content" source="media/msentramachinealwayspromptingforpassword-error/msentramachinealwayspromptingforpassword.png" alt-text="Screenshot of the error code shown in the Body section of the Run a flow built with Power Automate for desktop page.":::

## Cause

Power Automate for desktop can't validate your Microsoft Entra ID (formerly Azure Active Directory) credentials on the machine. This issue is typically caused by a group policy setting on your machine.

## Recommended resolution (public preview)

There are 2 possible mitigations that are leveraging [RDS AAD Auth Security](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-rdpbcgr/dc43f040-d75d-49a9-90c6-0c9999281136) to obtain “RD tokens” to open a windows session.

### Using RDS AAD Authentication with a user/password and MFA exception
This first option uses RD tokens obtained from a username and password. It requires an MFA exception for the desktop flow connection accounts.
It is available since PAD 2.49 and is activated by default when Network Level Authentication policy is set, or can be enforced with a PAD registry setting.

To activate at PAD level only when using 
 
|Registry Path|Value|Type|Value|
|Computer\HKEY_LOCAL_MACHINE\SOFTWARE\WOW6432Node\Microsoft\Power Automate Desktop\Service|UseRdsAadAuthentication|DWORD-32|1|
 
To activate Network Level Authentication at machine level 
|Registry Path|Value|Type|Value|Comment|
|Computer\HKEY_LOCAL_MACHINE\Software\Policies\Microsoft\Windows NT\Terminal Services|UserAuthentication|DWORD-32|1|NLA activated by policy (overrides the below)|
|Computer\HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp|UserAuthentication|DWORD-32|1|NLA activated on RDP-Tcp listener|(only if above   policy is missing)|

If activated at machine level, be aware that it impacts all the RD clients: check [how to configure remote desktop clients Use the Remote Desktop Connection app to connect to a remote PC using single sign-on with Microsoft Entra authentication | Microsoft Learn](https://learn.microsoft.com/windows-server/remote/remote-desktop-services/clients/remote-desktop-connection-single-sign-on#connect-to-a-remote-pc-using-single-sign-on-with-microsoft-entra-authentication)

### Using RDS AAD Authentication with a certificate
This second option uses RD tokens obtained from a certificate. It does not require MFA exception.
It is available since PAD 2.49 but requires PAD 2.52 when fPromptForPassword is enabled.
Check how to setup [Certificate Based Authentication for desktop flows](https://learn.microsoft.com/power-automate/desktop-flows/configure-certificate-based-auth).

### Hiding consent prompts for RDS AAD Authentication
Both options require [hiding the Remote Desktop MSEntra app consent prompts for a device group](https://learn.microsoft.com/power-automate/desktop-flows/run-unattended-desktop-flows#admin-consent-for-unattended-runs-using-cba-or-sign-in-credentials-with-nla-preview).

## Other possible resolution

To solve this issue, check the group policy setting on your machine.

1. Press the Windows key+<kbd>R</kbd> to open the **Run** dialog.
1. Type **gpedit.msc** and press <kbd>Enter</kbd> to open the Local Group Policy Editor.
1. Navigate to **Computer Configuration** > **Administrative Templates** > **Windows Components** > **Remote Desktop Services** > **Remote Desktop Session Host** > **Security**.
1. Look for the **Always prompt for password upon connection** setting.

   - If the setting is enabled, work with your IT department to disable the policy for that machine.

     > [!NOTE]
     > This value is also reflected in the registry at **Computer\HKEY_LOCAL_MACHINE\SOFTWARE\Policies\Microsoft\Windows NT\Terminal Services**. If the **fPromptForPassword** DWORD value for the **Terminal Services** key is set to **1**, the setting is enabled, and you need to work with your IT department to disable it (simply changing the registry value is generally not sufficient, as it might be reverted.)

   - If the **Always prompt for password upon connection** setting isn't enabled but you receive the error code, type **regedit** in the **Run** dialog to open the Registry Editor. In the Registry Editor, navigate to the **Computer\HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp** registry key. Then, look for the **fPromptForPassword** DWORD and set it to **0**. If the DWORD doesn't exist, create it and set its value to **0**.
