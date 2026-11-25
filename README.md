# staph-amr-nanopore-nf
---
title: "Prédiction génomique de la résistance aux antibiotiques chez *Staphylococcus aureus* à partir de séquençage Oxford Nanopore"
subtitle: "Rapport d'étape v1 : conception du pipeline et préparation des données"
author: "Aïnata Zida"
date: "Novembre 2026"
---

# Prédiction génomique de la résistance aux antibiotiques chez *Staphylococcus aureus* à partir de séquençage Oxford Nanopore

**Rapport d'étape v1 : conception du pipeline et préparation des données**

| | |
| --- | --- |
| **Auteure** | Aïnata Zida, ingénieure bio-informaticienne |
| **Version** | v1, novembre 2026 |
| **Statut** | Conception et préparation des données terminées ; implémentation Nextflow en cours |
| **Nature du projet** | Projet personnel, sur données publiques |

---

## Résumé

Ce projet vise à identifier les déterminants génétiques de la résistance aux antibiotiques chez *Staphylococcus aureus* à partir de lectures longues Oxford Nanopore, et à évaluer leur concordance avec des antibiogrammes de laboratoire.

Ce rapport couvre les trois premières phases du projet :

- **Cadrage scientifique** : formulation de l'objectif et des critères de sélection des données, à partir de l'analyse critique de la littérature.
- **Sélection et curation des données** : un jeu public de 18 isolats séquencés en Nanopore R10.4.1 a été retenu. L'audit de ses métadonnées a révélé des incohérences dans 7 des antibiotiques documentés. **4 antibiotiques fiables** ont été conservés.
- **Conception du pipeline** : 7 étapes, des règles de contrôle qualité chiffrées et sourcées, et un choix d'outils justifié par des benchmarks publiés et par les contraintes matérielles.

Aucun résultat d'analyse n'est encore présenté : l'implémentation en Nextflow constitue la prochaine étape.

---

## 1. Objectif

> Identifier les déterminants génétiques de la résistance aux antibiotiques (gènes acquis, mutations chromosomiques, plasmides) chez *Staphylococcus aureus* à partir de données Nanopore, afin d'**évaluer leur concordance avec les antibiogrammes** et d'**expliquer les discordances**.

La question centrale est donc : **dans quelle mesure le génome permet-il de prédire le phénotype de résistance, et comment expliquer les cas où la prédiction diverge de l'antibiogramme ?**

## 2. État d'avancement

| Phase | Contenu | Livrable | Statut |
| --- | --- | --- | --- |
| 1. Cadrage scientifique | Objectif, revue de la littérature, critères de données | Fiche projet | Terminée |
| 2. Données | Sélection, audit et curation d'un jeu public | Jeu de données justifié | Terminée |
| 3. Stratégie | Schéma d'analyse, règles de QC, choix des outils | Pipeline défendable | Terminée |
| 4. Implémentation | Nextflow DSL2, conteneurs, tests sur un isolat | Pipeline minimal fonctionnel | En cours |
| 5. Analyse | Concordance génotype / phénotype, discordances | Interprétation critique | En attente |
| 6. Reproductibilité | Tests, documentation, publication | Dépôt GitHub reproductible | En attente |

---

## 3. Cadrage scientifique

### 3.1 Étude de référence

La recherche bibliographique (PubMed) a retenu un article modèle selon deux critères : il combine **analyse génomique et antibiogrammes, avec une étape d'intégration**, et ses **données brutes sont déposées**.

**Kannekens-Jager et al. (2024)** décrivent une épidémie de SARM dans une clinique équine néerlandaise : 16 isolats séquencés en Nanopore (MinION, R9.4.1), CMI mesurées par microdilution (seuils CLSI), détection par ResFinder, comparaison classe d'antibiotiques par classe. L'article a servi de modèle de **méthode**, et ses **limites** ont fixé les critères de données :

| Limite de l'étude de référence | Conséquence pour ce projet |
| --- | --- |
| 16 isolats, tous SARM | Inclure des isolats **sensibles** comme contrôles |
| Deux clones épidémiques (11 ST398, 5 ST612) : observations non indépendantes | Exiger une **diversité génétique** (plusieurs ST) |
| Chimie Nanopore R9.4.1, plus sujette aux erreurs | Privilégier la chimie récente (R10.4.1) |

### 3.2 Règles de comparaison génotype / phénotype

L'analyse de l'étude de référence a fait apparaître quatre règles. L'une vient d'un cas qu'elle présente comme concordant : chez ST398, le gène *dfrK* (résistance au triméthoprime) est présent, alors que l'isolat est sensible à l'association triméthoprime-sulfaméthoxazole.

