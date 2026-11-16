# TP n°2 — Configuration des paramètres du Boot Loader GRUB2

**Module :** Administration Unix — 4IIR  
**Système demandé :** Red Hat Enterprise Linux 9  
**Objectif :** Comprendre le fonctionnement de GRUB2 et la génération de son fichier de configuration.

> **Important :** ce compte rendu présente les réponses et les commandes du TP. Les résultats réels des commandes et les captures d'écran doivent être ajoutés après leur exécution sur la machine RHEL 9.

---

## Partie 1 — Découverte de GRUB2 et du processus de démarrage

### 1. Rôle de GRUB2

GRUB2 (GRand Unified Bootloader version 2) est le chargeur de démarrage par défaut de RHEL 9. Il est lancé par le firmware BIOS ou UEFI. Il affiche le menu de démarrage, charge le noyau Linux et l'image initramfs en mémoire, puis transmet le contrôle au noyau avec ses paramètres de démarrage.

### 2. Identifier le mode de démarrage

**Commande :**
```bash
[ -d /sys/firmware/efi ] && echo "Mode UEFI" || echo "Mode BIOS (Legacy)"
```

**Interprétation :**
- Si la commande affiche `Mode UEFI`, le système démarre en mode UEFI.
- Si elle affiche `Mode BIOS (Legacy)`, le système démarre en mode BIOS traditionnel.

Emplacements indiqués dans le TP :
- BIOS : `/boot/grub2/grub.cfg`
- UEFI : `/boot/efi/EFI/redhat/grub.cfg`

Sur RHEL 9 en UEFI, le fichier situé dans `/boot/efi/EFI/redhat/grub.cfg` est un fichier relais qui renvoie vers le fichier principal `/boot/grub2/grub.cfg`.

**Résultat observé sur ma machine :** À compléter après exécution.

### 3. Lister le contenu de `/boot`

**Commandes :**
```bash
ls -l /boot
ls -l /boot/grub2
```

**Éléments à identifier :**
- `vmlinuz-<version>` : noyau Linux compressé.
- `initramfs-<version>.img` : système de fichiers racine temporaire utilisé au démarrage.
- `loader/entries/` : répertoire contenant les fichiers des entrées de démarrage au format BLS.
- `grub2/` : contient notamment `grubenv`, les polices et les modules GRUB2 ainsi que, en mode BIOS, le fichier `grub.cfg`.

### 4. Identifier le noyau en cours et les noyaux installés

**Commandes :**
```bash
uname -r
rpm -q kernel-core
```

- `uname -r` affiche la version du noyau actuellement en cours d'exécution.
- `rpm -q kernel-core` affiche les versions des paquets de noyau installés.

**Résultats observés :** À compléter après exécution.

---

## Partie 2 — Analyse de la configuration de GRUB2

La configuration de GRUB2 n'est pas modifiée directement dans `grub.cfg`. Ce fichier est généré avec `grub2-mkconfig` à partir de `/etc/default/grub` et des scripts présents dans `/etc/grub.d/`.

### 1. Analyser `/etc/default/grub`

**Commande :**
```bash
cat /etc/default/grub
```

| Variable | Rôle |
|---|---|
| `GRUB_TIMEOUT` | Durée d'affichage du menu avant le démarrage automatique. |
| `GRUB_DISTRIBUTOR` | Nom de la distribution affiché dans les entrées. |
| `GRUB_DEFAULT` | Entrée de démarrage sélectionnée par défaut. |
| `GRUB_DISABLE_SUBMENU` | Si `true`, les anciens noyaux ne sont pas placés dans un sous-menu. |
| `GRUB_TERMINAL_OUTPUT` | Type de sortie du terminal GRUB. |
| `GRUB_CMDLINE_LINUX` | Paramètres ajoutés à la ligne de commande du noyau. |
| `GRUB_DISABLE_RECOVERY` | Contrôle la génération des entrées de secours. |
| `GRUB_ENABLE_BLSCFG` | Si `true`, GRUB2 utilise la Boot Loader Specification (BLS). |

### 2. Lister les scripts de `/etc/grub.d`

**Commande :**
```bash
ls -l /etc/grub.d/
```

Les scripts sont exécutés dans l'ordre alphabétique par `grub2-mkconfig`.

| Script | Rôle |
|---|---|
| `00_header` | Définit les variables de base, le délai, le terminal et le chargement de grubenv. |
| `01_users` | Prend en compte le mot de passe défini avec `grub2-setpassword`. |
| `10_linux` | Génère les entrées des noyaux Linux ; sur RHEL 9, il utilise BLS. |
| `30_uefi-firmware` | Ajoute l'entrée d'accès à la configuration du firmware UEFI. |
| `40_custom` | Permet d'ajouter des entrées de menu personnalisées. |

---

## Partie 3 — Génération et modification de la configuration de GRUB2

> Effectuer ces manipulations dans une machine virtuelle de TP. Une erreur dans GRUB2 peut empêcher le démarrage. Faire une sauvegarde ou un snapshot avant toute modification.

### 1. Sauvegarder les fichiers

En session root :
```bash
cp -p /etc/default/grub /root/grub.default.bak
cp -p /etc/grub.d/40_custom /root/40_custom.bak
cp -p /boot/grub2/grub.cfg /root/grub.cfg.bak
```

