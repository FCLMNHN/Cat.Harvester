# Documentation technique — `Harvester_pipeline.py`

## Vue d'ensemble

`Harvester_pipeline.py` est un pipeline de moissonnage automatisé de métadonnées scientifiques. Il orchestre l'ensemble du flux de données depuis la collecte dans des entrepôts distants jusqu'à la publication sur la plateforme PNDB, en passant par la transformation et le transfert FTP.

Le pipeline prend en charge deux protocoles de collecte et gère plusieurs dizaines d'entrepôts institutionnels français et internationaux.

---

## Architecture du pipeline

Pour chaque entrepôt activé, le pipeline exécute séquentiellement les étapes suivantes :

```
Entrepôt distant
      │
      ▼
[Étape 1] Moissonnage  ──── OAI-PMH → fichier .xml
                        └── Direct   → fichier .json
      │
      ▼
[Étape 2] Normalisation XML (mode OAI-PMH uniquement)
      │
      ▼
[Étape 3] Téléchargement des métadonnées JSON individuelles
      │
      ▼
[Post-traitement] Transform_JSONToCSV.py  →  output_mapping*.csv
      │
      ▼
[Post-traitement] Transfert FTP  →  répertoire ODS/
      │
      ▼
[Post-traitement] Appel API PNDB  →  déclenchement du harvester
```

---

## Prérequis

### Dépendances Python

```
requests
python-dotenv
```

Les modules `xml.etree.ElementTree`, `os`, `time`, `argparse`, `datetime`, `re`, `json`, `urllib`, `subprocess`, `ftplib`, `glob` font partie de la bibliothèque standard Python.

### Variables d'environnement (fichier `.env`)

Toutes les credentials sont externalisées dans un fichier `.env` placé à la racine du projet :

| Variable        | Description                              |
|-----------------|------------------------------------------|
| `FTP_HOST`      | Hôte du serveur FTP                      |
| `FTP_PORT`      | Port FTP (entier)                        |
| `FTP_USER`      | Nom d'utilisateur FTP                    |
| `FTP_PASSWORD`  | Mot de passe FTP                         |
| `FTP_REMOTE_DIR`| Répertoire distant cible sur le serveur  |
| `API_TOKEN`     | Clé d'API pour l'authentification PNDB   |

### Scripts externes requis

Chaque dossier `Moissonnage XYZ/` doit contenir un script `Transform_JSONToCSV.py` qui transforme les fichiers JSON collectés en fichier `output_mapping*.csv`. Ce script est invoqué automatiquement en post-traitement.

---

## Structure des fichiers

```
Cat.Harvester/
├── Harvester_pipeline.py          ← ce script
├── .env                           ← variables d'environnement (non versionné)
├── Moissonnage RDG.INRAE/
│   ├── Transform_JSONToCSV.py
│   └── RDG.INRAE_JSONs/           ← JSONs collectés
├── Moissonnage PANGAEA/
│   ├── Transform_JSONToCSV.py
│   └── PANGAEA_JSONs/
└── ...
```

`BASE_DIR` est défini dynamiquement comme le répertoire contenant le script (`os.path.dirname(os.path.abspath(__file__))`), ce qui rend tous les chemins portables quel que soit l'hébergeur.

---

## Fonctions

### `harvest_all_oai_records`

```python
harvest_all_oai_records(base_url, metadata_prefix, set_spec, use_set, output_dir, output_filename) → str | None
```

Moissonne un entrepôt OAI-PMH via le verbe `ListRecords` en gérant la pagination par jetons de reprise (`resumptionToken`).

**Paramètres**

| Paramètre         | Type   | Description |
|-------------------|--------|-------------|
| `base_url`        | str    | URL de base de l'entrepôt OAI-PMH |
| `metadata_prefix` | str    | Format des métadonnées (ex. `oai_dc`, `dataverse_json`) |
| `set_spec`        | str    | Identifiant du set à moissonner |
| `use_set`         | bool   | Si `False`, le paramètre `set` est omis (tous les enregistrements) |
| `output_dir`      | str    | Répertoire de sortie |
| `output_filename` | str    | Nom du fichier XML de sortie (défaut : `all_oai_records.xml`) |

**Retour** : chemin du fichier XML créé, ou `None` si aucun enregistrement ou erreur.

**Comportement de récupération d'erreur** : en cas de réponse XML mal formée, le script tente d'extraire le jeton de reprise par expression régulière pour poursuivre le moissonnage.

**Pause** : 1 seconde entre chaque requête paginée (`time.sleep(1)`).

---

### `normalize_xml`

```python
normalize_xml(input_filepath) → str | None
```

