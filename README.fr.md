# media-stack

Une petite stack d'automatisation média auto-hébergée : **Transmission**
derrière un kill switch VPN, plus **Sonarr**, **Radarr**, **Prowlarr** et
**Bazarr**. Un seul `docker compose up -d`, aucun secret dans le dépôt,
identique sur macOS et Linux.

🇬🇧 [English version](README.md)

## Ce qui distingue ce montage

- **Transmission n'a aucune route vers Internet en dehors du VPN.** Il partage
  la pile réseau de gluetun (`network_mode: service:gluetun`) : si le tunnel
  tombe, Transmission passe hors ligne. Le kill switch est structurel : il
  tient même sans réglage pour l'imposer.
- **Tout écoute sur `127.0.0.1` par défaut.** Aucune interface web n'est
  exposée au réseau local ou à Internet tant que tu ne changes pas
  délibérément `BIND_ADDRESS`.
- **Un seul point de montage, donc les hardlinks fonctionnent.** Téléchargements
  et bibliothèque vivent sous le même répertoire hôte monté sur `/data` : Sonarr
  et Radarr importent par lien dur — pas de copie, pas d'espace disque doublé,
  le seed continue.
- **Aucun identifiant dans le dépôt.** `config/` et `.env` sont ignorés par
  git ; seul `.env.example` est publié.

## Prérequis

- Docker Engine (ou Docker Desktop) avec Compose v2.24+
- Un abonnement VPN supporté par
  [gluetun](https://github.com/qdm12/gluetun-wiki/tree/main/setup/providers)
  — les valeurs par défaut visent CyberGhost/OpenVPN, mais n'importe quel
  fournisseur convient
- De la place disque sous le répertoire visé par `DOWNLOADS_DIR`

## Installation

```bash
git clone https://github.com/maax6/media-stack.git
cd media-stack
cp .env.example .env
```

Édite `.env` :

| Variable | Remarques |
| --- | --- |
| `PUID` / `PGID` | `id -u` / `id -g`. Linux : souvent `1000/1000`. macOS : souvent `501/20`. |
| `TZ` | par ex. `Europe/Paris` |
| `DOWNLOADS_DIR` | Le répertoire hôte unique partagé par tous les services |
| `VPN_SERVICE_PROVIDER`, `OPENVPN_USER`, `OPENVPN_PASSWORD` | Depuis l'espace client de ton VPN |
| `TRANSMISSION_USER` / `TRANSMISSION_PASSWORD` | Obligatoires — ne laisse pas les valeurs par défaut |

Crée les répertoires média et, **pour les fournisseurs qui exigent un
certificat client (CyberGhost en fait partie)**, dépose la paire de fichiers —
gluetun ne démarrera pas sans :

```bash
mkdir -p "$DOWNLOADS_DIR"/{torrents,media/movies,media/tv}
mkdir -p config/gluetun
cp /chemin/vers/client.crt /chemin/vers/client.key config/gluetun/
```

Démarrage :

```bash
docker compose up -d
docker compose ps
```

Vérifie que le tunnel est bien monté et que Transmission est derrière — la
commande doit renvoyer l'IP du VPN, pas la tienne :

```bash
docker compose logs gluetun | grep -i "public ip"
```

## Interfaces web

Toutes liées à `127.0.0.1` par défaut ; les ports sont configurables dans `.env`.

| Application | URL |
| --- | --- |
| Transmission | http://127.0.0.1:19091 |
| Sonarr | http://127.0.0.1:18989 |
| Radarr | http://127.0.0.1:17878 |
| Prowlarr | http://127.0.0.1:19696 |
| Bazarr | http://127.0.0.1:16767 |

## Configuration au premier lancement

1. **Transmission** — répertoire de téléchargement : `/data/torrents`.
2. **Sonarr** — dossier racine `/data/media/tv`. Ajoute le client de
   téléchargement Transmission avec l'hôte `transmission`, port `9091`. Laisse
   *Remote Path Mapping* vide : tous les conteneurs voient le même chemin
   `/data`. Active les hardlinks dans *Media Management*.
3. **Radarr** — pareil, avec le dossier racine `/data/media/movies`.
4. **Prowlarr** — ajoute tes indexeurs, puis dans *Settings → Apps* ajoute
   Sonarr (`http://sonarr:8989`) et Radarr (`http://radarr:7878`) pour que les
   indexeurs se synchronisent automatiquement.
5. **Bazarr** — relie-le à Sonarr (`http://sonarr:8989`) et Radarr
   (`http://radarr:7878`), puis ajoute des fournisseurs de sous-titres.

## Modèle de sécurité — à lire avant d'exposer quoi que ce soit

**Ce qui est protégé :** tout le trafic pair-à-pair. Transmission ne peut pas
atteindre Internet en dehors du tunnel, par construction.

**Ce qui ne l'est pas :**

- **Sonarr, Radarr, Prowlarr et Bazarr utilisent ton IP réelle.** Ils parlent
  directement aux services de métadonnées et aux API d'indexeurs. Le trafic
  torrent — celui qui compte pour la visibilité auprès des pairs — reste dans
  le VPN. Faire passer Prowlarr par gluetun est possible mais volontairement
  écarté ici : les IP de sortie VPN partagées se font captcha et bloquer par
  beaucoup d'indexeurs, les noms de services Docker ne se résolvent plus dans
  l'espace réseau de gluetun, et le gain est mince puisque aucun trafic pair
  ne sort de ce conteneur.
- **Ces applications n'ont pas d'authentification durcie.** Garde
  `BIND_ADDRESS=127.0.0.1`. Pour un accès à distance, passe par un VPN vers ton
  réseau (Tailscale, WireGuard) ou un reverse proxy qui termine le TLS et
  impose une authentification. Ne redirige jamais ces ports depuis Internet.
- **`config/` contient de vrais secrets** dès que la stack a tourné : clés API,
  empreinte du mot de passe Transmission, clé privée VPN. Le dossier est ignoré
  par git — si tu le sauvegardes, traite cette sauvegarde comme sensible.

**Pas de port entrant.** Le port pair de Transmission n'est volontairement pas
publié : le client est en sortie seule. Les ratios sur trackers privés en
pâtiront. Si ton fournisseur VPN supporte la redirection de port, gluetun sait
la gérer — voir le wiki gluetun.

## Commandes du quotidien

```bash
docker compose logs -f            # suivre tous les logs
docker compose logs -f gluetun    # état du tunnel, IP publique, reconnexions
docker compose pull && docker compose up -d   # mettre à jour les images
docker compose down               # arrêt ; config et médias intacts
```

## Changer de machine

Copie `.env` et `config/` par un canal sûr (les deux contiennent des secrets),
puis `git clone` et `docker compose up -d`. Ajuste `PUID`/`PGID` et
`DOWNLOADS_DIR` pour le nouvel hôte. Sinon, repars de zéro : la stack recrée
son `config/` au premier lancement et tu reconfigures les applications à la
main.

## Cadre légal

Cette stack n'est que de la plomberie. Elle est livrée sans indexeur, sans
tracker et sans contenu. Télécharger des œuvres protégées sans autorisation est
illégal dans la plupart des pays — l'usage que tu en fais n'engage que toi.

## Licence

[MIT](LICENSE)
