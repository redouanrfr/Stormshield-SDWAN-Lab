# Lab SD-WAN & SLA Monitoring - Stormshield (WAN Core RIPv2)

Ce laboratoire démontre la mise en œuvre d'une solution **SD-WAN avec monitoring SLA** utilisant des pare-feux Stormshield (SNS). L'infrastructure repose sur un cœur de réseau (WAN Core) utilisant le routage dynamique pour assurer la connectivité inter-sites.

## 🌐 Cœur de Réseau & Routage
Le backbone est constitué de **4 routeurs Cisco** configurés en **RIPv2** :
- **Propagation dynamique** des préfixes réseaux entre le Site 1 et le Site 2.
- **Redondance des chemins** : Le WAN Core offre plusieurs routes (via R1 ou R2) que le SD-WAN Stormshield va piloter selon la qualité des liens.
- **Convergence** : Configuration optimisée pour une mise à jour rapide des tables de routage en cas de coupure d'un nœud du backbone.

## 🚀 Fonctionnalités SD-WAN
- **Monitoring SLA Actif** : Surveillance ICMP des sauts du WAN Core (Latence > 60ms, Jitter > 25ms).
- **Politiques de routage (PBR)** : Basculement automatique des flux critiques vers le lien de secours si le lien RIP principal se dégrade.
- **Résilience** : Temps de basculement complet observé entre 4 et 8 secondes selon le scénario.

## 🏗️ Architecture du Projet
- **/Architecture** : Schéma réseau détaillé (XML Diagrams.net) incluant le backbone RIPv2.
- **/Documentation** : 
    - [Documentation technique](Documentation/sdwan-lab-doc.md) (Plans d'adressage, configuration RIP).
    - [Rapport de tests](Documentation/tests.md) (Validation du failover).
- **/Demos** : Vidéo de démonstration des basculements en temps réel.

## 📺 Démonstration Vidéo
Une vidéo complète des tests de basculement est disponible dans le dossier `/Demos`.

- **[Lien vers la vidéo](Demos/sd-wan-stormshield.mp4)**
- **Points clés à observer** : 
    - Convergence RIPv2 lors de la coupure d'un routeur.
    - Réaction du Stormshield suite à l'injection de latence (60ms+).
    - Temps de basculement réel observé sur les pings.

## 📊 Scénarios de Validation
1. **Convergence RIPv2** : Test de reconstruction des routes après coupure d'un routeur backbone.
2. **Basculement sur latence** : Réaction du Stormshield face à une dégradation de lien sans coupure franche.
3. **Pénétration inter-sites** : Tests de flux via Kali Linux à travers le maillage RIPv2.

---
*Note : Pour des raisons de sécurité, les fichiers de configuration (secrets de fabrication) ne sont pas inclus en entier.*