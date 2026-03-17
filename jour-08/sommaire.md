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
