# Create svc-vmm user This is the account that the VMM service uses to operate in the background. like "ServiceAccount" `www-data` for nginx not Root user.
```powershell
Invoke-Command -VMName DC01 -Credential $domain -ScriptBlock {
    $c = $using:domain
    New-ADUser -Name svc-vmm -AccountPassword $c.Password -Enabled $true -PasswordNeverExpires $true
}
Invoke-Command -VMName VMM01 -Credential $domain -ScriptBlock {
    Add-LocalGroupMember -Group Administrators -Member LAB\svc-vmm
}
```

## Third Step:
-> After setting up the domain and DNS:
1) Join  HV01, HV02, HV03, HV04 to the domain.

```powershell
Get-NetAdapter
New-NetIPAddress -InterfaceAlias "Ethernet" -IPAddress 192.168.100.21 -PrefixLength 24 -DefaultGateway 192.168.100.1
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 192.168.100.10
Resolve-DnsName etisalat.local
Add-Computer -DomainName etisalat.local -NewName HV01 -Credential ETISALAT\Administrator -Restart
```

2) Check the DNS resolution, time, and authentication.
3) Prepare management and cluster accounts.
4) Run `Test-Cluster` on the four nodes.
5) Create the failover cluster.
6) Add the SAN LUNs, then configure the CSVs.
-> Note: The best practical approach is to set up the SAN and its paths from the outset—in parallel with the domain setup—but do not put the cluster into production until the validation process is complete.

## Fourth Step:
-> Connect the SAN with same Network with Hyper-V servers. and here there are two tpe:
1) iSCSI: It is possible to pass through Ethernet, but it is better to use a dedicated and isolated network or VLAN for storage, with independent paths and MPIO depending on the storage design.
2) Fiber Channel SAN: It uses Fabric and HBAs as its own connection methods, not just regular Ethernet.
-> Note: It doesn't recommanded connect the iSCSI to the same network as the VM traffic without proper design and separation. You also need to make sure that losing a Switch, NIC, or a single storage path doesn't stop data access.

# ###################################################################

## Install SCVMM:
1) Create two VM for SCVMM and his SQL.

```powershell
New-VM -Name VMM01 -Generation 2 -MemoryStartupBytes 16GB -SwitchName LabSwitch -NewVHDPath C:\Lab\VMs\VMM01\VMM01.vhdx -NewVHDSizeBytes 150GB
Set-VMProcessor VMM01 -Count 4
Set-VMMemory VMM01 -DynamicMemoryEnabled $false
Enable-VMIntegrationService -VMName VMM01 -Name "Guest Service Interface"
Add-VMDvdDrive -VMName VMM01 -Path C:\Lab\ISO\WS2025.iso
Set-VMFirmware VMM01 -FirstBootDevice (Get-VMDvdDrive -VMName VMM01)
Start-VM VMM01
vmconnect localhost VMM01
```

```powershell
Enter-PSSession -VMName VMM01 -Credential Administrator
Get-NetAdapter
New-NetIPAddress -InterfaceAlias "Ethernet" -IPAddress 192.168.100.30 -PrefixLength 24 -DefaultGateway 192.168.100.1
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 192.168.100.10
Resolve-DnsName etisalat.local
Add-Computer -DomainName etisalat.local -NewName VMM01 -Credential ETISALAT\Administrator -Restart
```

```powershell
New-VM -Name SQL01 -Generation 2 -MemoryStartupBytes 16GB -SwitchName LabSwitch -NewVHDPath C:\Lab\VMs\SQL01\SQL01.vhdx -NewVHDSizeBytes 256GB
Set-VMProcessor SQL01 -Count 4
Set-VMMemory SQL01 -DynamicMemoryEnabled $false
Enable-VMIntegrationService -VMName SQL01 -Name "Guest Service Interface"
Add-VMDvdDrive -VMName SQL01 -Path C:\Lab\ISO\WS2025.iso
Set-VMFirmware SQL01 -FirstBootDevice (Get-VMDvdDrive -VMName SQL01)
Start-VM SQL01
vmconnect localhost SQL01
```

1) Prepare SQL Server with a version supported by VMM 2025 on one VM recommanded always-on VM.

2) Install SCVMM on VM VMM01:
# Use an appropriate Service Account, and it is better to plan it from the beginning rather than using a personal Administrator account.
# Prepare the VMM Library and storage location for Templates and ISOs.
# Add the Hyper-V Cluster to the VMM Fabric, and each host is not a separate environment.
# Set Host Groups, Logical Networks, Logical Switches, IP Pools, and VM Networks according to the network design.
# The VMM Server must be a Domain Member, and the SQL Server must be compatible with the domain and connection requirements specified by Microsoft.

# ###################################################################

| **Windows ADK** + **WinPE add-on** | `https://learn.microsoft.com/windows-hardware/get-started/adk-install` |
1. **Download the Windows ADK 10.1.26100.9457 (September 2026)**
2. **Download the Windows PE add-on for the Windows ADK 10.1.26100.9457 (September 2026)**
| **VMM 2025** | `https://www.microsoft.com/en-us/evalcenter/download-system-center-2025` |

## Installing inside VMM01:
**1. unzip **SCVMM_2025 zip** then run **setup.exe** -> **Install**
**2. Select features to install**- **VMM management server**
**3. Product registration** Company name + **Product key:** 
**4. Accept License** and **Microsoft Update: Off** 
**5. Installation location:** Default ← **Prerequisites check**

**6. Database configuration**
| **Server name** | `VMM01` |
| **Use the following credentials** | (will use ETISALAT\Administrator or the use login) |
| **Instance name** | `MSSQLSERVER` |
| **New database** | `VirtualManagerDB` (default) |

**7. Configure service account and distributed key management**
| **Domain account** | ✅ |
| **User name** | `ETISALAT\svc-vmm` |
| **Password** | same password of ETISALAT\Administrator |
| **Store my keys in Active Directory** | ✅ ( use it only with VMM HA) |

**8. Port configuration:** Default
**9. Library configuration**- **Create a new library share** `MSSCVMMLibrary` (Default)
**10.** **Install** -> Maybe take 10-15 minutes

# ###################################################################