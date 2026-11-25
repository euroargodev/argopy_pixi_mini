# argopy_pixi_mini

## Objectif

Ce prototype s'inscrit dans l'étude de solutions alternatives à la gestion actuelle des environnements et des dépendances d'Argopy.

L'objectif est de tester Pixi sur une version réduite d'Argopy afin d'évaluer :

- La gestion des dépendances et des environnements ;
- La gestion de plusieurs versions de Python ;
- La construction et la distribution du package ;
- L'intégration dans GitHub Actions ;
- La reproductibilité des environnements.

## 1. Initialisation du projet

Version de Pixi utilisée : `0.84.2`.

```bash
pixi init argopy_pixi_mini --format pyproject
```

Le projet est configuré à partir d'un fichier `pyproject.toml`.

Contrairement au prototype uv, qui utilise `uv_build`, j'ai conservé Hatchling comme backend de construction.

Le nom du package distribué est `argopy_pixi_mini`, mais le nom du module importable reste `argopy`.

## 2. Intégration des modules d'Argopy

J'ai repris les mêmes modules que dans le prototype `argopy_uv_mini` afin de travailler sur un périmètre fonctionnel identique.

```text
src/
└── argopy/
    ├── __init__.py
    ├── errors.py
    ├── options.py
    └── utils/
        ├── __init__.py
        ├── assets.py
        ├── casting.py
        ├── checkers.py
        ├── chunking.py
        ├── geo.py
        └── lists.py
```

Pour indiquer à Hatchling le module à inclure dans la wheel, j'ai ajouté la configuration suivante :

```toml
[tool.hatch.build.targets.wheel]
packages = ["src/argopy"]
```

## 3. Gestion des dépendances

### Dépendances principales

```bash
pixi add numpy pandas xarray fsspec
```

Les dépendances sont récupérées depuis Conda-forge et ajoutées dans la section `[tool.pixi.dependencies]` du fichier `pyproject.toml`.

Pixi génère également un fichier `pixi.lock` permettant de verrouiller les dépendances résolues.

### Dépendances de développement

```bash
pixi add --feature dev pytest
pixi workspace environment add dev --feature dev
```

Pixi utilise un système de *features* permettant de regrouper des dépendances et de les combiner pour créer différents environnements.

## 4. Intégration et exécution des tests

J'ai repris les mêmes tests que dans le prototype uv :

- `test_utils_chunking.py`
- `test_utils_checkers.py`

```bash
pixi run -e dev pytest tests/ -v
```

Résultat : **20 tests réussis.**

## 5. Construction de la wheel

Pixi étant principalement un gestionnaire de packages et d'environnements, j'ai utilisé le frontend de construction Python `build`, associé au backend Hatchling, pour générer une wheel.

```bash
pixi exec --spec python-build -- python -m build --wheel
```

La wheel générée est :

```text
dist/argopy_pixi_mini-0.1.0-py3-none-any.whl
```

J'ai ensuite vérifié son contenu :

```bash
unzip -l dist/*.whl
```

Les modules attendus sont bien présents dans la wheel.

À noter : `pixi build` est destiné à la construction de packages Conda, et non de wheels Python.

## 6. Installation indépendante de la wheel

Afin de vérifier que le package peut être utilisé indépendamment de Pixi, j'ai créé un environnement virtuel Python externe au projet.

```bash
python3 -m venv /tmp/argopy-pixi-test

/tmp/argopy-pixi-test/bin/python -m pip install \
    dist/argopy_pixi_mini-0.1.0-py3-none-any.whl
```

### Problème rencontré

Lors de la première installation, l'importation des modules échouait avec l'erreur suivante :

```text
ModuleNotFoundError: No module named 'numpy'
```

Les dépendances ajoutées avec `pixi add` étaient présentes dans `[tool.pixi.dependencies]`, mais pas dans les métadonnées Python du projet.

J'ai donc ajouté manuellement les dépendances dans la section `[project]` :

```toml
dependencies = [
    "numpy",
    "pandas",
    "xarray",
    "fsspec"
]
```

Après reconstruction et réinstallation de la wheel, les imports fonctionnent correctement.

### Observation

Dans cette configuration, il existe deux déclarations des dépendances :

- Les dépendances nécessaires à la gestion de l'environnement Pixi ;
- Les dépendances nécessaires à la distribution du package Python.

Cette séparation pourrait représenter une contrainte de maintenance pour Argopy, notamment lorsqu'il faut modifier les bornes de versions des dépendances.

## 7. Gestion de plusieurs plateformes

J'ai ajouté les plateformes suivantes dans la configuration du workspace :

```toml
[tool.pixi.workspace]
channels = ["conda-forge"]
platforms = [
    "linux-64",
    "win-64",
    "osx-64",
    "osx-arm64"
]
```

Puis j'ai actualisé le lockfile :

```bash
pixi lock
```

Un seul fichier `pixi.lock` permet ainsi de stocker les résolutions associées aux différentes plateformes et aux différents environnements.

## 8. Gestion de plusieurs versions de Python

J'ai créé deux features permettant de définir les versions de Python utilisées pour les tests.

