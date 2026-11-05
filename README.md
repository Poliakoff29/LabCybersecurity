# pfSense · Kali · Ubuntu · Wazuh — Lab d'attaque, de détection et de défense

Un lab maison en VirtualBox / Hyper-V pour pratiquer, de bout en bout, la mise en place d'un pare-feu périmétrique, une attaque par déni de service, une reconnaissance réseau suivie d'un brute-force SSH, une injection SQL sur une application web vulnérable protégée par un WAF, et la supervision de plusieurs machines avec un SIEM (Wazuh).

| | |
|---|---|
| **Durée estimée** | 1 à 2 jours |
| **Hyperviseurs** | VirtualBox + Hyper-V |
| **Machines** | pfSense · Kali · Ubuntu · Windows 10 · Windows Server · Wazuh |

> **Note sur les images** : chaque emplacement d'image ci-dessous est inséré avec la syntaxe Markdown standard `![description](nom-fichier.png)`. Place simplement tes captures d'écran, renommées exactement comme indiqué, dans le même dossier que ce fichier (ou dans un sous-dossier `images/` en ajustant les chemins) — elles s'afficheront automatiquement sur GitHub.

## Sommaire

- [00. Présentation & architecture](#00-présentation--architecture)
- [01. Prérequis](#01-prérequis)
- [02. Configuration réseau des VM](#02-configuration-réseau-des-vm)
- [03. Installation de pfSense](#03-installation-de-pfsense)
- [04. Scénario 1 — DoS & mitigation](#04-scénario-1--déni-de-service-dos--mitigation)
- [05. Scénario 2 — Recon & brute-force SSH](#05-scénario-2--reconnaissance--brute-force-ssh)
- [06. Scénario 3 — SQLi & WAF](#06-scénario-3--injection-sql--protection-waf)
- [07. Scénario 4 — Supervision Wazuh](#07-scénario-4--supervision-avec-wazuh)
- [08. Dépannage réseau (MTU)](#08-dépannage-réseau--le-problème-de-mtu)
- [09. Conclusion & suite](#09-conclusion--pistes-dextension)

---

## 00. Présentation & architecture

Ce lab part d'un montage simple — un pare-feu pfSense entre un attaquant (Kali) et une victime (Ubuntu) — et l'étend en quatre scénarios indépendants, chacun ciblant une couche différente de la sécurité : réseau (DoS), identité (brute-force), application (injection SQL/WAF), et détection (SIEM).

| Machine | Rôle | Réseau |
|---|---|---|
| pfSense | Pare-feu périmétrique | WAN (pont) + LAN (interne) |
| Kali Linux | Attaquant | Pont (même réseau physique que le WAN) |
| Ubuntu Server | Victime / hôte applicatif (DVWA, ModSecurity) | LAN interne |
| Wazuh (OVA) | Serveur SIEM (manager + indexer + dashboard) | Pont |
| Windows 10 / Server 2019 | Postes supervisés (agents Wazuh) | Pont (via Hyper-V, commutateur externe) |

> **Note d'architecture** — Le lab a été construit en deux temps : d'abord toutes les machines derrière pfSense (réseau interne isolé, « LabNet_dmz »), pour les scénarios 1 à 3 ; puis, pour le scénario Wazuh, les machines ont été temporairement rebasculées en accès direct par pont (pfSense éteint) afin de simplifier le déploiement d'un SIEM multi-OS. Les deux approches sont documentées ci-dessous — à toi de choisir celle qui convient à ton objectif.

## 01. Prérequis

- VirtualBox ≥ 7.x (et, pour les VM Windows ici, Hyper-V activé en parallèle)
- ISO : pfSense CE 2.6/2.7, Kali Linux, Ubuntu Server 22.04+
- OVA Wazuh (téléchargé depuis le site officiel)
- Au moins 16 Go de RAM côté hôte si toutes les VM tournent en même temps (Wazuh seul recommande 8 Go)
- Une connexion Internet stable — voir la section [Dépannage réseau](#08-dépannage-réseau--le-problème-de-mtu) si tu rencontres des coupures

## 02. Configuration réseau des VM

Plan d'adressage utilisé pour la partie « derrière pfSense » du lab :

| Segment | Adaptateur VirtualBox | Plage |
|---|---|---|
| WAN (pfSense) | Accès par pont | DHCP, réseau physique de la box |
| Kali | Accès par pont | DHCP, même réseau physique |
| LAN interne (LabNet_dmz) | Réseau interne | 192.168.1.0/24 |
| pfSense LAN | Réseau interne | 192.168.1.1 (DHCP .100–.199) |
| Ubuntu | Réseau interne | 192.168.1.101 (DHCP) |

1. **pfSense** — Adaptateur 1 : Accès par pont (carte physique). Adaptateur 2 : Réseau interne, nom `LabNet_dmz` (le nom doit être identique, au caractère près, sur toutes les VM internes).

   ![Onglet Réseau de la VM pfSense, adaptateurs 1 et 2](pfsense-network-settings.png)

2. **Kali** — Adaptateur 1 uniquement : Accès par pont, même carte physique.

   ![Onglet Réseau de la VM Kali](kali-network-settings.png)

3. **Ubuntu** — Adaptateur 1 uniquement : Réseau interne, `LabNet_dmz`.

   ![Onglet Réseau de la VM Ubuntu](ubuntu-network-settings.png)

> **⚠️ Piège rencontré** — Sous Windows, activer Hyper-V (pour les VM Windows 10/Server) change le point d'accrochage réseau de VirtualBox : la carte physique n'apparaît plus directement dans « Accès par pont », remplacée par une carte `Hyper-V Virtual Ethernet Adapter`. Il faut alors rebasculer toutes les VM VirtualBox déjà en pont sur cette nouvelle carte pour qu'elles retrouvent l'accès réseau.

## 03. Installation de pfSense

1. Démarrer la VM sur l'ISO, laisser l'installation par défaut, retirer l'ISO avant le redémarrage.

2. Au premier boot, assigner les interfaces : `vtnet0` (ou `em0`) = WAN, `vtnet1` (`em1`) = LAN.

   ![Écran console récapitulatif WAN/LAN après assignation](pfsense-interface-assignment.png)

3. **Accès à l'interface web depuis le WAN** (nécessaire une seule fois, pour la configuration initiale) : entrer dans le shell console (option 8), taper `pfctl -d` pour désactiver temporairement le pare-feu, se connecter en HTTPS sur l'IP WAN avec `admin / pfsense`, puis créer immédiatement une règle d'autorisation GUI avant de réactiver le pare-feu avec `pfctl -e`.

   ```
   Firewall ▸ Rules ▸ WAN ▸ Add
     Action      : Pass
     Protocol    : TCP
     Source      : Network — ton-réseau-local/24
     Destination : WAN address
     Dest. Port  : HTTPS (443)
   ```

   ![Règle d'autorisation d'accès GUI depuis le WAN](pfsense-wan-gui-rule.png)

4. **Réseaux réservés** — si ton WAN obtient une IP privée (double NAT, fréquent avec une box 4G/LTE), pfSense bloque ce trafic par défaut. Décocher *Interfaces ▸ WAN ▸ Reserved Networks ▸ Block private networks / Block bogon networks*.

5. **DHCP LAN** : *Services ▸ DHCP Server ▸ LAN*, plage 192.168.1.100–192.168.1.199, DNS = celui de ta box.

## 04. Scénario 1 — Déni de service (DoS) & mitigation

Démonstration d'un flood ICMP puis SYN depuis Kali vers Ubuntu, observé et bloqué côté pfSense.

### Règle d'autorisation Kali → Ubuntu

Route statique sur Kali vers le réseau interne, via l'IP WAN de pfSense :

```bash
sudo ip route add 192.168.1.0/24 via <IP_WAN_PFSENSE>
```

Règle pfSense autorisant Kali à atteindre Ubuntu :

```
Firewall ▸ Rules ▸ WAN ▸ Add
  Action: Pass | Protocol: Any
  Source: <IP Kali> | Destination: <IP Ubuntu>
```

![Règle WAN autorisant Kali vers Ubuntu](pfsense-wan-rule-kali-ubuntu.png)

![Ping réussi de Kali vers Ubuntu, confirmant route et règle](kali-to-ubuntu-ping-success.png)

### Observer le flood (tshark — Ubuntu Server sans interface graphique)

> **Pourquoi tshark et pas Wireshark** — Sur un Ubuntu Server (sans environnement de bureau), l'interface graphique de Wireshark ne peut pas s'afficher. `tshark` est son équivalent en ligne de commande — et c'est aussi l'outil qu'on utiliserait sur un vrai serveur en production.

```bash
sudo apt install -y tshark
sudo tshark -i <interface> -f icmp
```

Depuis Kali, dans un autre terminal :

```bash
sudo hping3 -1 --flood 192.168.1.101          # flood ICMP
sudo hping3 --flood -S -p 80 192.168.1.101    # flood SYN (variante)
```

![tshark capturant le flot massif de paquets ICMP pendant l'attaque](tshark-icmp-flood-attack.png)

![tshark capturant le flood SYN sur le port 80](tshark-syn-flood-attack.png)

### Bloquer l'attaque

```
Firewall ▸ Rules ▸ WAN ▸ Add (en HAUT de la liste)
  Action: Block | Protocol: Any
  Source: <IP Kali> | Destination: <IP Ubuntu>
  Log: activé | Description: "Block Kali DoS"
```

L'ordre compte : pfSense évalue les règles de haut en bas et s'arrête à la première correspondance — la règle de blocage doit donc être au-dessus de la règle d'autorisation.

![tshark ne reçoit plus aucun paquet ICMP pendant l'attaque](tshark-attack-blocked.png)

![Status - System Logs - Firewall, entrées de blocage](pfsense-firewall-logs-block.png)

![Flood SYN également bloqué, 0 paquet reçu](tshark-syn-flood-blocked.png)

![Logs pfSense confirmant le blocage du SYN flood](pfsense-logs-syn-flood-blocked.png)

## 05. Scénario 2 — Reconnaissance & brute-force SSH

Avant de rejouer ce scénario, désactiver temporairement la règle « Block Kali DoS » sur pfSense (elle bloquerait aussi ce trafic) — à réactiver une fois le scénario terminé.

### Scan Nmap

```bash
sudo nmap -sS -sV -O -p- 192.168.1.101
```

![Résultat du scan : ports ouverts, version SSH, OS détecté](nmap-scan-results.png)

### Brute-force SSH (Hydra)

> **⚠️ Compte de démonstration** — Ne jamais tester un brute-force sur un compte réel. Créer un compte dédié avec un mot de passe volontairement faible : `sudo adduser labuser` (mot de passe : par ex. `password123`).

```bash
hydra -l labuser -P password.txt ssh://192.168.1.101
```

Attention à la casse des options : `-P` majuscule pour une liste de mots de passe (fichier), `-p` minuscule pour un mot de passe unique littéral.

![Hydra ayant trouvé le mot de passe du compte de test](hydra-ssh-bruteforce-success.png)

### Détection & mitigation (fail2ban)

```bash
sudo grep 'Failed password' /var/log/auth.log | tail -20

sudo apt install -y fail2ban
sudo nano /etc/fail2ban/jail.local
```

```ini
[sshd]
enabled = true
port = ssh
maxretry = 3
bantime = 600
findtime = 120
```

```bash
sudo systemctl restart fail2ban
sudo fail2ban-client status sshd
```

![IP de Kali listée dans la section Banned IP list](fail2ban-ssh-ban-status.png)

![Connexion SSH refusée depuis Kali après le bannissement](ssh-blocked-after-fail2ban.png)

## 06. Scénario 3 — Injection SQL & protection WAF

### Déployer DVWA (application vulnérable, via Docker)

```bash
curl -fsSL https://get.docker.com -o get-docker.sh && sudo sh get-docker.sh
sudo usermod -aG docker $USER

docker run -d --name dvwa -p 8080:80 vulnerables/web-dvwa
```

Connexion DVWA : `admin / password`. Mettre le niveau de sécurité sur **Low** (onglet DVWA Security) pour que la vulnérabilité soit exploitable facilement.

![Page SQL Injection de DVWA, formulaire vide](dvwa-sqli-page.png)

### Exploitation

Manuelle, dans le champ User ID :

```
1' OR '1'='1
```

![La requête retourne tous les utilisateurs au lieu d'un seul](dvwa-manual-sqli-success.png)

Automatisée, avec sqlmap :

```bash
sqlmap -u "http://192.168.1.101:8080/vulnerabilities/sqli/?id=1&Submit=Submit" \
  --cookie="PHPSESSID=<valeur>; security=low" --dbs --batch
```

![Bases de données découvertes automatiquement par sqlmap](sqlmap-database-enumeration.png)

### Mise en place du WAF (Nginx + ModSecurity + OWASP CRS)

> **Choix technique** — SafeLine WAF (mentionné dans le guide de départ) a été remplacé par Nginx + ModSecurity suite à des coupures réseau répétées vers les serveurs de son éditeur. ModSecurity est de toute façon un standard largement reconnu de l'industrie et s'installe entièrement via les dépôts Ubuntu.

```bash
sudo apt install -y nginx libnginx-mod-http-modsecurity modsecurity-crs
sudo sed -i 's/SecRuleEngine DetectionOnly/SecRuleEngine On/' /etc/nginx/modsecurity.conf

sudo mkdir -p /etc/nginx/modsec
sudo nano /etc/nginx/modsec/main.conf
```

```apache
Include /etc/nginx/modsecurity.conf
Include /etc/modsecurity/crs/crs-setup.conf
Include /usr/share/modsecurity-crs/rules/*.conf
```

Reverse-proxy devant DVWA, sur le port 8081 :

```bash
sudo nano /etc/nginx/sites-available/dvwa-waf
```

```nginx
server {
    listen 8081;
    modsecurity on;
    modsecurity_rules_file /etc/nginx/modsec/main.conf;

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

```bash
sudo ln -s /etc/nginx/sites-available/dvwa-waf /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

![Accès normal à DVWA via le port 8081, à travers le WAF](dvwa-through-waf-normal-access.png)

### Vérifier le blocage

Rejouer la même injection (`1' OR '1'='1`) sur le port 8081 — ModSecurity doit renvoyer une erreur 403 avant même d'atteindre DVWA.

![Erreur 403 Forbidden renvoyée par ModSecurity](modsecurity-sqli-blocked.png)

![Journal d'audit ModSecurity détaillant la détection d'injection SQL](modsecurity-audit-log-sqli.png)

## 07. Scénario 4 — Supervision avec Wazuh

Pour simplifier le déploiement multi-OS (Ubuntu, Windows 10, Windows Server), pfSense a été éteint pour cette partie et toutes les machines rebasculées en accès direct par pont — voir la note d'architecture en introduction.

### Déploiement du serveur Wazuh (OVA)

1. Importer l'OVA dans VirtualBox (Fichier ▸ Importer un appliance virtuel).
2. Adaptateur 1 en Accès par pont, même réseau physique que les autres machines.
3. Prévoir au minimum **8 Go de RAM et 4 vCPU** — l'indexer (OpenSearch) est gourmand et un sous-dimensionnement provoque des timeouts de démarrage.
4. Démarrer la VM, noter l'IP et les identifiants du dashboard affichés en console.

   ![Écran de bienvenue console avec URL et identifiants](wazuh-boot-console.png)

### Déployer un agent Linux (Ubuntu)

Dashboard Wazuh ▸ Endpoints ▸ Deploy new agent ▸ choisir le paquet (DEB amd64), renseigner l'adresse du **serveur Wazuh** (pas celle de la machine à surveiller), nommer l'agent, puis copier/coller les commandes générées sur la machine Ubuntu.

```bash
sudo systemctl status wazuh-agent
```

![Agent Ubuntu listé Active dans le dashboard](wazuh-linux-agent-active.png)

### Déployer un agent Windows & activer le FIM

Même procédure (package MSI) pour Windows 10 et Windows Server. Pour surveiller un dossier en temps réel (File Integrity Monitoring), éditer `ossec.conf` sur l'agent :

```xml
<syscheck>
    <disabled>no</disabled>
    ...
    <directories realtime="yes" report_changes="yes" check_all="yes">C:\Users\<user>\Documents</directories>
</syscheck>
```

> **⚠️ Piège rencontré** — Deux erreurs classiques : (1) oublier de passer `<disabled>` à `no` au niveau du module (le nouveau dossier ajouté en dessous ne sert à rien si le module entier reste désactivé) ; (2) une syntaxe XML invalide (attributs sans signe `=`) empêche l'agent de charger correctement sa configuration.

Redémarrer le service puis vérifier les logs pour confirmer la prise en compte :

```powershell
# PowerShell (admin)
Restart-Service -Name wazuh-agent
Get-Content "C:\Program Files (x86)\ossec-agent\ossec.log" -Tail 50
```

Les résultats apparaissent dans le dashboard, module **Integrity Monitoring** (pas dans « Events » générique).

## 08. Dépannage réseau — le problème de MTU

Rencontré pendant le lab : Ubuntu arrivait à résoudre les noms de domaine (DNS fonctionnel) mais aucune connexion TCP n'aboutissait (`apt`, `curl` en timeout). Diagnostic et correction :

1. Mesurer le MTU réel du lien (box 4G/LTE notamment, souvent inférieur à 1500) depuis l'hôte physique :

   ```powershell
   # Windows
   ping -f -l 1472 8.8.8.8   # -f = Don't Fragment, -l = taille du paquet
   ```

   Redescendre progressivement la taille jusqu'à trouver la valeur maximale qui passe.

2. Appliquer une valeur sûre sur les interfaces pfSense : *Interfaces ▸ WAN* et *Interfaces ▸ LAN*, champ MTU (ex. 1400).

3. Ajuster le MSS clamping : *System ▸ Advanced ▸ Firewall & NAT*, « Firewall Maximum MSS » = MTU − 40 (ex. 1360).

> **✅ Résultat** — Après ajustement, les connexions TCP aboutissent normalement (`curl` renvoyant une vraie réponse HTTP au lieu d'un timeout). Un bon réflexe de diagnostic réseau à retenir : un ping qui passe mais une connexion TCP qui échoue évoque presque toujours un problème de MTU/fragmentation plutôt qu'un blocage pur et simple.

## 09. Conclusion & pistes d'extension

Ce lab couvre, de bout en bout : filtrage périmétrique (pfSense), déni de service et sa mitigation, reconnaissance et brute-force avec détection/réponse automatisée (fail2ban), sécurité applicative avec un WAF (Nginx/ModSecurity, OWASP CRS), et supervision centralisée (Wazuh, FIM, agents multi-OS).

Pistes pour aller plus loin :

- Connecter un agent IA local (LLM via LM Studio/AnythingLLM) au serveur Wazuh via le protocole MCP, pour assister le tri et l'analyse des alertes sans envoyer de données vers un service externe.
- Ajouter des règles de corrélation Wazuh personnalisées déclenchées par les scénarios 1 à 3 (DoS, brute-force, SQLi) pour une détection de bout en bout.
- Remettre les machines Wazuh/Windows derrière pfSense pour unifier l'architecture du lab.

---

*Guide rédigé à partir d'un lab personnel. Les adresses IP, mots de passe de démonstration et noms d'hôtes sont propres à cet environnement et à remplacer par les tiens.*
