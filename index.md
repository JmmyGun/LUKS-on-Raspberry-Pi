## LUKS su Raspberry Pi + sblocco e avvio automatico

***Ultimo aggiornamento: Marzo 2024.***

Questa guida spiega come crittografare la partizione root di una scheda SD con Raspberry Pi OS utilizzando LUKS. Il processo richiede un Raspberry Pi con Raspberry Pi OS sulla scheda SD e una memoria USB con almeno la stessa capacità della scheda SD. È importante avere un backup della scheda SD, nel caso in cui qualcosa vada storto, poiché verrà sovrascritta, così come della memoria USB.

La crittografia del disco completo è abbastanza semplice da eseguire con le moderne distribuzioni Linux. Raspberry Pi è un'eccezione perché la partizione di avvio non include la maggior parte dei programmi e dei moduli del kernel necessari. D'altra parte, è importante utilizzare la crittografia del disco con Raspberry Pi perché la scheda SD può essere estratta dall'unità e il suo contenuto letto abbastanza facilmente.

### Requisiti

Kernel Linux 5.0 o successivo. Puoi verificarlo con questo comando:
```
uname -s -r
```
Installa i programmi necessari:
```
sudo apt install busybox cryptsetup initramfs-tools resize2fs cryptsetup-initramfs
```
I microprocessori dei computer Raspberry Pi non includono l'accelerazione AES. Ciò significa che è necessaria un'implementazione software di AES, che è lenta con un hardware modesto. Fortunatamente esiste una recente alternativa, Adiantum, che funziona velocemente nel software. Il kernel Linux 5.0 o successivo include i moduli del kernel crittografico necessari per Adiantum. Puoi verificare che ogni modulo sia presente e caricato con questo comando:
```
cryptsetup benchmark -c xchacha20,aes-adiantum-plain64
```
L'output, se tutto è a posto, sarà simile a questo:
```
# Tests are approximate using memory only (no storage IO).
#            Algorithm |       Key |      Encryption |      Decryption
xchacha20,aes-adiantum        256b       111.2 MiB/s       114.6 MiB/s
```
### Preparazione avvio di Linux

Bisogna specificare alcuni programmi che devono essere inclusi nell' 'initramfs'. Lo facciamo con un nuovo file:
```
/etc/initramfs-tools/hooks/luks_hooks
```
e deve contenere questo contenuto:
```bash
#!/bin/sh -e
PREREQS=""
case $1 in
        prereqs) echo "${PREREQS}"; exit 0;;
esac

. /usr/share/initramfs-tools/hook-functions

copy_exec /sbin/resize2fs /sbin
copy_exec /sbin/fdisk /sbin
copy_exec /sbin/cryptsetup /sbin
```
I programmi sono 'resize2fs', 'fdisk' e 'cryptsetup'.
Il file deve essere reso eseguibile:
```
sudo chmod +x /etc/initramfs-tools/hooks/luks_hooks
```
L'initramfs per Raspberry Pi OS non include di default i moduli del kernel per LUKS e la crittografia. Bisogna configurare i moduli del kernel da aggiungere. Modifica:
```
/etc/initramfs-tools/modules
```
e le seguenti righe con i nomi dei moduli del kernel aggiunti:
```
algif_skcipher
xchacha20
adiantum
aes_arm
sha256
nhpoly1305
dm-crypt
```
Rigenera ‘initramfs’:
```
sudo update-initramfs -u
```
Verifica la presenza dei programmi nell'initramfs con il seguente comando:
```
lsinitramfs /boot/initramfs.gz | grep -P "sbin/(cryptsetup|resize2fs|fdisk)"
```
Verifica che i moduli del kernel siano presenti nell'initramfs con il seguente comando:
```
lsinitramfs /boot/initramfs.gz | grep -P "(algif_skcipher|chacha|adiantum|aes-arm|sha256|nhpoly1305|dm-crypt)"
```

### Preparazione dell'avvio
Prima di riavviare il Raspberry Pi, è necessario apportare alcune modifiche ai file. Queste modifiche servono per istruire il processo di avvio affinché utilizzi un filesystem root crittografato. Dopo aver effettuato le modifiche precedenti, il Raspberry Pi si avvierà correttamente. Tuttavia, dopo aver apportato le modifiche ai seguenti file, il Raspberry Pi non avvierà l'interfaccia desktop finché non sarà completato l'intero processo di crittografia della partizione radice e la configurazione di LUKS. Nel caso in cui si verifichino problemi dopo aver apportato modifiche ai successivi quattro file e prima della crittografia della partizione radice, è possibile ripristinare le modifiche nei quattro file e il Raspberry Pi si avvierà normalmente. È consigliabile fare una copia di questi file prima di apportare modifiche.

**File: /boot/cmdline.txt**

