# Cours Complet : Azure Firewall – Guide Pratique et Théorique

## Sommaire

1. [Introduction à Azure Firewall](#1-introduction-à-azure-firewall)
   - 1.1 Qu’est-ce qu’Azure Firewall ? (What)
   - 1.2 Pourquoi utiliser Azure Firewall ? (Why)
   - 1.3 Quand déployer Azure Firewall ? (When)
   - 1.4 Alternatives et cas où ne pas l’utiliser (If not)

2. [Architecture et Concepts Clés](#2-architecture-et-concepts-clés)
   - 2.1 Règles et collections de règles
   - 2.2 Ordre de traitement des règles
   - 2.3 AzureFirewallSubnet
   - 2.4 Politique de pare-feu (Firewall Policy)

3. [Prérequis](#3-prérequis)

4. [Étape 1 : Création du Groupe de Ressources](#4-étape-1--création-du-groupe-de-ressources)

5. [Étape 2 : Création du Réseau Virtuel et des Sous-réseaux](#5-étape-2--création-du-réseau-virtuel-et-des-sous-réseaux)

6. [Étape 3 : Création de la Machine Virtuelle de Test](#6-étape-3--création-de-la-machine-virtuelle-de-test)

7. [Étape 4 : Déploiement du Firewall et de sa Politique](#7-étape-4--déploiement-du-firewall-et-de-sa-politique)

8. [Étape 5 : Création de la Route par Défaut](#8-étape-5--création-de-la-route-par-défaut)

9. [Étape 6 : Configuration des Règles d’Application](#9-étape-6--configuration-des-règles-dapplication)

10. [Étape 7 : Configuration des Règles Réseau](#10-étape-7--configuration-des-règles-réseau)

11. [Étape 8 : Configuration d’une Règle DNAT (Destination NAT)](#11-étape-8--configuration-dune-règle-dnat-destination-nat)

12. [Étape 9 : Modification des Serveurs DNS de la Carte Réseau](#12-étape-9--modification-des-serveurs-dns-de-la-carte-réseau)

13. [Étape 10 : Test du Firewall](#13-étape-10--test-du-firewall)

14. [Conclusion et Bonnes Pratiques](#14-conclusion-et-bonnes-pratiques)

---

## 1. Introduction à Azure Firewall

### 1.1 Qu’est-ce qu’Azure Firewall ? (What)

**Azure Firewall** est un service de sécurité réseau cloud-native, managé par Microsoft, qui protège les ressources déployées dans vos réseaux virtuels Azure . C’est un pare-feu avec état (*stateful*) en tant que service (*Firewall-as-a-Service*), ce qui signifie qu’il inspecte l’intégralité du trafic réseau et maintient une table des connexions actives pour prendre des décisions intelligentes.

Il offre une haute disponibilité intégrée et une scalabilité automatique (il s’adapte à la charge sans aucune configuration de votre part). Il peut être centralisé pour contrôler le trafic sortant (de l’intérieur vers Internet), entrant (d’Internet vers l’intérieur) et le trafic entre les réseaux virtuels.

### 1.2 Pourquoi utiliser Azure Firewall ? (Why)

Le contrôle du trafic réseau est un pilier de toute stratégie de sécurité. Voici pourquoi Azure Firewall est un choix pertinent  :
- **Filtrage granulai**re : Il permet de créer des règles très précises basées sur des adresses IP, des protocoles, des ports, mais aussi sur des noms de domaine complets (FQDN) via des règles d’application.
- **Politique centralisée** : Avec Azure Firewall Manager, vous pouvez gérer des politiques de sécurité de manière centralisée pour plusieurs pare-feu à travers différentes régions et abonnements.
- **Inspection TLS** : Il peut déchiffrer, inspecter et rechiffrer le trafic sortant pour détecter les menaces cachées dans le trafic HTTPS.
- **Intelligence contre les menaces** : Il peut être intégré aux flux d’intelligence contre les menaces de Microsoft pour alerter ou bloquer le trafic vers des domaines ou adresses IP malveillants.
- **Journalisation et analyse** : Il s’intègre nativement avec Azure Monitor pour fournir des logs détaillés sur le trafic autorisé et refusé.

### 1.3 Quand déployer Azure Firewall ? (When)

Vous devriez envisager Azure Firewall dans les situations suivantes :
- **Vous devez sécuriser le trafic sortant (South-North)** : Par exemple, pour limiter les sites web que les VM de votre subnet peuvent visiter .
- **Vous voulez exposer des services internes de manière sécurisée** : En utilisant le DNAT (Destination Network Address Translation) pour rediriger le trafic entrant vers une VM spécifique, sans exposer directement l’IP de la VM.
- **Vous mettez en place une architecture Hub & Spoke** : Azure Firewall est la pièce centrale pour centraliser la sécurité de tous vos réseaux spokes.
- **Vous avez besoin d’isoler des environnements** (Prod, Dev, Test) avec des politiques de filtrage différentes.
- **Vous souhaitez remplacer des appliances virtuelles (NVA)** complexes et coûteuses à gérer par un service managé.

### 1.4 Alternatives et cas où ne pas l’utiliser (If not)

Azure Firewall n’est pas toujours la solution adaptée :
- **Pour une sécurité de couche application très fine (WAF)** : Si vous avez besoin de protéger vos applications web contre les injections SQL ou le XSS, le **WAF (Web Application Firewall)** intégré à Application Gateway ou Azure Front Door est plus approprié.
- **Pour un filtrage très basique entre sous-réseaux** : Si vous avez juste besoin de bloquer un port spécifique entre deux sous-réseaux dans un petit environnement, un simple **Network Security Group (NSG)** peut suffire et être plus simple à configurer.
- **Pour des besoins de routage très complexes** : Une appliance virtuelle (NVA) comme un Cisco ASA ou un pfSense pourrait offrir plus de flexibilité, mais avec une charge de gestion bien supérieure.
- **Coût** : Azure Firewall a un coût fixe plus un coût par Go de données traitées. Pour des volumes très faibles, une VM faisant office de pare-feu peut être plus économique, mais au prix d’une fiabilité et d’une scalabilité moindres.

---

## 2. Architecture et Concepts Clés

Avant de passer à la pratique, il est essentiel de comprendre quelques concepts fondamentaux.

### 2.1 Règles et collections de règles
Les règles définissent le trafic autorisé ou refusé. Elles sont regroupées en **collections de règles** :
- **Règles NAT (DNAT)** : Traduisent une destination IP/publique (IP du firewall) vers une IP privée interne (ex: RDP vers une VM).
- **Règles réseau** : Filtrent le trafic basé sur les adresses IP, protocoles (TCP, UDP) et ports. Elles sont idéales pour des services non-HTTP/HTTPS (ex: DNS, RDP).
- **Règles d’application** : Filtrent le trafic basé sur des noms de domaine complets (FQDN) comme `www.google.com` .

### 2.2 Ordre de traitement des règles
L’ordre d’évaluation des règles est crucial et suit une logique stricte  :
1.  **Règles NAT (DNAT)** : Appliquées en premier pour le trafic entrant.
2.  **Règles réseau** : Appliquées ensuite. Si une règle réseau autorise le trafic, les règles d’application ne sont pas évaluées.
3.  **Règles d’application** : Si le trafic est HTTP/HTTPS et qu’aucune règle réseau ne l’a autorisé, les règles d’application sont alors évaluées.
4.  **Règles d’infrastructure** : Collections de règles internes à Azure, autorisées par défaut.
5.  **Déni par défaut** : Si aucune règle ne correspond, le trafic est **refusé**.

### 2.3 AzureFirewallSubnet
Le firewall doit être déployé dans un sous-réseau dédié nommé **obligatoirement** `AzureFirewallSubnet`. Ce sous-réseau doit avoir une taille minimale de `/26` (soit 64 adresses) pour permettre la scalabilité du service .

### 2.4 Politique de pare-feu (Firewall Policy)
C’est une ressource Azure qui contient l’ensemble des règles (NAT, réseau, application) et des paramètres (intelligence contre les menaces, DNS, etc.). Elle permet de gérer les règles de manière centralisée et de les appliquer à plusieurs firewalls. C’est la méthode moderne de gestion .

---

## 3. Prérequis

Pour suivre ce tutoriel, vous aurez besoin de :
- Un abonnement Azure actif (vous pouvez créer un compte gratuit).
- Azure CLI installé et configuré, ou l’accès à [Azure Cloud Shell](https://shell.azure.com/) directement depuis le portail.
- Une paire de clés SSH (si vous utilisez Linux) ou des identifiants pour une VM Windows.

---

## 4. Étape 1 : Création du Groupe de Ressources

Le **groupe de ressources** est un conteneur logique qui rassemblera toutes les ressources de ce TP.

```bash
# Définir des variables
RG="rg-firewall-tutorial"
LOCATION="francecentral"  # Choisissez une région proche de vous

# Créer le groupe de ressources
az group create --name $RG --location $LOCATION
```

---

## 5. Étape 2 : Création du Réseau Virtuel et des Sous-réseaux

Nous allons créer un VNet nommé `vnet-firewall` avec deux sous-réseaux :
- `AzureFirewallSubnet` : pour le firewall (taille /26).
- `Workload-SN` : pour la machine virtuelle de test (taille /24) .

```bash
VNET_NAME="vnet-firewall"
# Sous-réseau pour le firewall (nom OBLIGATOIRE)
FIREWALL_SUBNET="AzureFirewallSubnet"
# Sous-réseau pour la VM de workload
WORKLOAD_SUBNET="Workload-SN"

# Créer le VNet
az network vnet create \
    --resource-group $RG \
    --name $VNET_NAME \
    --address-prefix 10.0.0.0/16

# Créer le sous-réseau pour le firewall (taille minimale /26)
az network vnet subnet create \
    --resource-group $RG \
    --vnet-name $VNET_NAME \
    --name $FIREWALL_SUBNET \
    --address-prefix 10.0.1.0/26

# Créer le sous-réseau pour la charge de travail
az network vnet subnet create \
    --resource-group $RG \
    --vnet-name $VNET_NAME \
    --name $WORKLOAD_SUBNET \
    --address-prefix 10.0.2.0/24
```

---

## 6. Étape 3 : Création de la Machine Virtuelle de Test

Créez une VM Linux (Ubuntu) dans le sous-réseau `Workload-SN`. **Elle n’aura pas d’IP publique** . Nous l’appellerons `srv-work`.

```bash
VM_NAME="srv-work"
ADMIN_USER="azureuser"

# Créer la VM sans IP publique
az vm create \
    --resource-group $RG \
    --name $VM_NAME \
    --image Ubuntu2204 \
    --vnet-name $VNET_NAME \
    --subnet $WORKLOAD_SUBNET \
    --admin-username $ADMIN_USER \
    --generate-ssh-keys \
    --public-ip-address "" \
    --size Standard_B1s

# Récupérer l'adresse IP privée de la VM (elle sera utile plus tard)
VM_PRIVATE_IP=$(az vm show --resource-group $RG --name $VM_NAME --show-details --query privateIps -o tsv)
echo "L'adresse IP privée de la VM de test est : $VM_PRIVATE_IP"
```

> **Note** : Si vous êtes bloqué par le nombre de vCPU dans votre région, vous pouvez choisir une autre taille comme `Standard_B1s`.

---

## 7. Étape 4 : Déploiement du Firewall et de sa Politique

Cette étape comprend la création d’une IP publique pour le firewall, la création de la ressource firewall, et la création d’une politique de pare-feu .

```bash
# Nom de la politique de pare-feu
POLICY_NAME="fw-test-pol"
# Nom du firewall
FIREWALL_NAME="test-fw01"
# Nom de l'IP publique pour le firewall
FW_PUBLIC_IP="fw-pip"

# 1. Créer une IP publique Standard
az network public-ip create \
    --resource-group $RG \
    --name $FW_PUBLIC_IP \
    --sku Standard \
    --location $LOCATION

# 2. Créer la politique de pare-feu
az network firewall policy create \
    --resource-group $RG \
    --name $POLICY_NAME \
    --location $LOCATION

# 3. Créer le firewall en l'associant à la politique et au VNet
az network firewall create \
    --resource-group $RG \
    --name $FIREWALL_NAME \
    --location $LOCATION \
    --sku AZFW_VNet \
    --firewall-policy $POLICY_NAME

# 4. Créer la configuration IP du firewall
az network firewall ip-config create \
    --resource-group $RG \
    --firewall-name $FIREWALL_NAME \
    --name "fw-ip-config" \
    --vnet-name $VNET_NAME \
    --public-ip-address $FW_PUBLIC_IP

# 5. Récupérer les adresses IP publique et privée du firewall (à noter)
az network firewall show \
    --resource-group $RG \
    --name $FIREWALL_NAME \
    --query "[ipConfigurations[0].privateIPAddress, ipConfigurations[0].publicIPAddress]"
```

Notez bien les valeurs retournées pour la suite.

---

## 8. Étape 5 : Création de la Route par Défaut

Pour que le trafic sortant de la VM de test passe par le firewall, nous devons créer une **route par défaut** (`0.0.0.0/0`) pointant vers l’IP privée du firewall. Cette route sera associée à la table de routage du sous-réseau `Workload-SN` .

```bash
ROUTE_TABLE_NAME="firewall-route"
# Remplacez par l'IP privée de votre firewall récupérée à l'étape précédente
FW_PRIVATE_IP="<IP_PRIVEE_DU_FIREWALL>"

# 1. Créer une table de routage
az network route-table create \
    --resource-group $RG \
    --name $ROUTE_TABLE_NAME \
    --location $LOCATION \
    --disable-bgp-route-propagation false

# 2. Créer la route par défaut (0.0.0.0/0) avec le next hop "VirtualAppliance"
az network route-table route create \
    --resource-group $RG \
    --route-table-name $ROUTE_TABLE_NAME \
    --name "fw-default-route" \
    --address-prefix 0.0.0.0/0 \
    --next-hop-type VirtualAppliance \
    --next-hop-ip-address $FW_PRIVATE_IP

# 3. Associer la table de routage au sous-réseau de travail
az network vnet subnet update \
    --resource-group $RG \
    --vnet-name $VNET_NAME \
    --name $WORKLOAD_SUBNET \
    --route-table $ROUTE_TABLE_NAME
```

---

## 9. Étape 6 : Configuration des Règles d’Application

Nous allons créer une règle d’application qui autorise la VM de test (source `10.0.2.0/24`) à accéder uniquement à `www.google.com` en HTTP/HTTPS . Tout autre site sera bloqué.

```bash
# Ajouter une collection de règles d'application à la politique
az network firewall policy rule-collection-group collection add-filter-collection \
    --policy-name $POLICY_NAME \
    --resource-group $RG \
    --rule-collection-group-name "DefaultApplicationRuleCollectionGroup" \
    --name "App-Coll01" \
    --collection-priority 200 \
    --action Allow \
    --rule-name "Allow-Google" \
    --rule-type ApplicationRule \
    --source-addresses "10.0.2.0/24" \
    --protocols "http=80" "https=443" \
    --target-fqdns "www.google.com"
```

> **Important** : Les règles d’application ne sont évaluées que si le trafic n’a pas été autorisé par une règle réseau. Le firewall a aussi des règles d’infrastructure intégrées .

---

## 10. Étape 7 : Configuration des Règles Réseau

Pour que la VM puisse résoudre les noms de domaine (DNS), elle doit pouvoir contacter un serveur DNS. Nous allons créer une règle réseau autorisant le trafic UDP vers les serveurs DNS publics `209.244.0.3` et `209.244.0.4` sur le port 53 .

```bash
az network firewall policy rule-collection-group collection add-filter-collection \
    --policy-name $POLICY_NAME \
    --resource-group $RG \
    --rule-collection-group-name "DefaultNetworkRuleCollectionGroup" \
    --name "Net-Coll01" \
    --collection-priority 200 \
    --action Allow \
    --rule-name "Allow-DNS" \
    --rule-type NetworkRule \
    --source-addresses "10.0.2.0/24" \
    --protocols "UDP" \
    --destination-addresses "209.244.0.3" "209.244.0.4" \
    --destination-ports "53"
```

---

## 11. Étape 8 : Configuration d’une Règle DNAT (Destination NAT)

Pour tester l’accès entrant, nous allons configurer une règle DNAT. Elle permettra de se connecter en SSH à notre VM privée (`srv-work`) en utilisant l’IP publique du firewall et un port spécifique (par exemple, le port `2222`). Le firewall redirigera le trafic vers le port 22 de la VM privée .

```bash
# Récupérer l'IP publique du firewall
FW_PUBLIC_IP_ADDRESS=$(az network public-ip show --resource-group $RG --name $FW_PUBLIC_IP --query ipAddress -o tsv)

echo "L'adresse IP publique du firewall est : $FW_PUBLIC_IP_ADDRESS"

# Ajouter une règle DNAT
az network firewall policy rule-collection-group collection add-nat-collection \
    --policy-name $POLICY_NAME \
    --resource-group $RG \
    --rule-collection-group-name "DefaultDnatRuleCollectionGroup" \
    --name "Dnat-Coll01" \
    --collection-priority 200 \
    --action Dnat \
    --rule-name "SSH-DNAT" \
    --rule-type NatRule \
    --source-addresses "*" \
    --protocols "TCP" \
    --destination-addresses $FW_PUBLIC_IP_ADDRESS \
    --destination-ports "2222" \
    --translated-address $VM_PRIVATE_IP \
    --translated-port "22"
```

---

## 12. Étape 9 : Modification des Serveurs DNS de la Carte Réseau

Pour que la VM utilise les serveurs DNS que nous avons autorisés dans la règle réseau (et non les DNS Azure par défaut), nous devons modifier la configuration DNS de sa carte réseau .

```bash
# Récupérer le nom de la carte réseau de la VM
NIC_NAME=$(az vm show --resource-group $RG --name $VM_NAME --query 'networkProfile.networkInterfaces[0].id' -o tsv | xargs basename)

# Configurer des DNS personnalisés sur la carte réseau
az network nic update \
    --resource-group $RG \
    --name $NIC_NAME \
    --dns-servers "209.244.0.3" "209.244.0.4"

# Redémarrer la VM pour appliquer les changements (optionnel mais recommandé)
az vm restart --resource-group $RG --name $VM_NAME
```

> **Note sur DNS Proxy** : Azure Firewall peut aussi agir comme un proxy DNS, ce qui simplifie la configuration. Si vous l’activez, vous définissez l’IP privée du firewall comme serveur DNS sur la carte réseau . Cette approche est plus évolutive dans un environnement de production.

---

## 13. Étape 10 : Test du Firewall

Nous allons maintenant tester la configuration pour nous assurer que tout fonctionne comme prévu.

### 13.1 Test de la règle DNAT (accès entrant)
Depuis votre machine locale, essayez de vous connecter en SSH à la VM via l’IP publique du firewall sur le port 2222 .
```bash
ssh -p 2222 $ADMIN_USER@$FW_PUBLIC_IP_ADDRESS
```
La connexion doit aboutir. Cela prouve que la règle DNAT fonctionne.

### 13.2 Test des règles d’application (accès sortant)
Une fois connecté à la VM, testez l’accès aux sites web :
```bash
# Ceci doit fonctionner
curl -v http://www.google.com
# Ceci doit être bloqué par le firewall
curl -v http://www.microsoft.com
```
Vous devriez voir la page de Google, mais une erreur de timeout ou de refus pour Microsoft.

### 13.3 Test de la résolution DNS
Vérifiez que la VM utilise bien les DNS configurés :
```bash
cat /etc/resolv.conf
# Vous devriez voir les adresses 209.244.0.3 et 209.244.0.4
```

---

## 14. Conclusion et Bonnes Pratiques

Félicitations ! Vous avez déployé et configuré un Azure Firewall complet. Vous avez appris à :

1.  Créer une infrastructure réseau sécurisée (VNet, subnets).
2.  Déployer un firewall managé avec une politique centralisée.
3.  Forcer le trafic d’un sous-réseau à passer par le firewall (UDR).
4.  Créer des règles d’application pour le filtrage web.
5.  Créer des règles réseau pour autoriser des services comme le DNS.
6.  Exposer un service interne via DNAT.
7.  Configurer les DNS d’une VM.

**Bonnes pratiques à retenir :**
- **Toujours utiliser une Firewall Policy** plutôt que les règles classiques pour une gestion centralisée.
- **Nommer correctement vos ressources** (ex: `AzureFirewallSubnet` est obligatoire).
- **Planifier vos plages d’adresses IP** pour éviter les chevauchements, surtout si vous prévoyez du peering.
- **Surveiller les logs** : Activez les diagnostics sur le firewall pour envoyer les logs vers Log Analytics, Storage Account ou Event Hubs.
- **Principe du moindre privilège** : Commencez avec des règles restrictives et ouvrez au fur et à mesure, en analysant les logs.
- **Séparer les environnements** : Utilisez des politiques différentes pour la production, le développement et les tests.

Pour nettoyer toutes les ressources et éviter des frais inutiles, supprimez le groupe de ressources :
```bash
az group delete --name $RG --yes --no-wait
```

---

**Résumé des commandes Azure CLI pour Azure Firewall :**
```bash
# Créer un RG
az group create --name $RG --location $LOCATION

# Créer un VNet et subnets
az network vnet create ...
az network vnet subnet create --name AzureFirewallSubnet ...

# Créer IP publique, politique, firewall
az network public-ip create ...
az network firewall policy create ...
az network firewall create ...
az network firewall ip-config create ...

# Créer une route par défaut
az network route-table create ...
az network route-table route create ...
az network vnet subnet update --route-table ...

# Ajouter des règles
az network firewall policy rule-collection-group collection add-filter-collection ... # App/Network
az network firewall policy rule-collection-group collection add-nat-collection ...    # DNAT

# Configurer DNS sur la VM
az network nic update --dns-servers ...
```