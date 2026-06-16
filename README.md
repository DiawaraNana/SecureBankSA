# 🏦 SecureBank SOC Lab — Séance 1 : SIEM & Détection d'Intrusion

[![Cybersécurité](https://img.shields.io/badge/Cybersécurité-Ingénierie-red)](https://euromed.ac.ma)
[![Lab](https://img.shields.io/badge/Lab-VM-0052CC)]()
[![SOC Tier 1](https://img.shields.io/badge/SOC-Tier_1-orange)]()
[![Stack](https://img.shields.io/badge/Stack-pfSense%20%7C%20Suricata%20%7C%20Wazuh%20%7C%20Elastic-green)]()
[![License](https://img.shields.io/badge/License-EUROMED-blue)]()

> **Construire, opérer et analyser le premier niveau défensif d'un Security Operations Center à travers un scénario bancaire réaliste.**

---

## 📋 Table des matières

- [Contexte du scénario](#-contexte-du-scénario)
- [Objectifs pédagogiques](#-objectifs-pédagogiques)
- [Architecture du lab](#-architecture-du-lab)
- [Prérequis](#-prérequis)
- [Déploiement de l'infrastructure](#-déploiement-de-linfrastructure)
- [Simulation de l'attaque (Red Team)](#-simulation-de-lattaque-red-team)
- [Détection et analyse (Blue Team)](#-détection-et-analyse-blue-team)
- [Livrables attendus](#-livrables-attendus)
- [Ressources](#-ressources)
- [Auteurs](#-auteurs)

---

## 🎯 Contexte du scénario

**SecureBank SA** est une banque en ligne marocaine gérant **300 000 comptes clients**. Son infrastructure héberge :

- Un **portail web bancaire** (Apache + PHP)
- Une **API de paiement** (REST JSON)
- Un **back-office interne** (accès SSH réservé aux sysadmins)
- Une **base de données Oracle** contenant les données financières

Ce matin à **06h47**, le système de monitoring détecte des anomalies réseau sur la DMZ. La DSI active le **SOC d'urgence**. Vous êtes les analystes **Tier 1** en charge de l'investigation.

🔴 INCIDENT ACTIF
├── 06h47 — Scan réseau détecté sur DMZ
├── 07h03 — 247 tentatives SSH sur 192.168.50.10
└── 07h15 — Injection SQL suspectée sur /api/login


---

## 🎓 Objectifs pédagogiques

À l'issue de ce TP, vous serez capable de :

1. **Déployer** une architecture SOC complète (pfSense + Suricata + Wazuh + Elastic Stack)
2. **Configurer** la collecte et la normalisation des logs (ECS)
3. **Simuler** une attaque multi-phase (Recon → Brute-force → SQLi)
4. **Détecter** et **corréler** les alertes dans Kibana
5. **Créer** une règle de détection EQL (Kill Chain)
6. **Rédiger** un rapport d'incident format PICERL

---

## ️ Architecture du lab

Le lab repose sur **4 machines virtuelles** interconnectées sur un réseau interne isolé.
┌─────────────────────────────────────────────────────────────────────┐
│ LAB SECUREBANK — 4 VMs / RÉSEAU INTERNE ISOLÉ │
└─────────────────────────────────────────────────────────────────────┘
WAN/NAT (rouge) LAN 192.168.50.0/24 (Host-Only)
│ │
┌────▼─────┐ ┌─────▼──────┐
│ VM-KALI │ │ VM-pfSense │
│ Red Team │── WAN/NAT ──▶│ 192.168.50.1│
│ nmap │ │ Suricata │
│ Hydra │ │ IDS │
│ sqlmap │ ─────┬──────┘
└──────────┘ │
syslog:514 │
▼
┌──────────────────────────┐
│ VM-SIEM │
│ 192.168.50.100 │
│ Elasticsearch :9200 │
│ Kibana :5601 │
│ Wazuh-manager │
│ Filebeat │
└──────────┬───────────────┘
│
┌────────────────┼────────────────┐
│ Wazuh │ │ Filebeat
▼ ▼ ▼
┌──────────┐ ┌──────────┐ ┌──────────────┐
│ VM-WEB │ │ Analyste │ │ (autres agents)│
│192.168.50│ │ SOC │ └──────────────┘
│.10 │ │ Kibana │
│Apache │ │ :5601 │
│DVWA·SSH │ │ Vous ! │
──────────┘ └──────────┘


### 🖥️ Spécifications des VMs

| VM | OS | RAM | Rôle | Services | IP |
|----|----|-----|------|----------|-----|
| **VM-KALI** | Kali 2024 | 2 Go | 🗡️ Red Team | nmap, Hydra, sqlmap | NAT (WAN) |
| **VM-pfSense** | FreeBSD | 512 Mo | 🛡️ Firewall + IDS | pfSense, Suricata IDS | WAN: NAT / LAN: 192.168.50.1 |
| **VM-WEB** | Ubuntu 22.04 | 1 Go |  Cible | Apache, DVWA, SSH, Wazuh-agent, Filebeat | 192.168.50.10 |
| **VM-SIEM** | Ubuntu 22.04 | 4 Go |  SOC | Elasticsearch, Kibana, Wazuh-manager, Filebeat | 192.168.50.100 |

### 🔀 Flux réseau

| Flux | Couleur | Description |
|------|---------|-------------|
| **Trafic LAN** | 🔵 Bleu | Communications normales entre VMs du LAN |
| **Trafic attaquant** | 🔴 Rouge pointillé | Flux malveillants depuis VM-KALI via pfSense |
| **Flux logs/agents** | 🟢 Vert pointillé | Remontée syslog (UDP:514) et agents Wazuh/Filebeat vers VM-SIEM |

---

## 📦 Prérequis

### Matériel

- **Hôte** : Machine avec minimum **16 Go RAM**, processeur 4 cœurs, 100 Go disque
- **Hyperviseur** : VMware Workstation / VirtualBox / Proxmox
- **Réseau** : Configuration Host-Only pour le LAN 192.168.50.0/24

### Logiciels

- [pfSense 2.7+](https://www.pfsense.org/)
- [Kali Linux 2024](https://www.kali.org/)
- [Ubuntu Server 22.04 LTS](https://ubuntu.com/download/server)
- [DVWA](https://github.com/digininja/DVWA) (à installer sur VM-WEB)

### Connaissances

- Notions TCP/IP et routage
- Commandes Linux de base
- Compréhension des protocoles HTTP, SSH, DNS

---

## 🚀 Déploiement de l'infrastructure

### Étape 1 — pfSense + Suricata (30 min)


# 1. Installer pfSense sur VM avec 2 interfaces réseau
# 2. Accéder à la WebGUI : http://192.168.50.1 (admin/pfsense)
# 3. Installer Suricata : System → Package Manager → Suricata

Configuration Suricata :

Services → Suricata → Global Settings
├── ETOpen Rules          : ✓ Enable
├── Update Interval       : 1 day
├── Interface             : WAN
├── EVE JSON Log          : ✓ Enable
├── Block Offenders       : ✗ NON (IDS passif)
└── Rules activées :
    ├── emerging-scan         # Détection Nmap
    ├── emerging-exploit      # Détection exploits
    ├── emerging-web_server   # Attaques web
    ├── emerging-sql          # SQLi
    ── emerging-dos          # DDoS
Vérification :
```bash
# Sur pfSense (SSH)
ls -lh /var/log/suricata/suricata_WAN*/eve.json
# → Le fichier doit exister et être non vide
```
###  Étape 2 — Stack Elastic sur VM-SIEM (25 min)
# Dépôt Elastic 8.x
```
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch \
  | gpg --dearmor -o /usr/share/keyrings/elastic-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/elastic-keyring.gpg] \
  https://artifacts.elastic.co/packages/8.x/apt stable main" \
  | tee /etc/apt/sources.list.d/elastic-8.x.list

apt update && apt install -y elasticsearch kibana filebeat
```
Configuration Elasticsearch (/etc/elasticsearch/elasticsearch.yml) :

network.host: 0.0.0.0
http.port: 9200
xpack.security.enabled: false   # ⚠️ Lab uniquement
cluster.name: securebank-soc

Configuration Kibana (/etc/kibana/kibana.yml) :

server.host: "0.0.0.0"
server.port: 5601
elasticsearch.hosts: ["http://localhost:9200"]
```
systemctl enable --now elasticsearch kibana
sleep 60
curl -s http://localhost:9200/_cluster/health?pretty
```
# → "status" : "green"

### Étape 3 — Filebeat : collecte Suricata + syslog (20 min)
Sur pfSense : Status → System Logs → Settings

Enable Remote Logging : ✓
Remote Log Server     : 192.168.50.100
Remote Syslog Port    : 514
Syslog Contents       : ✓ Firewall Events, General

Sur VM-SIEM (/etc/filebeat/filebeat.yml) :
filebeat.modules:
  - module: suricata
  - module: apache
  - module: nginx
  - module: system

filebeat.inputs:
  - type: syslog
    protocol.udp.host: "0.0.0.0:514"
    tags: ["pfsense", "securebank-fw"]

output.elasticsearch:
  hosts: ["localhost:9200"]
  index: "securebank-%{[agent.name]}-%{+yyyy.MM.dd}"

filebeat modules enable suricata apache nginx system
filebeat setup
systemctl enable --now filebeat

# Vérification
curl "http://localhost:9200/_cat/indices?v"
# → securebank-filebeat-* doit avoir docs.count > 0

### Étape 4 — Wazuh Manager + Agent (20 min)
Sur VM-SIEM (Wazuh Manager) :
```
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | apt-key add -
echo "deb https://packages.wazuh.com/4.x/apt/ stable main" \
  | tee /etc/apt/sources.list.d/wazuh.list
apt update && apt install -y wazuh-manager
systemctl enable --now wazuh-manager
```
Intégration Wazuh → Elasticsearch (/var/ossec/etc/ossec.conf) :
<integration>
  <name>elastic</name>
  <hook_url>http://localhost:9200</hook_url>
  <alert_format>json</alert_format>
</integration>

Sur VM-WEB (Agent Wazuh) :
```
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | apt-key add -
echo "deb https://packages.wazuh.com/4.x/apt/ stable main" \
  | tee /etc/apt/sources.list.d/wazuh.list
apt update

WAZUH_MANAGER="192.168.50.100" \
WAZUH_AGENT_NAME="web01-securebank" \
  apt install -y wazuh-agent

systemctl enable --now wazuh-agent
```
Vérification (sur VM-SIEM) :
```
/var/ossec/bin/agent_control -l
```
# → ID:001  Name: web01-securebank  Status: Active ✓

### ⚔️ Simulation de l'attaque (Red Team)
⚖️ Rappel éthique et légal : Ces commandes ciblent exclusivement les VMs de votre lab personnel isolé. Exécuter ces outils sur des systèmes tiers sans autorisation constitue une infraction pénale (Code pénal marocain, loi 07-03).

### Phase 1 — Reconnaissance (06h47)

# VM-KALI → Suricata (emerging-scan)
```
nmap -sV -sC -O -p- 192.168.50.10
nikto -h http://192.168.50.10
```
MITRE : T1046 — Network Service Scanning
### Phase 2 — Brute-force SSH (07h03)
# VM-KALI → Wazuh (Règle 5712)

```
hydra -l admin -P /usr/share/wordlists/rockyou.txt \
      -t 4 -f ssh://192.168.50.10
```
🎯 MITRE : T1110.001 — Brute Force: Password Guessing
### Phase 3 — Injection SQL (07h15)

# VM-KALI → Suricata (ET SQL)
# Pré-requis : DVWA installé sur VM-WEB, security level = "low"
```
sqlmap -u "http://192.168.50.10/dvwa/vulnerabilities/sqli/?id=1&Submit=Submit" \
       --cookie="security=low;PHPSESSID=VOTRE_SESSION" \
       --dbs --batch
```
MITRE : T1190 — Exploit Public-Facing Application

🔍 Détection et analyse (Blue Team)
Vérification dans Kibana
Accéder à http://192.168.50.100:5601 → Security → Alerts

Requêtes KQL utiles

# Tous les événements de l'attaquant
source.ip: "185.220.101.47"

# Alertes Suricata critiques
event.module: suricata AND suricata.eve.alert.severity: (1 OR 2)

# Brute-force SSH (Wazuh)
event.module: wazuh AND rule.groups: authentication_failed

# Requêtes POST suspectes sur l'API
url.path: "/api/login" AND http.request.method: POST AND http.response.status_code: 400

Création de la règle EQL (Kill Chain)
Kibana → Security → Rules → Create → Event Correlation

sequence by source.ip with maxspan=30m
  /* Phase 1 & 2 : brute-force SSH */
  [authentication where event.outcome == "failure" and destination.port == 22]
  [authentication where event.outcome == "failure" and destination.port == 22]
  [authentication where event.outcome == "failure" and destination.port == 22]

  /* Phase 3 : tentative SQLi */
  [network where event.module == "suricata" and suricata.eve.alert.category like "*sql*"]

Paramètres :
Name : SecureBank — Kill Chain Recon+BF+SQLi
Severity : Critical
Risk Score : 95
MITRE : T1046, T1110.001, T1190
Schedule : Every 5 min / Last 35 min

📚 Ressources
Documentation Elastic SIEM
Documentation Wazuh
Documentation Suricata
MITRE ATT&CK Framework
Emerging Threats Open Rules
Bank Al-Maghrib — Directives cybersécurité
👥 Auteurs
Cybersécurité Ingénierie — Séance 1/4
Laboratoire SOC — EUROMED Fès
Encadrant : Professeur AMAMOU AHMED
Étudiants : DIAWARA Nana
📅 Date : Mai 2026
Scénario : SecureBank SA — Incident en cours
