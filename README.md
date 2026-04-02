# Documentation utilisateur — Scraper de tracteurs d'occasion
**Projet Groupe 4 · Site cible : europe-camions.com**

---

## Table des matières

1. [Vue d'ensemble](#1-vue-densemble)
2. [Prérequis](#2-prérequis)
3. [Structure du projet](#3-structure-du-projet)
4. [Fonctionnement détaillé](#4-fonctionnement-détaillé)
   - 4.1 [Initialisation et configuration](#41-initialisation-et-configuration)
   - 4.2 [Scraping de la première page](#42-scraping-de-la-première-page)
   - 4.3 [Pagination automatique](#43-pagination-automatique)
   - 4.4 [Construction du DataFrame](#44-construction-du-dataframe)
   - 4.5 [Segmentation des prix](#45-segmentation-des-prix)
   - 4.6 [Export CSV](#46-export-csv)
5. [Données collectées](#5-données-collectées)
6. [Fichiers de sortie](#6-fichiers-de-sortie)
7. [Limitations connues et points d'attention](#7-limitations-connues-et-points-dattention)
8. [Exemple de résultat](#8-exemple-de-résultat)
9. [Exemple de résultat](#9-Nettoyage-de-données)

---

## 1. Vue d'ensemble

Ce notebook Python réalise le **scraping automatisé** des annonces de tracteurs d'occasion publiées sur le site [europe-camions.com](https://www.europe-camions.com/tracteur-occasion/1-31/annonces-tracteur.html).

Il parcourt toutes les pages de résultats, extrait les caractéristiques techniques et commerciales de chaque annonce, puis exporte les données dans plusieurs fichiers CSV segmentés selon le type de prix.

**Flux global :**

```
Site web (europe-camions.com)
        │
        ▼
  Scraping page par page (BeautifulSoup + requests)
        │
        ▼
  Dictionnaire Python → DataFrame pandas
        │
        ▼
  Nettoyage & segmentation par type de prix
        │
        ▼
  Export en 4 fichiers CSV
```

---

## 2. Prérequis

### Bibliothèques Python requises

| Bibliothèque | Rôle |
|---|---|
| `pandas` | Manipulation des données tabulaires |
| `requests` | Envoi des requêtes HTTP au site |
| `beautifulsoup4` | Parsing du HTML des pages |
| `re` | Expressions régulières pour le nettoyage |
| `time` | Pause entre les requêtes (politesse serveur) |
| `numpy` | Gestion des valeurs manquantes (`np.nan`) |

### Installation

```bash
pip install pandas requests beautifulsoup4 numpy
```

### Environnement recommandé

- Python 3.8+
- Jupyter Notebook ou JupyterLab
- Connexion internet stable (le scraping peut durer plusieurs minutes selon le nombre de pages)

---

## 3. Structure du projet

Le notebook est organisé en **5 cellules principales** :

| Cellule | Rôle |
|---|---|
| 1 — Imports | Chargement des bibliothèques |
| 2 — Scraping page 1 | Initialisation du dictionnaire et extraction de la première page |
| 3 — Boucle de pagination | Parcours automatique de toutes les pages suivantes |
| 4 — Construction du DataFrame | Mise en forme et nettoyage initial |
| 5 — Segmentation & export | Séparation par type de prix et sauvegarde CSV |

---

## 4. Fonctionnement détaillé

### 4.1 Initialisation et configuration

**URL de départ :**
```
https://www.europe-camions.com/tracteur-occasion/1-31/annonces-tracteur.html
```

Un dictionnaire `data` est créé avec une clé par caractéristique à collecter. Chaque clé correspond à un champ affiché sur le site :

```python
data = {
    "Réf. client :", "Description", "Kilométrage",
    "Date de 1ere immatriculation", "Energie", "Marque",
    "Modèle", "Pays", "Prix", "Vendeur", "Localisation",
    "Photo", "Poids à vide", "PTC", "Essieux", "Puissance", "Etat"
}
```

---

### 4.2 Scraping de la première page

Le code charge la page d'accueil et repère les blocs d'annonces via la classe CSS `google-tag` (chaque bloc correspond à une annonce).

Pour chaque annonce, il extrait :

- **Prix** → balise `<span class="meta-price">`
- **Vendeur** → balise `<span class="title-vendeur">`
- **Localisation** → balise `<span class="d-none">`
- **Description** → balise `<h2>`
- **Photo** → attribut `src` de la balise `<img>`
- **Caractéristiques techniques** → balises `<td>` en paires clé/valeur

> **Principe des paires `<td>`** : Le tableau HTML contient des lignes où la première cellule est le nom d'une caractéristique (ex. `"Kilométrage"`) et la cellule suivante sa valeur (ex. `"220 000 km"`). Le code vérifie si le contenu d'une cellule correspond à une clé du dictionnaire, et si oui, récupère la cellule adjacente.

---

### 4.3 Pagination automatique

Après avoir scrapé une page, le script recherche le bouton "page suivante" :

```python
next_button = extract_contenu.select_one(
    "div.btn-next-pagination.next.btn.text-center.no-decoration"
)
```

Il utilise l'attribut `data-ihref` de ce bouton pour construire l'URL de la page suivante. La boucle continue jusqu'à ce qu'il n'y ait plus de bouton suivant, ou qu'une URL déjà visitée soit rencontrée (protection anti-boucle infinie via un ensemble `pages_vues`).

Une **pause de 0,5 seconde** (`time.sleep(0.5)`) est insérée entre chaque page pour ne pas surcharger le serveur.

> **Note :** Lors de l'exécution, le scraper a parcouru jusqu'à 475 pages avant de détecter une URL déjà visitée et de s'arrêter.

---

### 4.4 Construction du DataFrame

```python
df = pd.DataFrame(data.values(), index=data.keys()).T
df["Puissance"] = df["Puissance"].replace(r"[^\d]", "", regex=True)
```

La colonne `Puissance` est nettoyée pour ne conserver que les chiffres (suppression des unités textuelles comme `"ch"` ou `"cv"`).

---

### 4.5 Segmentation des prix

Les annonces présentent quatre formats de prix distincts. Le code crée un DataFrame séparé pour chacun :

| DataFrame | Condition de filtrage | Description |
|---|---|---|
| `LocationParMois` | Contient `"mois en location"` | Véhicules en location avec prix mensuel |
| `PrixSurDemande` | Contient `"Prix sur demande"` | Prix non affiché, à demander au vendeur |
| `PrixLocationSurDemande` | Contient `"Tarif de location sur demande"` | Location à tarif non affiché |
| `Achat` | Aucune des conditions ci-dessus | Prix d'achat exprimé en euros |

> ⚠️ **Avertissement** : La cellule de segmentation contient une erreur de syntaxe à la ligne de nettoyage du prix dans `Achat` (mélange de `str.extract` et `replace`). Cette ligne doit être corrigée avant exécution.

---

### 4.6 Export CSV

Chaque DataFrame est exporté en CSV sans index :

```python
LocationParMois.to_csv("LocationParMois.csv", index=False)
PrixSurDemande.to_csv("PrixSurDemande.csv", index=False)
PrixLocationSurDemande.to_csv("PrixLocationSurDemande.csv", index=False)
Achat.to_csv("Achat.csv", index=False)
```

Les fichiers sont créés dans le répertoire de travail courant du notebook.

---

## 5. Données collectées

Voici la liste complète des 17 champs extraits pour chaque annonce :

| Champ | Type | Exemple |
|---|---|---|
| `Réf. client :` | Texte | `00013161` |
| `Description` | Texte | `Tracteur Mercedes Arocs 2046` |
| `Kilométrage` | Texte | `220 000 km` |
| `Date de 1ere immatriculation` | Texte | `31/01/2018` |
| `Energie` | Texte | `Gazoil` |
| `Marque` | Texte | `Mercedes` |
| `Modèle` | Texte | `2046` |
| `Pays` | Texte | `FRANCE` |
| `Prix` | Texte | `45 800 EURHT` ou `Prix sur demande` |
| `Vendeur` | Texte | `BERROYER` |
| `Localisation` | Texte | `Vendée` |
| `Photo` | URL | `https://photo.static-viamobilis.com/...` |
| `Poids à vide` | Texte | `7.51 Tonnes` |
| `PTC` | Texte | `19 Tonnes` |
| `Essieux` | Texte | `4x2` |
| `Puissance` | Numérique (après nettoyage) | `460` |
| `Etat` | Texte | `Occasion` ou `Neuf` |

---

## 6. Fichiers de sortie

| Fichier | Contenu |
|---|---|
| `LocationParMois.csv` | Annonces de location avec loyer mensuel chiffré |
| `PrixSurDemande.csv` | Annonces sans prix affiché (achat) |
| `PrixLocationSurDemande.csv` | Annonces de location sans tarif affiché |
| `Achat.csv` | Annonces avec prix d'achat en euros |

---

## 7. Limitations connues et points d'attention

### Erreur de syntaxe à corriger

La ligne suivante dans la cellule de segmentation est incorrecte et empêchera l'exécution :

```python
# ❌ Code actuel (erroné)
Achat["Prix"] = Achat["Prix"].replace(str.extract(r'(\d+)')r"[^\d]", "", regex=True)

# ✅ Correction suggérée
Achat["Prix"] = Achat["Prix"].replace(r"[^\d]", "", regex=True)
```

### Avertissement `SettingWithCopyWarning`

Les affectations sur les sous-DataFrames (`LocationParMois["Prix"] = ...`) génèrent un avertissement pandas. Pour l'éviter, utiliser `.loc` :

```python
LocationParMois.loc[:, "Prix"] = LocationParMois["Prix"].replace(r"[^\d]", "", regex=True)
```

### Longueurs de colonnes incohérentes

Certaines annonces peuvent ne pas afficher tous les champs (ex. pas de photo, pas de kilométrage). Cela peut provoquer des colonnes de longueurs différentes lors de la construction du DataFrame. Il est conseillé de vérifier avec `df.shape` et de gérer les valeurs manquantes avec `pd.DataFrame.from_dict(data, orient='index').T` si nécessaire.

### Dépendance à la structure HTML du site

Si le site modifie sa structure HTML (classes CSS, organisation des `<td>`), le scraper peut cesser de fonctionner ou retourner des données vides. Une vérification périodique est recommandée.

### Durée d'exécution

Le script a parcouru 475 pages lors de l'exécution documentée. Avec une pause de 0,5 s par page, cela représente environ **4 à 8 minutes** d'exécution selon la connexion.

---

## 8. Exemple de résultat

Aperçu des 5 premières lignes du DataFrame final :

| Réf. client | Description | Kilométrage | Date immat. | Marque | Prix | Puissance | Etat |
|---|---|---|---|---|---|---|---|
| 00013161 | Tracteur Mercedes Arocs 2046 | 220 000 km | 31/01/2018 | Mercedes | Prix sur demande | 460 | Occasion |
| 00013489 | Tracteur DAF XF 530 | 538 000 km | 18/12/2019 | DAF | 45 800 EURHT | 530 | Occasion |
| VO25-1966 | Tracteur Volvo F6 460 FH | 511 000 km | 04/04/2019 | Volvo | Prix sur demande | 460 | Occasion |
| VO26-2072 | Tracteur Iveco S-WAY 490 | 416 350 km | 21/10/2022 | Iveco | Prix sur demande | 490 | Occasion |
| VO25-1984 | Tracteur produits dangereux Renault | 74 km | 12/06/2024 | Renault | Prix sur demande | 480 | Neuf |

---
### Nettoyage des données 
# Projet de Nettoyage de Données - Marché Poids Lourds

Ce projet vise à nettoyer et harmoniser les données de véhicules (tracteurs routiers) provenant de différentes sources de vente et de location.

## Sources de données
Le projet traite quatre fichiers principaux :
- `Achat.csv` : Véhicules en vente directe.
- `LocationParMois.csv` : Véhicules en location avec tarifs mensuels.
- `PrixSurDemande.csv` : Véhicules en vente (prix non affiché).
- `PrixLocationSurDemande.csv` : Véhicules en location (prix non affiché).

## Étapes de nettoyage
Le traitement automatisé via Python (Pandas) a permis de réaliser les opérations suivantes :

### 1. Typage et Conversion
- **Kilométrage** : Suppression du suffixe "km" et des espaces, conversion en entier.
- **Poids (Vide/PTC)** : Extraction des valeurs numériques et conversion en tonnes (float).
- **Dates** : Conversion des dates de première immatriculation au format `datetime`.
- **Prix** : Conversion en format numérique (NaN pour les mentions textuelles "sur demande").

### 2. Normalisation Textuelle
- Nettoyage des colonnes `Marque`, `Modèle`, `Energie`, and `Localisation` (suppression des espaces blancs).
- Standardisation de la colonne `Etat` (Occasion/Neuf).

### 3. Qualité des données
- **Déduplication** : Suppression des doublons basée sur la référence client unique.
- **Nettoyage des colonnes** : Suppression de la colonne `Photo` (non nécessaire pour l'analyse).
- **Indicateurs** : Ajout d'une colonne `Prix_connu` pour faciliter le filtrage lors de l'analyse marketing.

## Fichiers générés
Le script produit quatre fichiers nettoyés :
- `Achat_clean.csv`
- `Location_clean.csv`
- `PrixSurDemande_clean.csv`
- `PrixLocationSurDemande_clean.csv`

*Documentation générée à partir du notebook `Code_Scraping_Groupe4.ipynb` — Groupe 4*
