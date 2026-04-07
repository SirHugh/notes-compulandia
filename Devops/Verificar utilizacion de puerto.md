## Verificar que proceso utiliza un puerto especifico

### Con ss (recomendado, más moderno)
``` bash
ss -tlnp | grep :8080
```
### Con lsof
```bash
lsof -i :8080
```

### Con fuser
``` bash 
fuser 8080/tcp`
```

### Ver PID + proceso de todos los puertos en escucha
```bash 
ss -tlnp
```

## Eliminar el proceso que usa un puerto
### Opción 1: con fuser (directo)
```bash
fuser -k 8080/tcp
```
### Opción 2: obtener el PID primero y luego matar
```bash 
lsof -t -i :8080          # devuelve solo el PID
kill $(lsof -t -i :8080)  # SIGTERM (graceful)
kill -9 $(lsof -t -i :8080)  # SIGKILL (forzado)
```

### Opción 3: con ss + awk
```bash
ss -tlnp | grep :8080
kill -9 <PID>
```

### Ejemplo:
```bash
$ ss -tlnp | grep :9090
LISTEN 0  128  0.0.0.0:9090  0.0.0.0:*  users:(("prometheus",pid=3821,fd=8))

$ kill -9 3821
```


# Comandos ufw


