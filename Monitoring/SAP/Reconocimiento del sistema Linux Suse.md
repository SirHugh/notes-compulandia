# Versión exacta del sistema

``` bash
# cat /etc/os-release | grep -E "NAME|VERSION"
NAME="SLES"
VERSION="15-SP4"
VERSION_ID="15.4"
PRETTY_NAME="SUSE Linux Enterprise Server 15 SP4"
CPE_NAME="cpe:/o:suse:sles:15:sp4"
```
# Verificar conectividad a internet desde la VM
curl -s --max-time 5 https://github.com 2>&1 | head -3

# Ver si zypper tiene repositorios activos
zypper repos

# Ver si node_exporter ya existe en el sistema
which node_exporter 2>/dev/null || echo "no instalado"

# Arquitectura del sistema
uname -m