# Scan-antivirus-station-blanche

**Les deux script on était tester sur :**

1. LTSP version: 23.02-1+deb12u1
2. Debian 12

**1 - Script : Ce script surveille l'ajout de nouveaux périphériques de stockage via ```udevadm``` et, lorsqu'un périphérique est détecté, il attend que celui-ci soit prêt. Ensuite, il lance un script de scan antivirus sur le périphérique détecté dans un terminal graphique.**

**2 - Script : Ce script vérifie et monte un périphérique USB, puis effectue une analyse antivirus avec clamAV sur le périphérique monté. Si le périphérique est correctement monté, il analyse les fichiers et supprime les éléments malveillants, suspects ou indésirables. Après l'analyse, le périphérique est démonté en toute sécurité, et le script attend le retrait physique du périphérique avant de se terminer.**

------------------------------------------------------------------------------

**1 - script :**

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

------------------------------------------------------------------------------

**2 - Script :**

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