```toml
[tool.pixi.feature.py312.dependencies]
python = "3.12.*"

[tool.pixi.feature.py313.dependencies]
python = "3.13.*"

[tool.pixi.environments]
dev = ["dev"]
py312 = ["dev", "py312"]
py313 = ["dev", "py313"]
```

Vérification des versions :

```bash
pixi run --locked -e py312 python --version
pixi run --locked -e py313 python --version
```

Pixi permet également de définir des tâches directement dans le `pyproject.toml` :

```toml
[tool.pixi.tasks]
test = "pytest tests/ -v"
```

Les tests peuvent alors être exécutés avec :

```bash
pixi run --locked -e py312 test
pixi run --locked -e py313 test
```

Les 20 tests passent dans les deux environnements.

## 9. Intégration continue

J'ai mis en place un workflow GitHub Actions permettant de tester le projet sur trois systèmes d'exploitation et deux versions de Python.

| Système | Python 3.12 | Python 3.13 |
|---|---|---|
| Ubuntu | Passed | Passed |
| Windows | Passed | Passed |
| macOS | Passed | Passed |

Chaque job réalise les opérations suivantes :

- Installation de Pixi ;
- Création de l'environnement et exécution des tests à partir du lockfile ;
- Affichage des packages installés ;
- Construction de la wheel.

L'ensemble des six jobs passe.

La construction de la wheel utilise cependant un environnement temporaire créé par `pixi exec`, indépendant de l'environnement de test sélectionné dans la matrice.

## 10. Analyse des environnements

### Versions des dépendances

L'analyse des logs GitHub Actions sous Ubuntu donne les résultats suivants :

| Package | py312 | py313 |
|---|---|---|
| Python | 3.12.16 | 3.13.18 |
| NumPy | 2.5.5 | 2.5.5 |
| Pandas | 3.1.1 | 3.1.1 |
| Xarray | 2026.9.0 | 2026.9.0 |
| Fsspec | 2026.9.0 | 2026.9.0 |
| Pytest | 9.2.0 | 9.2.0 |

Les deux environnements utilisent les mêmes versions des principales dépendances, mais certains builds diffèrent.

Par exemple :

| Package | Build py312 | Build py313 |
|---|---|---|
| NumPy | py312he827f4e_1 | py313hb5f73ae_1 |
| Pandas | np2py312h91ec553_0 | np2py313h4900a6c_0 |
| Xarray | pyh5ded981_0 | pyh5ded981_0 |

Pixi sélectionne des builds compatibles avec chaque version de Python.

### Dépendances natives

Les logs montrent également la présence de bibliothèques natives telles que :

- OpenBLAS ;
- BLAS ;
- LAPACK ;
- libgfortran ;
- libgcc ;
- OpenSSL.

Ces dépendances sont résolues automatiquement par Pixi.

### Coexistence de Conda et PyPI

Le package local `argopy_pixi_mini` est installé en mode éditable comme package Python, tandis que ses dépendances sont installées depuis Conda-forge.

Les logs ne montrent pas de double installation des dépendances depuis PyPI et Conda-forge.

### Occupation de l'espace disque

J'ai remarqué que la commande `pixi list` affichait une taille particulièrement importante pour le package local :

- Environnement py312 : 1.02 GiB ;
- Environnement py313 : 879.38 MiB.

J'ai donc vérifié la taille des différents répertoires du projet en local :

```bash
du -sh .
du -sh .pixi
du -sh src/
du -sh dist/
```

| Répertoire | Taille |
|---|---:|
| Projet complet | 1,6 G |
| .pixi/ | 1,6 G |
| src/ | 536 K |
| dist/ | 28 K |

La quasi-totalité de l'espace disque occupé par le projet provient du répertoire `.pixi`.

Mon hypothèse est que la taille affichée par `pixi list` pour le package éditable tient compte du contenu du répertoire local, notamment des environnements Pixi. Cette hypothèse reste à confirmer.

Ces résultats ne permettent pas encore de comparer la consommation disque de Pixi à celle d'uv.

## 11. Bilan

### Points positifs

- Gestion des dépendances Python et natives ;
- Gestion de plusieurs environnements au sein d'un même projet ;
- Gestion multiplateforme avec un seul lockfile ;
- Possibilité de combiner plusieurs features ;
- Définition de tâches réutilisables ;
- Intégration fonctionnelle dans GitHub Actions ;
- Distribution possible sous forme de wheel Python.

### Points de vigilance

- Double déclaration des dépendances dans la configuration actuelle ;
- Utilisation d'un frontend et d'un backend supplémentaires pour construire une wheel Python ;
- Environnement de construction temporaire non verrouillé par le lockfile du projet ;
- Occupation disque à étudier ;
- Configuration des features et des environnements plus détaillée.

### Perspectives

Ce prototype valide la faisabilité technique de l'utilisation de Pixi sur un sous-ensemble d'Argopy.

Il reste à étudier :

- La gestion des environnements pinned et upstream ;
- La mise à jour des dépendances ;
- Les performances par rapport à uv ;
- La migration de l'intégralité d'Argopy ;
- Les contraintes de maintenance à long terme.