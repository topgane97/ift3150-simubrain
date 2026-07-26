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

*Dernière mise à jour : semaine 12 (fin juillet 2026).*

Cette page présente l'implémentation telle qu'elle existe dans le dépôt : l'architecture, les composantes développées, l'état d'avancement et les décisions de conception. Le détail des tests et de la validation empirique vit dans la page [Évaluation](../evaluation/). ; le déroulé chronologique des tâches vit dans la page [Suivi](../suivi/).

## Architecture générale

Le code est organisé en src-layout (`src/` et `tests/` comme dossiers frères à la racine), packagé sous le module `simubrain`. L'architecture matérialise l'hypothèse du projet : trois couches séparées de façon étanche, avec des dépendances qui ne vont que dans un sens.

| Couche | Emplacement | Responsabilité |
|--------|-------------|----------------|
| **Hasard** | `src/simubrain/rng.py` | Dérive et distribue les flux aléatoires à partir d'une seule graine. Ne connaît ni DEVS, ni les modèles. |
| **Modèles DEVS** | `src/simubrain/models/` | La sémantique de simulation, exprimée en modèles atomiques et couplés sur PyPDEVS. Reçoit le hasard, ne le crée pas. |
| **Vérification** | `src/simubrain/analysis.py` | Compare l'observé au théorique. N'importe ni PyPDEVS, ni les modèles, ni le RNG : ne connaît que des listes de temps, des signaux en escalier et des paramètres. |
| **Expériences** | `src/simubrain/experiments/` | Le harnais en ligne de commande qui assemble une simulation, mesure, imprime et trace. C'est la seule couche qui voit les trois autres. |

Aucune flèche ne remonte : les modèles reçoivent du hasard, la vérification reçoit des résultats, et seule la couche d'expériences a le droit de tout assembler. C'est ce qui permet à la couche de vérification d'observer n'importe quelle source future sans modification, et aux modèles d'être testés isolément.

La structure expérimentale suit le pattern **Experimental Frame** de Zeigler : la source joue le rôle de generator, le `Transducer` celui de la sonde (l'observation), et la terminaison est gérée par `setTerminationTime` (le rôle d'acceptor).

Stack technique : Python 3.11, PyPDEVS 2.4.2 (installation editable), numpy, scipy (tests statistiques) et matplotlib pour le calcul et les figures, pytest pour les tests, ruff pour le linting et le formatage.

## Composantes réalisées

### rng.py (RandomStream)

Gestion centralisée et reproductible du tirage aléatoire, injectée dans les modèles plutôt qu'instanciée à l'intérieur. Une racine `RandomStream` est construite à partir d'une seule graine entière ; chaque source aléatoire de la simulation en est ensuite dérivée par *label*, jamais re-seedée à la main.

L'implémentation s'appuie sur `SeedSequence` de numpy, qui n'est pas un générateur mais une **graine dérivable** : un nœud dans un arbre, dont on peut extraire des enfants indépendants. Seules les feuilles produisent réellement des nombres.

Deux opérations de dérivation : `spawn(label)` retourne un générateur numpy prêt à l'emploi, pour les modèles feuilles (un neurone) ; `spawn_stream(label)` retourne un `RandomStream` enfant qui peut lui-même être dérivé, pour les propriétaires intermédiaires qui distribuent des sous-flux à leurs sous-composants (un modèle couplé qui sème ses nœuds).

La dérivation est **order-independent** : `spawn("spikes")` produit le même sous-flux quel que soit l'ordre des autres labels dérivés. Concrètement, la clé de dérivation est le chemin d'étiquettes menant au nœud, pas un compteur d'appels. C'est la propriété qui garde les modèles multi-sources (MMPP) et les réseaux (G-networks) reproductibles quand des composants sont ajoutés, réordonnés ou retirés. La clé vient d'un hachage BLAKE2b du label, et non du `hash()` natif de Python qui est salé par processus (`PYTHONHASHSEED`) et casserait la reproductibilité d'une exécution à l'autre.

Corollaire souvent utile : deux assemblages différents qui dérivent leurs générateurs le long de chemins d'étiquettes différents obtiennent des générateurs différents, même à graine racine égale. C'est ce qui explique que le MMPP en une brique et le MMPP en plusieurs briques ne produisent pas la même trace, sans que l'ordre d'instanciation y soit pour quoi que ce soit.

### PoissonNeuron