1. Un gène isolé ne garantit pas la résistance à une **association** de molécules.
2. La résistance ne provient pas uniquement de gènes acquis : les **mutations chromosomiques** doivent être recherchées.
3. La comparaison porte sur **la molécule exacte testée**, pas sur une classe d'antibiotiques.
4. Le génotype indique un **potentiel** ; l'antibiogramme mesure un **résultat**.

### 3.3 Grille d'analyse des discordances

Une discordance entre prédiction et antibiogramme sera analysée à trois niveaux, sans présumer que l'un des deux résultats est exact :

| Niveau | Causes possibles |
| --- | --- |
| Biologique | Gène non exprimé ; gène tronqué ou non fonctionnel ; association de molécules dont une seule est ciblée |
| Bio-informatique | Contamination par une autre espèce porteuse du gène ; seuils de détection inadaptés ; perte d'un plasmide à l'assemblage ; erreur de séquençage |
| Laboratoire | Inoculum inadapté ; résistance hétérogène ; référentiel d'interprétation (CLSI ou EUCAST) |

---

## 4. Données

### 4.1 Stratégie de recherche

Les lectures brutes sont déposées dans SRA ou ENA, mais les antibiogrammes y figurent rarement. La recherche est donc **partie des articles** combinant Nanopore et antibiogrammes, plutôt que des archives de séquences, où la plupart des projets n'ont pas de phénotype associé.

### 4.2 Jeu de données retenu

**Dabernig-Heinz et al. (2025), *Scientific Data*** : jeu de données Nanopore de quatre espèces bactériennes (BioProject PRJNA1091452), avec métadonnées et antibiogrammes déposés sur Zenodo.

| Critère | Évaluation |
| --- | --- |
| Lectures Nanopore accessibles | Cellules R10.4.1, chimie V14 ; lectures déjà basecallées sur SRA (pas de basecalling, donc pas de GPU nécessaire) |
| Antibiogramme par isolat | CMI et interprétations S/I/R |
| Référentiel | EUCAST 2023 |
| Valeurs de CMI | Permettent de vérifier les interprétations |
| Diversité génétique | Isolats d'origines variées (Allemagne, Nigeria, États-Unis, Royaume-Uni ; 2006 à 2020), à confirmer par MLST |
| Souches de référence | COL, Newman et ATCC 10832, dont les génomes complets sont publiés |

### 4.3 Curation des métadonnées

L'audit des deux fichiers de métadonnées a conduit à six décisions :

| Constat | Décision |
| --- | --- |
| En-tête de colonne fusionné, en contradiction avec la note du fichier sur les accessions « nouvelles » | Hypothèse vérifiée sur SRA (ordre chronologique des accessions) : une seule série d'accessions est propre à ce jeu |
| Deux séries d'accessions, issues de deux laboratoires (Lab1 et Lab2), pour 3 isolats seulement | Analyse principale sur la série commune aux 18 isolats, pour éviter un **effet de lot** ; l'autre série sert de **test de reproductibilité** |
| SA70 est un *Staphylococcus simulans*, sans antibiogramme | Exclu. Le pipeline vérifiera l'espèce de chaque assemblage |
| Profondeur hétérogène (ex. SA62 : environ 39x pour la série retenue) | Profondeur mesurée par isolat ; exclusion sous 20x |
| Interprétations S/R incompatibles avec les CMI pour 7 antibiotiques (ex. vancomycine « R » avec CMI ≤ 0,5 mg/L) | Antibiotiques exclus (voir ci-dessous) |
| Antibiotiques sans contraste résistant / sensible | Exclus : la concordance ne peut y être évaluée |

**Audit des antibiogrammes.** Chaque interprétation a été confrontée à sa CMI et aux seuils EUCAST. Pour 7 antibiotiques, les étiquettes S/R semblent avoir glissé d'une ligne à l'autre du tableau. Par exemple, les isolats classés « R » au triméthoprime-sulfaméthoxazole sont exactement les isolats résistants à l'érythromycine. Les CMI paraissent restées en place, mais faute de pouvoir l'établir, ces antibiotiques ont été exclus. Une réinterprétation à partir des CMI pourra faire l'objet d'une analyse complémentaire, présentée à part.

**Antibiotiques retenus** (18 isolats) :

