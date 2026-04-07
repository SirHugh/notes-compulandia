### Windows 
1. generar llave
``` shell
ssh-keygen -t ed25519
```

2. Copiar llave al servidor Linux desde Windows
``` powershell
type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh usuario@ip-del-servidor "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```


