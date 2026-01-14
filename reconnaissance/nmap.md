# Nmap

## Scans basiques

```bash
nmap <IP>                    # Scan simple
nmap -sV -sC <IP>           # Version + scripts
nmap -p- <IP>               # Tous les ports
nmap -F <IP>                # Scan rapide
nmap -A <IP>                # Scan agressif
```

## Types de scan

```bash
nmap -sS <IP>               # SYN scan (stealth)
nmap -sT <IP>               # TCP connect
nmap -sU <IP>               # UDP scan
```

## Ports

```bash
nmap -p 22,80,443 <IP>      # Ports spécifiques
nmap -p 1-1000 <IP>         # Plage
nmap -p- <IP>               # Tous les ports
```

## Détection

```bash
nmap -O <IP>                # OS detection
nmap -sV <IP>               # Version services
```

## Scripts NSE

```bash
nmap --script vuln <IP>     # Scripts vulns
nmap --script http-* <IP>   # Scripts HTTP
nmap --script-help <nom>    # Aide script
```

## Output

```bash
nmap -oN scan.txt <IP>      # Normal
nmap -oX scan.xml <IP>      # XML
nmap -oG scan.grep <IP>     # Greppable
nmap -oA scan <IP>          # Tous formats
```

## Options utiles

```bash
nmap -T4 <IP>               # Timing (0-5)
nmap -v <IP>                # Verbose
nmap --open <IP>            # Ports ouverts seulement
nmap -Pn <IP>               # Skip ping
```

## Exemples combinés

```bash
# Scan complet
sudo nmap -sS -sV -O -p- -T4 -oA full_scan <IP>

# Scan web rapide
nmap -p 80,443 -sV --script http-* <IP>

# Scan UDP commun
sudo nmap -sU -p 53,161,500 <IP>
```