Modèle DEVS atomique pur source (aucun port d'entrée) qui émet un spike sur son port `output` à chaque tirage. Les intervalles inter-spikes sont i.i.d. exponentiels de moyenne $1/\lambda$.

C'est le cas de base du projet : par rapport à un DEVS déterministe, seul le calcul de l'avance du temps change, passant d'une constante à un tirage exponentiel. Le reste de la structure atomique (`outputFnc` émet le spike, `intTransition` retourne l'état suivant) reste identique. L'invariant `rate > 0` est vérifié à la construction.

Le modèle applique le **pré-tirage** : l'intervalle est tiré à l'avance dans l'état et `timeAdvance` se contente de le lire. Cette correction, décrite plus bas dans les décisions de conception, était une dette technique de la première version, et sa résorption a conditionné la réutilisation du modèle dans la famille G-networks.

### Transducer

Sonde DEVS atomique purement passive : son avance du temps retourne l'infini, donc elle ne se déclenche jamais d'elle-même et ne produit aucune sortie. À chaque événement reçu sur son port `input`, elle ajoute un couple `(temps, payload)` à une liste interne, exposée pour l'analyse hors-ligne via les propriétés `event_times` et `count`.

Le composant est volontairement agnostique au domaine : le payload peut être n'importe quel objet, ce qui rend la sonde réutilisable dans n'importe quel réseau DEVS. C'est la brique la plus stable du projet : elle sert aux trois familles sans une ligne de modification. La famille G-networks en a fourni la preuve la plus nette, puisque la sonde y enregistre des **longueurs de file** et non des spikes, sans que son code change.

Pour dater les événements, elle accumule `self.elapsed`, le délai que PyPDEVS positionne juste avant chaque transition externe. C'est le pattern d'horloge absolue dans un modèle passif.

### SpikeTrainExperiment

Modèle DEVS couplé minimal qui câble une source à une sonde : `PoissonNeuron.output` vers `Transducer.input`. Après simulation, le train de spikes enregistré est accessible via la propriété `spike_times`.

### MMPPNeuron (version en une brique)

Modèle DEVS atomique d'un neurone dont le taux de décharge instantané est modulé par une chaîne de Markov à temps continu (CTMC) cachée. Quand la CTMC est dans l'état $i$, les spikes suivent un processus de Poisson homogène de taux `rates[i]` ; la chaîne saute entre états selon la matrice génératrice $Q$.

Le modèle est générique dans le nombre d'états : le mécanisme est identique pour tout $n$, seules les données (`rates`, `Q`) portent le compte d'états. Le cas classique à deux états (repos / actif) est une simple instanciation, pas une classe spéciale.

Le cœur est un mécanisme à deux horloges concurrentes (*dual-clock*). Une horloge de spike $\text{Exp}(\text{rates}[\text{current}])$ donne le délai jusqu'au prochain spike ; une horloge de transition $\text{Exp}(-Q[\text{current}, \text{current}])$ donne le délai jusqu'au prochain saut de la CTMC, où $-Q[\text{current}, \text{current}]$ est le taux de sortie total de l'état courant. L'avance du temps retourne le minimum des deux délais restants.

Si l'horloge de spike gagne, le modèle émet un spike, re-tire seulement l'horloge de spike, garde l'état CTMC inchangé et **décrémente** l'horloge de transition du temps écoulé plutôt que de la re-tirer, pour ne pas retarder artificiellement un saut déjà en attente. Si l'horloge de transition gagne, le modèle saute vers un nouvel état échantillonné selon $Q[\text{current}]$, puis re-tire les deux horloges. L'absence de mémoire de l'exponentielle rend les deux traitements équivalents en loi.

