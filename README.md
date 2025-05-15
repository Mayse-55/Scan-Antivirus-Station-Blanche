# 🧾 Informations

1. 📦 LTSP version : `23.02-1+deb12u1`  
2. 🐧 Distribution : Debian 12

---

# ⚙️ Description des scripts

**📄 Ce script utilise ```udevadm``` pour surveiller en temps réel l’ajout de nouveaux périphériques de stockage (comme des clés USB ou disques externes). Lorsqu’un périphérique est détecté par le système, le script attend que celui-ci soit entièrement initialisé et prêt à être utilisé.**

**Une fois prêt, le script lance automatiquement le deuxiéme script qui lui lance un scan antivirus avec ClamAV en ouvrant un terminal graphique pour exécuter un script de scan dédié. Cette analyse permet de détecter et signaler toute présence de fichiers infectés ou malveillants sur le périphérique connecté.**

**Cette automatisation garantit une protection efficace contre les menaces véhiculées via les supports amovibles, tout en offrant une interface visuelle facilitant le suivi de l’analyse pour l’utilisateur.**

---

## 📌 À savoir

- Ce script doit être **ajouté et exécuté automatiquement au démarrage de la session utilisateur**.  
- Il est écrit en **Bash** et doit être lancé avec les droits nécessaires.
```bash
chmod +x /etc/script/autoscan
# Remplacez /etc/script/autoscan par le chemin réel de votre script
```

---

**🐧 Script Bash :**
```bash
MOUNT_POINT="/mnt/autoscan"
SCRIPT="/etc/antivirus/autoscan.sh"

# Boucle de surveillance
/usr/bin/udevadm monitor --subsystem-match=block | while read -r line; do
    echo "$line" | grep -q "add" && {
        sleep 1  # Attente que le périphérique soit prêt

        # Cherche le dernier périphérique de type 'sdX1'
        DEVICE=$(lsblk -o NAME,TYPE -nr | grep "part" | tail -n1 | awk '{print $1}')

        # Si rien trouvé, on ignore
        [ -z "$DEVICE" ] && continue

        # Lance le script de scan dans un terminal graphique
        xterm -T "Scan USB" -e "bash $SCRIPT /dev/$DEVICE"
    }
done
```
---

# 📋 Deuxiéme Script

## 📌 À savoir

- Il est écrit en **Bash** et doit être lancé avec les droits nécessaires.
```bash
chmod +x /etc/script/autoscan
# Remplacez /etc/script/autoscan par le chemin réel de votre script
```

---

**1️⃣ Pour faire fonctionner le deuxiéme script correctement il faut modifier Visudo :**
```bash
visudo
```
**2️⃣ Ajouter cela et Enregistrer :**
```bash
nomdevotreprofil     ALL=(ALL) NOPASSWD: /bin/mount, /bin/umount, /bin/mkdir, /bin/chown, /bin/clamscan
# Changer "nomdevotreprofil" par le profil que vous avez créer
```
---

**🐧​ - Script Bash :**
```bash                                                                                                                                                                                                                                                                                     
#!/bin/bash

# Variable

DEVICE="$1"
MOUNT_POINT="/mnt/autoscan"

# Vérifie que le périphérique existe

if [ ! -b "$DEVICE" ]; then
    echo "[!] Le périphérique $DEVICE n'existe pas."
    exit 1
fi

# Trouve le périphérique parent (ex: /dev/sdb à partir de /dev/sdb1)

PARENT_DEVICE=$(lsblk -no PKNAME "$DEVICE")
PARENT_PATH="/dev/$PARENT_DEVICE"

# Création du dossier si besoin

sudo mkdir -p "$MOUNT_POINT"

# Inistialisation du systeme avec information utilisateur

echo "=== INITIALISATION DU SYSTEME ==="
echo ""
echo "[!] Attention Tous les éléments suivants seront supprimés :"
echo ""
echo " • Virus et Logiciels malveillants"
echo " • Fichiers suspects"
echo " • Contenu indésirable"
echo ""

# Attente de 3 seconds

sleep 3

# Montage de la cle usb

echo "=== MONTAGE DU PERIPHERIQUE ==="
echo ""
if ! sudo mount "$DEVICE" "$MOUNT_POINT"; then
    echo ""
    echo "[!] Erreur : Impossible de monté le périphérique $DEVICE"
    echo ""
    echo "[*] Veuillez retirer physiquement le périphérique... "

    # Attente du retrait en cas d'erreur du montage

    while [ -b "$DEVICE" ]; do
        sleep 1
    done

    exit 1
else
    echo "[✓] Succès : Périphérique monté sur $MOUNT_POINT"
fi

# Vérification clamAV

echo ""
echo "[*] Vérification clamAV en cours..."
sleep 3
sudo clamscan -r --bell --remove "$MOUNT_POINT"

# Demontage de la cle usb

echo ""
echo "=== DEMONTAGE DU PERIPHERIQUE ==="
if sudo umount "$MOUNT_POINT"; then
    echo ""
    echo "[✓] Succès : Périphérique démontée"
else
    echo ""
    echo "[!] Erreur : Impossible de démonté le périphérique"
    echo ""
    echo "[*] Veuillez retirer physiquement le périphérique... "

    # Attente du retrait en cas d'erreur de démontage

    while [ -b "$DEVICE" ]; do
        sleep 1
    done

    exit 1
fi

# Fin du script

echo ""
echo "=== VERIFICATION TERMINEE ==="
echo ""
echo "[*] Veuillez retirer physiquement le périphérique... "

# Attente du retrait de la cle usb

while [ -b "$DEVICE" ]; do
    sleep 1
done

exit 0
```
------------------------------------------------------------------------------

![Capture d'écran 2025-05-14 123722](https://github.com/user-attachments/assets/7e64c044-ddb4-4168-b91c-59ba8ef67d7e)
