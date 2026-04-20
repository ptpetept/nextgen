$certs = @()
$certs += Get-ChildItem Cert:\CurrentUser\Root | Where-Object { $_.Subject -like "*Proxy SSL Interception*" }
$certs += Get-ChildItem Cert:\LocalMachine\Root | Where-Object { $_.Subject -like "*Proxy SSL Interception*" }
$certs += Get-ChildItem Cert:\CurrentUser\CA   | Where-Object { $_.Subject -like "*<organisation>*" }
$certs += Get-ChildItem Cert:\LocalMachine\CA   | Where-Object { $_.Subject -like "*<organisation>*" }

$pem = foreach ($c in $certs | Sort-Object Thumbprint -Unique) {
  "-----BEGIN CERTIFICATE-----"
  [Convert]::ToBase64String($c.RawData, 'InsertLineBreaks')
  "-----END CERTIFICATE-----"
}
$pem | Set-Content -Encoding ascii C:\certs\corp-bundle.pem