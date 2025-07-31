# CTF Nebula  

## Level00
> This level requires you to find a Set User ID program that will run as the “flag00” account. You could also find this by carefully looking in top level directories in / for suspicious looking directories. Alternatively, look at the find man page.

To access this level, log in as level00 with the password of level00.
--- 

**SOLUZIONE**
*Cercare un pò in `/` tutti i file dell'utente `flag00`*

```
find / -user flag00 2>/dev/null
```

La costa interessante è che riusciamo a trovare un file appartenente a `flag00` nel seguente percorso : `/rofs/bin/.../flag00`

se lo analizziamo, vediamo che ha il bit SETUID impostato. Semplicemente eseguendolo, ottieniamo l'accesso come `flag00`, abbiamo così catturato il flag.



## Level01
> Questo esercizio ci chiede di analizzare il programma dato e di comprenderne la vulnerabilità che permette l'esecuzione di programmi arbitriari.


Step:
- Si impostano TUTTI gli UID e GID ai valori effettivi(flag01) tramite `setresgid` e `setresuid`

*Cosa succede quando si impostano tutti gli UID/GID al valore di quello effettivo?*

- visto che l'uid/gid effettivo è quello del creatore, si ha una vera e propria elevazione dei privilegi
- poiché anche quello *salvato* è impostato allo stesso valore del privilegiato, l'elevazione è come se fosse **PERMANENTE** (come se fosse fatta tramite seteuid())

IDEA, cambiare il path aggiungendo prima il mio programma, così env va a pescare il mio programma come quello da eseguire.

Creare nella propria home un file  `echo` con dentro solo `/bin/sh`.

Modificare la PATH tramite `export PATH=/home/level01:$PATH`

Eseguire il file `flag01` trovato in `/home/flag01`. Facendo ciò ottieniamo una shell. Se fai `whoami` vedi che sei flag01. Eseguendo `getflag` riesci poi ad ottenre il flag.

---

## Level02
> There is a vulnerability in the below program that allows arbitrary programs to be executed, can you find it? To do this level, log in as the level02 account with the password level02. Files for this level can be found in /home/flag02. 

Viene costruita una stringa contenente un comando UNIX : `/bin/echo $USER is cool` dove USER viene presa dalle variabili d'ambiente.

Viene poi stampata la stringa costruita.

E infine eseguita tramite `system(buffer)`

