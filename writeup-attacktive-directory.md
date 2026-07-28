# Writeup — TryHackMe : Attacktive Directory

**Room :** https://tryhackme.com/room/attacktivedirectory

**Objectif :** Compromettre un environnement Active Directory vulnérable en enchaînant énumération, attaques Kerberos et extraction d'identifiants.

**Cible :** Contrôleur de domaine du domaine `spookysec.local`

---

## 1. Préparation de l'environnement d'attaque

Avant de commencer l'énumération, on installe les outils nécessaires sur Kali.

### 1.1 Impacket

Impacket est une collection de classes Python qui permet de dialoguer directement avec les protocoles réseau Windows bas niveau (SMB, MSRPC, Kerberos, NTLM) sans avoir à les réimplémenter. C'est la boîte à outils de référence pour attaquer un AD.

Scripts que je vais probablement utiliser dans ce lab :

| Script | Utilité |
|---|---|
| `secretsdump.py` | Extraction des hashs de mots de passe (SAM locale ou base NTDS.dit du DC) |
| `psexec.py` / `wmiexec.py` / `smbexec.py` | Obtenir un shell distant avec des identifiants valides (ou un hash via Pass-the-Hash) |
| `GetNPUsers.py` | AS-REP Roasting (récupérer un hash crackable pour un compte sans pré-authentification Kerberos) |
| `GetUserSPNs.py` | Kerberoasting (récupérer un hash crackable pour un compte de service) |

**Installation :** 
```bash
sudo apt update
sudo apt install python3-impacket impacket-scripts
```

### 1.2 BloodHound + Neo4j

BloodHound sert à cartographier visuellement les relations et privilèges dans un AD (qui peut administrer quoi, chemins d'escalade de privilèges, etc.). Neo4j est la base de données graphe sur laquelle BloodHound s'appuie.

```bash
sudo apt install bloodhound neo4j
```

*(À utiliser plus tard dans le lab, une fois des identifiants obtenus, pour visualiser les chemins d'attaque vers Domain Admin.)*

### 1.3 Kerbrute

Kerbrute permet de faire du brute-force/énumération contre le service Kerberos (port 88) : découverte de noms d'utilisateurs valides, brute-force de mots de passe, password spraying. Il est plus discret que d'autres méthodes car il exploite une particularité du protocole Kerberos (les réponses diffèrent selon qu'un utilisateur existe ou non), sans générer d'échec de connexion classique.

```bash
# Télécharger la dernière version
wget https://github.com/ropnop/kerbrute/releases/latest/download/kerbrute_linux_amd64

# Rendre exécutable
chmod +x kerbrute_linux_amd64

# Déplacer dans le PATH
sudo mv kerbrute_linux_amd64 /usr/local/bin/kerbrute
```

---

## 2. Énumération

### 2.1 Scan Nmap

Premier réflexe classique : identifier les ports/services ouverts sur la cible pour orienter la suite de l'énumération.
```bash
nmap -sC -sV -p- <IP_DU_DC>
```
Sur un contrôleur de domaine typique, on s'attend à voir : 53 (DNS), 88 (Kerberos), 135/139/445 (RPC/SMB), 389/636 (LDAP/LDAPS), 3268/3269 (Global Catalog).

### 2.2 Énumération SMB (139/445)

**Outil : `enum4linux`**

Utilité : outil tout-en-un pour énumérer des informations via SMB/RPC sur une machine Windows — partages disponibles, utilisateurs, groupes, politique de mots de passe, informations sur le domaine, etc. C'est l'outil de référence pour un premier passage sur les ports 139/445.

```bash
enum4linux -a <IP_DU_DC>
```
`-a` = mode "tout énumérer" (utilisateurs, partages, politique de mot de passe, groupes...).

### 2.3 Énumération Kerberos (88) — découverte d'utilisateurs

Le port 88 ouvert signale la présence de Kerberos, ce qui permet d'utiliser Kerbrute pour découvrir des comptes valides sans avoir besoin d'identifiants au préalable.

```bash
./kerbrute userenum --dc <IP_DU_DC> -d spookysec.local <FICHIER_WORDLIST_USERNAMES>
```
- `--dc` : IP du contrôleur de domaine à cibler
- `-d` : nom du domaine (`spookysec.local`)
- dernier argument : wordlist de noms d'utilisateurs à tester

Résultat attendu : une liste de comptes utilisateurs valides confirmés dans le domaine, à conserver pour l'étape suivante.

---

## 3. Exploitation Kerberos — AS-REP Roasting

### 3.1 Principe

L'AS-REP Roasting cible les comptes ayant l'attribut **"Ne pas nécessiter la pré-authentification Kerberos"** (`DONT_REQ_PREAUTH`) activé.

Fonctionnement normal de Kerberos vs. cas vulnérable :
1. Normalement, avant de délivrer un TGT (Ticket Granting Ticket), le KDC exige une preuve d'identité (pré-authentification) — typiquement un timestamp chiffré avec le hash NT du mot de passe de l'utilisateur.
2. Si la pré-authentification est désactivée sur un compte, **n'importe qui** peut demander un TGT pour ce compte sans fournir de mot de passe.
3. Le TGT renvoyé (plus précisément la portion AS-REP) est chiffré avec le hash NT du compte ciblé.
4. On récupère ce blob chiffré et on tente de le craquer **hors ligne** (donc sans limite de tentatives, sans lockout AD) pour retrouver le mot de passe en clair.

### 3.2 Mise en pratique

Avec la liste d'utilisateurs valides obtenue via Kerbrute, on teste l'AS-REP Roasting sur chacun avec le script Impacket dédié :

```bash
GetNPUsers.py spookysec.local/ -no-pass -usersfile <fichier_utilisateurs_valides> -dc-ip <IP_DU_DC>
```
- `-no-pass` : on ne fournit pas de mot de passe, on interroge juste le KDC
- `-usersfile` : liste des noms d'utilisateurs à tester (issue de l'énumération Kerbrute)
- Pour les comptes vulnérables, le script renvoie un hash au format `$krb5asrep$...`, prêt à être passé à un outil de cracking.

### 3.3 Craquage du hash

```bash
hashcat -m 18200 <fichier_hash.txt> <wordlist> --force
# ou
john --format=krb5asrep --wordlist=<wordlist> <fichier_hash.txt>
```
`-m 18200` = mode hashcat spécifique aux hashs AS-REP.

---

## Notes personnelles / points à retenir

- L'AS-REP Roasting ne nécessite **aucun identifiant préalable** — seule une liste de noms d'utilisateurs valides suffit, d'où l'intérêt de bien énumérer les usernames en amont avec Kerbrute.
- Toujours privilégier le craquage **hors ligne** dès que possible : plus rapide, indétectable côté AD, pas de risque de lockout de compte.
- Garder une trace de chaque hash/mot de passe trouvé au fur et à mesure — ils serviront de pivot pour les étapes suivantes (Kerberoasting, accès SMB, etc.).