Les paramètres sont validés à la construction : `rates` strictement positifs, $Q$ carrée et compatible avec `rates`, lignes de somme nulle, hors-diagonale non négative, taux de sortie strictement positif (pas d'état absorbant). Deux flux RNG indépendants sont dérivés par label, `spawn("spikes")` pour l'horloge de spike et `spawn("transitions")` pour les durées de séjour et le choix de l'état suivant.

### MMPPExperiment

Modèle DEVS couplé qui câble un `MMPPNeuron` à un `Transducer`, sur le même patron que `SpikeTrainExperiment`. La duplication entre les quatre frames d'expérience est une décision documentée plutôt qu'un oubli ; l'argument complet et sa condition de révision vivent dans le docstring de module de ce fichier, et sont résumés plus bas dans les décisions de conception.

### MarkovChain

Extraction du mécanisme de commutation caché à l'intérieur du `MMPPNeuron`, sorti en modèle atomique autonome. La chaîne saute entre états selon $Q$ et publie, à chaque saut, le **taux de décharge** du nouvel état sur son port `rate_out`. Elle ne sait rien des spikes.

Émettre le taux plutôt que l'indice d'état est une décision de couplage faible : la source en aval reçoit un simple nombre et n'a jamais besoin de connaître `rates`, ni l'existence même d'une chaîne.

Le modèle applique le **pré-tirage** de façon poussée : non seulement la durée de séjour est tirée à l'avance dans l'état, mais l'**état de destination** l'est aussi. DEVS appelle `outputFnc` avant `intTransition`, et cette fonction doit être sans effet de bord ; pré-tirer la destination est ce qui lui permet de publier `rates[next_state]` en se contentant de lire. La transition interne valide ensuite le saut et pré-tire la destination et la durée suivantes.

### ModulatedPoissonNeuron

Source de Poisson dont le taux peut être changé en cours de simulation par un message reçu sur son port `rate_in`. Entre deux changements, elle décharge comme un Poisson homogène ; à réception d'un nouveau taux, elle re-tire son intervalle en attente sous ce taux.

C'est la contrepartie décomposée de l'horloge de spike interne du `MMPPNeuron`. Le `PoissonNeuron` d'origine est laissé intact : ce modèle ajoute un port d'entrée et donc une nature mixte (source et récepteur), ce qui justifie une classe séparée plutôt qu'une modification (principe ouvert/fermé).

Le re-tirage à réception plutôt que la mise à l'échelle du résidu est exact par absence de mémoire de l'exponentielle, et strictement moins de machinerie (YAGNI).

### DecomposedMMPPExperiment (version en plusieurs briques)

Modèle couplé qui assemble les trois sous-modèles :

```text
MarkovChain.rate_out ──► ModulatedPoissonNeuron.rate_in
ModulatedPoissonNeuron.output ──► Transducer.input
```

La chaîne et la source puisent dans deux sous-flux frères indépendants (`"markov"`, `"poisson"`). Le taux initial de la source est fixé à `rates[initial_state]`, ce qui la synchronise avec la chaîne à $t = 0$ sans dépendre de l'ordre d'arrivée des messages.

Les deux écritures du MMPP sont **statistiquement équivalentes, non identiques trace pour trace**, et il vaut la peine d'être précis sur la raison. Ce n'est pas une question d'ordre de consommation : la dérivation par étiquette est order-independent par construction, et l'ordre d'instanciation ne peut donc rien changer. Ce qui diffère est le **chemin de dérivation** de chaque générateur. Le monolithe descend par `"neuron"` puis `"spikes"` et `"transitions"` ; le décomposé descend par `"markov"` menant à `"transitions"` et par `"poisson"` menant à `"spikes"`. Les rôles se correspondent un pour un, mais les clés diffèrent, donc les générateurs sont amorcés différemment. S'y ajoute une divergence interne : la `MarkovChain` pré-tire sa destination dès la construction, là où le monolithe la tire au moment du saut, donc la séquence de tirages à l'intérieur d'un même générateur n'est pas la même non plus. C'est exactement ce que le test d'équivalence est conçu pour trancher.

### GQueue (file de Gelenbe)

Modèle DEVS atomique d'une file d'attente à un serveur recevant deux types d'arrivées de Poisson : des clients **positifs** sur `positive_in`, qui rejoignent la file et sont servis, et des signaux **négatifs** sur `negative_in`, qui détruisent un client en attente ou s'évanouissent sans effet si la file est vide.

Le client négatif ne porte aucun travail : c'est un pur signal d'annihilation. C'est ce qui distingue un G-network d'un réseau de Jackson, et c'est la raison pour laquelle la charge stationnaire vaut $\rho = \lambda^+/(\mu + \lambda^-)$ : la destruction agit comme un second canal de départ, donc elle grossit le dénominateur.

L'état est réduit à $\{n, t_{\text{service}}, \text{publishing}\}$, sans identité de client : rien dans la grandeur validée (la longueur moyenne) ne demande de distinguer les clients, donc YAGNI interdit une liste de clients tant qu'une discipline autre que sans mémoire n'est pas requise. Le drapeau `publishing` n'est pas une donnée du domaine mais un artefact de la contrainte DEVS décrite plus bas.

L'invariant $n = 0 \iff t_{\text{service}} = \infty$ est centralisé dans une méthode `_service_clock(n)` unique, de sorte qu'aucun appelant n'ait à s'en souvenir. Il retire une branche du `timeAdvance`, qui reste une lecture pure, et il est testé explicitement sur chacun des chemins qui vident la file (départ, destruction).

Le modèle publie sa longueur sur `length_out` à chaque changement. Il ne calcule aucune moyenne : le `Transducer` enregistre les couples $(t, n)$ et la couche de vérification intègre l'escalier. La couche de vérification n'apprend donc jamais ce qu'est une file.

Un port `departure_out` publie un marqueur à chaque fin de service. Il est présent pour le routage dans un réseau multi-nœuds, et laissé **non câblé** dans l'expérience à un seul nœud : le câbler par anticipation serait spéculatif.

### GQueueExperiment

Modèle couplé qui assemble une ou deux sources de Poisson, la file et la sonde :

```text
PoissonNeuron("positive").output ──► GQueue.positive_in
PoissonNeuron("negative").output ──► GQueue.negative_in    (si lambda- > 0)
GQueue.length_out ─────────────────► Transducer.input
```

Les composants aléatoires puisent dans des sous-flux frères indépendants (`"positive"`, `"negative"`, `"queue"`), donc l'expérience reste reproductible quel que soit l'ordre d'instanciation.

Aucune des deux sources ne sait ce qu'est un G-network : ce sont des `PoissonNeuron` ordinaires, identiques, et le signe est porté **entièrement par le port** sur lequel le couplage se termine. C'est le bénéfice de la décision antérieure de garder le `PoissonNeuron` comme source pure non typée.

**Le cas limite M/M/1.** Le modèle accepte `lambda_minus = 0`, et il traite ce cas en **omettant entièrement** la source négative et son câblage, plutôt qu'en instanciant une source silencieuse. La décision découle directement de la précédente : puisque le signe vit dans le port, « pas de canal de destruction » est littéralement « pas de fil vers `negative_in` ». L'attribut `negative_source` vaut alors `None` et reste défini, pour qu'un appelant puisse le tester sans risque d'`AttributeError`. Comme la dérivation des flux se fait par étiquette, omettre la source négative ne déplace ni le flux positif ni celui du service : les deux configurations restent directement comparables, ce qu'un test vérifie en comparant l'état initial de la source positive dans les deux cas.

### analysis.py (couche de vérification)

Regroupe les utilitaires de comparaison observé contre théorique : les dataclasses `EmpiricalStats` et `TheoreticalStats`, le calcul des intervalles inter-spikes (`compute_isi`), les statistiques mesurées et prédites, et les fonctions de tracé. Pour le neurone de Poisson homogène, `make_validation_figure` assemble une figure à trois panneaux (raster, distribution des ISI contre $\text{Exp}(\lambda)$, comptage cumulé $N(t)$ contre $\lambda t$).

La couche a été étendue trois fois sans casser le code existant.

**Pour le MMPP.** `stationary_distribution(Q)` résout $\pi Q = 0$ sous $\sum_i \pi_i = 1$ ; `effective_rate(rates, Q)` en déduit le taux effectif $\bar\lambda$. La figure MMPP (`make_mmpp_validation_figure`) réutilise les panneaux raster et comptage cumulé, agnostiques au processus, mais remplace l'histogramme ISI par une version sans superposition : pour un processus modulé, la loi des ISI est une mixture sur-dispersée, et superposer $\text{Exp}(\bar\lambda)$ suggérerait à tort un ajustement qui ne tient pas.

**Pour la moyenne de la file de Gelenbe.** L'extension était plus exigeante, puisque la grandeur validée change de nature : $\rho$ est une charge stationnaire, donc la validation porte sur une longueur de file moyenne dans le temps, alors que la couche ne savait lire que des trains d'événements. `time_average(records, duration, initial_value)` intègre un signal constant par morceaux sans savoir ce qu'il représente. Son paramètre `initial_value` est obligatoire : la fonction ne devine pas la valeur du signal avant le premier changement enregistré. L'appelant de la file passe 0, et le fait qu'il doive le dire rend l'hypothèse visible. `gqueue_utilization` et `gqueue_mean_length` calculent $\rho$ et $\mathbb{E}[N] = \rho/(1-\rho)$, et rejettent une configuration instable ($\rho \geq 1$), pour laquelle aucun régime stationnaire n'existe.

**Pour la loi entière de la file.** La moyenne n'est que le premier moment, et deux lois différentes peuvent la partager. `gqueue_length_distribution(lambda_plus, mu, lambda_minus, max_length)` donne la loi stationnaire géométrique $P(N = n) = (1-\rho)\rho^n$, tronquée et **non normalisée** de façon explicite : le docstring déclare que la somme vaut $1 - \rho^{M+1}$ et non 1, pour qu'un appelant qui compare à un histogramme normalisé sache quelle masse lui manque dans la queue. Son pendant empirique est `time_weighted_histogram(records, duration, initial_value)`, qui fait exactement le même découpage en paliers que `time_average` mais range chaque durée dans un dictionnaire indexé par la valeur au lieu de tout sommer dans un accumulateur. La pondération par le temps et non par le nombre de changements est le point : une longueur atteinte souvent mais quittée immédiatement pèse presque rien dans une loi stationnaire, qui est par définition une fraction de temps.

Toutes les fonctions théoriques (`effective_rate`, `theoretical_stats`, `gqueue_mean_length`, `gqueue_length_distribution`) ne consomment **aucune sortie de simulation** : elles ne dépendent que des paramètres, ce qui est la condition pour que leur résultat soit une prédiction et non une description. Un test vérifie d'ailleurs que les deux formes closes de la file sont cohérentes entre elles, en confirmant que $\sum_n n\,P(N = n)$ redonne bien $\mathbb{E}[N]$ : ajouter la loi n'introduit pas une seconde vérité concurrente de la première.

### Les harnais d'expérience

Quatre points d'entrée en ligne de commande, sur le même patron : parser les arguments, simuler, imprimer la comparaison, écrire la figure.

| Commande | Ce qu'elle valide |
|----------|-------------------|
| `python -m simubrain.experiments.run_basic_experiment` | Neurone de Poisson contre $\lambda$ |
| `python -m simubrain.experiments.run_mmpp_experiment` | MMPP en une brique contre $\bar\lambda$ |
| `python -m simubrain.experiments.run_decomposed_mmpp_experiment` | MMPP en plusieurs briques contre $\bar\lambda$ |
| `python -m simubrain.experiments.run_gqueue_experiment` | File de Gelenbe contre $\mathbb{E}[N] = \rho/(1-\rho)$ |

Pour les runners MMPP et G-queue, les paramètres du modèle sont fixés dans le code plutôt qu'exposés en arguments : ces harnais existent pour valider un cas de référence, pas pour explorer des configurations arbitraires.

Le runner de la file expose deux arguments propres à la nature stationnaire de sa validation. `--warmup` (100 s par défaut) écarte le transitoire initial avant de moyenner, puisque la file démarre vide et que ce n'est pas un tirage de la loi stationnaire. Ce chiffre n'est pas arbitraire : le temps de relaxation est de l'ordre de $1/[(\mu + \lambda^-)(1-\rho)^2] \approx 0.19$ s au cas de référence, donc cent secondes représentent plusieurs centaines de temps de relaxation pour un coût de 5 % de la fenêtre. `--plot-window` (20 s par défaut) borne l'extrait tracé : une estimation stationnaire demande un horizon qui contient des dizaines de milliers de changements, et les tracer tous produit une bande pleine dont le bord supérieur est une enveloppe de maxima locaux, pas une trajectoire. La moyenne reste calculée sur la fenêtre entière, indépendamment de ce qui est tracé.

Il n'y a délibérément **pas de cinquième runner** pour les campagnes multi-graines ni pour le cas limite M/M/1. Ces validations assertent une propriété et ne produisent aucune figure, donc elles vivent dans `tests/test_gqueue_conformance.py`. Un cinquième runner aurait dupliqué le quatrième à un paramètre près.

## État d'avancement

Le code est **depuis la fin de la semaine 12**. Réalisé et committé :

- **l'architecture en trois couches** : hasard injecté (`rng.py`), modèles DEVS, vérification indépendante du simulateur ;
- **la famille Poisson** : source, sonde, assemblage, expérience, validée contre $\lambda$ ;
- **la famille MMPP en une brique** : neurone dual-clock générique, validé contre $\bar\lambda$ ;
- **la famille MMPP en plusieurs briques** : `MarkovChain` + `ModulatedPoissonNeuron` + `Transducer`, validée contre $\bar\lambda$ ;
- **le critère d'équivalence** : test de Kolmogorov-Smirnov entre les deux écritures du MMPP, sur dix graines indépendantes ;
- **la famille G-networks** : `GQueue` atomique, `GQueueExperiment` couplé réutilisant deux `PoissonNeuron` comme sources, validée contre $\mathbb{E}[N] = \rho/(1-\rho)$ ;
- **la campagne de conformité multi-graines** : huit graines à 1500 s avec intervalle de confiance de Student, qui remplace la tolérance relative sur une graine unique ;
- **le cas limite M/M/1** ($\lambda^- = 0$) comme contrôle croisé sur le même dispositif ;
- **la validation contre la loi stationnaire entière** de la file, et non contre son seul premier moment ;
- **l'extension de la couche de vérification aux signaux en escalier** (`time_average`, `time_weighted_histogram`), qui reste agnostique au domaine ;
- **la résorption de la dette technique du `PoissonNeuron`** (pré-tirage), prérequis à sa réutilisation dans la file de Gelenbe ;
- une suite pytest de 142 tests couvrant les invariants structurels et déterministes des trois couches, plus les campagnes statistiques (détail dans Évaluation).

À venir :

- diagrammes UML (classes en couches) et modèle C4, et synthèse de l'architecture et des trois critères sur les trois familles ;
- rapport final et présentation.

## Décisions de conception

### Hasard injecté plutôt qu'instancié

Plutôt que chaque modèle instancie son propre `np.random.default_rng(seed)`, une racine `RandomStream` est construite à partir d'une graine et **injectée** dans les modèles, qui dérivent leurs générateurs par label. Un modèle déclare ce dont il a besoin, il ne se sert pas tout seul.

Cela isole les flux entre composants, garde les exécutions reproductibles, centralise le tirage en un point d'échange unique, et rend chaque modèle testable isolément avec un flux contrôlé. La dérivation par label est order-independent, condition nécessaire pour les modèles multi-sources et les réseaux, et c'est elle qui rend le cas M/M/1 directement comparable au cas de référence malgré la source manquante.

### Pré-tirage dans l'état

DEVS exige que le calcul de l'avance du temps et la fonction de sortie soient **purs** : appelables plusieurs fois, même réponse, aucune modification. Un modèle stochastique doit pourtant tirer. La contradiction se résout en tirant à l'avance et en rangeant le résultat dans l'état : les fonctions pures ne tirent plus, elles lisent.

Sans cette règle, le simulateur qui interroge le modèle plusieurs fois avant d'agir obtient une réponse différente à chaque appel, consomme le flux à chaque interrogation, et la simulation produit des résultats faux sans jamais planter. C'est un invariant testé explicitement (`test_time_advance_is_pure`).

Le `PoissonNeuron` a longtemps fait exception : il tirait directement dans son `timeAdvance`. Le tirage y était inoffensif tant que le modèle restait une source isolée, dont l'avance du temps n'est interrogée qu'une fois par événement, mais serait devenu un bug silencieux dès son insertion dans un modèle couplé recevant des entrées. La famille G-networks a rendu la correction obligatoire, puisqu'elle réutilise exactement ce modèle dans un assemblage. La dette est résorbée.

Le `MarkovChain` pousse le pré-tirage à sa conclusion : la destination du prochain saut est elle aussi pré-tirée, sans quoi `outputFnc` ne pourrait pas publier le taux du prochain état de façon pure.

### Découplage de la couche de vérification

`analysis.py` ne dépend ni des modèles DEVS, ni du simulateur, ni du RNG. Cela garde la vérification stable et lui permet d'observer toute source future sans changement.

L'extension au MMPP l'a confirmé en pratique : les panneaux raster et comptage cumulé ont été réutilisés tels quels, seul le panneau ISI a dû être spécialisé. L'extension à la file de Gelenbe a été l'épreuve la plus sévère, puisque la grandeur validée n'est plus un train d'événements mais une charge stationnaire. Trois options se présentaient : reconstruire $N(t)$ depuis les arrivées et les départs, ce qui aurait fait fuir la sémantique file dans la couche de vérification ; échantillonner périodiquement, ce qui aurait introduit un biais de discrétisation et un modèle de plus à valider ; ou publier la longueur et intégrer l'escalier avec une fonction agnostique. La troisième a été retenue, et la couche est sortie étendue sans être contaminée. Le passage à la loi entière n'a rien changé à cet équilibre : `time_weighted_histogram` ignore autant ce qu'est une file que `time_average`.

Corollaire strict : les fonctions théoriques ne prennent que des paramètres en entrée, jamais un résultat de simulation. Une prédiction qui regarde l'observation cesse d'être une prédiction.

### Le signe porté par le port, pas par la charge utile

Dans la file de Gelenbe, un client positif et un signal négatif arrivent avec exactement le même payload. Ce qui les distingue est le **port** sur lequel le couplage se termine.

L'alternative aurait été de typer le message (`("spike", +1)` contre `("spike", -1)`) et de faire lire le signe par la file. Cela aurait forcé les deux sources à connaître les G-networks, alors que ce sont des neurones de Poisson ordinaires qui n'ont aucune raison de savoir à quoi ils servent. Porter le sens par le port fait du routage la responsabilité de l'assemblage, ce qui est exactement le découpage que le principe de responsabilité unique demande, et permet de réutiliser le `PoissonNeuron` sans une ligne de modification.

### Omettre la source négative plutôt que la faire taire

Le cas limite M/M/1 demandait que `GQueueExperiment` accepte $\lambda^- = 0$, ce que le `PoissonNeuron` refuse puisqu'il valide `rate > 0`. Trois options ont été pesées.

Relâcher la garde `rate > 0` casserait un invariant correct et testé : un processus de Poisson de taux nul n'est pas un processus dégénéré, c'est l'absence de processus, et son intervalle inter-arrivées serait infini. Affaiblir un composant pour la commodité d'un appelant est le mauvais sens de la dépendance. Ajouter une classe `SilentSource` créerait une classe entière pour un seul cas de test, jamais réutilisée ailleurs (YAGNI).

L'option retenue rend le câblage du flux négatif **conditionnel** dans le modèle couplé. C'est la seule des trois qui exprime une vérité du modèle plutôt qu'un contournement : puisque le signe vit dans le port, l'absence de canal de destruction *est* l'absence de fil. En termes de conception, on préfère modifier la structure plutôt qu'ajouter un cas particulier dans le comportement, ce qui évite un état « source présente mais inactive » qui n'a aucun sens dans le domaine.

La dette assumée est le type de l'attribut : `negative_source` est un `PoissonNeuron | None` documenté. L'alternative, ne pas définir l'attribut du tout dans ce cas, serait pire, puisqu'elle donnerait une classe dont l'interface change selon les paramètres.

### État transitoire pour publier depuis une transition externe

Un modèle DEVS atomique ne peut pas émettre de sortie depuis `extTransition`. La file aurait donc publié sa longueur aux seuls départs, ratant toutes les montées de $n$, et le signal enregistré aurait été inexploitable.

Deux issues existaient. Faire écouter les arrivées par une seconde sonde et reconstruire $N(t)$ dans le runner aurait fait fuir la sémantique file dans la couche de vérification, ce que l'architecture interdit. La solution retenue est le patron DEVS standard : sur une arrivée, la file lève un drapeau `publishing` et s'auto-réveille avec un `timeAdvance` nul ; ce pas transitoire publie la nouvelle longueur, baisse le drapeau, et le service en cours reprend intact. Le service pendant n'est ni consommé ni re-tiré par ce passage, ce qu'un test vérifie explicitement.

### Décrémenter le résiduel plutôt que le re-tirer

Sur toute arrivée dans une file occupée, le temps de service restant est décrémenté du temps écoulé plutôt que re-tiré. Sous un service exponentiel, les deux sont équivalents en loi. Décrémenter reste pourtant correct quelle que soit la loi de service, alors que re-tirer ne l'est que sous l'exponentielle : le choix évite d'enfouir silencieusement une hypothèse que la classe ne déclare pas.

Le même raisonnement gouverne l'horloge de transition du `MMPPNeuron`.

### Ne pas surcharger la confluence, mais l'asserter

PyPDEVS applique par défaut `intTransition` puis `extTransition` quand un départ et une arrivée coïncident exactement. C'est la sémantique voulue pour la file : le départ s'accomplit, puis l'arrivée simultanée s'applique à l'état résultant. Surcharger `confTransition` pour réécrire ce comportement aurait été du bruit.

Le projet ne s'en remet pas au défaut pour autant : deux tests l'assertent directement. Une file de longueur 1 qui termine son service à l'instant exact où un client positif arrive doit finir à 1, et une file de longueur 1 qui termine son service quand un négatif arrive doit finir à 0 avec le négatif perdu. Si une version future de la librairie changeait l'ordre, les tests le diraient au lieu de laisser la sémantique dériver en silence.

### Émission du taux plutôt que de l'état

Le `MarkovChain` publie un taux, pas un indice d'état. La source en aval reçoit un nombre qu'elle sait interpréter sans rien connaître de la chaîne, de `rates`, ni du nombre de régimes. C'est le couplage le plus faible possible entre les deux briques, et c'est ce qui rend le `ModulatedPoissonNeuron` réutilisable devant n'importe quel autre pilote de taux.

### Nouvelle classe plutôt que modification (ouvert/fermé)

Le `ModulatedPoissonNeuron` aurait pu être obtenu en ajoutant un port d'entrée au `PoissonNeuron`. Cela aurait changé la nature du modèle existant, de source pure à source-récepteur, et cassé sa sémantique. Une classe séparée préserve le modèle validé et isole la nouvelle responsabilité.

### Ne pas factoriser les frames d'expérience

Les quatre modèles couplés partagent la même forme : instancier des sous-modèles, dériver des sous-flux frères par étiquette, câbler la sortie de la source vers l'entrée de la sonde, exposer un accesseur en lecture seule sur les enregistrements. La règle des trois est donc atteinte, et c'est précisément le point où la discipline du projet demanderait normalement de factoriser. La duplication est pourtant laissée en place, et la décision est écrite plutôt que subie, puisqu'une décision non écrite est indiscernable d'un oubli.

La part réellement commune est plus petite qu'elle n'en a l'air : trois lignes. Ce qui diffère est tout le reste, à savoir les signatures de constructeur (`rate`, puis `rates, Q`, puis `lambda_plus, lambda_minus, service_rate`), le nombre de sources, le nombre de couplages, et la sémantique des accesseurs. Ce dernier point est le plus décisif : `spike_times` jette les payloads et ne garde que les dates, alors que `length_records` garde les couples parce que le payload *est* la mesure. Les nommer pareil dans une classe de base suggérerait une interchangeabilité qui n'existe pas, ce qui est exactement le type de fuite d'abstraction que le docstring de `analysis.py` avait déjà dû corriger une fois.

S'y ajoute le calendrier : refactorer quatre modèles couplés et leurs quatre modules de tests la semaine du gel, sans livrer aucune capacité nouvelle en échange, achèterait de la propreté au prix du risque. Une dette prise sciemment dans un prototype de recherche et une dette prise par inadvertance dans un produit livré ne sont pas le même objet.

La condition de révision est énoncée plutôt que laissée implicite : un cinquième frame dont la composition de sources correspondrait exactement à un existant, même arité et même sémantique d'accesseur, ou l'arrivée d'un second type de sonde que tous les frames devraient supporter. Ni l'un ni l'autre n'existe aujourd'hui. L'argument complet vit dans le docstring de module de `mmpp_experiment.py`, c'est-à-dire à l'endroit où la question se pose au lecteur du code.

### Stockage de la liste d'événements hors de l'état

La liste d'événements du `Transducer` est gardée comme attribut d'instance plutôt que dans l'objet d'état, pour éviter de copier une liste de plus en plus grande à chaque transition. C'est un écart assumé à la règle « les transitions retournent un nouvel état », justifié par le coût quadratique qu'elle induirait ici.

### Généricité du MMPP, et YAGNI ailleurs

Le neurone MMPP prend `rates` et $Q$ de taille arbitraire, le mécanisme dual-clock étant identique pour tout nombre d'états. Le cas deux états est un appel, pas une classe. C'est le seul point d'extension anticipé, à coût nul.

Le reste suit YAGNI : pas de Strategy configurable pour la loi de tirage (l'exponentielle *est* le processus dans les trois familles), pas de base commune entre les frames d'expérience, pas de Factory pour le RNG tant qu'un seul backend existe, pas d'identité de client dans la file tant que la grandeur validée n'en demande aucune, et pas de câblage du port `departure_out` tant qu'aucun second nœud n'existe pour le recevoir.

La file de Gelenbe illustre aussi la règle inverse. Contrairement au MMPP, elle est livrée **uniquement en version décomposée** : l'assemblage est l'objet même du modèle, et un monolithe qui simulerait la file, ses arrivées et son service dans une seule classe n'aurait aucun sens pour un formalisme dont la raison d'être est le réseau.

### Invariants explicites et transitions sans effet de bord

Les gardes de validité (`rate > 0`, structure de $Q$, `service_rate > 0`) sont vérifiées à la construction, **une seule fois, par le composant qui les possède**. Le modèle couplé de la file ne revalide pas `lambda_plus` ni `service_rate`, qui sont rejetés par les sous-modèles auxquels ils sont confiés ; il ne valide que `lambda_minus`, dont la valeur nulle a un sens structurel que les sous-modèles ne connaissent pas. Recopier une règle métier à deux endroits garantit qu'un jour l'un des deux sera oublié.

Les fonctions de transition retournent un nouvel état plutôt que de muter l'état courant. Cela facilite le raisonnement et rend les transitions testables sans simulateur : on force un état, on appelle la transition, on inspecte ce qu'elle retourne.

L'invariant $n = 0 \iff t_{\text{service}} = \infty$ de la file va plus loin que la validation à la construction : il doit tenir après **chaque** transition. Il est centralisé dans une méthode unique plutôt que répété à chaque branche, et testé sur chacun des chemins qui vident la file.