# SimuBrAIn: Méta-formalisme stochastique pour la simulation cérébrale

!!! abstract "Projet IFT 3150 — Été 2026"

    **Étudiant :** Ryan Chahri, 
    **Superviseur académique :** Eugène Syriani (UdeM), 
    **Expert :** Alexandre Muzy (CNRS), 
    **Encadrant :** Abdelhamid (maîtrise).

## Contexte

SimuBrAIn est un projet de recherche international **UdeM (Canada) / CNRS (France)** dont l'objectif à long terme est de construire des **jumeaux numériques personnalisés du cerveau humain**. Le projet s'articule autour de trois axes complémentaires :

- **Axe 1: Modèles fondés sur la simulation** *(Canada, UdeM)*
- **Axe 2: Modèles fondés sur l'IA** *(France, CNRS)*
- **Axe 3: Modèles hybrides simulation-IA** *(collaboration)*

!!! note "Mon rôle"

    Mon projet s'inscrit dans l'**Axe 1, Objectif 2 : conception d'un nouveau méta-formalisme de modélisation stochastique à événements discrets.** Je travaille comme assistant de recherche aux côtés d'Abdelhamid, étudiant à la maîtrise, pour **implémenter en Python** le méta-formalisme conçu par l'équipe.

## Problématique

Le méta-formalisme **DEVS** (*Discrete Event System Specification*) fournit aujourd'hui une grammaire déclarative robuste pour modéliser des systèmes à événements discrets **déterministes**. Cependant, la modélisation du cerveau est intrinsèquement **stochastique** : l'activité neuronale (spikes, transmission synaptique) est mieux décrite par des processus aléatoires.

Le défi est double :

1. **Unification formelle**: il n'existe pas aujourd'hui de méta-formalisme unique capable d'embrasser de façon composable les divers formalismes stochastiques pertinents (processus de Poisson, chaînes de Markov, files d'attente).
2. **Passage à l'échelle**: un modèle du cerveau doit pouvoir assembler des neurones et synapses hétérogènes à très grande échelle (≈ 10¹¹ neurones, 10¹⁴ synapses), ce qui exige des modèles autonomes, composables et réutilisables, ainsi qu'un moteur de simulation efficace.

## Proposition et objectifs

Notre proposition est d'étendre DEVS pour qu'il devienne un **méta-formalisme stochastique unifié**, capable d'embrasser :

- les **processus ponctuels** (notamment Poisson, utilisés en neurosciences computationnelles) ;
- les **chaînes de Markov à temps continu** (CTMC) et processus Markov-modulés (MMPP) ;
- les **files d'attente** et processus GSMP (G-networks de Gelenbe).

Deux hypothèses de modélisation guident l'efficacité du simulateur :

!!! tip "Hypothèses clés"

    1. **Asynchronie temporelle**: deux spikes ne se produisent jamais exactement simultanément, ce qui élimine les conflits de synchronisation.
    2. **Suivi d'activité**: détection automatique des changements pertinents pour ne calculer que le nécessaire.

**Objectifs concrets :**

- Implémenter et valider des modèles atomiques stochastiques de référence (neurone de Poisson, transducer, modèle couplé).
- Étendre vers des formalismes plus riches (MMPP, G-networks) et en implémenter au moins un, validé.
- Implémenter un neurone stochastique exprimé directement avec le méta-formalisme et en mesurer la performance.

## Méthodologie envisagée

Le projet progresse de façon incrémentale, du modèle le plus simple vers le méta-formalisme complet :

| Étape | Phase | Contenu |
|:-----:|-------|---------|
| 1 | **Familiarisation (PyPDEVS)** | Neurone de Poisson, transducer, modèle couplé |
| 2 | **Extension queue-based** | File recevant les spikes |
| 3 | **Modèles stochastiques avancés** | Markov/MMPP et/ou G-networks de Gelenbe |
| 4 | **Méta-formalisme** | Neurone stochastique via le méta-formalisme unifié, puis benchmarks |

Le travail se fait en **Python** avec la librairie **PyPDEVS**, sous gestion de version **Git/GitHub**, avec rencontres hebdomadaires de supervision.

## Validation et évaluation

Chaque modèle implémenté est validé avant de passer à l'étape suivante :

- **Correction statistique**: comparaison du **taux empirique** (mesuré par simulation, vue comptage du processus) au **taux théorique** attendu, pour vérifier que les modèles reproduisent les bonnes distributions.
- **Performance**: benchmarks mesurant l'efficacité du simulateur, en particulier pour le neurone stochastique exprimé via le méta-formalisme.
- **Tests unitaires**: chaque modèle est vérifié par des tests unitaires sur le comportement attendu de la librairie Pydevs.
- **Reproductibilité**: dépôt GitHub finalisé avec README, tests et exemples reproductibles.
## Échéancier

!!! info "Suivi détaillé"
    Le suivi complet est disponible dans la page [Suivi de projet](suivi.md).

### Plan prévisionnel

| Période | Activités | Livrable / Jalon |
|---------|-----------|------------------|
| **Semaine 1**<br>4 → 8 mai | Setup du site web du cours · environnement conda + PyPDEVS · dépôt GitHub | Environnement opérationnel |
| **Semaines 2–4**<br>11 → 29 mai | Familiarisation PyPDEVS (neurone Poisson, transducer, modèle couplé) · validation taux empirique vs théorique · extension queue-based (file recevant les spikes) | Neurone de Poisson, transducer, modèle couplé et File recevant les spikes |
| **Semaines 5–8**<br>1ᵉʳ → 26 juin | Comprendre Markov et MMPP (*Markov-Modulated Poisson Process*) · comprendre les G-networks de Gelenbe · implémenter un de ces modèles + valider | Markov/MMPP et/ou G-networks de Gelenbe et Mise en commun I-II|
| **Semaines 9–12**<br>29 juin → 24 juillet | Implémentation du neurone stochastique via le méta-formalisme · benchmarks de performance | neurone stochastique via le méta-formalisme, benchmarks de performance et Mise en commun III |
| **Semaine 13**<br>27 → 31 juillet | Foire : kiosque de démonstration · finalisation du dépôt (README, exemples reproductibles) | Démo + repo finalisé |
| **Semaines 14–15**<br>3 → 14 août | Rédaction du rapport final · présentation finale (25 min) | Rapport + présentation |

## Navigation du site

- **[Suivi](suivi.md)**: Journal de bord et avancement
- **[Études préliminaires](analyse.md)**: Analyse et recherche
- **[Réalisation](realisation.md)**: Implémentation
- **[Évaluation](evaluation.md)**: Résultats et validation