Normalise un fichier XML OAI-PMH en corrigeant le double-encodage des entités HTML (`&amp;amp;` → `&amp;`) et en ré-indentant le document.

**Paramètres**

| Paramètre        | Type | Description |
|------------------|------|-------------|
| `input_filepath` | str  | Chemin du fichier XML à normaliser (modifié sur place) |

**Retour** : le chemin du fichier normalisé, ou `None` en cas d'erreur.

---

### `download_metadata_from_oai_xml`

```python
download_metadata_from_oai_xml(input_filepath, output_directory, target_subject=None)
```

Extrait les URLs de téléchargement depuis un fichier XML OAI-PMH et télécharge les métadonnées JSON correspondantes. Supporte deux sources :

- **Dataverse** : URL extraite de l'attribut `directApiCall` de l'élément `<metadata>`.
- **PANGAEA** : URL reconstruite à partir du DOI contenu dans l'identifiant OAI (`oai:pangaea.de:doi:…`), en appelant `https://doi.pangaea.de/{doi}?format=metadata_jsonld`.

**Paramètres**

| Paramètre          | Type       | Description |
|--------------------|------------|-------------|
| `input_filepath`   | str        | Fichier XML source |
| `output_directory` | str        | Répertoire de sortie des JSONs |
| `target_subject`   | str / None | Filtre thématique (non implémenté dans ce cas, réservé) |

---

### `harvest_direct_search`

```python
harvest_direct_search(base_url, question, subtree, output_dir, output_filename) → str
```

Moissonne un entrepôt Dataverse via l'API Search (`/api/search`) avec pagination automatique, par pages de 10 résultats.

**Paramètres**

| Paramètre         | Type | Description |
|-------------------|------|-------------|
| `base_url`        | str  | URL de base de l'entrepôt (ex. `https://entrepot.recherche.data.gouv.fr/oai`) |
| `question`        | str  | Requête de recherche (ex. `*`, ou terme spécifique) |
| `subtree`         | str  | Sous-arborescence Dataverse à interroger |
| `output_dir`      | str  | Répertoire de sortie |
| `output_filename` | str  | Nom du fichier JSON de listing |

**Retour** : chemin du fichier JSON de listing créé.

**Pause** : 0,5 seconde entre les pages (`time.sleep(0.5)`).

---

### `_process_downloads`

```python
_process_downloads(urls_with_ids, output_directory, target_subject=None)
```

Fonction interne de téléchargement. Itère sur une liste de tuples `(url, persistent_id)`, télécharge chaque JSON Dataverse, applique un filtre thématique si `target_subject` est défini, et sauvegarde les fichiers.

**Filtrage thématique** : compare les valeurs du champ `subject` (dans `datasetVersion.metadataBlocks.citation.fields`) aux mots-clés de `target_subject` (liste séparée par des virgules, insensible à la casse).

---

### `download_metadata_from_direct_json`

```python
download_metadata_from_direct_json(input_filepath, output_directory, base_url, target_subject=None)
```

Lit le fichier de listing JSON produit par `harvest_direct_search`, construit les URLs d'export Dataverse (`/api/datasets/export?exporter=dataverse_json&persistentId=…`) et délègue le téléchargement à `_process_downloads`.

---

### `rename_json_to_done`

```python
rename_json_to_done(filepath)
```

Renomme le fichier de listing JSON en ajoutant l'extension `.done` une fois le traitement terminé, afin d'éviter un re-traitement accidentel.

---

### `run_transform_and_upload`

```python
run_transform_and_upload(repo_name: str, output_dir: str, harvester_uid: str)
```

Orchestre le post-traitement après moissonnage :

1. **Transformation** : invoque `Transform_JSONToCSV.py` (situé dans le répertoire parent de `output_dir`) via `subprocess.run`, avec un timeout de 10 minutes.
2. **Détection du CSV** : recherche dans le répertoire parent tout fichier `output_mapping*.csv` dont la date de modification est celle du jour.
3. **Transfert FTP** : envoie le CSV vers le répertoire `ODS/` du serveur FTP configuré.
4. **Appel API** : déclenche le harvester PNDB correspondant via `POST https://www.pndb.fr/api/automation/v1.0/harvesters/{harvester_uid}/start/`.

**Comportement si `harvester_uid` est vide** : l'appel API est ignoré (avertissement affiché).

---

## Configuration des entrepôts

Les entrepôts sont déclarés dans le dictionnaire `available_repositories` (dans le bloc `__main__`). Seuls ceux avec `"harvest": True` sont traités.

### Paramètres d'un entrepôt

