# Versión del sistema
Get-ComputerInfo | Select-Object OsName, OsVersion, OsArchitecture

# Verificar conectividad internet
Test-NetConnection github.com -Port 443

# Ver si ya existe windows_exporter
Get-Service | Where-Object {$_.Name -like "*exporter*"}
Get-Command windows_exporter 2>$null