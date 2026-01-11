# URL2Stream 🎬

**URL2Stream** transforme n'importe quel site web en flux vidéo, permettant d'afficher des interfaces web complexes (dashboards, pages de statut, etc.) via des systèmes de diffusion vidéo traditionnels comme DVB-T. Le format de sortie et le codec sont configurables selon vos besoins.
Conçu initiallement pour intégration avec [DVB-Tx](https://github.com/TrusterBSW/DVB-Tx) afin de diffuser des dashboards (Grafana, Zabbix, etc.) via DVB.

## 📋 Table des matières

- [Démarrage rapide](#-démarrage-rapide)
- [Architecture](#-architecture)
- [Configuration](#-configuration)
- [Structure du projet](#-structure-du-projet)
- [xdotool Cheat Sheet](#-xdotool-cheat-sheet)


## 🚀 Démarrage rapide

### 💻 Avec encodage CPU (libx264)

```bash
cd Docker
docker compose build -t url2stream
docker run -d url2stream:latest
```

### 🚀 Avec encodage GPU NVIDIA (NVENC)

```bash
cd Docker-GPU
docker compose build -t url2stream-gpu
docker run -d -hwaccels --gpus all url2stream-gpu:latest
```

### ⚙️ Variables d'environnement personnalisées

```bash
docker run \
  -e FIREFOX_URL="http://grafana.local:3000/d/dashboard" \
  -e XVFB_WIDTH=2560 \
  -e XVFB_HEIGHT=1440 \
  -e FFMPEG_CRF=20 \
  -e FFMPEG_FRAMERATE=60 \
  url2stream:latest
```


## 🏗️ Architecture

```
┌─────────────────────────────────────┐
│      Docker Container               │
├─────────────────────────────────────┤
│  ┌──────────────────────────────┐   │
│  │  XVFB (Serveur X virtuel)    │   │
│  │  :99 → 1920x1080x24          │   │
│  └────────────┬─────────────────┘   │
│               │                     │
│  ┌────────────▼──────────────────┐  │
│  │  Firefox (Mode Kiosk)         │  │
│  │  - Charge URL cible           │  │
│  │  - Exécute scripts optionnels │  │
│  │  - Affichage auto-refresh     │  │
│  └────────────┬──────────────────┘  │
│               │                     │
│  ┌────────────▼──────────────────┐  │
│  │  FFmpeg (x264 ou NVENC)       │  │
│  │  - Encode XVFB → vidéo        │  │
│  │  - Output: flux UDP ou fichier│  │
│  └────────────┬──────────────────┘  │
│               │                     │
└───────────────┼─────────────────────┘
                │
         ┌──────▼───────┐
         │  Flux video  │
         └──────────────┘
```

### 🔄 Flux de traitement

1. **XVFB** crée un serveur X virtuel offrant un framebuffer en mémoire
2. **Firefox** démarre en mode kiosk, accède à l'URL fournie et affiche la page
3. **Optionnel** : Un script bash (xdotool) peut automatiser des interactions (navigation, saisie, clics)
4. **FFmpeg** capture l'écran XVFB et encode en vidéo (codec configurable)
5. **Output** : Le flux est envoyé en UDP réseau ou sauvegardé en fichier selon configuration

### 🐳 Images Docker

Deux images Docker sont fournies, optimisées pour différents scénarios :

#### 💻 `Docker/` (CPU - libx264)
- Encodage H.264 par **libx264** (CPU)
- Compatible sur tout matériel
- Consommation CPU plus élevée
- Usage : Déploiements standards, machines sans GPU

**Exemple de performance :** Xeon E5-2670v2 → 250% CPU

#### 🚀 `Docker-GPU/` (GPU NVIDIA - NVENC)
- Encodage H.264 par **NVENC** (GPU NVIDIA)
- Accélération matérielle GPU
- Consommation CPU très réduite, GPU modéré
- Usage : Production, performances élevées

**Exemple de performance :** Quadro P400 → 10% CPU, 5% GPU

#### 🔧 Installation NVIDIA Container Runtime

La runtime NVIDIA est **obligatoire** pour l'image GPU. Consultez la [documentation officielle NVIDIA](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html) pour l'installation.

## ⚙️ Configuration

Toutes les variables d'environnement sont définies dans `docker-compose.yml` (CPU) et `docker-compose-gpu.yml` (GPU).

| Variable | Valeur par défaut (CPU) | Valeur par défaut (GPU) | Description |
|----------|--------------|--------------|-------------|
| `DISPLAY` | `:99` | `:99` | 🖥️ Numéro d'affichage XVFB |
| `XVFB_WIDTH` | `1920` | `1920` | 📏 Largeur écran virtuel en pixels |
| `XVFB_HEIGHT` | `1080` | `1080` | 📏 Hauteur écran virtuel en pixels |
| `XVFB_DEPTH` | `24` | `24` | 🎨 Profondeur couleur en bits |
| `FFMPEG_FRAMERATE` | `24` | `24` | ⏱️ Framerate en fps |
| `FFMPEG_CRF` | `26` | `26` | 🎬 Qualité vidéo (0-51, plus bas = meilleur) |
| `FFMPEG_CODEC` | `libx264` | `h264_nvenc` | 🔀 Codec : `libx264` (CPU) ou `h264_nvenc` (GPU) |
| `FFMPEG_PRESET` | `ultrafast` | `fast` | ⚡ Preset d'encodage |
| `FFMPEG_CONTAINER` | `mpegts` | `mpegts` | 📦 Format conteneur vidéo |
| `FFMPEG_OUTPUT_URI` | `udp://172.17.0.1:9000` | `udp://172.17.0.1:9000` | 🔗 Destination : URI réseau ou chemin fichier |
| `FIREFOX_URL` | `https://example.com/` | `https://example.com/` | 🌐  URL du site web à capturer |
| `FIREFOX_PROFILE` | — | — | 👤 **Optionnel**. Chemin profil Firefox existant |
| `CUSTOM_SCRIPT` | — | — | 📜 **Optionnel**. Chemin script bash pour automation |

### 🔑 Exemple de configurations 


#### 🔗 `FFMPEG_OUTPUT_URI`

Destination du flux vidéo. Peut être une adresse UDP réseau ou un chemin fichier local.

**Exemples :**
```bash
-e FFMPEG_OUTPUT_URI="udp://172.17.0.1:9000"
-e FFMPEG_OUTPUT_URI="/data/output.ts"
```

#### 👤 `FIREFOX_PROFILE`

Non défini par défaut, permet de partager les cookies et sessions d'un profil Firefox desktop au conteneur. Utile pour accéder à des pages nécessitant une authentification.

**Usage :**
```bash
docker run -v ./data:/data \
  -e FIREFOX_PROFILE=/data/firefox_profile \
  -e FIREFOX_URL=http://status.infra.ex/logged-dashboard \
  url2stream:latest
```

#### 📜 `CUSTOM_SCRIPT`

Non défini par défaut, permet de definir un chemin vers un script bash exécuté après le chargement de la page. Permet d'automatiser des interactions avec le navigateur via **xdotool**.

**Usage :**
```bash
docker run -v ./data/:/data \
  -e CUSTOM_SCRIPT=/data/scripts/auto-interact.sh \
  url2stream:latest
```

Exemple de script :
```bash
#!/bin/bash
# Attendre le chargement complet
sleep 3

# Obtenir l'ID de fenêtre Firefox
WID=$(xdotool search --name "Firefox" | head -1)
xdotool windowfocus $WID

# Deplacer la souris aux coordoné 960x540 pixel, et clic
xdotool mousemove 960 540 click 1

# Saisir du texte
xdotool type "monmotdepasse"
xdotool key Return
```

## 📦 Structure du projet

```
.
├── Docker/                 # 💻 Build CPU (libx264)
    └── docker-compose.yml  # ⚙️ Config CPU
├── Docker-GPU/             # 🚀 Build GPU NVIDIA (NVENC)
    └── docker-compose.yml  # 🚀 Config GPU
└── README.md               # 📖 Documentation
```



## 🎮 xdotool Cheat Sheet

**xdotool** automatise le navigateur virtuel.

### 🖱️ Mouvements et clics

```bash
# Positionner la souris (x, y)
xdotool mousemove 960 540

# Clics
xdotool click 1 # Clic gauche
xdotool click 2 # Clic molette
xdotool click 3 # Clic droit

# Positionner la souris (x, y) et cliquer
xdotool mousemove 960 540 click 1

# Scroll vers le bas
xdotool click --repeat 5 5
```

### ⌨️ Saisie de texte

```bash
# Taper du texte simple
xdotool type "Bonjour"

# Taper avec délai (30ms entre caractères)
xdotool type --delay 30 "Texte lent"
```

### 🔑 Touches spéciales

```bash
# Touches individuelles
xdotool key Tab
xdotool key space
xdotool key Return

# Répéter une touche
xdotool key --repeat 3 BackSpace
```