| Clé               | Type   | Description |
|-------------------|--------|-------------|
| `harvester_uid`   | str    | Identifiant du harvester PNDB (vide si non applicable) |
| `harvest`         | bool   | `True` pour activer le moissonnage |
| `Type_moissonnage`| str    | `"OAI-PMH"` ou `"Direct"` |
| `base_url`        | str    | URL racine de l'entrepôt |
| `metadata_prefix` | str    | Préfixe de métadonnées OAI-PMH |
| `set_spec`        | str    | Identifiant du set OAI-PMH |
| `use_set`         | bool   | Activer le filtrage par set |
| `question`        | str    | Requête API Search (mode Direct) |
| `subtree`         | str    | Sous-arborescence Dataverse (mode Direct) |
| `description`     | str    | Libellé lisible de l'entrepôt |
| `subject`         | str    | Filtre thématique (virgules pour plusieurs valeurs) |
| `output_dir`      | str    | Chemin de sortie (relatif à `BASE_DIR`) |

### Entrepôts référencés

| Clé                    | Description                                      | Mode       |
|------------------------|--------------------------------------------------|------------|
| `recherche_data_gouv`  | Entrepôt principal recherche.data.gouv.fr        | OAI-PMH    |
| `inrae`                | INRAE                                            | OAI-PMH    |
| `pangaea`              | PANGAEA                                          | OAI-PMH    |
| `datacite_ubfc`        | DataCite UBFC                                    | OAI-PMH    |
| `uga`                  | Université Grenoble Alpes                        | OAI-PMH    |
| `umontpellier`         | Université Montpellier                           | OAI-PMH    |
| `ulille`               | Université Lille                                 | OAI-PMH    |
| `sorbonne-univ`        | Sorbonne Université                              | OAI-PMH    |
| `upoitiers`            | Université Poitiers                              | Direct     |
| `ird`                  | IRD (DataSuds)                                   | OAI-PMH    |
| `ird_geo`              | GéoIRD                                           | OAI-PMH    |
| `cirad`                | CIRAD                                            | OAI-PMH    |
| `ubfc`                 | Université Bourgogne Franche-Comté               | OAI-PMH    |
| `univ-rennes`          | Université de Rennes                             | Direct     |
| `cnrs`                 | CNRS                                             | Direct     |
| `upsaclay`             | Université Paris-Saclay                          | Direct     |
| `data-bfc`             | UBFC (API Search)                                | Direct     |
| `ubo`                  | Université de Bretagne Occidentale               | Direct     |
| `ephe-psl`             | EPHE                                             | Direct     |
| `zenodo`               | Zenodo                                           | OAI-PMH    |
| `hal`                  | HAL                                              | OAI-PMH    |

---

## Flux d'exécution détaillé (bloc `__main__`)

1. Lecture du dictionnaire `available_repositories` et sélection des entrepôts avec `harvest: True`.
2. **Nettoyage** : suppression de tous les fichiers présents dans chaque `output_dir` activé.
3. Pour chaque entrepôt sélectionné :
   - **Mode `Direct`** : `harvest_direct_search` → `download_metadata_from_direct_json` → `rename_json_to_done`
   - **Mode `OAI-PMH`** : `harvest_all_oai_records` → `normalize_xml` → `download_metadata_from_oai_xml`
   - Dans les deux cas : `run_transform_and_upload`
4. Les erreurs par entrepôt sont interceptées individuellement ; le pipeline passe à l'entrepôt suivant sans s'arrêter.

---

## Gestion des erreurs

| Situation | Comportement |
|-----------|--------------|
| Erreur réseau HTTP | Interruption du moissonnage pour l'entrepôt concerné |
| XML mal formé | Tentative de récupération du jeton par regex, sinon arrêt |
| `Transform_JSONToCSV.py` absent | Avertissement, post-traitement ignoré |
| Timeout subprocess (> 10 min) | Avertissement, post-traitement ignoré |
| Aucun CSV du jour trouvé | Avertissement, FTP ignoré |
| Erreur FTP | Erreur loggée, passage au CSV suivant |
| `harvester_uid` vide | Avertissement, appel API ignoré |
| Erreur inattendue (entrepôt) | Passage à l'entrepôt suivant |

---

## Activation d'un entrepôt

Pour activer un entrepôt, passer son champ `harvest` de `False` à `True` dans le dictionnaire `available_repositories` :

```python
"inrae": {
    "harvest": True,   # ← activer
    ...
}
```

Pour ajouter un nouvel entrepôt, créer une entrée avec les mêmes clés, créer le répertoire de sortie correspondant et y placer un script `Transform_JSONToCSV.py`.