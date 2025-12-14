# ============================================================
# Hyper-V SET & Cluster Network Builder
#
# This script builds:
#   - A SET-based vSwitch (no Management OS attached)
#   - Management OS vNICs for CLUSTER TRAFFIC ONLY
#
# Assumptions (important):
#   - You already have a separate physical MGMT NIC configured
#   - These vNICs do NOT get default gateways (by design)
#   - Bandwidth weights are used to protect CSV & LM traffic
#
# This is meant to be run ONCE on a new host, not repeatedly.
# ============================================================

Write-Host "=== Hyper-V SET & Cluster vNIC Builder ===" -ForegroundColor Cyan
Write-Host ""
Write-Host "Quick sanity check before we start:" -ForegroundColor Gray
Write-Host "  • Separate physical MGMT NIC is already configured" -ForegroundColor Gray
Write-Host "  • These vNICs are for cluster traffic only" -ForegroundColor Gray
Write-Host "  • No default gateways will be configured" -ForegroundColor Gray
Write-Host ""

# ----------------------------
# Create the SET vSwitch
# ----------------------------

$switchName = Read-Host "Enter the name for the SET vSwitch"

$nicInput = Read-Host "Enter physical NICs for SET (comma separated, e.g. set1,set2)"
$physicalNics = $nicInput.Split(",") | ForEach-Object { $_.Trim() }

Write-Host ""
Write-Host "Creating SET vSwitch '$switchName'..." -ForegroundColor Yellow
Write-Host "Management OS will NOT be attached (this is intentional)." -ForegroundColor DarkGray

New-VMSwitch `
    -Name $switchName `
    -NetAdapterName $physicalNics `
    -EnableEmbeddedTeaming $true `
    -AllowManagementOS $false `
    -MinimumBandwidthMode Weight

# ----------------------------
# Bandwidth guidance (human, not dogmatic)
# ----------------------------

Write-Host ""
Write-Host "Bandwidth weight guidance (nothing is enforced, just smart defaults):" -ForegroundColor Cyan
Write-Host "  • CSV / Storage        : 50–60  (this keeps the cluster alive)" -ForegroundColor Gray
Write-Host "  • Live Migration (LM)  : 25–35  (fast moves without killing storage)" -ForegroundColor Gray
Write-Host "  • Replica (Optional)  : 10–20  (async, should never starve CSV)" -ForegroundColor Gray
Write-Host "  • Cluster / Heartbeat : 5–10   (small but important)" -ForegroundColor Gray
Write-Host "  • Aim for a total around 100" -ForegroundColor DarkGray
Write-Host ""

# ----------------------------
# How many cluster networks?
# ----------------------------

$vnicCount = Read-Host "How many cluster vNICs do you want to create (excluding MGMT)?"
$vnicCount = [int]$vnicCount

# ----------------------------
# Optional Replica network
# ----------------------------

$useReplica = Read-Host "Will this host use Hyper-V Replica traffic? (y/n)"

if ($useReplica -match '^[Yy]') {
    Write-Host ""
    Write-Host "Replica network will be included." -ForegroundColor Cyan
    Write-Host "This is optional and only needed if you're actually replicating VMs." -ForegroundColor DarkGray
    Write-Host "Recommended weight: 10–20 (Replica is async and can yield)." -ForegroundColor DarkGray
    $vnicCount++
}

# ----------------------------
# Build each cluster vNIC
# ----------------------------

for ($i = 1; $i -le $vnicCount; $i++) {

    Write-Host ""
    Write-Host "--------------------------------------------" -ForegroundColor DarkGray
    Write-Host "Configuring cluster vNIC $i of $vnicCount" -ForegroundColor Cyan
    Write-Host "--------------------------------------------" -ForegroundColor DarkGray

    # Friendly naming goes a long way when troubleshooting at 2am
    $vnicName = Read-Host "Enter vNIC name (CSV, LM, REPL, CLS, etc.)"

    # Suggest a weight based on common naming patterns
    $weightSuggestion = switch -Regex ($vnicName) {
        'CSV|STOR|STORAGE' { '50–60 (CSV / Storage)' }
        'LM|LIVE'          { '25–35 (Live Migration)' }
        'REPL|REPLICA'     { '10–20 (Replica)' }
        'CLS|CLUSTER|HB'   { '5–10 (Heartbeat)' }
        default            { 'Custom (just keep total ≈ 100)' }
    }

    $weight = Read-Host "Enter bandwidth weight for $vnicName (recommended: $weightSuggestion)"
    $weight = [int]$weight

    $vlanId = Read-Host "Enter VLAN ID for $vnicName"
    $vlanId = [int]$vlanId

    Write-Host "IP configuration (no gateway will be set):" -ForegroundColor Gray
    $ipAddress = Read-Host "  IP address"
    $prefix = Read-Host "  Prefix length (e.g. 24)"
    $prefix = [int]$prefix

    Write-Host ""
    Write-Host "Creating vNIC '$vnicName'..." -ForegroundColor Yellow

    # Create Management OS vNIC
    Add-VMNetworkAdapter `
        -ManagementOS `
        -SwitchName $switchName `
        -Name $vnicName

    # Clean up the ugly 'vEthernet (Name)' adapter naming
    Rename-NetAdapter `
        -Name "vEthernet ($vnicName)" `
        -NewName $vnicName

    # Apply bandwidth reservation
    Set-VMNetworkAdapter `
        -ManagementOS `
        -Name $vnicName `
        -MinimumBandwidthWeight $weight

    # VLAN tagging
    Set-VMNetworkAdapterVlan `
        -ManagementOS `
        -VMNetworkAdapterName $vnicName `
        -Access `
        -VlanId $vlanId

    # IP address (intentionally no default gateway)
    New-NetIPAddress `
        -InterfaceAlias $vnicName `
        -IPAddress $ipAddress `
        -PrefixLength $prefix
}

Write-Host ""
Write-Host "=== Cluster network configuration complete ===" -ForegroundColor Green
Write-Host "Next steps (recommended):" -ForegroundColor Cyan
Write-Host "  • Verify NIC order and names in Failover Cluster Manager" -ForegroundColor Gray
Write-Host "  • Assign cluster network roles (CSV, LM, etc.)" -ForegroundColor Gray
Write-Host "  • Test Live Migration and CSV failover" -ForegroundColor Gray