Contiene una riga con dei parametri. Uno di essi è 'root', che specifica la posizione della partizione root. Per il Raspberry Pi di solito è '/dev/mmcblk0p2', ma può anche essere un altro dispositivo (o lo stesso) specificato come "PARTUUID=xxxxx". Il valore di 'root' deve essere cambiato in '/dev/mapper/sdcard'. Ad esempio, se 'root' è:
```
root=/dev/mmcblk0p2
```
deve essere cambiato in:
```
root=/dev/mapper/sdcard
```
inoltre, alla fine della riga, separato da uno spazio, deve essere aggiunto:
```
cryptdevice=/dev/mmcblk0p2:sdcard
```

**File: /etc/fstab**

Il dispositivo per la partizione root (‘/’) deve essere cambiato con il mapper. 
Ad esempio, se il dispositivo per la root è:
```
/dev/mmcblk0p2
```
deve essere cambiato con:
```
/dev/mapper/sdcard
```

**File: etc/crypttab**

Alla fine del file, una nuova riga deve essere aggiunta con il seguente contenuto:
```
sdcard	/dev/mmcblk0p2	none	luks
```

Tutto è pronto per il riavvio. Dopo il riavvio, Raspberry Pi OS non riuscirà a avviarsi perché abbiamo configurato una partizione root che non esiste ancora. Dopo diversi messaggi uguali che indicano il fallimento, verrà visualizzata la shell ‘initramfs’.

### Crittografia della partizione root

Nella shell ‘initramfs’, possiamo verificare l'utilizzo di ‘cryptsetup’ con i moduli kernel dei cifrari:
```
cryptsetup benchmark -c xchacha20,aes-adiantum-plain64
```
Se il test viene completato con successo, tutto è a posto. Ora dobbiamo copiare la partizione root della scheda SD nella memoria USB. L'idea è quella di avere una copia della partizione root della scheda SD nella memoria USB, creare un volume crittografato nella partizione root (il contenuto verrà perso) e copiare la partizione root dalla memoria USB nella partizione crittografata della scheda SD. Tieni presente che il contenuto precedente della memoria USB verrà perso. Prima di farlo, controlliamo la partizione root e riduciamo le dimensioni del suo filesystem al minimo possibile, in modo che l'operazione di copia richieda meno tempo. Il comando per controllare e correggere eventuali errori è:
```
e2fsck -f /dev/mmcblk0p2
```
Una volta terminato, dobbiamo ridimensionare il filesystem. È estremamente importante annotare attentamente le dimensioni del filesystem dopo il ridimensionamento, in particolare il numero di blocchi da 4k:
```
resize2fs -fM -p /dev/mmcblk0p2
```
Un esempio dell'output del programma è:
```
Resizing the filesystem on /dev/mmcblk0p2 to 524288 (4k) blocks.
```
Il numero da annotare è '524288' nell'esempio.
Una volta completato il ridimensionamento, otterremo un valore del checksum dell'intera partizione. Faremo lo stesso dopo ogni operazione di copia tra la scheda SD e la memoria USB. Se il checksum è lo stesso, la copia sarà identica. Creiamo il checksum per la partizione root da copiare. Ricordati di sostituire 'XXXXX' con il numero di blocchi da 4 kB ottenuto dopo il ridimensionamento del filesystem. È utile aggiungere "time " prima di "dd" per ottenere quanto tempo impiega l'operazione. Quando vengono calcolati i restanti checksum, permette di sapere, più o meno, quanto tempo ci vorrà.
```
time dd bs=4k count=XXXXX if=/dev/mmcblk0p2 | sha1sum
```
Annota il valore SHA1.
Collega ora la memoria USB al Raspberry Pi. Verranno stampate alcune informazioni sulla memoria USB. Se non è collegata un'altra memoria USB, il dispositivo per la memoria sarà probabilmente /dev/sda. Può essere verificato con:
```
fdisk -l /dev/sda
```
Stiamo per utilizzare l'intera memoria USB, /dev/sda. Questa operazione copierà il filesystem root della scheda SD nel dispositivo di memoria USB, cancellando tutto il suo contenuto (ricorda di sostituire 'XXXXX' con il numero di blocchi da 4k ottenuto dopo il ridimensionamento del filesystem):
```
time dd bs=4k count=XXXXX if=/dev/mmcblk0p2 of=/dev/sda
```
Durante la copia da o verso schede SD e memorie USB potrebbe comparire un messaggio che indica che il task worker è stato bloccato per diversi secondi. Può essere ignorato se i checksum sono corretti.

Ora abbiamo il filesystem root originale e la copia. Abbiamo anche il checksum dell'originale. Dobbiamo calcolare il checksum della copia e confrontarli. Possiamo considerarli una copia esatta se i checksum coincidono. Il comando per calcolare il checksum della copia è (ricordati di sostituire 'XXXXX' con il numero di blocchi da 4k che hai ottenuto dopo il ridimensionamento del filesystem):
```
time dd bs=4k count=XXXXX if=/dev/sda | sha1sum
```
Se i checksum sono corretti, è il momento di criptare il filesystem root della scheda SD, per creare il volume LUKS utilizzando 'cryptsetup'. Ci sono molti parametri e possibili valori per la crittografia. Questo è il comando che ho scelto:
```
cryptsetup --type luks2 --cipher xchacha20,aes-adiantum-plain64 --hash sha256 --iter-time 5000 –-key-size 256 --pbkdf argon2i luksFormat /dev/mmcblk0p2
```
Ulteriori informazioni sui parametri possono essere trovate qui:
<https://man7.org/linux/man-pages/man8/cryptsetup.8.html>

