# Cloudflare Dynamic DNS Updater
# ===============================

# SETTINGS
$cloudflareApiToken = "44HnYA*********FLi****IQL_****"
$zoneName = "your zone name"

# List all records you want to update
$recordNames = @(
    "@",
    "your a record 1",
    "your a record 2",
    "your a record 3"
)
# TXT SPF record name
$spfRecordName = "Your spf record if you have one"          # for txt records used for email servers, for example   v=spf1 ip4:59.5.61.44 a:yourdomain.com -all

# TTL and proxied settings for A records
$ttlDefault = 1
$proxiedDefault = $false

# ----------------------------

# Get public IPv4 with multiple fallback providers
function Get-PublicIP {
    $providers = @(
        "https://api.ipify.org?format=json",
        "https://ip.seeip.org/json",
        "https://ifconfig.me/ip",
        "https://checkip.amazonaws.com",
        "https://ipv4.icanhazip.com",
        "https://ipinfo.io/ip"
    )

    foreach ($url in $providers) {
        try {
            $resp = Invoke-RestMethod -Uri $url -TimeoutSec 5
            if ($resp -is [string]) { $ip = $resp.Trim() }
            elseif ($resp.ip) { $ip = $resp.ip }
            elseif ($resp.address) { $ip = $resp.address }
            if ($ip -match '^(?:[0-9]{1,3}\.){3}[0-9]{1,3}$') { return $ip }
        } catch { }
    }

    throw "Unable to retrieve public IP from any provider."
}

# Normalize record names to FQDN
function Normalize-ToFQDN($inputName, $zoneName) {
    if ($inputName -eq "@") { return $zoneName }
    if ($inputName -match "\.$zoneName$") { return $inputName }
    return "$inputName.$zoneName"
}

# Update SPF TXT record: replaces ip4:xxx.xxx.xxx.xxx with current IP
function Update-SPFRecord {
    param(
        [string]$zoneId,
        [string]$recordName,
        [string]$newIp,
        [string]$apiToken
    )

    Write-Host "`n[TXT] Checking SPF record for: $recordName" -ForegroundColor Yellow

    $url = "https://api.cloudflare.com/client/v4/zones/$zoneId/dns_records?type=TXT&name=$recordName"
    $response = Invoke-RestMethod -Uri $url -Headers @{
        "Authorization" = "Bearer $apiToken"
        "Content-Type"  = "application/json"
    }

    if ($response.result.Count -eq 0) {
        Write-Host "No SPF TXT record found for $recordName" -ForegroundColor Red
        return
    }

    $record = $response.result[0]
    $recordId = $record.id
    $oldValue = $record.content

    $newValue = $oldValue -replace "ip4:\d{1,3}(\.\d{1,3}){3}", "ip4:$newIp"

    if ($newValue -eq $oldValue) {
        Write-Host "SPF record unchanged — no update needed." -ForegroundColor Green
        return
    }

    Write-Host "Updating SPF record:" -ForegroundColor Cyan
    Write-Host "Old: $oldValue" -ForegroundColor DarkGray
    Write-Host "New: $newValue" -ForegroundColor White

    $body = @{
        type    = "TXT"
        name    = $recordName
        content = $newValue
        ttl     = 1
    } | ConvertTo-Json

    Invoke-RestMethod -Method PUT `
        -Uri "https://api.cloudflare.com/client/v4/zones/$zoneId/dns_records/$recordId" `
        -Headers @{
            "Authorization" = "Bearer $apiToken"
            "Content-Type"  = "application/json"
        } -Body $body

    Write-Host "SPF record updated ?" -ForegroundColor Green
}

# ----------------------------
# MAIN

# Get Zone ID
$zoneUrl = "https://api.cloudflare.com/client/v4/zones?name=$zoneName"
$zoneResponse = Invoke-RestMethod -Uri $zoneUrl -Headers @{
    "Authorization" = "Bearer $cloudflareApiToken"
    "Content-Type"  = "application/json"
}
$zoneId = $zoneResponse.result[0].id
Write-Host "Zone: $zoneName (ID: $zoneId)" -ForegroundColor Cyan

# Get Public IP
try {
    $publicIp = Get-PublicIP
    Write-Host "`nPublic IP detected: $publicIp" -ForegroundColor Cyan
}
catch {
    Write-Host $_ -ForegroundColor Red
    exit
}

# Normalize all A records to FQDN
$normalizedRecords = $recordNames | ForEach-Object { Normalize-ToFQDN $_ $zoneName }

# Process each A record
foreach ($record in $normalizedRecords) {

    Write-Host "`nProcessing A record: $record" -ForegroundColor Yellow

    $dnsUrl = "https://api.cloudflare.com/client/v4/zones/$zoneId/dns_records?type=A&name=$record"
    $dnsResponse = Invoke-RestMethod -Uri $dnsUrl -Headers @{
        "Authorization" = "Bearer $cloudflareApiToken"
        "Content-Type"  = "application/json"
    }

    if ($dnsResponse.result.Count -gt 0) {
        # Record exists
        $recordId = $dnsResponse.result[0].id
        $currentIp = $dnsResponse.result[0].content

        if ($currentIp -ne $publicIp) {
            Write-Host "Updating existing record ? $publicIp"
            $body = @{
                type    = "A"
                name    = $record
                content = $publicIp
                ttl     = $ttlDefault
                proxied = $proxiedDefault
            } | ConvertTo-Json

            Invoke-RestMethod -Method PUT `
                -Uri "https://api.cloudflare.com/client/v4/zones/$zoneId/dns_records/$recordId" `
                -Headers @{
                    "Authorization" = "Bearer $cloudflareApiToken"
                    "Content-Type"  = "application/json"
                } -Body $body

            Write-Host "A record updated ?" -ForegroundColor Green
        }
        else {
            Write-Host "IP unchanged — no update needed." -ForegroundColor Green
        }
    }
    else {
        # Record does not exist ? create
        Write-Host "Creating A record ? $publicIp"
        $body = @{
            type    = "A"
            name    = $record
            content = $publicIp
            ttl     = $ttlDefault
            proxied = $proxiedDefault
        } | ConvertTo-Json

        Invoke-RestMethod -Method POST `
            -Uri "https://api.cloudflare.com/client/v4/zones/$zoneId/dns_records" `
            -Headers @{
                "Authorization" = "Bearer $cloudflareApiToken"
                "Content-Type"  = "application/json"
            } -Body $body

        Write-Host "A record created ?" -ForegroundColor Green
    }
}

# Update SPF TXT record
Update-SPFRecord -zoneId $zoneId -recordName $spfRecordName -newIp $publicIp -apiToken $cloudflareApiToken

Write-Host "`nAll records processed successfully." -ForegroundColor Cyan
