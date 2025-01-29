# Solution for Hyper-V Default Switch Changing IP After Restart

## Problem
When you search for this issue on Google, you will likely come across the following explanation:

> "The Hyper-V Default Switch will randomly use one of these IP address ranges based on the host system IP address:
>
> Start IP: 192.168.0.0 – End IP: 192.168.255.255  
> Start IP: 172.17.0.0 – End IP: 172.31.255.255  
>
> The Default Switch has a built-in local collision detection mechanism to determine if an IP address in these ranges is already assigned to a network interface on the Local Host system and will try to find a (/28 – 255.255.255.240) address space within one of these ranges that is not already allocated to any network interfaces.
>
> The Default Switch on the Host system always assigns the first IP address of the chosen range to the virtual Ethernet Adapter on the Host and uses this address as the Default Gateway and DNS server provided to the hosted VMs connected to it.
>
> You cannot remove or manage "DHCP" on the default switch."

This means that every time your system restarts, the IP of the Default Switch may change, making it difficult to use static IP addresses in your virtual machines without creating additional network adapters.

## Insight
It is quite interesting that this remains an unresolved issue. However, for those who need one or more static IPs without using DHCP, the solution is usually "simple." Because of this, it seems that not much attention has been given to the issue. 

In this context, I felt the need to share this "simple" solution here.

## Understanding the Issue
Before jumping to the solution, let's analyze the situation. As you know, the internet functions by packets "hopping" from one place to another. To check which IP range has been assigned after a restart, you can use the following PowerShell command:

```powershell
Get-NetIPAddress -InterfaceIndex (Get-NetAdapter -Name 'vEthernet (Default Switch)').ifIndex
```

For example, my output was:

```
IPAddress         : fe80::b663:bae6:6053:6df7%23
InterfaceIndex    : 23
InterfaceAlias    : vEthernet (Default Switch)
AddressFamily     : IPv6
Type              : Unicast
PrefixLength      : 64
PrefixOrigin      : WellKnown
SuffixOrigin      : Link
AddressState      : Preferred
ValidLifetime     :
PreferredLifetime :
SkipAsSource      : False
PolicyStore       : ActiveStore

IPAddress         : 172.26.32.1
InterfaceIndex    : 23
InterfaceAlias    : vEthernet (Default Switch)
AddressFamily     : IPv4
Type              : Unicast
PrefixLength      : 20
PrefixOrigin      : Manual
SuffixOrigin      : Manual
AddressState      : Preferred
ValidLifetime     :
PreferredLifetime :
SkipAsSource      : False
PolicyStore       : ActiveStore
```

As seen above, the assigned virtual gateway received an IPv4 address of `172.26.32.1`.

Now, let's examine the packet hopping process by running the following command:

```powershell
Get-NetRoute
```

The output I received:

```
ifIndex DestinationPrefix                              NextHop                                  RouteMetric ifMetric PolicyStore
------- -----------------                              -------                                  ----------- -------- -----------
16      255.255.255.255/32                             0.0.0.0                                          256 25       ActiveStore
23      255.255.255.255/32                             0.0.0.0                                          256 5000     ActiveStore
15      255.255.255.255/32                             0.0.0.0                                          256 25       ActiveStore
4       255.255.255.255/32                             0.0.0.0                                          256 40       ActiveStore
19      255.255.255.255/32                             0.0.0.0                                          256 65       ActiveStore
3       255.255.255.255/32                             0.0.0.0                                          256 20       ActiveStore
1       255.255.255.255/32                             0.0.0.0                                          256 75       ActiveStore
16      224.0.0.0/4                                    0.0.0.0                                          256 25       ActiveStore
23      224.0.0.0/4                                    0.0.0.0                                          256 5000     ActiveStore
15      224.0.0.0/4                                    0.0.0.0                                          256 25       ActiveStore
4       224.0.0.0/4                                    0.0.0.0                                          256 40       ActiveStore
19      224.0.0.0/4                                    0.0.0.0                                          256 65       ActiveStore
3       224.0.0.0/4                                    0.0.0.0                                          256 20       ActiveStore
1       224.0.0.0/4                                    0.0.0.0                                          256 75       ActiveStore
4       192.168.1.255/32                               0.0.0.0                                          256 40       ActiveStore
4       192.168.1.5/32                                 0.0.0.0                                          256 40       ActiveStore
4       192.168.1.0/24                                 0.0.0.0                                          256 40       ActiveStore
23      172.26.47.255/32                               0.0.0.0                                          256 5000     ActiveStore
23      172.26.32.1/32                                 0.0.0.0                                          256 5000     ActiveStore
23      172.26.32.0/20                                 0.0.0.0                                          256 5000     ActiveStore
```

