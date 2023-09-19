# Après l'effort...

## Énoncé

```
Après une dure journée de travail, rien ne vaut une bonne bouteille de pinard entre amis. Vous voyez le responsable de l'approvisionnement qui arrive les mains derrière le dos. 
Apparemment, il va falloir travailler plus que ça pour enfin pouvoir picoler !

Le fichier une fois décompressé fait 2 Go. Bon courage !
```
```
Créateur : *OwlyDuck*

Un grand merci à *Smyler* pour la découverte de `losetup -fP`, et tout le reste :)
```

## Solution

### Identification

Le .img nous fait déjà dire que c'est une image disque sans trop se mouiller.

Pour obtenir plus d'informations on peut donc utiliser la commande `file` :
```bash
➜  /tmp file cle_de_gege.img 
cle_de_gege.img: DOS/MBR boot sector; partition 1 : ID=0x83, start-CHS (0x0,32,33), end-CHS (0x3e,86,24), startsector 2048, 999424 sectors; partition 2 : ID=0x83, start-CHS (0x3e,86,25), end-CHS (0x15,21,16), startsector 1001472, 3192832 sectors
```
ou bien celle-ci :
```bash
[poussin@Poule tmp]$ fdisk -l cle_de_gege.img 
Disque cle_de_gege.img : 2 GiB, 2147483648 octets, 4194304 secteurs
Unités : secteur de 1 × 512 = 512 octets
Taille de secteur (logique / physique) : 512 octets / 512 octets
taille d'E/S (minimale / optimale) : 512 octets / 512 octets
Type d'étiquette de disque : dos
Identifiant de disque : 0x7ee6443f

Périphérique     Amorçage   Début     Fin Secteurs Taille Id Type
cle_de_gege.img1             2048 1001471   999424   488M 83 Linux
cle_de_gege.img2          1001472 4194303  3192832   1,5G 83 Linux
```
ou bien celle-ci :
```bash
[poussin@Poule tmp]$ mmls cle_de_gege.img 
DOS Partition Table
Offset Sector: 0
Units are in 512-byte sectors

      Slot      Start        End          Length       Description
000:  Meta      0000000000   0000000000   0000000001   Primary Table (#0)
001:  -------   0000000000   0000002047   0000002048   Unallocated
002:  000:000   0000002048   0001001471   0000999424   Linux (0x83)
003:  000:001   0001001472   0004194303   0003192832   Linux (0x83)
```

Il y a donc deux partitions dans la clé, on va devoir les monter séparément si on veut avancer. Le hint est donné dans l'énoncé : `losetup -fP`

### Recherche de la clé LUKS

Lorsqu'on essaye de monter la première, on obtient ce message d'erreur :
```bash
➜  /tmp sudo losetup -fP cle_de_gege.img
➜  /tmp ls /dev | grep loop
loop0
loop0p1
loop0p2
loop-control
➜  /tmp sudo mount /dev/loop0p1 /mnt/test 
mount: /mnt/test: type de système de fichiers « crypto_LUKS » inconnu.
       dmesg(1) peut avoir plus d'informations après un échec de l'appel système du montage.
```
Le type de fichier ne semble pas supporté, il va falloir se renseigner dessus. D'après Wikipedia: LUKS, pour Linux Unified Key Setup, est le standard associé au noyau Linux pour le chiffrement de disque créé par Clemens Fruhwirth.

Notre bon vieux Gégé est donc un cachotier ! Déjà, c'était bizarre d'avoir deux partitions sur sa clé USB, mais là en plus il en a protégé une avec un mot de passe ! Il va falloir chercher plus loin.

On peut ouvrir que la deuxième, et on tombe sur le pdf extrait d'un site, expliquant comment marche cryptsetup, et une image sans informations dessus.

Mais on ne s'arrête pas là ! Peut-être que des fichiers ont été supprimés !
En effet, quand un fichier est supprimé, son espace mémoire n'est modifié que lorsque l'espace 'libéré' est utilisé à nouveau. Il existe deux types de méthodes pour récupérer des fichiers :
 - Le carving : Consiste à chercher des magic bytes dans la mémoire sans se soucier du système de fichiers. Si ça commence comme une photo de canard, que ça a la taille d'une photo de canard, et le bruit d'une photo de canard, alors c'est sûrement une photo de canard.
  - Le File undelete : Typiquement ce que fait TestDisk. Cet outil est aussi utilisé pour réparer des partitions, mais pour nous il va surtout reconnaitre (potentiellement avec notre aide) le système de fichier utilisé et va retrouver les fichiers supprimés de cette manière

Pour savoir utiliser testdisk :
`https://www.cgsecurity.org/wiki/TestDisk:_undelete_file_for_FAT`
Testdisk et photorec nous donnent la solution au bout d'un moment, soit directement dans l'image, soit dans les exifs

**Alternate PATH :** `strings cle_de_gege | grep -i pass`


Passphrase LUKS : `S0mEtIm3s_re4d1Ng_1s_En0uGh`

### Trouver le flag

On peut utiliser cryptsetup pour ouvrir l'unique photo sur la partition. 

les commandes ressembleront typiquement à ça:
```bash
sudo cryptsetup open /dev/loop0p1 cle_secrete_de_gege
sudo mount /dev/mapper/cle_secrete_de_gege /mnt/
```

Ou alors utiliser l'explorateur de fichier qui fait généralement le taff.

[Inspiration du challenge](https://linuxconfig.org/usb-stick-encryption-using-linux)

Flag : `Star{Un_p3u_dE_v1n_D4nS_mOn_LUKS}`

## Création
```bash  
  262  fallocate -l 2G /tmp/cle_de_gege.img
  263  fdisk /tmp/cle_de_gege.img
  265  sudo losetup -fP /tmp/cle_de_gege.img 
  266  sudo cryptsetup luksFormat /dev/loop0p1
  267  sudo cryptsetup open /dev/loop0p1 cle_secrete_de_gege
  268  sudo mkfs.fat /dev/mapper/cle_secrete_de_gege 
  269  sudo mount /dev/mapper/cle_secrete_de_gege /mnt/cle_secrete_de_gege/
  271  sudo cp ./HackademINT/StarHack2023/apres_l_effort/flag.jpg /mnt/cle_secrete_de_gege/
  272  sudo umount /mnt/cle_secrete_de_gege 
  273  sudo cryptsetup luksClose /dev/mapper/cle_secrete_de_gege
  274  sudo mkfs.fat /dev/loop0p2
  275  sudo mkdir /mnt/cle_de_gege
  277  sudo mount /dev/loop0p2 /mnt/cle_de_gege/
  278  sudo cp ./HackademINT/StarHack2023/apres_l_effort/almostLost.jpg /mnt/cle_de_gege/
  279  sudo cp ./HackademINT/StarHack2023/apres_l_effort/wine-bottle-puzzle.jpg /mnt/cle_de_gege/
  280  sudo cp ./HackademINT/StarHack2023/apres_l_effort/dm-crypt_Device\ encryption\ -\ ArchWiki.pdf /mnt/cle_de_gege/
  281  cd /mnt/cle_de_gege/
  286  sudo rm almostLost.jpg
  288  cd
  289  sudo umount /mnt/cle_de_gege 
  290  sudo losetup -d /dev/loop0 
```