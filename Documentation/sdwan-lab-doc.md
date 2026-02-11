# LAB SD-WAN / SLA-STORMSHIELD
## Documentation Technique Complète

---

## 📋 Table des matières

1. [Vue d'ensemble](#vue-densemble)
2. [Architecture réseau](#architecture-réseau)
3. [Plan d'adressage](#plan-dadressage)
4. [Configuration SD-WAN](#configuration-sd-wan)
5. [Configuration des routeurs](#configuration-des-routeurs)
6. [Tests et validation](#tests-et-validation)
7. [Scénarios de failover](#scénarios-de-failover)
8. [Troubleshooting](#troubleshooting)

---

## 🎯 Vue d'ensemble

### Objectif du lab

Ce laboratoire démontre la mise en œuvre d'une solution **SD-WAN avec monitoring SLA** utilisant des pare-feux Stormshield pour assurer la haute disponibilité et le basculement automatique entre plusieurs liens WAN.

### Composants principaux

- **2 sites distants** (Site 1 et Site 2) équipés de pare-feux Stormshield
- **4 routeurs Cisco** formant un WAN core avec RIPv2
- **Liens redondants** avec monitoring SLA actif
- **Basculement automatique** Main/Backup basé sur ICMP
- **Infrastructure de test** incluant Kali Linux et Metasploitable

### Cas d'usage

✅ Redondance WAN multi-opérateur  
✅ Basculement automatique en cas de défaillance  
✅ Monitoring de qualité de service (SLA)  
✅ Tests de sécurité inter-sites  

---

## 🏗️ Architecture réseau

### Topologie générale

```
Site 1                    WAN CORE (RIPv2)                    Site 2
┌─────────────┐          ┌──────────────┐          ┌─────────────┐
│Stormshield-1│          │              │          │Stormshield-2│
│             │───MAIN───│  R1      R3  │───MAIN───│             │
│  40.40.x.x  │          │              │          │  50.50.x.x  │
│             │─BACKUP───│  R2      R4  │─BACKUP───│             │
└─────────────┘          └──────────────┘          └─────────────┘
      │                   30.30.1.0/24                     │
      │                                                     │
   PC3 + Meta-1                                        PC1 + PC4
```

### Zones de sécurité

| Zone            | Description         | Réseau                     |
|-----------------|---------------------|----------------------------|
| **Site 1 LAN**  | Réseau local Site 1 | 40.40.1.0/24, 40.40.2.0/24 |
| **Site 2 LAN**  | Réseau local Site 2 | 50.50.1.0/24, 50.50.2.0/24 |
| **WAN Core**    | Backbone RIPv2      | 30.30.1.0/24               |
| **WAN Site 1**  | Liens vers R1/R2    | 10.10.1.0/24, 10.10.2.0/24 |
| **WAN Site 2**  | Liens vers R3/R4    | 20.20.1.0/24, 20.20.2.0/24 |
| **DMZ Pentest** | Zone Kali/PC2       | 60.60.1.0/24               |

---

## 📊 Plan d'adressage

### Site 1 - Stormshield-1

| Interface | Adresse IP     | Passerelle    | Destination      | Rôle        |
|-----------|----------------|---------------|------------------|-------------|
| **e0**    | DHCP (Cloud2)  | -             | Internet         | WAN externe |
| **e1**    | 10.10.1.0/24   | 10.10.1.254   | R1               | WAN Main    |
| **e2**    | 10.10.2.1/24   | 10.10.2.254   | R2               | WAN Backup  |
| **e3**    | 40.40.2.254/24 | -             | PC3              | LAN         |
| **e4**    | 40.40.1.254/24 | -             | Metasploitable-1 | LAN         |

### Site 2 - Stormshield-2

| Interface | Adresse IP     | Passerelle    | Destination | Rôle        |
|-----------|----------------|---------------|-------------|-------------|
| **e0**    | DHCP (Cloud1)  | -             | Internet    | WAN externe |
| **e1**    | 20.20.1.1/24   | 20.20.1.254   | R3          | WAN Main    |
| **e2**    | 20.20.2.1/24   | 20.20.2.254   | R4          | WAN Backup  |
| **e3**    | 50.50.1.254/24 | -             | PC1 (usr)   | LAN         |
| **e4**    | 50.50.2.254/24 | -             | PC4 (srv)   | LAN         |

### Routeurs WAN Core

#### R1 (Main Site 1)
```
g1/0: 10.10.1.254/24  → Stormshield-1
g2/0: 30.30.1.1/24    → Switch1 (RIPv2)
```

#### R2 (Backup Site 1)
```
g1/0: 10.10.2.254/24  → Stormshield-1
g2/0: 30.30.1.2/24    → Switch1 (RIPv2)
```

#### R3 (Main Site 2)
```
g1/0: 20.20.1.254/24  → Stormshield-2
g2/0: 30.30.1.3/24    → Switch1 (RIPv2)
```

#### R4 (Backup Site 2)
```
g1/0: 20.20.2.254/24  → Stormshield-2
g2/0: 30.30.1.4/24    → Switch1 (RIPv2)
g3/0: 60.60.1.254/24  → Switch2 (DMZ Pentest)
```

### Équipements terminaux

| Équipement              | Adresse IP   | Passerelle  | Description          |
|-------------------------|--------------|-------------|----------------------|
| **PC3**                 | 40.40.2.1/24 | 40.40.2.254 | VPCS Site 1          |
| **Metasploitable-1**    | 40.40.1.1/24 | 40.40.1.254 | Cible vulnérable     |
| **PC1**                 | 50.50.1.1/24 | 50.50.1.254 | VPCS Site 2 (user)   |
| **PC4**                 | 50.50.2.1/24 | 50.50.2.254 | VPCS Site 2 (server) |
| **PC2**                 | 60.60.1.2/24 | 60.60.1.254 | VPCS DMZ             |
| **Kali-1**              | 60.60.1.1/24 | 60.60.1.254 | Plateforme pentest   |

---

## ⚙️ Configuration SD-WAN

### Stormshield-1 - Configuration SLA

#### Paramètres de monitoring

```
Object name: sd-wan-router
Detection method: ICMP
Timeout: 1 seconde
Interval: 2 secondes
Failures before degradation: 2
```

#### Seuils SLA

| Métrique                | Seuil | Impact                 |
|-------------------------|-------|------------------------|
| **Latency**             | 60 ms | Dégradation si dépassé |
| **Jitter**              | 25 ms | Dégradation si dépassé |
| **Packet loss rate**    | 5%    | Dégradation si dépassé |
| **Unavailability rate** | 5%    | Dégradation si dépassé |

#### Routes configurées

| Gateway       | IP          | Poids | Statut | Rôle   |
|---------------|-------------|-------|--------|--------|
| **router-r1** | 10.10.1.254 | 1     | Active | Main   |
| **router-r2** | 10.10.2.254 | 1     | Standby| Backup |

#### Politique de basculement

- **Load balancing**: No load balancing
- **Enable backup gateways**: When all gateways cannot be reached
- **If no gateways available**: Do not route

### Stormshield-2 - Configuration SLA

#### Paramètres de monitoring

```
Object name: sd-wan-router
Detection method: ICMP
Timeout: 1 seconde
Interval: 3 secondes
Failures before degradation: 3
```

#### Seuils SLA (identiques)

| Métrique                | Seuil |
|-------------------------|-------|
| **Latency**             | 60 ms |
| **Jitter**              | 25 ms |
| **Packet loss rate**    | 5%    |
| **Unavailability rate** | 5%    |

#### Routes configurées

| Gateway       | IP          | Poids | Statut  | Rôle   |
|---------------|-------------|-------|---------|--------|
| **router-r3** | 20.20.1.254 | 1     | Active  | Main   |
| **router-r4** | 20.20.2.254 | 1     | Standby | Backup |

#### Politique de basculement

- **Load balancing**: By connection
- **Enable backup gateways**: When all gateways cannot be reached
- **If no gateways available**: Default route

### Différences entre Storm-1 et Storm-2

| Paramètre                       | Stormshield-1 | Stormshield-2 |
|---------------------------------|---------------|---------------|
| **Interval**                    | 2s            | 3s            |
| **Failures before degradation** | 2             | 3             |
| **Load balancing**              | Non           | By connection |
| **No gateway action**           | Do not route  | Default route |

---

## 🔧 Configuration des routeurs

### R1 - Configuration complète

```cisco
hostname R1
!
interface GigabitEthernet1/0
 ip address 10.10.1.254 255.255.255.0
 negotiation auto
!
interface GigabitEthernet2/0
 ip address 30.30.1.1 255.255.255.0
 negotiation auto
!
router rip
 version 2
 timers basic 5 15 15 30
 redistribute static metric 2
 network 10.0.0.0
 network 30.0.0.0
 no auto-summary
!
ip route 0.0.0.0 0.0.0.0 GigabitEthernet2/0
ip route 40.40.1.0 255.255.255.0 10.10.1.1
ip route 40.40.2.0 255.255.255.0 10.10.1.1
```

### R2 - Configuration complète

```cisco
hostname R2
!
interface GigabitEthernet1/0
 ip address 10.10.2.254 255.255.255.0
 negotiation auto
!
interface GigabitEthernet2/0
 ip address 30.30.1.2 255.255.255.0
 negotiation auto
!
router rip
 version 2
 timers basic 5 15 15 30
 network 10.0.0.0
 network 30.0.0.0
 no auto-summary
!
ip route 0.0.0.0 0.0.0.0 GigabitEthernet2/0
ip route 40.40.1.0 255.255.255.0 10.10.2.1
ip route 40.40.2.0 255.255.255.0 10.10.2.1
```

### R3 - Configuration complète

```cisco
hostname R3
!
interface GigabitEthernet1/0
 ip address 20.20.1.254 255.255.255.0
 negotiation auto
!
interface GigabitEthernet2/0
 ip address 30.30.1.3 255.255.255.0
 negotiation auto
!
router rip
 version 2
 redistribute static metric 2
 network 20.0.0.0
 network 30.0.0.0
 no auto-summary
!
ip route 0.0.0.0 0.0.0.0 GigabitEthernet2/0
ip route 50.50.1.0 255.255.255.0 20.20.1.1
ip route 50.50.2.0 255.255.255.0 20.20.1.1
```

### R4 - Configuration complète

```cisco
hostname R4
!
interface GigabitEthernet1/0
 ip address 20.20.2.254 255.255.255.0
 ip rip advertise 5
 negotiation auto
!
interface GigabitEthernet2/0
 ip address 30.30.1.4 255.255.255.0
 negotiation auto
!
interface GigabitEthernet3/0
 ip address 60.60.1.254 255.255.255.0
 ip rip advertise 5
 negotiation auto
!
router rip
 version 2
 timers basic 5 15 15 30
 redistribute static metric 2
 network 20.0.0.0
 network 30.0.0.0
 network 60.0.0.0
 no auto-summary
!
ip route 0.0.0.0 0.0.0.0 GigabitEthernet2/0
ip route 50.50.1.0 255.255.255.0 20.20.2.1
ip route 50.50.2.0 255.255.255.0 20.20.2.1
```

### Points clés de la configuration RIPv2

- **Version**: RIPv2 pour support VLSM
- **Timers optimisés**: 5/15/15/30 pour convergence rapide
- **Redistribution**: Routes statiques vers les LANs
- **No auto-summary**: Préservation des sous-réseaux
- **Default route**: Via g2/0 vers le WAN core

---

## ✅ Tests et validation

### 1. Vérification de base

#### Sur Stormshield-1
```bash
# Vérifier l'état SD-WAN
MONITOR / SD-WAN → REAL-TIME
# Statut attendu:
# router-r1: Active, SLA Good
# router-r2: Standby, SLA Good
```

#### Sur R1
```cisco
R1# show ip interface brief
R1# show ip route rip
R1# show ip rip database
R1# ping 30.30.1.2  # Vers R2
R1# ping 30.30.1.3  # Vers R3
```

### 2. Tests de connectivité inter-sites

#### Depuis PC3 (Site 1)
```bash
# Vers Site 2
ping 50.50.1.1    # PC1
ping 50.50.2.1    # PC4

# Traceroute pour vérifier le chemin
traceroute 50.50.1.1
# Chemin attendu: 40.40.2.254 → 10.10.1.254 → 30.30.1.x → 20.20.1.254 → 50.50.1.1
```

#### Depuis Kali-1
```bash
# Vers Site 1
ping 40.40.1.1    # Metasploitable-1
ping 40.40.2.1    # PC3

# Vers Site 2
ping 50.50.1.1    # PC1
ping 50.50.2.1    # PC4
```

### 3. Validation SLA

#### Mesure de latence
```bash
# Depuis Stormshield-1
ping -c 100 10.10.1.254  # Vers R1
ping -c 100 10.10.2.254  # Vers R2

# Statistiques attendues:
# - Latence moyenne < 10ms (LAN)
# - Jitter < 5ms
# - Packet loss: 0%
```

#### Surveillance continue
```bash
# Monitoring temps réel sur Stormshield
MONITOR / SD-WAN → REAL TIME GRAPH
# Observer pendant 5-10 minutes
```

---

## 🔄 Scénarios de failover

### Scénario 1: Panne du lien Main (R1)

#### Simulation
```cisco
# Sur R1
R1(config)# interface GigabitEthernet1/0
R1(config-if)# shutdown
```

#### Comportement attendu

| Temps    | Événement                      | État                     |
|----------|--------------------------------|--------------------------|
| **T+0**  | Interface R1 g1/0 down         | -                        |
| **T+2s** | 1er ICMP timeout sur Storm-1   | -                        |
| **T+4s** | 2ème ICMP timeout              | Dégradation détectée     |
| **T+5s** | Basculement vers R2            | router-r2 devient Active |
| **T+6s** | Trafic rerouté via 10.10.2.254 | Connectivité restaurée   |

#### Vérification
```bash
# Sur Stormshield-1
MONITOR / SD-WAN
# router-r1: Inactive/Down
# router-r2: Active, SLA Good

# Test de connectivité
ping 50.50.1.1  # Doit fonctionner via R2
```

#### Restauration
```cisco
# Sur R1
R1(config)# interface GigabitEthernet1/0
R1(config-if)# no shutdown
```

Après ~6-8 secondes, router-r1 redevient Active.

### Scénario 2: Dégradation SLA (latence)

#### Simulation avec netem (Linux)
```bash
# Sur l'interface entre Storm-1 et R1
tc qdisc add dev eth0 root netem delay 100ms

# Ajouter du jitter
tc qdisc change dev eth0 root netem delay 100ms 30ms
```

#### Comportement attendu
- Latence mesurée > 60ms (seuil)
- Jitter > 25ms (seuil)
- SLA Status passe à "Degraded"
- Stormshield peut basculer selon la politique

#### Nettoyage
```bash
tc qdisc del dev eth0 root
```

### Scénario 3: Panne totale Site 1 WAN

#### Simulation
```cisco
# Sur R1 et R2
R1(config)# interface GigabitEthernet1/0
R1(config-if)# shutdown
R2(config)# interface GigabitEthernet1/0
R2(config-if)# shutdown
```

#### Comportement Stormshield-1
- Tous les gateways injoignables
- Action selon config: "Do not route"
- Site 1 isolé du WAN
- LAN interne reste fonctionnel

### Scénario 4: Test de charge et load balancing

#### Sur Stormshield-2 (By connection)
```bash
# Générer du trafic depuis PC1 et PC4
for i in {1..100}; do
  curl http://target-ip &
done

# Observer la répartition
MONITOR / SD-WAN → REAL TIME
# Connexions réparties entre R3 et R4 selon disponibilité
```

---

## 🔍 Troubleshooting

### Problème: SLA Status "Bad" alors que le lien fonctionne

**Causes possibles:**
1. Latence réseau trop élevée (> 60ms)
2. Perte de paquets (> 5%)
3. Congestion réseau
4. Problèmes de routage asymétrique

**Diagnostic:**
```bash
# Mesures détaillées
ping -c 100 -i 0.2 10.10.1.254
mtr 10.10.1.254

# Vérifier les compteurs
show interface GigabitEthernet1/0
# Chercher: errors, drops, overruns
```

**Solutions:**
- Ajuster les seuils SLA si environnement WAN réel
- Vérifier la bande passante disponible
- Analyser avec Wireshark les délais

### Problème: Pas de basculement lors de panne

**Vérifications:**
```bash
# 1. Configuration SD-WAN active?
MONITOR / SD-WAN → Vérifier "SD-WAN SLA: Active"

# 2. Backup gateway configuré?
Configure routing → Vérifier router-r2 présent

# 3. Politique de failover correcte?
Advanced configuration → "When all gateways cannot be reached"
```

### Problème: Routes RIPv2 non propagées

**Diagnostic:**
```cisco
# Sur R1
R1# show ip protocols
R1# show ip rip database
R1# debug ip rip
```

**Points à vérifier:**
- Version RIPv2 (pas v1)
- `no auto-summary` configuré
- Network statements corrects
- Pas de passive-interface bloquant

### Problème: PC ne peut pas joindre l'autre site

**Tests systématiques:**
```bash
# 1. Gateway local OK?
ping 40.40.2.254  # Depuis PC3

# 2. Routeur WAN joignable?
ping 10.10.1.254  # Vers R1

# 3. WAN core fonctionnel?
# Depuis Stormshield-1
ping 30.30.1.3  # Vers R3

# 4. Routeur remote OK?
ping 20.20.1.254  # Vers R3

# 5. Gateway remote OK?
ping 50.50.1.254  # Gateway Site 2

# 6. Destination finale
ping 50.50.1.1  # PC1
```

### Commandes de diagnostic utiles

#### Stormshield
```bash
# État SD-WAN temps réel
MONITOR / SD-WAN

# Logs système
LOGS / SYSTEM → Filtrer "router" ou "SLA"

# Table de routage
CONFIGURATION / SYSTEM / Routing

# Tests réseau
CONFIGURATION / OBJECTS / Network objects → Test connectivity
```

#### Routeurs Cisco
```cisco
# État général
show ip interface brief
show ip route
show ip protocols

# RIPv2 spécifique
show ip rip database
show ip route rip

# Statistiques interfaces
show interface GigabitEthernet1/0
show interface counters errors

# Debugging
debug ip routing
debug ip rip
```

---

## 📈 Métriques de performance

### Temps de basculement mesurés

| Scénario                      | Temps de détection      | Temps total de bascule | Perte de paquets |
|-------------------------------|-------------------------|------------------------|------------------|
| **Panne lien Main (R1)**      | 4s (2 timeouts × 2s)    | 5-6s                   | 4-6 paquets      |
| **SLA dégradé**               | Variable (dépend durée) | 3-5s                   | Minimale         |
| **Restauration lien**         | 2-3s (reprise ICMP)     | 6-8s                   | 0-2 paquets      |

### Overhead SLA monitoring

- **Bande passante**: ~64 bytes/ping × (1/2s ou 1/3s) = négligeable
- **Charge CPU**: < 1% sur Stormshield
- **Impact latence**: < 1ms

---

## 🎓 Exercices pratiques

### Exercice 1: Tester le failover
1. Vérifier l'état initial (R1 Active)
2. Générer du trafic continu (ping)
3. Couper R1
4. Mesurer le temps de basculement
5. Vérifier la reprise sur R2
6. Restaurer R1
7. Observer le retour automatique

### Exercice 2: Stress test SLA
1. Configurer iperf entre sites
2. Saturer la bande passante
3. Observer l'impact sur les métriques SLA
4. Noter le comportement du basculement

### Exercice 3: Pentest inter-sites
1. Depuis Kali-1, scanner Metasploitable-1
2. Analyser le routage des paquets
3. Identifier les flux traversant le WAN core
4. Tester différents chemins (via R1 ou R2)

---

## 📚 Références

### Documentation Stormshield
- Guide d'administration SD-WAN
- Configuration avancée du routage
- Monitoring SLA et métriques

### RFC et standards
- RFC 2453: RIP Version 2
- Best practices SD-WAN
- ICMP monitoring standards

### Outils utilisés
- **GNS3**: Émulation réseau
- **Cisco IOS**: 15.2
- **Stormshield**: Pare-feu nouvelle génération
- **VPCS**: Virtual PC Simulator
- **Kali Linux**: Plateforme pentest

---

## ✍️ Notes finales

### Points forts de la configuration

✅ Redondance complète des liens WAN  
✅ Basculement automatique vérifié  
✅ Monitoring SLA temps réel  
✅ Infrastructure de test complète  
✅ Séparation claire des zones réseau  

### Améliorations possibles

🔧 Ajouter du load balancing actif/actif  
🔧 Implémenter IPSEC entre sites  
🔧 Configurer QoS sur les routeurs  
🔧 Ajouter monitoring SNMP centralisé  
🔧 Mettre en place syslog centralisé  

### Prochaines étapes

1. Documenter les tests de charge
2. Créer des playbooks Ansible pour déploiement
3. Mettre en place Grafana pour monitoring
4. Automatiser les tests de failover

---

**Auteur**: Lab SD-WAN Stormshield  
**Date**: Novembre 2025  
**Version**: 1.0