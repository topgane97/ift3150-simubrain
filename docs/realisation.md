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

*Dernière mise à jour : semaine 8 (fin juin 2026).*

Cette page présente l'implémentation telle qu'elle existe dans le dépôt : l'architecture, les composantes développées, l'état d'avancement et les décisions de conception. Le détail des tests et de la validation empirique vit dans la page Évaluation ; le déroulé chronologique des tâches vit dans la page Suivi.

## Architecture générale

Le code est organisé en src-layout (`src/` et `tests/` comme dossiers frères à la racine), packagé sous le module `simubrain`. Trois couches sont séparées de façon délibérée :

- les modèles DEVS (`src/simubrain/models/`) : la sémantique de simulation, exprimée comme modèles atomiques et couplés sur PyPDEVS ;
- la couche de validation (`src/simubrain/analysis.py`) : la comparaison empirique contre théorie, volontairement ignorante de DEVS (elle ne connaît que des listes de temps de spikes et un taux) ;
- la couche d'expériences (`src/simubrain/experiments/`) : le harnais en ligne de commande qui assemble une simulation, mesure, et écrit une figure.

Ce découpage donne un couplage faible entre les couches : la validation peut observer n'importe quelle source future de spikes (MMPP, G-networks) sans modification, et la gestion du RNG est centralisée dans une couche dédiée injectée dans les modèles.

La structure expérimentale suit le pattern Experimental Frame de Zeigler : la source de spikes joue le rôle de generator, le `Transducer` celui de la sonde (l'observation), et la terminaison de la simulation est gérée par `setTerminationTime` (le rôle d'acceptor).

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

### rng.py (RandomStream)

Gestion centralisée et reproductible du tirage aléatoire, injectée dans les modèles plutôt qu'instanciée à l'intérieur. Une racine `RandomStream` est construite à partir d'une seule graine entière ; chaque source aléatoire de la simulation en est ensuite dérivée par *label*, jamais re-seedée à la main.

Deux opérations de dérivation : `spawn(label)` retourne un générateur numpy prêt à l'emploi, pour les modèles feuilles (un neurone) ; `spawn_stream(label)` retourne un `RandomStream` enfant qui peut lui-même être dérivé, pour les propriétaires intermédiaires qui distribuent des sous-flux à leurs sous-composants (un modèle couplé qui sème ses nœuds).

La dérivation est order-independent : `spawn("spikes")` produit le même sous-flux quel que soit l'ordre des autres labels dérivés, ce qui garde les modèles multi-sources (MMPP) et les réseaux de modèles (G-networks) reproductibles même quand des composants sont ajoutés ou réordonnés. La clé de dérivation vient d'un hachage BLAKE2b du label, et non du `hash()` natif de Python qui est salé par processus (`PYTHONHASHSEED`) et casserait la reproductibilité d'une exécution à l'autre.

### MMPPNeuron

Modèle DEVS atomique d'un neurone dont le taux de décharge instantané est modulé par une chaîne de Markov à temps continu (CTMC) cachée. Quand la CTMC est dans l'état $i$, les spikes suivent un processus de Poisson homogène de taux `rates[i]` ; la chaîne saute entre états selon la matrice génératrice $Q$.

Le modèle est générique dans le nombre d'états : le mécanisme est identique pour tout $n$, seules les données (`rates`, `Q`) portent le compte d'états. Le cas classique à deux états (repos / actif) est une simple instanciation, pas une classe spéciale. Cela aligne le modèle sur le point d'extension justifié du méta-formalisme, à coût YAGNI nul.

Le cœur est un mécanisme à deux horloges concurrentes (*dual-clock*). Une horloge de spike $\text{Exp}(\text{rates}[\text{current}])$ donne le délai jusqu'au prochain spike ; une horloge de transition $\text{Exp}(-Q[\text{current}, \text{current}])$ donne le délai jusqu'au prochain saut de la CTMC, où $-Q[\text{current}, \text{current}]$ est le taux de sortie total de l'état courant. `timeAdvance` retourne le minimum des deux délais restants. Si l'horloge de spike gagne, le modèle émet un spike, re-tire seulement l'horloge de spike, garde l'état CTMC inchangé et décrémente l'horloge de transition du temps écoulé (sans la re-tirer, pour ne pas retarder artificiellement le saut en attente). Si l'horloge de transition gagne, le modèle saute vers un nouvel état échantillonné selon $Q[\text{current}]$, puis re-tire les deux horloges.

`timeAdvance` reste pur (aucun tirage, aucune mutation) : tous les tirages se font dans `__init__` et `intTransition`, conformément à l'invariant DEVS. Les paramètres sont validés à la construction : `rates` strictement positifs, $Q$ carrée et compatible avec `rates`, lignes de somme nulle, hors-diagonale non négative, taux de sortie strictement positif (pas d'état absorbant). Deux flux RNG indépendants sont dérivés par label, `spawn("spikes")` pour l'horloge de spike et `spawn("transitions")` pour les durées de séjour et le choix de l'état suivant.

### MMPPExperiment

Modèle DEVS couplé qui câble un `MMPPNeuron` à un `Transducer`, sur le même patron que `SpikeTrainExperiment`. La duplication entre les deux frames est laissée en place plutôt qu'abstraite : deux frames mono-source ne justifient pas encore une classe de base commune (YAGNI). Si un troisième modèle source apparaît, une base `SingleSourceExperiment` sera factorisée à ce moment-là.

