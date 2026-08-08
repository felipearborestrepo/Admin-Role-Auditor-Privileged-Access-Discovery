# Module Description
Built a PowerShell script that connects to Microsoft Entra ID via Microsoft Graph API, scans every active directory role in the tenant, identifies all role assignments, flags high risk privileged roles, and exports a prioritized report. Designed to give instant visibility into who has admin access — critical for security audits and M&A tenant discovery.

**Step 1 — Connected to Microsoft Graph**
- Connected to Entra ID tenant using two scopes
- RoleManagement.Read.All — new scope needed to read directory role assignments
- User.Read.All — needed to read user details on role members
- Without RoleManagement.Read.All you cannot read who has what admin role
```powershell
Connect-MgGraph -Scopes "RoleManagement.Read.All","User.Read.All" -NoWelcome
```
**Step 2 — Defined high risk roles list**
- Created an array of role names considered high risk
- These roles grant significant tenant-wide privileges
- Used later in the loop to classify each assignment as HIGH RISK or Standard
```powershell
$highRiskRoles = @(
    "Global Administrator"
    "Privileged Role Administrator"
    "Security Administrator"
    "User Administrator"
    "Password Administrator"
    "Application Administrator"
)
```
**Step 3 — Pulled all active directory roles**
- Used Get-MgDirectoryRole to retrieve all active roles in the tenant
- Only returns roles that have at least one member — inactive roles are excluded
- Found 17 active roles including Global Administrator, Security Administrator, User Administrator and more
- Confirmed count with Write-Host
```powershell
$roles = Get-MgDirectoryRole
Write-Host "Total active roles: $($roles.Count)" -ForegroundColor Green
```
**Step 4 — Created empty results list**
- Created empty list to collect every role assignment found during the scan
```powershell
$results = [System.Collections.Generic.List[PSCustomObject]]::new()
```
**Step 5 — Built the main scanning loop**
- Outer loop goes through every active directory role one at a time
- For each role called Get-MgDirectoryRoleMember to get all members assigned to that role
- Inner loop goes through every member of that role
- Used AdditionalProperties["displayName"] to get the user's name — role members return user details in AdditionalProperties dictionary instead of direct properties
- Used AdditionalProperties["userPrincipalName"] to get the user's UPN
- Checked if the role name exists in $highRiskRoles array using -contains
- Set risk level to "HIGH RISK" or "Standard" based on that check
- Printed high risk assignments in red and standard in cyan
- Added every assignment to $results as a custom object with five properties
```powershell
foreach ($role in $roles) {
    $members = Get-MgDirectoryRoleMember -DirectoryRoleId $role.Id
    foreach ($member in $members) {
        $displayName = $member.AdditionalProperties["displayName"]
        $upn         = $member.AdditionalProperties["userPrincipalName"]
        $isHighRisk  = $highRiskRoles -contains $role.DisplayName
        $riskLevel   = if ($isHighRisk) { "HIGH RISK" } else { "Standard" }
        $color       = if ($isHighRisk) { "Red" } else { "Cyan" }
        Write-Host "Role: $($role.DisplayName) | User: $displayName | Risk: $riskLevel" -ForegroundColor $color
        $results.Add([PSCustomObject]@{ ... })
    }
}
```
**Step 6 — Exported results to CSV**
- Exported all role assignments to AdminRoles_Report.csv on Desktop
- Sorted by RiskLevel first so HIGH RISK assignments appear at the top
- Then sorted by RoleName so assignments are grouped by role within each risk level
- Report includes: RoleName, DisplayName, UPN, MemberId, RiskLevel
```powershell
$results | Sort-Object RiskLevel, RoleName | Export-Csv -Path "$HOME/Desktop/AdminRoles_Report.csv" -NoTypeInformation
```
**Step 7 — Calculated summary counts**
- Filtered results by risk level using Where-Object
- () wraps each filter so PowerShell runs it first then counts the result
- Two counts used in the summary box
```powershell
$highRisk = ($results | Where-Object RiskLevel -eq "HIGH RISK").Count
$standard = ($results | Where-Object RiskLevel -eq "Standard").Count
```
**Step 8 — Printed formatted summary**
- Printed clean summary box showing total roles scanned, total assignments found, and breakdown by risk level
- High risk count in red, standard count in cyan
```powershell
Write-Host "  ADMIN ROLE AUDIT SUMMARY"
Write-Host "  Total roles scanned    : $($roles.Count)"
Write-Host "  Total role assignments : $($results.Count)"
Write-Host "  High risk assignments  : $highRisk"
Write-Host "  Standard assignments   : $standard"
```
**Results in PowerShell**

<img width="515" height="240" alt="Screenshot 2026-07-29 at 21 59 12" src="https://github.com/user-attachments/assets/3f99da32-9666-435f-99ff-03674af0519d" />

**Results in the CSV file**

<img width="1032" height="103" alt="Screenshot 2026-07-29 at 22 00 12" src="https://github.com/user-attachments/assets/afc8a036-e8ca-4ecd-a88e-1e22c210c3d8" />