Here, we can see that packets hopping to `**0.0.0.0**` are being sent to my modem at `**192.168.1.1**`, which then routes them to the internet. The range `**172.26.32.0/20**` belongs to our virtual adapter named **Default Switch**. The IP `**172.26.32.1/32**` acts as the **Gateway**, while `**172.26.47.255/32**` is likely the **Broadcast address**.

In my solution, instead of using `**/20 (255.255.240.0)**`, I will use `**/24 (255.255.255.0)**`. This means we expect to see:

- 🟢 `**192.168.100.0/24**`  *(Network Range)*
- 🏠 `**192.168.100.1/32**` *(Gateway)*
- 🚀 `**192.168.100.255/32**` *(Broadcast)*

If these addresses are missing, we won't have internet access, which can be fixed using `New-NetRoute`. If you need more details, you should search for this topic on Google, though it shouldn't be necessary.

---

### 🔧 Now, let's move on to the **"simple" solution:**

## Keeping Hyper-V Default Switch IP After Restart

My idea is to retain the IP address after a restart while ensuring that DHCP-based servers continue to function as before. Therefore, adding a new IP address with a single command is our real **"simple" solution.**

```powershell
$InterfaceIndex = (Get-NetAdapter -Name 'vEthernet (Default Switch)').ifIndex
New-NetIPAddress -InterfaceIndex $InterfaceIndex -IPAddress 192.168.100.1 -PrefixLength 24
```

After executing this command, you can check whether the expected routes have been added using `Get-NetRoute`. In my case, I found that they were successfully added.

### ⚠ Important Note:
Some modern physical switches use the **192.168.100.0** network. In this case, choosing a different IP address would be safer. Additionally, there is a possibility of occasional conflicts with the randomly assigned network. To prevent this, you could prepare a PowerShell script to remove the virtual switch's IP before applying this fix, though I chose not to.

At the end of this process, our **vEthernet (Default Switch)** will contain both the dynamically assigned network range and the manually assigned range. Now, we can assign **static IPs between 192.168.100.2 and 192.168.100.254** to our virtual machines, and they will have internet access.

---

## Automating the Process on Startup

Do we need to run this command manually every time?

Yes, but we can delegate this task to the operating system using **Task Scheduler (taskschd.msc).**

### Steps:
1. Create a folder: `C:\Users\%USERPROFILE%\StartupScripts`
2. Inside this folder, create a file: **Set-HyperVDefaultSwtichIP.ps1** and place the "simple" solution command inside.
3. Open **Task Scheduler (taskschd.msc).**
4. Click **Create Task** and configure:
   - **General / Name:** `Set-HyperVDefaultSwitchIP`
   - **General / Description:** Adds an additional IP range to Hyper-V Default Switch.
   - **Triggers / Add:**
     - **At log on** - This user.
   - **Actions / Add:**
     - **Start a program** - `powershell.exe -ExecutionPolicy Bypass -File "C:\Users\xxxYOURUSERHERExxx\StartupScripts\Set-HyperVDefaultSwitchIP.ps1"`

That's it! 🎉

---

## Why Use This Instead of Creating a Public Switch?

There can be various reasons for this. My reason is that I manage remote servers from the same **Hyper-V Manager** interface, and creating a **Public Switch** significantly slows down remote server management.

