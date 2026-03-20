# Cours de Préparation aux Certifications Azure : Questions Techniques Approfondies

## Sommaire

1. [Différence entre NSG et ASG](#1-différence-entre-nsg-et-asg)
2. [Blocage de l'accès à une VM depuis un subnet](#2-blocage-de-laccès-à-une-vm-depuis-un-subnet)
3. [Les NSG Azure sont-ils stateful ou stateless ?](#3-les-nsg-azure-sont-ils-stateful-ou-stateless-)
4. [Différence entre Azure Firewall et NSG](#4-différence-entre-azure-firewall-et-nsg)
5. [Avantages des Resource Groups dans Azure](#5-avantages-des-resource-groups-dans-azure)
6. [Différence entre Azure User Data et Custom Data](#6-différence-entre-azure-user-data-et-custom-data)
7. [Différence entre Azure Application Gateway et Azure Load Balancer](#7-différence-entre-azure-application-gateway-et-azure-load-balancer)
8. [Explication du flux réseau dans une architecture idéale](#8-explication-du-flux-réseau-dans-une-architecture-idéale)
9. [Rôle et cas d'usage d'Azure Bastion](#9-rôle-et-cas-dusage-dazure-bastion)

---

## 1. Différence entre NSG et ASG

### Network Security Group (NSG)

Un **NSG** est un pare-feu distribué stateful qui filtre le trafic réseau à destination et en provenance des ressources Azure . Il agit comme une liste de contrôle d'accès (ACL) qui peut être associée à un **subnet** ou à une **carte réseau (NIC)** .

**Caractéristiques clés :**
- **Filtrage au niveau TCP/IP** : règles basées sur la source, la destination, le port et le protocole
- **Priorité** : les règles sont évaluées par ordre de priorité (de 100 à 4096) 
- **État (stateful)** : si le trafic entrant est autorisé, le trafic sortant de réponse est automatiquement autorisé
- **Règles par défaut** : trafic interne VNet autorisé, trafic entrant externe bloqué par défaut 

### Application Security Group (ASG)

Un **ASG** est un objet logique qui permet de regrouper des machines virtuelles par **rôle applicatif** ou **fonction métier** . Il simplifie la gestion des règles NSG en évitant d'utiliser des adresses IP statiques.

**Caractéristiques clés :**
- **Regroupement logique** : on peut créer des groupes comme "WebServers", "AppServers", "DatabaseServers"
- **Référençable dans les règles NSG** : on utilise le nom de l'ASG comme source ou destination dans les règles 
- **Indépendant des adresses IP** : quand on ajoute ou retire des VM, les règles NSG restent valides
- **Même VNet requis** : toutes les NIC d'un ASG doivent être dans le même réseau virtuel 

### Tableau comparatif

| Critère | NSG | ASG |
|---------|-----|-----|
| **Rôle** | Filtrage du trafic | Regroupement logique de VM |
| **Niveau d'action** | Applique des règles de sécurité | Sert de référence dans les règles |
| **Association** | Subnet ou NIC | NIC uniquement |
| **Dépendance IP** | Utilise des adresses IP | Indépendant des IP |
| **Limite** | Jusqu'à 5 000 NSG par subscription | Doit être dans le même VNet |

**Cas d'usage typique** : Dans une application trois tiers, on crée trois ASG (Web, App, DB). On définit une règle NSG qui autorise le trafic du groupe "Web" vers le groupe "App" sur le port 8080, et du groupe "App" vers le groupe "DB" sur le port 1433. Ainsi, quand on ajoute des VM, on les affecte simplement au bon ASG sans modifier les règles .

---

## 2. Blocage de l'accès à une VM depuis un subnet

Pour bloquer l'accès à une VM depuis un subnet spécifique, on utilise les règles NSG. Plusieurs approches sont possibles selon l'architecture.

### Méthode 1 : Règle de refus explicite (Deny)

On ajoute une règle dans le NSG associé au subnet de la VM (ou directement à sa NIC) avec les paramètres suivants  :

```bash
az network nsg rule create \
  --resource-group $RG \
  --nsg-name $NSG_NAME \
  --name BlockSubnetAccess \
  --priority 150 \
  --source-address-prefixes 10.0.2.0/24 \
  --destination-port-ranges * \
  --access Deny \
  --protocol Tcp
```

### Méthode 2 : Principe du moindre privilège

On configure des règles d'autorisation très spécifiques et on laisse la règle par défaut "DenyAllInBound" bloquer tout le reste .

**Important** : Les règles sont évaluées par ordre de priorité (la plus petite valeur est évaluée en premier). Si une règle "Allow" existe avec une priorité plus élevée (nombre plus petit) qu'une règle "Deny", le trafic sera autorisé .

### Ordre d'évaluation typique

```
1. Règle 100 : Autoriser SSH depuis IP admin (Allow)
2. Règle 150 : Refuser tout trafic depuis subnet X (Deny) ← applicable
3. Règle 65000 : Autoriser trafic VNet (Allow par défaut)
```

Si une requête provient du subnet X et ne correspond à aucune règle Allow spécifique avant la règle 150, elle sera bloquée.

---

## 3. Les NSG Azure sont-ils stateful ou stateless ?

**Les NSG Azure sont stateful (avec état)** .

### Explication technique

Le terme **stateful** signifie que le NSG maintient une table des connexions actives et autorise automatiquement le trafic de retour pour une connexion établie, sans nécessiter de règle explicite dans le sens inverse .

**Exemple concret :**
- Vous créez une règle qui autorise le trafic entrant sur le port 80 (HTTP) depuis Internet
- Un client se connecte à votre serveur web
- Le serveur web répond avec du trafic sortant
- **Ce trafic sortant est automatiquement autorisé** par le NSG car il fait partie d'une connexion existante

### Comparaison avec une approche stateless

Si les NSG étaient stateless, vous devriez créer DEUX règles :
- Une règle entrante pour autoriser la requête HTTP
- Une règle sortante pour autoriser la réponse HTTP

**Avantage du stateful** : Simplification de la gestion et réduction du nombre de règles nécessaires.

**Exception** : Azure Firewall est aussi stateful, mais il opère à un niveau différent (voir section suivante).

---

## 4. Différence entre Azure Firewall et NSG

### Rôle et périmètre d'action

| Critère | Azure Firewall | NSG |
|---------|---------------|-----|
| **Périmètre** | Périmètre du réseau (entrée/sortie) | Interne au VNet  |
| **Niveau** | Application et réseau (L3-L7) | Réseau (L3-L4)  |
| **Filtrage** | FQDN, tags service, threat intelligence | IP, ports, protocoles |
| **État** | Stateful (complet) | Stateful (connexions) |
| **Coût** | Payant (service managé) | Gratuit  |
| **Haute disponibilité** | Intégrée, auto-scaling | Manuel via configuration |

### Fonctionnalités exclusives d'Azure Firewall

- **Filtrage par FQDN** : autoriser ou bloquer l'accès à des noms de domaine complets comme `*.google.com` 
- **Threat Intelligence** : blocage automatique des IP et domaines malveillants 
- **Inspection TLS** : déchiffrement et inspection du trafic HTTPS
- **DNAT (Destination NAT)** : exposer des services internes via l'IP publique du firewall
- **Centralisation** : une seule politique pour plusieurs VNets via Firewall Manager

### Ordre d'évaluation combiné

Si vous utilisez les deux, l'ordre dépend du chemin du trafic  :

- **Trafic entrant depuis Internet** : Azure Firewall est évalué en premier, puis NSG
- **Trafic interne entre VNets** : dépend de la configuration de routage
- **Trafic entre subnets** : NSG uniquement (sauf si routé via le firewall)

> **Règle d'or** : Le trafic doit être autorisé par TOUS les composants sur son chemin pour atteindre sa destination .

---

## 5. Avantages des Resource Groups dans Azure

Un **Resource Group** est un conteneur logique qui regroupe des ressources associées pour une solution Azure .

### Avantages clés

#### 1. Gestion unifiée du cycle de vie
- Toutes les ressources d'un même groupe peuvent être déployées, mises à jour ou supprimées ensemble
- Idéal pour les environnements éphémères (dev, test) : suppression en une seule opération

#### 2. Contrôle d'accès granulaire (RBAC)
- Appliquer des permissions au niveau du groupe (ex: "Contributeur" pour l'équipe Dev)
- Les droits s'héritent sur toutes les ressources du groupe 

#### 3. Organisation par projet/environnement
```bash
rg-payment-prod      # Production
rg-payment-qa        # Qualification  
rg-payment-dev       # Développement
```

#### 4. Gestion des coûts
- Visualisation des coûts agrégés par Resource Group
- Application de budgets et alertes par groupe

#### 5. Application de politiques (Azure Policy)
- Forcer des règles de conformité à l'échelle du groupe (ex: exiger des tags)

#### 6. Verrous (Locks)
- Empêcher la suppression accidentelle de ressources critiques
- Verrou "ReadOnly" ou "Delete" au niveau du groupe 

---

## 6. Différence entre Azure User Data et Custom Data

### Custom Data

**Définition** : Données fournies lors de la création de la VM, passées au système d'exploitation pendant le **provisioning** initial .

**Caractéristiques :**
- Encodage **Base64** obligatoire
- Taille maximale : **64 KB**
- Accessible à la VM uniquement au premier démarrage
- Non modifiable après création
- Pour Linux : stocké dans `/var/lib/waagent/CustomData`
- Pour Windows : fichier `%SYSTEMDRIVE%\AzureData\CustomData.bin`

### User Data

**Définition** : Données utilisateur accessibles via l'**Instance Metadata Service (IMDS)** , persistantes et modifiables pendant toute la vie de la VM.

**Caractéristiques :**
- Accessible via IMDS (endpoint `169.254.169.254`)
- Modifiable après création de la VM
- Utilisé pour cloud-init, scripts de configuration post-déploiement
- Approche moderne recommandée par Microsoft

### Tableau comparatif

| Critère | Custom Data | User Data |
|---------|-------------|-----------|
| **Moment d'accès** | Premier démarrage seulement | À tout moment |
| **Modifiable** | Non | Oui |
| **Persistance** | Provisoire | Permanente |
| **Accès** | Fichier local | API IMDS |
| **Cas d'usage** | Configuration initiale | Configuration dynamique |

> **Recommandation** : Pour les nouveaux déploiements, privilégiez **User Data** avec IMDS, qui offre plus de flexibilité .

---

## 7. Différence entre Azure Application Gateway et Azure Load Balancer

Ces deux services assurent la répartition de charge mais opèrent à des couches différentes du modèle OSI.

### Azure Load Balancer (L4)

**Type** : Équilibreur de charge de **couche 4** (transport) .

**Caractéristiques :**
- Répartition du trafic TCP/UDP
- Basé sur des règles (IP:port)
- Ne regarde pas le contenu des paquets
- Idéal pour le trafic non-HTTP (bases de données, serveurs de jeu, etc.)
- Types : public (entrée Internet) ou interne (entre VM)

**Cas d'usage typiques :**
- Distribution de trafic SQL Server
- Équilibrage de charge pour des serveurs de fichiers
- Haute disponibilité pour des applications legacy

### Application Gateway (L7)

**Type** : Équilibreur de charge de **couche 7** (application), aussi appelé ADC (Application Delivery Controller) .

**Caractéristiques :**
- Comprend le trafic HTTP/HTTPS
- Routage basé sur l'URL (ex: `/api/*` vers pool A, `/images/*` vers pool B)
- Termination SSL/TLS intégrée
- Web Application Firewall (WAF) inclus pour protéger contre les attaques applicatives
- Affinité de session (stickiness) via cookies

**Cas d'usage typiques :**
- Applications web multi-tenants
- Microservices avec routage par chemin
- Sites e-commerce nécessitant une protection WAF

### Tableau comparatif

| Critère | Load Balancer | Application Gateway |
|---------|---------------|---------------------|
| **Couche OSI** | L4 (Transport) | L7 (Application)  |
| **Protocoles** | TCP, UDP | HTTP, HTTPS |
| **Routage** | IP + port | URL, host header |
| **WAF intégré** | Non | Oui |
| **Terminaison SSL** | Non (nécessite back-end) | Oui |
| **Affinité de session** | Non | Oui (cookie-based) |

> **Citation clé** : "Application Gateway est un Application Delivery Controller (ADC) en tant que service, offrant diverses capacités d'équilibrage de charge de couche 7 pour vos applications" .

---

## 8. Explication du flux réseau dans une architecture idéale

Prenons le scénario suivant : une application web déployée dans un subnet web, avec une architecture de sécurité complète incluant Azure Firewall, WAF, NSG et ASG.

### Cheminement complet d'une requête utilisateur

```
Utilisateur → Internet → Azure DNS → Azure Firewall → Application Gateway (WAF) → Load Balancer interne → NSG subnet web → VM web → ASG/NSG back-end
```

### Étape par étape

#### 1. **Résolution DNS**
- L'utilisateur tape `www.monapp.com`
- Azure DNS résout le nom vers l'IP publique de l'Application Gateway (ou du Firewall selon l'architecture)

#### 2. **Azure Firewall (inspection périmétrique)**
- Le trafic arrive sur l'IP publique du Firewall
- Les règles DNAT redirigent vers l'Application Gateway
- Threat Intelligence vérifie si l'IP source est malveillante 
- Les règles réseau/filtrage FQDN sont appliquées

#### 3. **Application Gateway avec WAF**
- Réception du trafic HTTP/HTTPS
- WAF inspecte les requêtes pour détecter les attaques (SQL injection, XSS) 
- Routage basé sur l'URL (ex: `/api` vers un pool spécifique)
- Terminaison SSL (déchiffrement)
- Transmission vers le backend (VM web)

#### 4. **Load Balancer interne (optionnel)**
- Si plusieurs VM web, répartition de charge L4
- Sondes de santé pour vérifier la disponibilité

#### 5. **NSG du subnet web**
- Évaluation des règles entrantes 
- Doit autoriser le trafic provenant de l'Application Gateway (par IP ou service tag)
- Les règles par défaut autorisent le trafic VNet interne 

#### 6. **VM web**
- Réception de la requête
- Traitement applicatif

#### 7. **Communication vers les couches internes (ASG)**
- La VM web a besoin d'appeler l'API back-end (subnet app)
- Les ASG sont utilisés pour définir les règles :
  - Règle NSG autorisant le trafic du groupe "Web" vers le groupe "App" 
- Le trafic est filtré par le NSG du subnet app

#### 8. **Base de données**
- Uniquement accessible depuis le groupe "App" via règles ASG 
- NSG du subnet DB bloque tout autre trafic

### Principes de sécurité appliqués

- **Défense en profondeur** : multiples couches de sécurité
- **Zero Trust** : vérification à chaque étape
- **Moindre privilège** : chaque composant n'a accès qu'à ce dont il a besoin
- **Segmentation** : subnets distincts par fonction

---

## 9. Rôle et cas d'usage d'Azure Bastion

### Qu'est-ce qu'Azure Bastion ?

**Azure Bastion** est un service PaaS entièrement managé qui fournit une connectivité RDP et SSH sécurisée et transparente aux machines virtuelles directement depuis le portail Azure, **sans exposer d'IP publique** sur les VM .

### Architecture

- Déployé dans un VNet, dans un subnet dédié obligatoire : **AzureBastionSubnet** (taille minimale /26)
- Se connecte aux VM via leur **IP privée**
- Le trafic RDP/SSH est tunnelisé sur **HTTPS (port 443)** 

### Avantages clés

#### 1. **Élimination des IP publiques sur les VM**
- Plus besoin de jump box (serveur bastion traditionnel)
- Réduction de la surface d'attaque
- Protection contre le scan de ports

#### 2. **Sécurité renforcée**
- Authentification via Microsoft Entra ID (ex-Azure AD)
- Intégration possible avec l'authentification multifacteur (MFA)
- Journalisation des sessions (Premium SKU)

#### 3. **Simplicité d'utilisation**
- Accès direct depuis le portail Azure
- Pas de client supplémentaire à installer
- Session dans le navigateur (HTML5)

#### 4. **Haute disponibilité intégrée**
- Service managé avec SLA
- Possibilité de déployer dans plusieurs zones de disponibilité

### SKU disponibles

| SKU | Fonctionnalités |
|-----|-----------------|
| **Basic** | Connexions browser, accès VM via IP privée |
| **Standard** | Connexions client natif, partage de liens, ports personnalisés |
| **Premium** | Enregistrement de session, conformité avancée  |

### Quand utiliser Azure Bastion ?

✅ **Cas d'usage recommandés :**
- Administration de VM Windows/Linux sans IP publique
- Environnements de production avec exigences de sécurité strictes
- Conformité réglementaire nécessitant l'absence d'exposition publique
- Accès distant pour équipes support sans infrastructure complexe

❌ **Quand ne pas l'utiliser :**
- Besoin d'accès automatisé (CI/CD) : préférer des agents auto-hébergés
- Budget très contraint (coût horaire fixe)
- Architecture hybride nécessitant une connectivité permanente site-to-site (utiliser VPN Gateway)

### Comparaison : Bastion vs Jump Box traditionnelle

| Critère | Azure Bastion | VM Bastion (Jump Box) |
|---------|---------------|------------------------|
| **Gestion** | PaaS (Microsoft gère) | IaaS (vous gérez) |
| **Mise à l'échelle** | Automatique | Manuelle |
| **HA** | Intégrée | À configurer |
| **Coût** | Fixe (horaire) | VM + stockage |
| **Sécurité** | Haute (MS gère) | Variable (votre responsabilité) |

> **Citation** : "Azure Bastion permet des connexions RDP/SSH sécurisées et transparentes directement depuis le portail Azure, en utilisant des adresses IP privées, sans exposer de points de terminaison publics" .

---

## Conclusion

Ces concepts sont fondamentaux pour les certifications Azure (AZ-900, AZ-104, AZ-700) et les entretiens techniques. Retenez les points clés :

- **NSG** = pare-feu interne (gratuit) ; **ASG** = regroupement logique pour simplifier les règles
- Les **NSG sont stateful** (trafic de retour automatique)
- **Azure Firewall** = périmètre (payant) ; **NSG** = interne (gratuit)
- **Resource Groups** = conteneurs pour cycle de vie, RBAC, coûts
- **Custom Data** = premier démarrage ; **User Data** = modifiable via IMDS
- **Load Balancer** = L4 ; **Application Gateway** = L7 avec WAF
- **Azure Bastion** = accès sécurisé sans IP publique