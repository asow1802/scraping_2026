# Scraping & Analyse du Marché des Tracteurs Routiers

##  Présentation du projet

Dans le cadre du séminaire professionnel, nous avons réalisé un projet de **scraping et d’analyse de données** portant sur le marché des **tracteurs routiers d’occasion**.

L’objectif est de collecter, nettoyer et analyser des données issues du site :  
 https://www.europe-camions.com

Ces données seront ensuite exploitées pour produire des **analyses statistiques** et des **visualisations (dashboard)**.

---

##  Objectifs

- Scraper automatiquement les annonces de tracteurs routiers
- Extraire les informations pertinentes
- Nettoyer et structurer les données
- Stocker les données dans un fichier exploitable 
- Analyser les données
- Créer un dashboard interactif

---

##  Données collectées

Pour chaque véhicule, nous récupérons les informations suivantes :

- Description
- Année de mise en circulation
- Marque
- Kilométrage
- Prix
- Modèle
- Localisation du vendeur
- Pays
- Photos du camion
- Poids total en charge
- Poids - puissance moteur
- Boite de vitesse
- Type d'énergie
- Etat
- Essieux
- Réf. client

---

##  Technologies utilisées

- Python
- BeautifulSoup (scraping HTML)
- Requests (requêtes HTTP)
- Pandas (nettoyage et analyse des données)
- Looker studio (visualisation)

---

## Étapes du projet

1. Analyse du site web à scraper  
2. Identification des données utiles  
3. Développement du script de scraping  
4. Gestion de la pagination  
5. Extraction des données  
6. Nettoyage des données  
7. Stockage des données dans des fichiers CSV 
8. Analyse exploratoire  
9. Création du dashboard  

---

###  Gestion de la pagination  et Exctraction des données

Le site contenant plusieurs centaines de pages, une boucle a été mise en place pour :

- Naviguer automatiquement entre les pages
- Récupérer l’URL de la page suivante
- Éviter les boucles infinies en vérifiant les pages déjà visitées

Cela permet de scraper l’ensemble des annonces disponibles.

---

###  Nettoyage des données

Les données récupérées n’étant pas directement exploitables, un travail de nettoyage a été effectué :

- Suppression des caractères spéciaux (espaces, `EUR`, `€`)
- Conversion des prix en valeurs numériques
- Gestion des valeurs manquantes

---

###  Traitement des différents types de prix

Lors du scraping, plusieurs formats de prix ont été identifiés :

- Prix en euros
- Prix en location/mois
- Prix sur demande
- Prix en location sur demande

Afin de faciliter l’analyse :

- Les prix numériques ont été nettoyés et convertis
- Les annonces ont été classées selon leur type de prix
- Les données ont été séparées pour éviter les incohérences

---

###  Création des fichiers de données

À l’issue du scraping et du nettoyage :

- Les données ont été stockées dans des fichiers CSV
- Plusieurs fichiers ont été générés en fonction du type de prix
- Chaque fichier contient des données structurées et prêtes à être exploitées

Ces fichiers serviront de base pour les étapes suivantes d’analyse et de visualisation.

---

##  Organisation du projet

Le projet est réalisé en équipe avec une organisation inspirée des méthodes agiles :

- Scraping des données
- Nettoyage et structuration
- Analyse
- Visualisation
- Vérification des données

---

##  Sortie du projet

- Fichier CSV contenant les données 
- Dashboard interactif
- Code Python du scraping