Il comando richiederà una passphrase due volte (per conferma). È importante che la passphrase sia lunga e utilizzi diversi set di caratteri (lettere, numeri, simboli, maiuscole, minuscole, ecc.).
Dopo aver creato il volume LUKS, dobbiamo aprirlo e copiare il contenuto del filesystem root al suo interno. Il comando per aprire il volume LUKS è:
```
cryptsetup luksOpen /dev/mmcblk0p2 sdcard
```
Chiederà la passphrase scelta nella fase precedente. Una volta aperto, copiamo il filesystem root dalla memoria USB nel volume crittografato (ricordati di sostituire ‘XXXXX’ con il numero di blocchi da 4k ottenuti dopo il ridimensionamento del filesystem):
```
time dd bs=4k count=XXXXX if=/dev/sda of=/dev/mapper/sdcard
```
Dopo aver copiato, calcola nuovamente il checksum della copia per convalidarlo. Se i checksum coincidono, la copia è corretta (ricordati di sostituire ‘XXXXX’ con il numero di blocchi da 4k ottenuti dopo il ridimensionamento del filesystem):
```
time dd bs=4k count=XXXXX if=/dev/mapper/sdcard | sha1sum
```
Oltre al controllo del checksum, verifica anche il filesystem del volume LUKS:
```
e2fsck -f /dev/mapper/sdcard
```
Il fylesystem è ridotto. Ora espandilo alla dimensione della scheda SD:
```
resize2fs -f /dev/mapper/sdcard
```
Il processo è quasi terminato ora. La memoria USB può essere rimossa perché non è più necessaria. Esci dall'initramfs:
```
exit
```
Durante il processo di avvio, si entrerà nuovamente nella shell di initramfs. In questo momento bisogna aprire il volume LUKS appena creato per rendere disponibile il filesystem root:
```
cryptsetup luksOpen /dev/mmcblk0p2 sdcard
```
Dopo aver aperto il volume LUKS, dobbiamo uscire nuovamente e il sistema operativo Raspberry Pi OS si avvierà normalmente:
```
exit
```

### Avvio
Non vogliamo entrare nella shell di initramfs ogni volta che accendiamo il nostro Raspberry Pi. Possiamo fare in modo che Raspberry Pi OS chieda la password. Per farlo, è necessario rigenerare l'initramfs e riavviare:
```
sudo update-initramfs -u
```

Dopo il riavvio, dovrebbe apparire un messaggio di prompt, simile a "Please unlock disk...". Forse il prompt che chiede una password si perderà tra alcuni messaggi di avvio, ma puoi comunque inserire la tua passphrase e dovrebbe funzionare.

## Sblocco e avvio automatico della partizione root crittografata con LUKS tramite file chiave memorizzato nella partizione Boot ⚠️(NON SICURO)

In questa sezione della guida, viene spiegato come configurare il sistema per lo sblocco automatico della partizione root crittografata utilizzando LUKS tramite un file chiave. È importante tenere presente che questa procedura comporta il rischio di sicurezza associato al salvataggio del file chiave nella cartella di boot del sistema. Si consiglia di considerare attentamente i rischi e di adottare misure aggiuntive per proteggere la sicurezza del sistema.

Crea la cartella contenente la chiave:
```
sudo mkdir /boot/key && cd /boot/key 
```

Generazione della chiave
Crea la chiave e rendila in sola lettura:
```
sudo dd if=/dev/urandom of=keyfile bs=4096 count=4
```
rendila in sola lettura
```
sudo chmod 0400 keyfile
```
I dispositivi abilitati LUKS/dm_crypt possono contenere fino a 10 diverse keyfile/password. Quindi, oltre ad avere la password già impostata, aggiungi questo file chiave come metodo di autorizzazione aggiuntivo, ti verrà chiesto di inserire la passpharse.
```
sudo cryptsetup luksAddKey /dev/mmcblk0p2 keyfile
```
Puoi verificare l'aggiunta della chiave con il comando: 
```
sudo cryptsetup luksDump /dev/mmcblk0p2
```
Come risultato dovresti vedere i 2 keyslot "0", "1".
Puoi testare la chiave appena creata usandi il comando:
```
sudo cryptsetup open -v -d keyfile --test-passphrase /dev/mmcblk0p2
```
Il risultato dovrebbe essere simile a questo:
```
No usable token is available.
Key slot 2 unlocked.
Command successful.
```
Modifichiamo il mapper precedentemente creato quindi:
```
sudo nano /etc/crypttab
```
modifichiamo la prima riga da:
```
sdcard	/dev/mmcblk0p2	none	luks
```
così:
```
sdcard  /dev/mmcblk0p2  /boot/key/keyfile luks
```
Rigenera l'initramfs
```
sudo update-initramfs -u
```
Ora riavvia e il sistema dovrebbe avviarsi senza richiederti la pass se tutto è andato bene.


