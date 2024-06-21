## Monitoring activity of Check Point HA VMs using System Identity in Azure

### Motivation

What are real actions done by CPHA VMs? How to monitor them? This article will show you how to monitor the activity of CPHA VMs using Managed Service Identity (MSI) in Azure's Activity Log using PowerShell.

### Prerequisites

- Azure PowerShell
- Azure subscription
- Check Point HA cluster deployed in Azure

### Steps

```powershell

# login to Azure subscription, if needed
Connect-AzAccount

# assume we have VM cpha11 in resource group CPHA1 - need id of service principal of VM's system assigned identity
(Get-AzVM -ResourceGroupName CPHA1 -Name cpha11).Identity.PrincipalId
# 96effe93-cc60-4d14-b6a0-f4e26079f4c0

# similar for other cluster member cpha2
(Get-AzVM -ResourceGroupName CPHA1 -Name cpha12).Identity.PrincipalId
# 80c13982-495f-4876-9f69-48b7eb737438

# learn how to consume Azure Activity Log
get-help Get-AzActivityLog -detailed

# naive implementation - get all activity logs for last 24 hours and filter by caller (cpha1 and cpha2) and summarize by resource and operation
Get-AzActivityLog -StartTime (Get-Date).AddDays(-1) -EndTime (Get-Date)   | ? { ($_.Caller -eq "80c13982-495f-4876-9f69-48b7eb737438" ) -or ($_.Caller -eq "96effe93-cc60-4d14-b6a0-f4e26079f4c0" ) } | Group-Object ResourceId,OperationName | select Count,Name

# Count Name
# ----- ----
#    27 /subscriptions/f4ad5e85-ec75-4321-8854-ed7eb611f61d/resourceGroups/cpha1/providers/Microsoft.Network/networkInterfaces/cpha11-eth0, Create or Update Network Interface
#    12 /subscriptions/f4ad5e85-ec75-4321-8854-ed7eb611f61d/resourceGroups/cpha1/providers/Microsoft.Network/networkInterfaces/cpha11-eth1, Create or Update Network Interface
#    12 /subscriptions/f4ad5e85-ec75-4321-8854-ed7eb611f61d/resourcegroups/cpha1/providers/Microsoft.Network/networkInterfaces/cpha12-eth0, Create or Update Network Interface
#     6 /subscriptions/f4ad5e85-ec75-4321-8854-ed7eb611f61d/resourceGroups/cpha1/providers/Microsoft.Network/networkInterfaces/cpha12-eth1, Create or Update Network Interface

# we should filter caller close to source of data, and handle only data we need
Get-AzActivityLog -StartTime (Get-Date).AddDays(-1) -EndTime (Get-Date) -Caller "80c13982-495f-4876-9f69-48b7eb737438" | select ResourceId,OperationName | group-object ResourceId,OperationName -noelement | select Name,Count

# Name                                                                                                                                                                   Count
# ----                                                                                                                                                                   -----
# /subscriptions/f4ad5e85-ec75-4321-8854-ed7eb611f61d/resourceGroups/cpha1/providers/Microsoft.Network/networkInterfaces/cpha11-eth0, Create or Update Network Interface    12
# /subscriptions/f4ad5e85-ec75-4321-8854-ed7eb611f61d/resourceGroups/cpha1/providers/Microsoft.Network/networkInterfaces/cpha11-eth1, Create or Update Network Interface     6
# /subscriptions/f4ad5e85-ec75-4321-8854-ed7eb611f61d/resourceGroups/cpha1/providers/Microsoft.Network/networkInterfaces/cpha12-eth0, Create or Update Network Interface     9
# /subscriptions/f4ad5e85-ec75-4321-8854-ed7eb611f61d/resourceGroups/cpha1/providers/Microsoft.Network/networkInterfaces/cpha12-eth1, Create or Update Network Interface     6

# but what exactly is the call on NIC?
Get-AzActivityLog -StartTime (Get-Date).AddDays(-1) -EndTime (Get-Date) -Caller "80c13982-495f-4876-9f69-48b7eb737438" -ResourceId "/subscriptions/f4ad5e85-ec75-4321-8854-ed7eb611f61d/resourceGroups/cpha1/providers/Microsoft.Network/networkInterfaces/cpha11-eth0" | select -First 1 | fl *

# Properties have requestbody
Get-AzActivityLog -StartTime (Get-Date).AddDays(-1) -EndTime (Get-Date) -Caller "80c13982-495f-4876-9f69-48b7eb737438" -ResourceId "/subscriptions/f4ad5e85-ec75-4321-8854-ed7eb611f61d/resourceGroups/cpha1/providers/Microsoft.Network/networkInterfaces/cpha11-eth0" | % { $_.Properties.Content.requestbody }

# read one of the requestbody formatted
Get-AzActivityLog -StartTime (Get-Date).AddDays(-1) -EndTime (Get-Date) -Caller "80c13982-495f-4876-9f69-48b7eb737438" -ResourceId "/subscriptions/f4ad5e85-ec75-4321-8854-ed7eb611f61d/resourceGroups/cpha1/providers/Microsoft.Network/networkInterfaces/cpha11-eth0" | select -Last 4 | % { $_.Properties.Content.requestbody } | ConvertFrom-Json | ConvertTo-Json -Depth 10

# focus on ipConfigurations on eth0 - once with and without cluster-vip - when moved on cluster fail-over
```