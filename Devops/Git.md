## Dejar de trackear un achivo en git.

**Obs: Incluir en .gitignore**
### Archivo
git rm --cached archivo.txt

### Carpeta (recursivo)
git rm --cached -r carpeta/

git commit -m "Dejar de trackear archivo/carpeta"
```

Y si no querés que vuelva a trackearse, lo agregás al `.gitignore`:
```
archivo.txt
carpeta/