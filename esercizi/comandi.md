# Comandi utili
Trovare tutti i file con SETUID e SETGID attivo :
`````
```
find / -perm 6000 2>/dev/null
```


*Trovare lo ruid,rgid,euid,egid di un determinato processo*
#Avendo lanciato un processo `passwd` in un'altro terminale, provare a verificarne le credenziali tramite
```
ps -p $(pgrep -n passwd) -o ruid,rgid,euid,rgid
```

*Visualizzare l'algoritmo di controllo dei permessi su un file arbitrario*
```shell
namei -lx /etc/passwd
```