### analysis.py (couche de validation)

Regroupe les utilitaires de comparaison empirique contre théorie : les dataclasses `EmpiricalStats` et `TheoreticalStats`, le calcul des intervalles inter-spikes (`compute_isi`), les statistiques mesurées et prédites, et les fonctions de tracé. Pour le neurone de Poisson homogène, `make_validation_figure` assemble une figure à trois panneaux (raster, distribution des ISI contre $\text{Exp}(\lambda)$, comptage cumulé $N(t)$ contre $\lambda t$).

La couche a été étendue pour le MMPP sans casser le code Poisson existant. `stationary_distribution(Q)` résout $\pi Q = 0$ sous $\sum_i \pi_i = 1$ ; `effective_rate(rates, Q)` en déduit le taux effectif $\bar\lambda$. La figure MMPP (`make_mmpp_validation_figure`) réutilise les panneaux raster et comptage cumulé, agnostiques au processus, mais remplace l'histogramme ISI par une version sans superposition : pour un processus modulé, la loi des ISI est une mixture sur-dispersée, et superposer $\text{Exp}(\bar\lambda)$ suggérerait à tort un ajustement qui ne tient pas. Les résultats produits par cette couche et leur interprétation sont présentés dans la page Évaluation.

### run_basic_experiment

Harnais en ligne de commande qui assemble le tout pour le neurone de Poisson : il parse les arguments (taux, durée, graine, dossier de sortie), simule une expérience, imprime la comparaison des métriques, et écrit la figure de validation. Point d'entrée : `python -m simubrain.experiments.run_basic_experiment`.

### run_mmpp_experiment

Harnais en ligne de commande qui simule le neurone MMPP de référence (repos / actif), compare le compte de spikes observé au taux effectif $\bar\lambda = \sum_i \pi_i \, \text{rates}_i$, et écrit la figure de validation. Les taux et la matrice $Q$ sont fixés dans le runner plutôt qu'exposés en arguments : ce harnais existe pour valider le cas de référence, pas pour explorer des chaînes arbitraires. Point d'entrée : `python -m simubrain.experiments.run_mmpp_experiment`.

## État d'avancement

À ce stade, le volet Poisson homogène et le premier modèle stochastique modulé (MMPP) sont complétés et validés ; le travail à venir entame les G-networks et le méta-formalisme unifié.

Réalisé et committé :

- le neurone de Poisson homogène et sa simulation de bout en bout ;
- la sonde passive et le modèle couplé qui les relie ;
- la couche de validation et le harnais en ligne de commande ;
- la gestion centralisée du RNG (`RandomStream`), injectée dans les modèles et dérivée par label ;
- le neurone MMPP à dual-clock, son frame couplé, son harnais et les extensions de validation associées ;
- une suite pytest couvrant les invariants structurels et déterministes (détail dans Évaluation).

En cours et à venir :

- extension G-networks (réseaux de neurones aléatoires de Gelenbe) : modéliser horizontalement des populations en interaction via des signaux négatifs ;
- expression du neurone stochastique via le méta-formalisme unifié, puis benchmarks de performance.

## Décisions de conception

Découplage de la couche de validation : `analysis.py` ne dépend ni des modèles DEVS, ni du simulateur, ni du RNG. Cela garde la validation stable et lui permet d'observer toute source future de spikes sans changement.

Stockage de la liste d'événements hors de l'état du `Transducer` : la liste qui grossit est gardée comme attribut d'instance plutôt que dans l'objet d'état, pour éviter de copier une liste de plus en plus grande à chaque transition.

Gestion centralisée et injectée du RNG : plutôt que chaque modèle instancie son propre `np.random.default_rng(seed)`, un `RandomStream` racine est construit à partir d'une graine et injecté dans les modèles, qui dérivent leurs générateurs par label. Cela isole les flux entre composants, garde les exécutions reproductibles, et centralise le tirage en un point d'échange unique. La dérivation par label est order-independent (hachage BLAKE2b), ce qui reste reproductible quand des sources sont ajoutées ou réordonnées, condition nécessaire pour les modèles multi-sources (MMPP) et les réseaux à venir (G-networks).

Généricité du MMPP dès la conception : le neurone MMPP prend `rates` et $Q$ de taille arbitraire, le mécanisme dual-clock étant identique pour tout nombre d'états. Le cas deux états est un appel, pas une classe. C'est le seul point d'extension anticipé justifié (il fait partie de la portée du méta-formalisme), à coût nul ; le reste suit YAGNI, d'où l'absence de Strategy configurable dans `timeAdvance` (l'exponentielle est le processus) et l'absence de base commune entre les deux frames d'expérience tant qu'un troisième modèle source n'existe pas.

Pattern Experimental Frame : la séparation source / observation rend la sonde réutilisable et garde les responsabilités distinctes (un seul rôle par composant).

Invariants explicites et transitions sans effet de bord : les gardes de validité (`rate > 0`, structure de $Q$) sont vérifiées à la construction, et les fonctions de transition retournent un nouvel état plutôt que de muter l'état courant, ce qui facilite le raisonnement et le test.