Ces commandes créent des copies de sauvegarde des fichiers avant modification.

### 2. Modifier le délai d'affichage du menu

Ouvrir le fichier :
```bash
nano /etc/default/grub
```

Définir les valeurs suivantes :
```bash
GRUB_TIMEOUT=10
GRUB_TIMEOUT_STYLE=menu
```

- `GRUB_TIMEOUT=10` : affiche le menu pendant 10 secondes.
- `GRUB_TIMEOUT_STYLE=menu` : force l'affichage du menu.

Enregistrer avec `Ctrl+O`, valider avec Entrée, puis quitter avec `Ctrl+X`.

### 3. Définir le système de démarrage par défaut

Dans `/etc/default/grub`, définir :
```bash
GRUB_DEFAULT=1
```

Cela sélectionne la deuxième entrée du menu comme entrée par défaut (l'indexation commence à 0).

### 4. Ajouter les entrées Redémarrer et Arrêter

Ouvrir :
```bash
nano /etc/grub.d/40_custom
```

Ajouter les entrées suivantes à la fin du fichier :
```bash
menuentry "Redemarrer" {
    echo "Redemarrer le systeme..."
    reboot
}

menuentry "Arreter le systeme" {
    echo "Arret du systeme..."
    halt
}
```

Enregistrer et quitter l'éditeur.

### 5. Régénérer la configuration

Exécuter :
```bash
grub2-mkconfig -o /boot/grub2/grub.cfg
```

Cette commande régénère le fichier principal `grub.cfg` à partir des paramètres et des scripts de configuration.

Redémarrer ensuite la machine pour vérifier les modifications :
```bash
reboot
```

**Vérification :** vérifier l'affichage du menu pendant 10 secondes, l'entrée par défaut et la présence des deux entrées personnalisées.

---

## Partie 4 — Réinitialisation du mot de passe root

> À réaliser uniquement sur la machine virtuelle personnelle du TP, avec l'autorisation nécessaire.

1. Redémarrer la machine.
2. Dans le menu GRUB, sélectionner le système puis appuyer sur `e` pour modifier temporairement son entrée.
3. Repérer la ligne qui commence par `linux` et ajouter `rd.break` à la fin de cette ligne.
4. Démarrer avec `Ctrl+X`.
5. Dans le shell root, remonter `/sysroot` en lecture/écriture puis entrer dans le système :

```bash
mount -o remount,rw /sysroot
chroot /sysroot
```

6. Modifier le mot de passe root :

```bash
passwd
```

Saisir et confirmer le nouveau mot de passe.

7. Demander le ré-étiquetage SELinux au prochain démarrage :

```bash
touch /.autorelabel
```

8. Quitter le shell puis redémarrer :

```bash
exit
exit
```

**Explication :** `rd.break` interrompt le démarrage avant que initramfs ne passe le contrôle à systemd. Le montage en lecture/écriture et `chroot` permettent de modifier le mot de passe dans le système installé. `/.autorelabel` demande le ré-étiquetage SELinux nécessaire après la modification.

**Vérification :** ouvrir une session root avec le nouveau mot de passe.

---

## Partie 5 — Protection de l'interface GRUB par mot de passe

La protection de GRUB permet notamment :
- d'empêcher l'accès non autorisé au mode single-user ;
- de protéger l'accès à la console GRUB ;
- d'empêcher la modification des paramètres de démarrage par une personne non autorisée.

### 1. Définir le mot de passe GRUB

Exécuter en root :
```bash
grub2-setpassword
```

Saisir le mot de passe demandé, puis le confirmer.

Cette commande génère un mot de passe haché stocké dans :
```bash
/boot/grub2/user.cfg
```

### 2. Vérifier le fichier créé

```bash
cd /boot/grub2/
ls
cat user.cfg
```

Le fichier `user.cfg` contient le hachage du mot de passe GRUB, et non le mot de passe en clair.

### 3. Régénérer la configuration

```bash
grub2-mkconfig -o /boot/grub2/grub.cfg
```

### 4. Tester la protection

Redémarrer :
```bash
reboot
```

Dans le menu GRUB, sélectionner le système et appuyer sur `e`. GRUB demande alors le nom d'utilisateur et le mot de passe avant d'autoriser la modification de l'entrée.

### 5. Supprimer la protection (opération de TP)

Pour supprimer cette protection, le TP demande de supprimer le fichier :
```bash
rm /boot/grub2/user.cfg
```

Confirmer la suppression si le système le demande.

---

## Conclusion

Ce TP m'a permis d'étudier le rôle de GRUB2 dans le démarrage de RHEL 9, d'identifier les fichiers et scripts de configuration, de modifier les paramètres du menu et d'ajouter des entrées personnalisées. J'ai également étudié la procédure de réinitialisation du mot de passe root et la protection de GRUB2 par mot de passe.

---

## Éléments à compléter avant le dépôt

- [ ] Ajouter le résultat de la détection BIOS/UEFI.
- [ ] Ajouter les résultats de `uname -r` et `rpm -q kernel-core`.
- [ ] Ajouter quelques captures d'écran des commandes exécutées.
- [ ] Vérifier que les modifications GRUB ont été testées dans la VM.
- [ ] Ne jamais publier un mot de passe personnel ni le contenu sensible de `user.cfg`.