| Antibiotique | Résistants / sensibles | Effectif du plus petit groupe | Déterminant principal attendu |
| --- | --- | --- | --- |
| Céfoxitine / oxacilline | 12 / 6 | 6 | *mecA* |
| Benzylpénicilline | 14 / 4 | 4 | *blaZ* |
| Érythromycine | 4 / 14 | 4 | *ermC* |
| Gentamicine | 3 / 15 | 3 | *aac(6')-Ie-aph(2'')-Ia* |

Ces quatre déterminants sont des **gènes acquis**, portés par des éléments mobiles. Selon l'isolat, ils peuvent se trouver sur un plasmide ou intégrés au chromosome (CARD ; Lyon et Skurray, 1987). L'assemblage doit donc capturer à la fois le chromosome et les petits plasmides.

---

## 5. Stratégie d'analyse

### 5.1 Schéma du pipeline

```
FASTQ (accessions SRA, 18 isolats)   ← nf-core/fetchngs
   │
   ▼
1. Mesure de la profondeur          SeqKit       → profondeur par isolat ; exclusion < 20x
   │
   ▼
2. Filtrage, assemblage, polissage  Hybracter    → génome FASTA (chromosome + plasmides)
   │
   ▼
3. Contrôle qualité et espèce       CheckM2      → statut QC par isolat
                                    FastANI
   │      └─ Validation : souches de référence vs génomes publiés   dnadiff (MUMmer4)
   ▼
   ├──► 4a. Typage                  mlst         → isolat → ST
   └──► 4b. Résistances             ResFinder    → phénotype prédit par antibiotique
                    │
                    ▼
5. Comparaison                      R            ← antibiogrammes (4 antibiotiques)
   → tableau 2 × 2 par antibiotique, avec profondeur et ST de chaque isolat
```

Le pipeline sera écrit en **Nextflow DSL2**, avec un module et un conteneur par outil, et exécuté dans **GitHub Codespaces** (4 cœurs, 16 Go de RAM, 15 Go de stockage).

### 5.2 Règles de contrôle qualité

Les règles sont fixées **avant** l'examen des données, pour ne pas être ajustées aux résultats. Elles sont exhaustives et chiffrées. Chaque isolat reçoit un statut et une raison : aucun n'est écarté sans trace.

| Critère | Conforme | Source |
| --- | --- | --- |
| Profondeur de séquençage | ≥ 20x | FAQ de l'assembleur Flye |
| Taille totale de l'assemblage | 2,2 à 3,4 Mb | NCBI, fourchette attendue pour *S. aureus* (taxid 1280) |
| Taille du plus grand contig | 2,2 à 3,4 Mb (chromosome complet) | NCBI |
| Complétude / contamination | ≥ 99 % / ≤ 5 % | Nagy et al. (2026) |
| Espèce | ANI ≥ 95 % avec NCTC 8325, sur une fraction alignée suffisante | Seuil d'espèce usuel |

Au-dessus de 20x, l'hétérogénéité de profondeur est acceptée et documentée. On vérifiera si les discordances se concentrent sur les isolats les moins profonds.

### 5.3 Choix des outils

| Étape | Outil retenu | Justification | Alternatives écartées |
| --- | --- | --- | --- |
| Téléchargement | nf-core/fetchngs | Récupération par accession, vérification MD5, tableau d'échantillons | Modules écrits à la main |
| Profondeur | SeqKit | Statistiques de lectures légères | — |
| Assemblage | **Hybracter** (mode long) | Flye pour le chromosome, Plassembler pour les petits plasmides : récupération des plasmides de 96 % contre 56 % pour Flye seul (Nagy et al., 2026) | **Flye seul** : peine avec les plasmides < 10 kb, or pE194 (*ermC*) fait 3 728 pb. **Autocycler** : environ 146 h-CPU par isolat, soit environ 22 mois de quota de calcul pour 18 isolats |
| QC assemblage | CheckM2 | Complétude et contamination ; la longueur moyenne des gènes sert d'indicateur indirect des erreurs d'indel | — |
| Espèce | FastANI | Question binaire, outil léger | **Kraken2** : base réduite de 8 Go, incompatible avec le quota de stockage |
| Typage | mlst | Diversité génétique ; lancé après la vérification de l'espèce | — |
| Résistances | **ResFinder** | Meilleure exactitude équilibrée (0,80, moyenne sur 27 espèces ; Ring et al., 2026) ; prédiction de phénotype par antibiotique | **DeepARG** : pas de détection de mutations, spécificité de 0,52. **AMRFinderPlus** : performances inférieures dans ce benchmark |
| Validation | dnadiff (MUMmer4) | Liste SNP, indels et régions manquantes par rapport au génome publié | — |
| Comparaison | Script R | Tableaux 2 × 2 et visualisations | — |

### 5.4 Validation prévue

- **Exactitude de l'assemblage** : les assemblages des trois souches de référence (COL, Newman, ATCC 10832) seront comparés à leurs génomes publiés. Cela permettra de mesurer directement les erreurs, en particulier celles liées à la **méthylation de l'ADN**, signalées en R10.4.1 par l'étude multicentrique d'origine de ces données.
- **Reproductibilité expérimentale** : pour SA62, SA63 et SA67, séquencés par deux laboratoires, les prédictions de résistance seront comparées entre les deux jeux de lectures.
- **Reproductibilité informatique** : versions d'outils figées dans des conteneurs, exécution par Nextflow, rapports d'exécution (`trace`, `timeline`).

---

## 6. Limites identifiées

- **Effectifs réduits** : le plus petit groupe compte 3 isolats (gentamicine). Les concordances seront rapportées antibiotique par antibiotique, avec leurs effectifs, et considérées comme **indicatives**.
- **Quatre antibiotiques seulement**, tous liés à des gènes acquis : la détection des mutations chromosomiques ne sera pas évaluée sur ce jeu.
- **Benchmarks multi-espèces** : les performances citées pour ResFinder et Hybracter sont des moyennes sur d'autres espèces ou plusieurs espèces, pas des mesures chez *S. aureus*.
- **Métadonnées partiellement incohérentes** : les antibiotiques exclus pourront faire l'objet d'une analyse complémentaire si les auteurs confirment l'origine de l'erreur.
- **Ressources limitées** (16 Go de RAM, 15 Go de stockage) : traitement par lots, consommation à mesurer sur un isolat test.

## 7. Prochaines étapes

1. Mise en place de l'environnement (GitHub Codespaces, Nextflow, Docker).
2. Téléchargement d'un seul isolat test (SA62, souche de référence Newman).
3. Construction du pipeline **module par module**, chaque module validé avant l'ajout du suivant ; mesure de la taille des conteneurs et de la mémoire consommée.
4. Passage à l'échelle des 18 isolats, par lots.
5. Analyse de concordance et interprétation des discordances.
6. Tests automatisés, documentation utilisateur et publication du dépôt.

---

## Références

**Étude de référence et données**

- Kannekens-Jager M.M. et al. (2024). Methicillin-resistant *Staphylococcus aureus* outbreak in a Dutch equine referral clinic. *Journal of Medical Microbiology*, 73(8):001873. [doi:10.1099/jmm.0.001873](https://doi.org/10.1099/jmm.0.001873)
- Dabernig-Heinz J., Galeone V., Sedaghatjoo S., Steinmetz I., Kohler C., Hölzer M., Wagner G.E. (2025). A whole-genome sequencing dataset of nanopore raw signals for bacterial genotyping and methylation analysis. *Scientific Data*. [Article](https://www.nature.com/articles/s41597-025-06319-4)

**Benchmarks**

- Nagy D. et al. (2026). Nanopore long-read-only genome assembly of clinical Enterobacterales isolates is complete and accurate. *Microbial Genomics*, 12(2):001631. [doi:10.1099/mgen.0.001631](https://doi.org/10.1099/mgen.0.001631)
- Ring N. et al. (2026). Benchmarking nanopore-based strategies for antimicrobial resistance prediction. *Microbial Genomics*, 12(11):001885. [doi:10.1099/mgen.0.001885](https://doi.org/10.1099/mgen.0.001885)

**Déterminants de résistance**

- Lyon B.R., Skurray R. (1987). Antimicrobial resistance of *Staphylococcus aureus*: genetic basis. *Microbiological Reviews*, 51(1):88–134.
- Horinouchi S., Weisblum B. (1982). Nucleotide sequence and functional map of pE194. *Journal of Bacteriology*, 150(2):804–814. [doi:10.1128/jb.150.2.804-814.1982](https://doi.org/10.1128/jb.150.2.804-814.1982)
- Khan S.A., Novick R.P. (1983). Complete nucleotide sequence of pT181, a tetracycline-resistance plasmid from *Staphylococcus aureus*. *Plasmid*. [doi:10.1016/0147-619X(83)90039-2](https://doi.org/10.1016/0147-619X(83)90039-2)
- [CARD](https://card.mcmaster.ca), Comprehensive Antibiotic Resistance Database.

**Ressources et outils**

- [NCBI, Genome Size Check](https://ncbi.nlm.nih.gov/assembly/help/genome-size-check) ; [FAQ de Flye](https://github.com/mikolmogorov/Flye/blob/flye/docs/FAQ.md)
- Documentation : [Hybracter](https://hybracter.readthedocs.io/en/latest/), [nf-core/fetchngs](https://nf-co.re/fetchngs/latest/docs/usage/), [FastANI](https://github.com/ParBLiSS/FastANI), [MUMmer4](https://github.com/mummer4/mummer), [ResFinder](https://github.com/genomicepidemiology/resfinder)