IDEA:
- cambio il valore di user ma in cosa?
  - magari come se facessi una SQL injection, dove mettevo prima una cosa buona poi il resto codice malevolo
  - infatti in UNIX possiamo concatenare diversi comandi tramite `;`.
  - Attenzione però, se mettiamo solo `USER=';/bin/sh'` non funziona perché legge `/bin/sh is` come comando, perciiò una volta eseguita la nostra shell, dobbiamo troncare il comando con un'altro `;`

  infatti `USER=';/bin/sh;' e poi Semplicemente eseguire `getflag`

---

## Level03
> Check the home directory of flag03 and take note of the files there. There is a crontab that is called every couple of minutes.

Guardando nella cartella `/home/flag03` vediamo che c'è uno script ed una directory chiamata `writable_d` con i seguenti permessi `drwxrwxrwx`, quindi tutti ci posson scrivere e leggere.


Lo scritp invece contiene il seguente codice:

```
#!/bin/sh
for i in /home/flag03/writable.d/* ; do
    (ulimit -t 5; bash -x "$i")
  rm -f "$i"
done
```


- ciclo che si ripete per tutti i file in `writable_d`
- ogni file poi viene eseguito per 5 secondi tramite bash
- infine il file viene cancellato


Si evince da doeve si trova (`/home/flag03`) che lo script sta eseguendo in modo periodico e soprattutto con privilegi dell'utente `flag03`. 
- se vogliamo però possiamo barare nel seguente modo 
  - `su - nebula` diventiamo nebula così possiamo diventare root
  - `sudo -i` ora diventiamo root
  - `su - flag03` ora diventiamo flag03
  - `crontab -l`  per vedere se c'è un cronjob attivo


Mettendo insieme tutte le debolezze individuate finora: 
- permessi di accesso debolissimi a writable.d;
- esecuzione periodica degli script in writable.d con i privilegi di flag03;

L'utente `level03` può quindi eseguire codice arbitrario con privilegi di `flag03`


**IDEA** : creazione di una bash privilegiato
```
#!/bin/bash
cp /bin/bash /home/flag03/bash
chmod u+s /home/flag03/bash
```


Questo script crea una bash nella home dell'utente `flag03`.


**ATTENZIONE**
Questo script sarà eseguito con privilegi elevati. Ma per farlo devo copiarlo nella cartella `writable_d`

```
cp script.sh /home/flag03/writable_d
```

Per vedere la creazione fare 
```
watch -n 1 ls -l /home/flag03
```

Infine eseguire tale bash con l'opzione `-p` perché BASH di default abbassa i privilegi effettivi per motivi di sicurezza. L'opzione -p consente di mantenerli.

```
/home/flag03/bash -p
```


## Level04
> This level requires you to read the token file, but the code restricts the files that can be read. Find a way to bypass it :)


In questo esercizio si deve leggere un file tramite quel programma, ma se il nome e token e se non si hanno i permessi,non si può.


Permission denied. Really? Nope. Where do we control everything as any user? 
- in `/tmp/`

Se facciamo un link a quel token in /tmp assegnandogli un'altro nome, e eseuguiamo lo script su quel file, dovremmo riuscire.

```
ln -s /home/flag04/token /tmp/level04
./flag04 /tmp/level04
```


Si ottiene così un token di accesso. Usarlo come password per l'utente flag04 ed infine eseguire `getflag`.


## Level05

> Check the flag05 home directory. You are looking for weak directory permissions

Infatti se facciamo `ls -l /home` vediamo tutte le home e in particolare:

```
drwxr-x--- 4 flag05  level05  93 2012-08-18 06:56 flag05
```

Quindi la home di questo utente è accessibile anche dagli utenti del gruppo level05

Dentro alla home sua cè `.backup` nella quale c'è un `archivio tar` che si può copiare nella propria home oppure addirittura esportare direttamente tramite `-C /home/level05`

- scp -P 2222 level05@localhost:/home/flag05/.backup/* .
- oppure esportando l'archivio tar

Il contenuto dell'archivio sono delle chiavi SSH. Aggiungere l'dentità nel mio modo oppure tramite quello del prof

```
ssh -p 2222 -o PubkeyAcceptedKeyTypes=+ssh-rsa -i .ssh/id_rsa flag05@localhost
```



## Level06
> The flag06 account credentials came from a legacy unix system.


La password di flag06 è in un formato “legacy”: 
- è in /etc/passwd e non in /etc/shadow;
- è in un formato debole (hash debole senza salt).


Si crea localmente un file di nome flag06.hash con il
contenuto seguente:
ueqwOCnSGdsuM

Si usa il comando hashid* per identificare il tipo di hash.
`hashid -m flag06.hash`

Attenzione, metti anche il mode tramite `-m`

--File 'flag06.hash'--
Analyzing 'ueqwOCnSGdsuM'
[+] DES(Unix) [Hashcat Mode: 1500]
[+] Traditional DES [Hashcat Mode: 1500]
[+] DEScrypt [Hashcat Mode: 1500]


Si prova la rottura dell’hash con un **attacco a dizionario**. Si usa hashcat*, uno dei software di password cracking più famosi.

```
hashcat -a 0 -m 1500 flag06.hash rockyou.txt
```

Dove `-a 0` indica un attacco a dizionario.
Dove `-m 1500` inidica il tipo di hash --> DES


## Level07


## Level08

> World readable files strike again. Check what that user was up to, and use it to log into flag08 account.

Il file `capture.pcap` è di root ma è accessibile a tutti 

```
-rw-r--r-- 1 root   root    8302 2011-11-20 21:22 capture.pcap
```

Copiare questo file sul proprio pc fisico

Analizzalo con Wireshark.

- Statistiche → Gerarchia di protocolli
- Si selezionano tali frame cliccando con il tasto destro su Data e poi scegliendo Applica come filtro → Selezionati.
- Si seleziona il flusso TCP della conversazione. Si clicca sul primo frame col tasto destro e si seleziona Segui → Flusso TCP.

La password sembrerebbe essere "backdoor mate" in alfabeto leet. I punti sono caratteri non stampabili.
- infatti hai 0x7f e se vai nella tabella ascii a vedere, corrisponde a DEL in esadecimale
- gli altri caratteri, dipende da cosa c'è (anche là c'è un 7f)


