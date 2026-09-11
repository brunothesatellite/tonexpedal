# TONEX Pedal Controller

Contrôleur web single-page pour l'IK Multimedia TONEX Pedal. Gère les presets via USB MIDI et BLE MIDI, et lit les noms/configurations directement depuis le pédalier via l'interface série USB CDC.

Ordinateur :
![Interface PC](captures/tnx1.png)

Téléphone Android :
![Interface Android1](captures/android1.png)
![Interface Android2](captures/android2.png)


Démo vidéo PC :
[![Video PC](https://img.youtube.com/vi/ZrpM73ms7fk/0.jpg)](https://www.youtube.com/watch?v=ZrpM73ms7fk)

Démo vidéo Android :
[![Video Android](https://img.youtube.com/vi/XhKJ70A9dGQ/0.jpg)](https://www.youtube.com/watch?v=XhKJ70A9dGQ)

## Fonctionnalités

- **Grille 3×3** de presets assignables avec noms
- **Couleurs de presets** — assigner l'une des 9 couleurs LED (rouge, vert, ambre, jaune, cyan, bleu, rose, violet, blanc) à chaque tuile
- **Bibliothèque complète** des 150 presets (50 banks × 3 slots A/B/C)
- **Synchronisation USB** — lecture de tous les noms directement depuis le pédalier
- **Contrôle MIDI** — envoi de Bank Select + Program Change pour changer de preset
- **Contrôle BLE MIDI** — route Web Bluetooth expérimentale pour la même logique de preset, basée sur un service GATT et une caractéristique personnalisée, avec le même schéma logique Bank Select + Program Change quand le paquet BLE est accepté par le pédalier
- **Glisser-déposer** — assigner un preset à un bouton, swap entre boutons, ou supprimer via la corbeille
- **Édition** — double-clic pour renommer un preset
- **Recherche** filtrante dans la bibliothèque
- **Persistance** — configuration sauvegardée en localStorage
- **Responsive** — texte adaptatif via `container-type: inline-size` + unités `cqi`
- **Bascule bibliothèque** — chevron discret pour minimiser/étendre la bibliothèque de presets
- **Export/Import JSON** — exporte les noms de presets, les assignations de grille et les couleurs vers un fichier, importe sur une autre config
- **Support Android** — fonctionne sur Android Chrome via fallback WebUSB (Web Serial non disponible sur Android)

## Prérequis

| Composant | Version requise |
|-----------|----------------|
| Navigateur | Chrome 89+ ou Edge 89+ (Web MIDI + Web Serial API + Web Bluetooth BLE MIDI) |
| Système | Windows 10/11, Android (via WebUSB) |
| Pédalier | IK Multimedia TONEX Pedal (full size) |
| Câble | USB-C connecté au port USB du pédalier |
| BLE | La couche BLE MIDI nécessite une origine sécurisée telle que localhost/HTTPS et un navigateur compatible Web Bluetooth. Le device sélectionné doit exposer le service `03b80e5a-ede8-4b33-a751-6ce34ec4c700` et la caractéristique `7772e5db-3868-4112-a1a9-f2669d106bf3` |

> **Note** : L'API Web Serial nécessite HTTPS ou localhost. Servir via un serveur web local (ex: `https://mon-serveur/tonexpedal/`) ou `localhost`.

> **Note Android** : Sur Android, Web Serial n'est pas disponible — l'application utilise le fallback WebUSB pour la communication USB CDC. Le MIDI n'est pas disponible sur Android (pas d'API Web MIDI).
>
> **Note BLE expérimentale** : la couche BLE MIDI est une route expérimentalement documentée. Elle peut se déconnecter rapidement selon le PC, le système d’exploitation, ou la pile GATT du navigateur. La couche BLE ne doit pas être considérée comme stable au niveau production.

## Installation

### Option 1 — Serveur web local (recommandé)

Copier le dossier `tonexpedal/` dans le répertoire racine de votre serveur web, puis accéder via :
```
https://mon-serveur/tonexpedal/
```

### Option 2 — localhost avec un serveur simple

```bash
# Depuis le dossier tonexpedal/
npx serve -s . -l 3000
# ou
python -m http.server 3000
```

Puis ouvrir `http://localhost:3000`.

### Option 3 — Fichier statique (sans serveur)

Simplement double-cliquer sur `index.html` ou l'ouvrir via `file:///` dans votre navigateur.

## Utilisation

### Connexion MIDI

L'application supporte deux transports de contrôle distincts :

1. **USB MIDI / Web MIDI** — route USB-MIDI classique, stable et prioritaire sur la sélection du preset. C’est la route de référence pour le Bank Select + Program Change.
2. **BLE MIDI** — route Web Bluetooth expérimentale, visible par l'API du navigateur via un service GATT et une caractéristique dédiée. Elle reproduit la sémantique logique Bank Select + Program Change, mais la trame de paquet est différente de la route USB et doit être considérée comme une extension expérimentale.

La connexion BLE démarre à travers l’interface de scan Web Bluetooth de l’application. Une fois le device choisi, le flux de contrôle transporte la même abstraction logique `bank` + `slot` que le transport USB, mais l'encapsulation GATT sous Web Bluetooth est différente et le protocole est annoncé comme exploitable seulement en branche expérimentale.

1. Brancher le TONEX Pedal en USB
2. Ouvrir l'application dans Chrome/Edge
3. Sélectionner le device MIDI dans le menu déroulant **Device** ou lancer la recherche BLE si le routeur Web Bluetooth est actif
4. Choisir le canal MIDI (défaut : Ch 1)
5. Le statut passe à **Connecté** (point vert)

> **Note BLE** : la route BLE est expérimentale et peut se déconnecter rapidement selon le PC. De plus, la plage de presets `42C..49C` (`pc = 128..149`) ne fonctionne pas encore correctement sur ce flux et peut renvoyer la commande vers les presets 00A / au mauvais offset sur le pédalier.

### Synchronisation USB (lecture des presets)

1. Cliquer sur **Sync USB**
2. Sélectionner le port série TONEX Pedal dans le dialog
3. La progression s'affiche : Hello → State → Lecture des 150 presets
4. Les noms se remplissent automatiquement
5. Le bouton affiche **Terminé! X/150 presets lus**

### Export / Import JSON

- Cliquer sur **⬇ JSON** pour télécharger les noms de presets, les assignations de grille et les couleurs en `tonex-config.json`
- Cliquer sur **⬆ JSON** pour importer un fichier exporté
  - **Ancien format** (noms de presets uniquement) : importe les noms directement
  - **Nouveau format** (v2 avec config grille) : demande si on veut restaurer les assignations et couleurs

Format d'export (v2) :
```json
{
  "version": 2,
  "presets": {
    "0_A": "Trooper - 80s Pack",
    "0_B": "80s Lead - 80s Pack"
  },
  "buttons": {
    "0": { "bank": 0, "slot": "A" },
    "4": { "bank": 1, "slot": "B" }
  },
  "colors": {
    "0": "red",
    "4": "blue"
  }
}
```

### Grille 3×3

- **Clic simple** sur un bouton → envoie le Bank Select + Program Change au pédalier
- **Glisser** un preset de la bibliothèque → assigne au bouton
- **Glisser** un bouton vers un autre → swap les positions (les couleurs suivent)
- **Glisser** un bouton vers la corbeille → vide le bouton et sa couleur
- **Double-clic** → ouvre le modal d'édition (renommage)
- **Rond couleur** (coin supérieur droit) → cliquer pour assigner une couleur LED à la tuile

### Bibliothèque

- **Clic simple** → envoie le MIDI pour écouter le preset
- **Double-clic** → édite le nom
- **Recherche** → filtre par nom ou numéro de bank/slot
- **Glisser** vers la grille → assigne le preset
- **Chevron** (▶/◀) sur la bordure du panneau → minimise/étend la bibliothèque

## Architecture technique

### Fichiers

```
tonexpedal/
├── index.html          # Application single-file (HTML + CSS + JS)
├── favicon.svg         # Icône SVG
├── README.md           # Cette documentation
├── captures/
│   └── tnx1.png        # Capture d'écran de l'interface
└── V1.0/
    └── index.html      # Archive de la version 1.0
```

### Protocole MIDI

Le TONEX Pedal utilise 50 banks × 3 slots (A/B/C) = 150 presets.

| Preset # | Bank Select (CC#0) | Program Change |
|----------|-------------------|----------------|
| 0–127    | CC#0 = 0          | PC = preset#   |
| 128–149  | CC#0 = 1          | PC = preset# − 128 |

```
Bank Select : [0xB0 + channel, 0x00, value]
Program Change : [0xC0 + channel, PC]
```

### Protocole USB CDC Série (HDLC)

Le pédalier expose deux interfaces USB :
- **USB-MIDI** — pour les Bank Select / Program Change sur la route Web MIDI / USB-MIDI
- **USB CDC** — pour la communication série (lecture presets, paramètres)

### Transport BLE MIDI — architecture et trame

Le transport BLE est un chemin séparé et non réductible au flux USB. L’application ne construit pas un nouveau “midiOutput” d’USB, elle ouvre un canal GATT via `navigator.bluetooth.requestDevice()` puis accède au service `03b80e5a-ede8-4b33-a751-6ce34ec4c700` et à la caractéristique `7772e5db-3868-4112-a1a9-f2669d106bf3`.

La couche application manipule la table de presets dans un espace logique `bank × slot` converti ensuite en `pc` selon le calcul standard :

```
pc = bank × 3 + slotIndex
```

Pour le flux USB / Web MIDI, la sémantique de transport est bien connue :

```
[0xB0 + canal, 0x00, 0]  -> CC#0 Bank Select pour la première banque
[0xC0 + canal, PC]       -> Program Change
```

Pour le transport BLE, la couche d’application sérialise la sélection de preset par une trame GATT de paquet MIDI unique :

```
[0x80, midiCh, 0, bankVal, 0x80, 0xC0 + ch, pcVal]
```

avec :

- `midiCh = 0xB0 + canal`
- `bankVal = 0` pour la plage `0..127`
- `bankVal = 1` uniquement dans la fenêtre de cartographie haute `128..149`
- `pcVal = pc` pour `0..127`
- `pcVal = pc - 128` pour `128..149`

Cette trame est conservée dans l’architecture logique, mais elle doit être vue comme un paquet de transport BLE distinct du flux USB-MIDI classique. USB et BLE n’ont pas le même endpoint de transport, même si la couche applicative de presets dans `bank` + `slot` est la même.



#### Trame HDLC

```
[0x7E] [payload stuffed] [CRC_lo stuffed] [CRC_hi stuffed] [0x7E]
```

- **Délimiteur** : `0x7E`
- **Byte stuffing** : `0x7E` → `0x7D 0x5E`, `0x7D` → `0x7D 0x5D`
- **CRC-CCITT** : polynomial `0x8408`, init `0xFFFF`, résultat inversé (`~crc & 0xFFFF`)

#### Commandes

| Commande | Payload | Description |
|----------|---------|-------------|
| Hello | `b9 03 00 82 04 00 80 10 01 b9 02 02 10` | Initialisation connexion |
| Request State | `b9 03 00 82 06 00 80 10 03 b9 02 81 01 02 10` | Demande l'état courant |
| Request Preset (0–127) | `b9 03 81 00 02 82 06 00 80 10 03 b9 04 10 01 [index] 00` | Demande preset (17 octets) |
| Request Preset (128+) | `b9 03 81 00 02 82 06 00 80 10 03 b9 04 10 01 80 [index] 00` | Demande preset (18 octets, escape `0x80`) |

#### Réponse preset — Structure

```
[header] [B9 04 B9 02 BC 21] [nom 33 octets] [paramètres...]
                                           ↑ NAME_MARKER
```

La section paramètres commence par le marker `BA 03 BA 6D` (`PARAM_MARKER`), suivi de floats encodés `0x88` + 4 octets (little-endian) :

| Index paramètre | Octet offset (×5) | Description |
|----------------|-------------------|-------------|
| 17 | 85 | **AMP Enable** — 0.0 = off, >0.5 = on |
| 22 | 110 | **CAB Type** — 0.0 = off, 1.0 = VIR, 2.0 = Tone Model |

### Device ID

- **TONEX Pedal (full size)** : `0x10`
- TONEX One : `0x0B` (non supporté)

### Abstraction de transport (Support Android)

L'application utilise une couche d'abstraction de transport pour supporter à la fois **Web Serial** (desktop) et **WebUSB** (Android) :

```
transportSend(frame)      → Serial.write() ou USB.bulkTransferOut()
transportStartRead()      → boucle reader Serial ou boucle USB.bulkTransferIn()
transportIsOpen()         → serialPort.opened ou usbDevice.opened
transportDisconnect()     → serialPort.close() ou usbDevice.close()
```

**Flux de connexion :**
1. Essayer **Web Serial** en premier (desktop Chrome/Edge)
2. Si non disponible ou échec, fallback vers **WebUSB** (Android Chrome)
3. WebUSB affiche le sélecteur de device filtré par VID `0x1963` (IK Multimedia)

**Configuration WebUSB CDC :**
- Trouver l'interface CDC Communication (classe `0x02`) → control transfers (SET_LINE_CODING, SET_CONTROL_LINE_STATE)
- Trouver l'interface CDC Data (classe `0x0A`) → bulk endpoints pour les données HDLC
- Si la classe `0x0A` n'est pas trouvée, fallback sur toute interface avec des bulk endpoints

### Persistance

Tout est sauvegardé en `localStorage` sous la clé `tonex-state` :

```json
{
  "buttons": {
    "0": { "bank": 0, "slot": "A" },
    "4": { "bank": 1, "slot": "B" }
  },
  "midi": { "device": "ToneX MIDI Out", "channel": 0 },
  "presets": {
    "0_A": { "name": "Trooper - 80s Pack", "amp": true, "cab": false },
    "0_B": { "name": "80s Lead - 80s Pack", "amp": true, "cab": true }
  },
  "colors": {
    "0": "red",
    "4": "blue"
  }
}
```

## Dépannage

| Problème | Solution |
|----------|----------|
| Aucun device MIDI | Vérifier que le pédalier est branché. Chrome → `chrome://midi-devices` |
| Web Serial non disponible | Utiliser Chrome 89+ ou Edge 89+. Vérifier HTTPS/localhost |
| Sync USB échoue | Fermer tout autre logiciel utilisant le port série (IK Tonex, etc.) |
| Noms ne s'affichent pas | Relancer le Sync USB. Vérifier la console (F12) pour les erreurs |
| AMP/CAB toujours gris | Vérifier dans la console que les float32 sont correctement lus |
| Canvas vide | Recharger la page, le localStorage peut être corrompu |
| Android : Sync ne lit pas les données | Le fallback WebUSB devrait s'activer automatiquement. Vérifier les logs d'interface/endpoint dans la console |

## Bugs connus et limites de transport

La branche BLE MIDI est actuellement documentée comme une route expérimentale. Les limitations connues sont les suivantes :

- **La connexion BLE peut se déconnecter rapidement** : la couche BLE MIDI peut sembler se déconnecter après un ou quelques writes, en fonction du PC, du navigateur, et de la pile GATT locale. Cette observation est présentée comme une limite expérimentale, pas comme un comportement stable de production.
- **La plage `42C..49C` (`pc = 128..149`) n’est pas encore prise en charge correctement** : la route BLE semble renvoyer parfois sur le preset `00A` ou sur un preset de mauvais bank/slot. Le comportement de la branche BLE pour ces presets supérieurs est donc signalé comme un bug connu et non corrigé : le peset `>= 42C` n’est pas encore routé de façon fiable.
- **USB/Web MIDI reste la référence stable** : la route USB-MIDI et la route USB CDC de lecture des presets restent la base de travail qui doit être considérée comme la source de vérité. La branche BLE est seulement une route expérimentale de lecture/contrôle de preset en parallèle.

## Crédits

- Protocole USB CDC : reverse-engineered depuis [Builty/TonexOneController](https://github.com/Builty/TonexOneController)
- Documentation protocole : [vit3k/tonex_controller](https://github.com/vit3k/tonex_controller)
- Interface : IK Multimedia TONEX Pedal Controller v1.1

## Licence

Projet personnel — usage non commercial.
