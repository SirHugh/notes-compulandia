Pasos a ejecutar **en el router** antes de levantar el Compose.

---
## 1. Habilitar SNMP

```routeros
# Deshabilitar community "public" por defecto
/snmp community set [ find default=yes ] disabled=yes

# Crear community de solo lectura restringida al servidor de monitoreo
/snmp community add name=compulandia_ro \
    addresses=10.100.102.50/32 \
    read-access=yes \
    write-access=no

# Activar SNMP
/snmp set enabled=yes \
    contact="NOC Compulandia" \
    location="Asuncion-DataCenter"

# Verificar
/snmp print
/snmp community print
```

> Reemplazar `10.100.102.50/32` con la IP real del servidor donde correrá el Compose.

---

## 2. Habilitar API de RouterOS

```routeros
# Habilitar servicio API
/ip service enable api

# Restringir acceso solo desde el servidor de monitoreo
/ip firewall filter add \
    chain=input protocol=tcp dst-port=8728 \
    src-address=10.100.102.50 \
    action=accept \
    comment="API Prometheus" \
    place-before=0

# Verificar
/ip service print where name=api
```

---

## 3. Crear usuario de monitoreo

```routeros
/user group add name=prometheus \
    policy=read,api,!write,!policy,!test,!winbox,!password,!web,!sniff,!sensitive

/user add name=prometheus \
    group=prometheus \
    password=Compu2602# \
    comment="Prometheus monitoring"

/user print where name=prometheus
```

---

## 4. Verificar conectividad desde el servidor

```bash
# SNMP
snmpget -v2c -c compulandia_ro 10.100.102.1 sysDescr.0

# API
nc -zv 10.100.102.1 8728
```

Ambos deben responder antes de continuar.