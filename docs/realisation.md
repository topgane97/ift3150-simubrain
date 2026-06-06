---
title: Travail réalisé
---

<style>
    @media screen and (min-width: 76em) {
        .md-sidebar--primary {
            display: none !important;
        }
    }
</style>

# Réalisation

*Dernière mise à jour : fin de la semaine 5 (début juin 2026).*

Cette page présente l'implémentation telle qu'elle existe dans le dépôt : l'architecture, les composantes développées, l'état d'avancement et les décisions de conception. Le détail des tests et de la validation empirique vit dans la page Évaluation ; le déroulé chronologique des tâches vit dans la page Suivi.

## Architecture générale

Le code est organisé en src-layout (`src/` et `tests/` comme dossiers frères à la racine), packagé sous le module `simubrain`. Trois couches sont séparées de façon délibérée :

- les modèles DEVS (`src/simubrain/models/`) : la sémantique de simulation, exprimée comme modèles atomiques et couplés sur PyPDEVS ;
- la couche de validation (`src/simubrain/analysis.py`) : la comparaison empirique contre théorie, volontairement ignorante de DEVS (elle ne connaît que des listes de temps de spikes et un taux) ;
- la couche d'expériences (`src/simubrain/experiments/`) : le harnais en ligne de commande qui assemble une simulation, mesure, et écrit une figure.

Ce découpage donne un couplage faible entre les couches : la validation peut observer n'importe quelle source future de spikes (MMPP, G-networks) sans modification, et le refactor RNG à venir ne touchera que la couche modèles.

La structure expérimentale suit le pattern Experimental Frame de Zeigler : le `PoissonNeuron` joue le rôle de generator (la source de spikes), le `Transducer` celui de la sonde (l'observation), et la terminaison de la simulation est gérée par `setTerminationTime` (le rôle d'acceptor).

Stack technique : Python 3.11, PyPDEVS 2.4.2 (installation editable), numpy et matplotlib pour le calcul et les figures, pytest pour les tests, ruff pour le linting et le formatage.

## Composantes réalisées

### PoissonNeuron

Modèle DEVS atomique pur source (aucun port d'entrée) qui émet un spike sur son port `output` à chaque tirage. Les intervalles inter-spikes sont i.i.d. exponentiels de moyenne $1/\lambda$.

C'est ici que se concentre l'idée centrale du projet : par rapport à un DEVS déterministe, seul `timeAdvance` change, passant d'une constante à un tirage aléatoire `rng.exponential(scale=1.0 / rate)`. Le reste de la structure atomique (`outputFnc` émet le spike, `intTransition` retourne l'état suivant) reste identique. L'invariant `rate > 0` est vérifié à la construction.

### Transducer

Sonde DEVS atomique purement passive : son `timeAdvance` retourne l'infini, donc elle ne se déclenche jamais d'elle-même et ne produit aucune sortie. À chaque événement reçu sur son port `input`, elle ajoute un couple `(temps, payload)` à une liste interne, exposée pour l'analyse hors-ligne via les propriétés `event_times` et `count`.

Le composant est volontairement agnostique au domaine : le payload peut être n'importe quel objet, ce qui rend la sonde réutilisable dans n'importe quel réseau DEVS, pas seulement pour des trains de spikes.

### SpikeTrainExperiment

Modèle DEVS couplé minimal qui câble une source à une sonde : `PoissonNeuron.output` vers `Transducer.input`. Après simulation, le train de spikes enregistré est accessible via la propriété `spike_times`.

### analysis.py (couche de validation)

Regroupe les utilitaires de comparaison empirique contre théorie : les dataclasses `EmpiricalStats` et `TheoreticalStats`, le calcul des intervalles inter-spikes (`compute_isi`), les statistiques mesurées et prédites, et les fonctions de tracé assemblées dans `make_validation_figure` (figure à trois panneaux : raster, distribution des ISI contre $\text{Exp}(\lambda)$, comptage cumulé $N(t)$ contre $\lambda t$). Les résultats produits par cette couche et leur interprétation sont présentés dans la page Évaluation.

### run_basic_experiment

Harnais en ligne de commande qui assemble le tout : il parse les arguments (taux, durée, graine, dossier de sortie), simule une expérience, imprime la comparaison des métriques, et écrit la figure de validation. Point d'entrée : `python -m simubrain.experiments.run_basic_experiment`.

## État d'avancement

À ce stade, le volet Poisson homogène est complété et validé, et le travail en cours entame les extensions Markov/MMPP et G-networks.

Réalisé et committé :

- le neurone de Poisson homogène et sa simulation de bout en bout ;
- la sonde passive et le modèle couplé qui les relie ;
- la couche de validation et le harnais en ligne de commande ;
- une suite pytest couvrant les invariants structurels et déterministes (détail dans Évaluation).

En cours et à venir :

- refactor de la gestion du RNG : centraliser le tirage dans une classe dédiée injectée dans les modèles (injection de dépendances), pour offrir un point d'échange unique vers le backend MRG32k3a de L'Écuyer. Décision de conception encore ouverte sur la dérivation des graines (basée sur des labels contre compteur séquentiel) 
- extension MMPP (Markov-Modulated Poisson Process) : enrichir verticalement un neurone avec un taux dépendant de l'état 
- extension G-networks (réseaux de neurones aléatoires de Gelenbe) : modéliser horizontalement des populations en interaction 
- expression du neurone stochastique via le méta-formalisme unifié.

## Décisions de conception

Découplage de la couche de validation : `analysis.py` ne dépend ni des modèles DEVS, ni du simulateur, ni du RNG. Cela garde la validation stable à travers le refactor RNG à venir et lui permet d'observer toute source future de spikes sans changement.

Stockage de la liste d'événements hors de l'état du `Transducer` : la liste qui grossit est gardée comme attribut d'instance plutôt que dans l'objet d'état, pour éviter de copier une liste de plus en plus grande à chaque transition.

RNG par instance : chaque modèle crée son propre `np.random.default_rng(seed)`, ce qui isole les flux aléatoires entre neurones et rend les exécutions reproductibles quand une graine explicite est passée. C'est la base que le refactor à venir généralisera vers une gestion centralisée et injectée.

Pattern Experimental Frame : la séparation source / observation rend la sonde réutilisable et garde les responsabilités distinctes (un seul rôle par composant).

Invariants explicites et transitions sans effet de bord : `rate > 0` est vérifié à la construction, et les fonctions de transition retournent un nouvel état plutôt que de muter l'état courant, ce qui facilite le raisonnement et le test.