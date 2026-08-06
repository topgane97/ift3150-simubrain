---
title: Journal de Bord
---

<style>
    @media screen and (min-width: 76em) {
        .md-sidebar--primary { display: none !important; }
    }
</style>

# Journal de bord

Cette page rassemble mes **explorations techniques et conceptuelles**, semaine par
semaine : ce que j'ai cherché à comprendre en profondeur derrière chaque étape du
projet. Le suivi factuel des tâches (objectifs, accompli, prochaines étapes) vit
dans la page [Suivi](../suivi/).

## Semaine 1 et 2 (du 4 au 15 mai 2026) : mise en place, premiers pas DEVS et neurone de Poisson

Objectif d'exploration : mettre en place l'infrastructure du projet (site MkDocs du
cours, environnement conda, dépôt GitHub, PyPDEVS), puis comprendre la chaîne
complète qui mène au premier modèle : le squelette d'un modèle DEVS atomique en
Python, le neurone de Poisson (math et implémentation), et le rôle de la graine
pour la reproductibilité.

### Mise en place et outillage du projet

J'ai consolidé *pourquoi* chaque brique de l'environnement existe, au-delà du geste :

- **Site MkDocs (Material)** : suivi et documentation du projet publiés en continu
  (cette page en fait partie).
- **Conda** : un environnement dédié `simubrain` (Python 3.11, aligné avec la
  `target-version` de ruff). Règle d'or apprise : ne jamais installer dans `base`.
- **PyPDEVS en éditable** (`pip install -e .`) : mes éventuels patches de la
  librairie sont pris en compte sans réinstallation. Pour une dépendance stable on
  ferait simplement `pip install .`.
- **`src`-layout** (`src/` et `tests/` frères à la racine) : isole le code
  *importable* des tests, et évite d'importer accidentellement depuis le dossier
  courant plutôt que depuis le package installé.
- **`pyproject.toml`** remplace l'ancien `setup.py` : configuration **déclarative**
  du build, des métadonnées et des outils (ruff, pytest). Le bloc
  `[project.optional-dependencies] dev` **isole les outils de dev du runtime** : un
  utilisateur fait `pip install simubrain`, moi `pip install -e ".[dev]"`.
- **`__init__.py`** : marque chaque dossier comme *package* Python. C'est ce qui
  permettra au futur `transducer` d'écrire `from simubrain.models.poisson_neuron
  import PoissonNeuron`, et aux tests d'importer le package proprement.
- **GitHub CLI** : `gh repo create ... --source=. --remote=origin --private` crée le
  dépôt distant **et** ajoute le remote local en une seule commande.
- **Ruff** comme linter et formateur (sélection `E, F, W, I, N, UP, B, SIM`), et
  familiarisation avec les standards de déclaration de type dans les signatures.
- Compétences terminal au passage : `cat << 'EOF' > fichier`, `tree -a -I '.git'`
  pour visualiser l'arborescence, `git init`, `git branch -M main`, `git push`.

> Le runbook complet pas à pas (toutes les commandes) est conservé dans le `README`
> du dépôt ; cette page n'en garde que la logique.

### DEVS en pratique : l'exemple `policeman` et l'OO Python vs Java

J'ai étudié les exemples `policeman` et `trafficlight` du *tutorial_classic* de
PyPDEVS **ligne par ligne** pour saisir le squelette d'un modèle DEVS atomique :
l'état, `timeAdvance` (le délai jusqu'au prochain événement interne), `outputFnc`
(la sortie), `intTransition` (la transition interne) et `extTransition` (la
transition externe).

Côté Python orienté objet, plusieurs réflexes Java à désapprendre :

```python
from pypdevs.DEVS import AtomicDEVS          # import explicite, pas de wildcard

class PoissonNeuron(AtomicDEVS):             # héritage par parenthèses
    def __init__(self, name: str, rate: float, seed: int | None = None) -> None:
        super().__init__(name)               # pas `extends`, pas d'appel au parent à la Java
        # ...
```

`seed: int | None = None` m'a rappelé les *type hints* du cours IFT2035 : la
signature documente le contrat (un entier **ou** rien), sans l'imposer à l'exécution.

### Le neurone de Poisson (math + implémentation)

J'ai d'abord survolé le **LIF** (*leaky integrate-and-fire*), un neurone déterministe
à seuil, pour situer le contraste avec un modèle purement **stochastique**.

Le neurone de Poisson se comprend de deux façons équivalentes :

- **Vue comptage** : le nombre de spikes sur $[0, t]$ suit $N(t) \sim \text{Poisson}(\lambda t)$, avec $\mathbb{E}[N(t)] = \mathrm{Var}[N(t)] = \lambda t$.
- **Vue inter-arrivées** : les temps entre spikes (*interspike intervals*) sont distribués exponentiellement.

$$
T_i \sim \mathrm{Exp}(\lambda), \qquad \mathbb{E}[T_i] = \frac{1}{\lambda}
$$

Ici $\lambda$ est le **taux** en Hz (spikes/seconde) et $1/\lambda$ l'**intervalle
moyen** entre deux spikes. C'est cette valeur $1/\lambda$ qu'attend NumPy :

```python
def timeAdvance(self):
    # le seul élément qui change vs un DEVS déterministe :
    # une constante devient un tirage Exp(lambda)
    return self.rng.exponential(scale=1.0 / self.rate)   # scale = 1/rate, PAS rate
```

Piège retenu : `scale = 1/rate`, et non `rate`. Et c'est la **vue inter-arrivées**
qui est naturelle pour DEVS, puisque `timeAdvance` correspond au prochain intervalle
$T_i$.

> Note rétrospective : ce `timeAdvance` qui tire est exactement la dette technique
> résorbée en semaine 11. Le tirage doit vivre dans la transition, pas dans la
> lecture. Voir la section sur le pré-tirage plus bas.

### Graine, PRNG et reproductibilité

Une **graine** (*seed*) est le point de départ déterministe d'un générateur
pseudo-aléatoire : une même graine produit toujours la même séquence, donc des
résultats reproductibles par d'autres chercheurs (c'est un concept qu'Abdelhamid avait
souligné pour la reproduction d'expériences). Intuition via un LCG jouet
$x_{n+1} = (5x_n + 3) \bmod 16$, en partant de $7$ : $6, 1, 8, 11, 10, \dots$

L'idée qui généralise ce LCG jouet est celle de **sous-flux** (*substreams*)
indépendants : une même graine racine, mais des sous-séquences garanties disjointes,
une par composant. C'est ce qui donne à la fois le parallélisme et la
reproductibilité. Trajectoire d'implémentation envisagée, où la couche RNG reste
échangeable en un seul endroit :

```python
self.rng = np.random.default_rng(seed)        # aujourd'hui (NumPy)
self.rng = AutreBackend.substream(neuron_id)  # plus tard, si nécessaire
```

**Difficultés**

- Le concept de graine et de PRNG est revenu à plusieurs sessions : clarifié par le LCG jouet, mais encore à consolider du côté des sous-flux.
- La syntaxe d'héritage en Python (réflexe `extends` de Java).
- Faire le pont entre le neurone de Poisson *mathématique* (les $T_i$) et son *implémentation*, en passant d'une vue comptage à une vue inter-arrivées.

## Semaine 3 à 5 (du 18 mai au 5 juin 2026) : tests, modèle couplé et validation statistique

Objectif d'exploration : finaliser et tester le neurone de Poisson, puis construire le
`Transducer` et le modèle couplé pour fermer le triangle pédagogique du DEVS de base
(source, puits et couplage), et enfin valider statistiquement que la simulation
reproduit bien la théorie. Le but n'est pas un modèle plus sophistiqué, mais une
**infrastructure de validation réutilisable** pour tout le reste du projet.

### Tests unitaires et niveaux de vérification

J'ai séparé deux niveaux de vérification, distinction que je veux garder pour tout le
projet :

- **pytest** couvre les **invariants structurels et déterministes** : instanciation
  par défaut, levée d'erreur sur un `rate` invalide (via `pytest.raises`, qui vérifie
  l'invariant `rate > 0`), reproductibilité à graine fixe, et `timeAdvance` toujours
  strictement positif.
- La **validation statistique** (empirique vs théorique) est d'une autre nature et vit
  **dans l'expérience**, pas dans les tests unitaires : on ne teste pas qu'un tirage
  aléatoire vaut une valeur précise, on vérifie qu'une distribution est la bonne.

> Note rétrospective : cette seconde moitié a évolué. La validation statistique est
> revenue dans pytest à partir de la semaine 12, mais **dans des modules séparés** et
> sous une forme qui n'assert jamais sur un tirage isolé : une campagne de graines et
> un intervalle de confiance. La distinction de fond tient, c'est son emplacement qui
> a changé.

Workflow avant chaque commit : `ruff check .` (et `ruff check . --fix`), `ruff format`,
puis `pytest -v`. La suite démarre avec le neurone (`test_instantiation_default`,
`test_invalid_rate_raises`, `test_seed_reproducibility`, `test_time_advance_positive`),
puis grandit avec le transducer et le modèle couplé (instanciation des sous-modèles,
enregistrement chronologique des événements, comptage cohérent) : 12 tests au vert.

### Conventional Commits

J'ai adopté les *Conventional Commits* comme standard : un préfixe de type par commit,
à l'impératif présent, un commit propre par livrable validé. Les types : `feat`
(fonctionnalité visible), `fix` (correction de bug), `docs` (documentation seule),
`test` (tests seuls), `refactor` (réécriture sans changement de comportement), `perf`
(performance), `chore` (maintenance hors-code : dépendances, config, structure du
dépôt), `ci` (pipeline d'intégration) et `style` (formatage pur). En pratique `style`
sert rarement seul, puisque `ruff format` tourne avant chaque commit et le formatage
part avec le commit `feat` correspondant.

### Le pattern Experimental Frame (Zeigler)

`SpikeTrainExperiment` n'est pas une structure arbitraire : il instancie l'**Experimental
Frame** de Zeigler, la forme canonique d'une expérience en DEVS, en trois rôles :

- **Generator** : le système sous test qui produit les stimuli, ici le `PoissonNeuron`
  (le train de spikes).
- **Acceptor** : décide quand arrêter. Absent comme modèle ici, car il n'y a pas de
  condition d'arrêt structurelle ; c'est `Simulator.setTerminationTime` qui joue ce
  rôle depuis l'extérieur.
- **Transducer** : la sonde, qui observe et restitue les résultats.

Le **modèle couplé** consiste justement à relier ces atomiques (`PoissonNeuron` vers
`Transducer`) : c'est le couplage du triangle.

### `self.elapsed` et l'horloge absolue d'un modèle passif

Le `Transducer` est un modèle **passif** : son `timeAdvance` est infini et il ne réagit
qu'aux événements externes. Pour dater les spikes reçus, j'ai utilisé `self.elapsed`,
le **delta** entre la transition précédente du modèle et l'instant courant. PyPDEVS
positionne cet attribut **juste avant** d'appeler `extTransition` : on ne l'initialise
pas et on ne le met pas à jour, on le **lit** seulement. Pour reconstruire le temps
absolu, on accumule :

```python
t_absolu = t_absolu_precedent + self.elapsed
```

C'est le pattern d'horloge absolue dans un modèle DEVS passif. Les transitions
retournent un **nouvel état** (souvent un dictionnaire conservé dans l'état du modèle)
plutôt que de muter l'état en place.

### Validation statistique : empirique vs théorique

L'expérience se lance par exemple ainsi :

```bash
python -m simubrain.experiments.run_basic_experiment --rate 20 --duration 60 --seed 42
```

et produit une figure à trois panneaux : le **raster** des spikes, la distribution des
**ISI** comparée à $\mathrm{Exp}(\lambda)$, et le **comptage cumulé** $N(t)$ comparé à
$\lambda t$.

Le point conceptuel central concerne le **bruit attendu**. Le comptage suit
$N(T) \sim \text{Poisson}(\lambda T)$, donc son écart-type est $\sqrt{\lambda T}$ et son
erreur relative décroît lentement :

$$
\frac{\sqrt{\lambda T}}{\lambda T} = \frac{1}{\sqrt{\lambda T}}
$$

Concrètement, avec $\lambda = 30$ et $T = 2$, on a $\lambda T = 60$, un écart-type
$\sqrt{60} \approx 7.7$ (soit 13 % de la moyenne) : un comptage observé de 63 est à
$+0.4\,\sigma$, parfaitement normal. Avec une fenêtre 20 fois plus longue
($\lambda T = 1200$), l'écart-type relatif tombe à $\sqrt{1200}/1200 \approx 2.9\,\%$ et
la courbe cumulée colle à la droite « comme sur des rails ». De même, l'histogramme ISI
est lisse quand chaque bin est bien rempli, et en dents de scie quand il n'y a que
quelques valeurs par bin (le bruit d'échantillonnage domine).

L'intervalle de confiance à 95 % du comptage se calcule donc :

```python
half_width = 1.96 * np.sqrt(expected_count)
```

où `1.96` est le quantile de la loi normale à 95 % et le $\sqrt{\cdot}$ vient de ce que,
pour une loi de Poisson, la variance égale la moyenne. Pour $\lambda T = 60$ :
$60 \pm 1.96 \cdot 7.7 = [45, 75]$, et 63 tombe dedans, donc aucun bug.

À retenir pour le rapport : une seule réalisation courte ne « valide » rien à elle
seule, car elle peut dévier visiblement de la théorie sans qu'il y ait de bug. Une
validation rigoureuse passe par une fenêtre longue (qui réduit la variance) ou par
plusieurs réalisations dont on vérifie que la moyenne des comptages tombe dans le
CI95 % et que la distribution des ISI agrégés colle.

> Cette dernière phrase est, avec le recul, le programme des semaines 12 et 14. C'est
> exactement ce que sont devenues les campagnes multi-graines, et c'est la seule
> conclusion de cette semaine qui a directement dicté une décision d'architecture
> deux mois plus tard.

**Difficultés**

- S'assurer de passer $1/\lambda$ (et non $\lambda$) à NumPy, dont la convention `scale` est la moyenne.
- Le retour d'état sous forme de dictionnaire conservé dans l'état du modèle.
- Bien saisir le rôle du `Transducer` et ce qu'un modèle couplé signifie au sens DEVS.
- Côté expérience, cerner quoi afficher et quelles données sont réellement importantes à analyser.
- Généraliser le seed en classe me donne du fil à retordre, mais m'amène à réfléchir davantage comme un ingénieur logiciel, ce qui est positif.

## Semaine 6 à 8 (du 8 au 26 juin 2026) : Markov, MMPP et RNG reproductible en réseau

Objectif d'exploration : passer du neurone de Poisson homogène à un neurone dont le
taux est modulé par une chaîne de Markov cachée (MMPP), en construisant d'abord la
carte conceptuelle qui situe Markov, MMPP et les G-networks les uns par rapport aux
autres, puis en consolidant la couche RNG en une classe reproductible à l'échelle
d'un réseau. La semaine 7 était une pause (examens finaux), donc la matière se
concentre sur les semaines 6 et 8.

### Situer Markov DEVS, MMPP et les G-networks

Avant d'implémenter quoi que ce soit, j'ai eu besoin de comprendre que ces trois
noms ne vivent pas au même niveau. Les confondre menait à une roadmap floue.

- **Markov DEVS** (chapitres 21-22 de Zeigler) n'est pas un modèle biologique mais
  un *patron de spécification* : DEVS où les durées de séjour dans chaque état sont
  tirées d'une exponentielle dont le taux dépend de l'état courant, gouvernées par
  une chaîne de Markov à temps continu (CTMC). C'est la mécanique générique.
- **MMPP** (*Markov-Modulated Poisson Process*) est un modèle pour *un* neurone. Le
  neurone tire encore ses spikes selon un Poisson, mais le taux $\lambda$ n'est
  plus constant : une CTMC cachée fait basculer le système entre régimes (repos
  $\lambda_0 = 5$ Hz, actif $\lambda_1 = 40$ Hz) et fixe le $\lambda$ courant.
  Deux processus imbriqués : la chaîne module, le Poisson génère. C'est l'extension
  directe de mon `PoissonNeuron`.
- **G-networks** (Gelenbe) sont d'un tout autre ordre : un modèle de *réseau* de
  files d'attente, pas d'un neurone isolé. Leur signature est le **signal négatif**
  (un spike inhibiteur qui *retire* du travail à la file destinataire, là où un
  client positif en ajoute). C'est la métaphore excitation / inhibition, et Gelenbe
  a explicitement relié ses *Random Neural Networks* à ce formalisme.

La hiérarchie que je retiens :

| Concept | Niveau | Ce qu'il modélise |
|---|---|---|
| Markov DEVS | méta-formalisme / spécification | mécanique CTMC générique dans DEVS |
| MMPP | un neurone | taux Poisson modulé par une CTMC |
| G-networks | un réseau | neurones couplés, signaux $\pm$ |

Conséquence pour la roadmap : MMPP est ma prochaine étape (un neurone, dans la
continuité directe du Poisson), les G-networks viennent après (un réseau, donc un
saut d'échelle), et Markov DEVS est le socle commun dont les deux héritent.

### RandomStream : reproductibilité par arbre de graines

Le refactor du RNG en classe (`rng.py`) répond à une question qui devient centrale
dès qu'on quitte le neurone isolé : comment garder une simulation reproductible
quand elle contient plusieurs sources aléatoires, et surtout quand on *ajoute* une
source sans vouloir perturber les autres ?

L'idée que j'ai dû assimiler est celle de `SeedSequence` de NumPy. Une
`SeedSequence` n'est pas un générateur : c'est une **graine dérivable**, un noeud
dans un arbre. La racine porte la graine entière (42) ; on en dérive des enfants,
et des petits-enfants, chaque noeud produisant un flux statistiquement indépendant.
Seules les feuilles (les `Generator`) tirent réellement des nombres.

Deux opérations structurent l'arbre. `spawn(label)` rend un générateur prêt à
l'emploi, pour un modèle feuille (un neurone qui consomme du hasard).
`spawn_stream(label)` rend un `RandomStream` enfant, pour un propriétaire
intermédiaire (un réseau qui distribue des sous-flux à ses noeuds sans consommer
lui-même).

```python
root = RandomStream(seed=42)
layer = root.spawn_stream("layer_1")   # noeud intermédiaire
neuron_A = layer.spawn("spikes")       # feuille, générateur réel
```

Le point conceptuel qui m'a demandé le plus de réflexion est la **dérivation par
nom plutôt que par compteur**. La position d'un noeud dans l'arbre est encodée par
le hachage de son label, pas par un compteur incrémental. Conséquence : `spawn`
devient *order-independent*. Dériver `"spikes"` avant ou après `"voltage"` donne
toujours le même sous-flux pour `"spikes"`. C'est exactement la garantie
nécessaire pour un réseau : ajouter un neurone à un G-network ne doit pas
invalider les trains de spikes des neurones déjà présents.

Le dernier maillon est le choix de **BLAKE2b** pour hacher les labels plutôt que
le `hash()` natif de Python. Ce dernier est salé par processus
(`PYTHONHASHSEED`) comme mesure de sécurité, donc `hash("spikes")` change d'une
exécution à l'autre et casserait toute reproductibilité inter-run. BLAKE2b, hachage
cryptographique, rend toujours la même sortie pour la même entrée, sur toute
machine. C'est ce qui fait qu'une exécution entière est déterministe à partir d'un
seul entier.

### De la CTMC au MMPP : ce qui module et ce qu'on observe

Le coeur de la semaine 8 a été de comprendre la CTMC sous le MMPP, en trois briques
qui s'empilent.

**L'absence de mémoire de l'exponentielle.** La loi $\text{Exp}(\lambda)$ est la
seule loi continue vérifiant

$$
P(T > s + t \mid T > s) = P(T > t).
$$

Après avoir attendu $s$ sans événement, le temps résiduel est distribué comme un
tirage frais : le temps déjà écoulé n'informe pas sur le temps restant. Ce n'est
pas un détail, c'est précisément ce qui rend le processus *markovien* en temps
continu : l'état courant suffit à décrire le futur, l'historique n'ajoute rien.

**La matrice génératrice $Q$.** Elle encode toute la dynamique de la chaîne
cachée. Ses trois règles ne sont pas arbitraires, elles découlent du sens de chaque
terme. Sur mon exemple repos / actif :

$$
Q = \begin{pmatrix} -0.5 & 0.5 \\ 2.0 & -2.0 \end{pmatrix}.
$$

- $Q[i,j] \ge 0$ hors diagonale : un taux de saut, donc jamais négatif.
- $Q[i,i] = -\sum_{j \ne i} Q[i,j]$ : chaque ligne somme à zéro, la diagonale
  n'est pas un vrai taux mais une case comptable qui absorbe le reste de sa ligne.
- $-Q[i,i] > 0$ : taux de sortie strictement positif, sinon l'état est
  *absorbant* (on n'en sort jamais). Mes validations rejettent exactement ce cas.

Ce qui casse l'intuition, c'est l'asymétrie : $Q[0,1] = 0.5$ mais
$Q[1,0] = 2.0$. La ligne est le point de départ, la colonne l'arrivée, et les
deux rôles ne sont jamais interchangeables. Biologiquement ça se tient : basculer du
repos vers l'actif (rare, $0.5$/s) n'a pas de raison d'avoir le même taux que
retomber de l'actif vers le repos (rapide, $2.0$/s, l'actif est instable).

**La distribution stationnaire $\pi$ et le taux effectif.** $\pi_i$ est la
fraction de temps passée dans l'état $i$ à long terme, solution de

$$
\pi Q = 0, \qquad \sum_i \pi_i = 1.
$$

Intuitivement, à l'équilibre le flux repos $\to$ actif égale le flux actif $\to$
repos : $\pi_0 \cdot 0.5 = \pi_1 \cdot 2.0$, ce qui donne $\pi = (0.8, 0.2)$.
Le neurone passe 80 % de son temps au repos, ce qui fait sens puisqu'il y reste en
moyenne 4 fois plus longtemps ($2$ s contre $0.5$ s). C'est ici que la théorie
devient prédiction testable, via le **taux effectif** pondéré par le temps de
séjour :

$$
\bar\lambda = \sum_i \pi_i \, \text{rates}_i = 0.8 \cdot 5 + 0.2 \cdot 40 = 12 \text{ Hz},
$$

soit 720 spikes attendus sur 60 s. C'est le maillon qui relie la chaîne cachée aux
spikes qu'on compte.

Pour résoudre $\pi$ numériquement, j'ai retenu deux idées sans entrer dans le
détail du code : $\pi Q = 0$ seule a une infinité de solutions proportionnelles,
donc on remplace l'une des équations (redondante, puisque les lignes de $Q$
somment à zéro) par la contrainte de normalisation $\sum_i \pi_i = 1$ ; et on
résout par moindres carrés (`lstsq`) plutôt que par inversion exacte, comme filet
de sécurité contre les imprécisions flottantes de $Q$ (ici la solution est exacte,
mais un $Q$ issu d'un calcul antérieur pourrait être légèrement incohérent).

**Le piège théorique à ne pas rater : l'ISI n'est pas exponentiel.** Pour un
Poisson, l'ISI suit $\text{Exp}(\lambda)$. Pour un MMPP, non : c'est une
**mixture** (parfois état lent, longs intervalles ; parfois état rapide, courts
intervalles), donc une distribution *sur-dispersée*. La coïncidence trompeuse est
que la *moyenne* de l'ISI vaut bien $1/\bar\lambda$, mais l'égalité des moyennes
n'entraîne pas l'égalité des lois. Superposer une $\text{Exp}(\bar\lambda)$ sur
l'histogramme serait statistiquement faux, ce qui justifie que
`make_mmpp_validation_figure` s'en abstienne là où la figure Poisson le peut.

### Le mécanisme dual-clock

L'implémentation du MMPP repose sur deux horloges concurrentes. Une horloge de spike
$\text{Exp}(\text{rates}[\text{current}])$ donne le délai jusqu'au prochain spike ;
une horloge de transition $\text{Exp}(-Q[\text{current},\text{current}])$ donne le
délai jusqu'au prochain saut de la CTMC. `timeAdvance` retourne le minimum des deux.

Le point que je voulais vraiment comprendre est *pourquoi on peut décrémenter
l'horloge perdante au lieu de la re-tirer*. Quand le spike gagne, on émet le spike,
on re-tire seulement l'horloge de spike, et on **décrémente** l'horloge de
transition du temps écoulé plutôt que de la re-tirer à neuf. La justification est
l'absence de mémoire : le taux de sortie de l'état courant n'a pas changé, donc le
temps résiduel après avoir attendu `elapsed` suit exactement la même loi qu'un
tirage frais. Décrémenter ou re-tirer donnent donc la même distribution, mais
décrémenter évite de retarder artificiellement un saut déjà en attente.

**Le cas du saut d'état est différent, et je l'ai d'abord mal formulé.** Quand
l'horloge de transition gagne, le régime change, donc le taux de décharge change
aussi. Le délai de spike en attente est jeté et un délai frais est tiré sous le
nouveau taux. Dire que « les deux suivent la même loi » serait ici **faux** :
$\text{Exp}(5)$ et $\text{Exp}(40)$ sont deux lois distinctes. Ce que l'absence de
mémoire donne réellement, c'est que le résiduel **ne porte aucune information** :
le temps déjà attendu ne dit rien sur le temps restant. Le jeter ne perd donc
rien, et le bon délai suivant est un tirage frais sous le nouveau taux. La nuance
est mince mais elle change l'argument, et j'ai dû corriger les docstrings
concernés en semaine 14.

Piège d'implémentation confirmé au passage : `numpy.random.exponential` prend
`scale = 1/`$\lambda$ (la moyenne), pas $\lambda$. Un neurone à 40 Hz s'écrit
`exponential(scale=1/40)`, jamais `scale=40`.

**Difficultés**

- Cartographier la roadmap Markov / MMPP / G-networks : comprendre que ce sont trois
  niveaux distincts (spécification, neurone, réseau) et non trois modèles au même
  rang.
- Saisir la structure en arbre de `rng.py` : pourquoi la dérivation par label plutôt
  que par compteur, et en quoi l'*order-independence* est la condition de
  reproductibilité dans un réseau de neurones.
- Comprendre le MMPP autant sur le plan théorique (matrice $Q$, $\pi$,
  $\bar\lambda$) que sur le plan de l'implémentation matricielle.
- Cerner comment analyser correctement les données du MMPP après l'expérience, en
  particulier pourquoi l'ISI ne se compare pas à une exponentielle.
- Formuler correctement l'argument d'absence de mémoire selon que le taux change ou
  non : deux situations qui se ressemblent et ne se justifient pas pareil.

## Semaine 9 à 12 (du 29 juin au 24 juillet 2026) : décomposition du MMPP, équivalence statistique et famille G-networks

Objectif d'exploration : quitter le neurone isolé pour la question qui commande tout le projet, celle de la composabilité. Un même processus peut s'écrire en une brique ou en plusieurs briques reliées ; comment démontrer que les deux écritures font la même chose ? Cela a demandé de comprendre le test de Kolmogorov-Smirnov et ses pièges, puis de porter la même exigence de preuve sur une famille d'un tout autre ordre, les files de Gelenbe, en terminant par une campagne de conformité multi-graines qui remplace une tolérance pragmatique par un vrai critère statistique. La semaine 9 était surtout consacrée aux présentations (mise en commun 2, groupe SimuBrAIn), donc la matière technique se concentre sur les semaines 10 à 12.

### Le pré-tirage : pourquoi une fonction de lecture ne doit jamais tirer

Avant d'aborder la décomposition, il faut poser une règle que toute la suite utilise. DEVS exige que `timeAdvance` et `outputFnc` soient **pures** : appelables plusieurs fois de suite, même réponse, aucune modification de l'état. Or un modèle stochastique doit bien tirer quelque part. La contradiction se résout en tirant **à l'avance** et en rangeant le résultat dans l'état : les fonctions pures ne tirent plus, elles lisent.

Sans cette règle, un simulateur qui interroge le modèle plusieurs fois avant d'agir obtient une réponse différente à chaque appel, consomme le flux aléatoire à chaque interrogation, et produit des résultats faux sans jamais planter. C'est le pire mode de défaillance possible : silencieux.

Le `PoissonNeuron` a longtemps fait exception, en tirant directement dans son `timeAdvance`. Le tirage y était inoffensif tant que le modèle restait une source isolée, dont le simulateur n'interroge l'avance du temps qu'une fois par événement. Il serait devenu un bug dès son insertion dans un modèle couplé recevant des entrées, où l'avance du temps est recalculée autour des événements des voisins. La famille G-networks a rendu la correction obligatoire, puisqu'elle réutilise exactement ce modèle comme source des deux flux de la file. La dette a donc été résorbée en semaine 11, avant la file de Gelenbe et non après.

### Décomposer le MMPP : deux écritures, un seul processus

Le MMPP en une brique cachait deux mécanismes dans une seule classe : la chaîne de Markov qui module, et le Poisson qui décharge. La version décomposée les sépare en deux modèles atomiques reliés. La `MarkovChain` publie le taux du régime courant sur un port de sortie ; le `ModulatedPoissonNeuron` reçoit ce taux sur un port d'entrée et décharge en conséquence.

Le point conceptuel que j'ai dû assimiler est qu'**émettre le taux plutôt que l'indice d'état** est ce qui garde le couplage faible. Si la chaîne publiait « je suis dans l'état 1 », la source aurait besoin de connaître la table `rates` pour traduire. En publiant directement `rates[next_state]`, la source reçoit un nombre qu'elle sait interpréter sans rien savoir de la chaîne, ni même de son existence. C'est le découplage le plus fort possible entre les deux briques.

Le `MarkovChain` pousse le pré-tirage plus loin que les modèles précédents. DEVS appelle `outputFnc` **avant** `intTransition`. Pour publier le taux du prochain état sans le tirer dans `outputFnc`, il faut avoir pré-tiré non seulement la durée de séjour, mais aussi **l'état de destination** lui-même, et l'avoir rangé dans l'état. La transition interne ne fait ensuite que valider le saut déjà décidé et pré-tirer le suivant.

Un piège d'implémentation confirmé par des tests qui échouaient : le contrat de message de PyPDEVS. La sortie doit être `{port: [valeur]}` et la lecture `inputs[port][0]`. J'avais d'abord écrit `{port: valeur}`, ce qui plantait dans le solveur avec un `TypeError: 'float' object is not iterable` au moment où il tentait d'étendre le sac de messages du port destinataire. Les deux échecs de tests suivants venaient du même contrat mal intégré, mais du côté des tests eux-mêmes : l'un passait `{n.rate_in: 40.0}` à `extTransition` au lieu de `{n.rate_in: [40.0]}`, l'autre assertait sur la liste entière au lieu de son premier élément. Uniformiser le contrat sur les modèles **et** sur leurs tests a réglé les trois.

Le cas d'une charge utile de type chaîne mérite d'être signalé, parce qu'il est plus dangereux que l'exception. Une chaîne étant itérable, `{port: "spike"}` ne lève rien : le solveur étend le sac avec cinq messages distincts, `'s'`, `'p'`, `'i'`, `'k'`, `'e'`. Le contrat violé ne plante pas, il fabrique du faux en silence. C'est exactement le genre de défaillance que le projet cherche à rendre impossible.

### Le point le plus subtil : deux sens de « pareil »

Voici le cœur intellectuel de ces semaines. On pourrait croire que deux écritures équivalentes du même processus doivent produire **exactement la même suite d'événements**. C'est faux, et c'est le piège central.

La raison est plus fine que « les deux versions tirent dans un ordre différent », et il faut être précis ici, parce que l'imprécision contredirait l'argument central de `rng.py`. La dérivation des flux se fait **par étiquette**, jamais par compteur : elle est donc *order-independent* par construction, et l'ordre d'instanciation ne peut pas être la cause. Ce qui diffère est le **chemin d'étiquettes** menant à chaque générateur :

- monolithe : racine, puis `"neuron"`, puis `"spikes"` et `"transitions"` ;
- décomposé : racine, puis `"markov"` menant à `"transitions"`, et `"poisson"` menant à `"spikes"`.

Les rôles se correspondent un pour un, mais les clés de dérivation ne sont pas les mêmes, donc les générateurs sont amorcés différemment et ne produisent pas les mêmes nombres. S'y ajoute une seconde source de divergence, interne cette fois : la `MarkovChain` pré-tire son état de destination dès la construction, alors que le monolithe le tire au moment du saut, donc la séquence de tirages à l'intérieur d'un même générateur diffère aussi.

Leurs suites de spikes **diffèrent donc nécessairement**, même à graine égale. Exiger qu'elles soient identiques déclarerait fausse **toute** décomposition, y compris les correctes.

Le bon critère n'est donc pas « même suite » mais « même loi de probabilité ». Deux dés honnêtes ne donnent pas la même suite de résultats ; ce sont pourtant le même dé. C'est cette égalité en loi qu'il faut tester.

### Le test de Kolmogorov-Smirnov et son interprétation

Le KS à deux échantillons répond à une question précise : deux échantillons proviennent-ils de la même distribution ? Il ne compare pas des moyennes isolément, mais la forme entière de la loi, via la statistique

$$
D = \sup_x |F_1(x) - F_2(x)|,
$$

le plus grand écart vertical entre les deux fonctions de répartition empiriques. Sous l'hypothèse nulle (même loi), $D$ suit une distribution connue, ce qui donne une p-value.

Le point de rigueur que je veux retenir pour la présentation : **un KS non significatif ne prouve pas l'hypothèse nulle**. Il échoue à la rejeter. C'est de la *non-réfutation*, pas une confirmation. On ne conclut jamais « les deux modèles sont identiques », mais « aucun test statistique n'est parvenu à les distinguer, à ce pouvoir statistique et sur ces graines ».

Cette réserve est d'autant plus nécessaire que le pouvoir du test est ici faible : deux échantillons de dix valeurs ne détectent qu'un écart distributionnel important. Le dire renforce la conclusion plutôt que de l'affaiblir, puisque c'est précisément ce qui interdit de lire une non-réfutation comme une preuve. Paramétrer sur dix graines reste un gain réel par rapport à une graine unique, qui pourrait passer par chance sans qu'on puisse le voir.

> Le nombre de graines est passé à trente en semaine 14, et le raisonnement qui a
> présidé à ce choix est développé dans la section correspondante. La réserve
> ci-dessus, elle, ne change pas de nature : elle s'assouplit sans disparaître.

### Le piège de l'autocorrélation, et pourquoi on n'ajuste jamais le seuil

Le KS sur les ISI bruts d'un même run échouait sur certaines graines, avec des p-values de l'ordre de $10^{-3}$ alors que l'écart maximal entre les courbes était inférieur à 7 % ($D$ sous 0.07). Ces valeurs proviennent d'un banc de simulation jetable monté pour explorer le problème, et non de PyPDEVS : elles donnent l'ordre de grandeur du phénomène, pas une mesure de référence. Ce qui a été confirmé sous PyPDEVS 2.4.2, c'est que la version corrigée passe. J'ai d'abord cru à une vraie différence entre les modèles. C'en était une fausse.

La cause est une **hypothèse brisée**, pas une découverte. Le KS suppose des échantillons i.i.d. Or les ISI d'un même run sont **autocorrélés** : les longs intervalles se groupent dans l'état lent de la CTMC, les courts dans l'état rapide. Avec environ 1400 ISI corrélés par run, la fonction de répartition empirique sous-estime sa propre variance, le test croit disposer de bien plus d'information indépendante qu'il n'en a réellement, et il rejette sur des écarts triviaux.

Deux issues se présentaient. La mauvaise : baisser le seuil $\alpha$ jusqu'à ce que le test passe, ce qui est du *p-hacking* et n'a aucune place dans du code de dépôt. La bonne : corriger l'**entrée** du test, pas son seuil. La séquence intra-run viole l'hypothèse i.i.d., donc on ne la donne pas au KS. On agrège plutôt **une observation par run indépendant** : le compte de spikes par graine (qui teste l'échelle et la sur-dispersion) et l'ISI moyen par graine (un résumé scalaire de la position de la loi). Chaque entrée du KS devient alors i.i.d. entre graines. Le coût est explicite et assumé : on perd de la résolution sur la forme fine de la loi des ISI, résolution qu'on laisse aux figures de comptage cumulé.

### La sur-dispersion, signature du MMPP et non défaut

La première exécution du décomposé, à graine 42 sur 60 secondes, a donné 666 spikes pour une attente de 720, soit un cheveu sous la borne basse de l'intervalle de confiance de Poisson [667, 773]. Une valeur hors intervalle n'a rien d'alarmant en soi, puisque cela arrive 5 % du temps par construction, mais tomber exactement à la frontière méritait un diagnostic plutôt qu'un haussement d'épaules. Deux hypothèses, et elles ne sont pas équivalentes : soit c'est le hasard de cette graine, soit la décomposition perd des spikes de façon systématique, le suspect naturel étant le tirage frais de l'intervalle à chaque changement de taux.

Le test discriminant est simple : relancer sur d'autres graines. Si les comptes se répartissent des deux côtés de 720, c'est le hasard ; s'ils sont tous serrés en dessous, c'est un biais. Les graines 1, 7 et 100 ont donné 824, 623 et 821. Les quatre comptes tombent hors de l'intervalle, mais **des deux côtés** de la valeur attendue, deux en dessous et deux au-dessus. Pas de biais, donc, et rien à corriger.

Ce qui est instructif est **l'amplitude** de la dispersion, bien plus large que ce qu'un Poisson pur prédirait. C'est exactement la **sur-dispersion**, la signature du MMPP. L'intervalle de confiance de Poisson suppose variance égale à la moyenne ; un processus modulé ajoute de la variance au-delà, parce que sur un horizon court la CTMC ne fait que quelques transitions et chaque run attrape une fraction différente de temps passé dans chaque régime. Voir les comptes s'étaler sur $\pm 100$ là où un Poisson resterait à $\pm 50$ n'est pas un bug : c'est la démonstration empirique que la modulation fait quelque chose. Que les quatre graines sortent de l'intervalle, et non une sur vingt, est la mesure directe du fait que le gabarit ne convient pas.

Conséquence pratique : le CI de Poisson est le mauvais gabarit pour valider le compte d'un MMPP sur horizon court, et le bon critère reste la convergence du taux $N(t)/t \to \bar\lambda$ sur horizon long. C'est pourquoi le test d'intégration du décomposé mesure sur 400 secondes et non sur 60 : la demi-largeur relative de l'intervalle décroît en $1/\sqrt{\bar\lambda T}$, donc l'horizon long resserre la mesure autour de $\bar\lambda$ sans qu'on ait à toucher au seuil.

> Ce diagnostic est celui qui a le plus directement dicté le travail de la semaine
> 14 : le gabarit inadapté a fini par être remplacé, pas contourné.

### Passer à un autre ordre : les files de Gelenbe

Les G-networks ne sont pas un neurone de plus mais un modèle de **réseau**. Une file de Gelenbe reçoit deux types d'arrivées de Poisson : des clients **positifs** qui rejoignent la file et sont servis, et des signaux **négatifs** qui détruisent un client en attente, ou s'évanouissent sans effet si la file est vide. Le client négatif ne porte aucun travail : c'est un pur signal d'annihilation, la métaphore de l'inhibition neuronale.

Le point mathématique central est la place de $\lambda^-$ dans la charge stationnaire :

$$
\rho = \frac{\lambda^+}{\mu + \lambda^-}.
$$

L'intuition qui rend la formule évidente : un client quitte la file de **deux** façons, soit servi (taux $\mu$), soit détruit (taux $\lambda^-$). Du point de vue de la longueur de file, peu importe pourquoi il part. La destruction est donc un **second canal de sortie**, ce qui l'inscrit au dénominateur, avec le service, jamais au numérateur. L'erreur naturelle serait d'écrire $(\lambda^+ - \lambda^-)/\mu$, comme si les négatifs annulaient les positifs à l'entrée ; c'est faux, puisqu'un négatif sur file vide se perd sans rien détruire. Une fois $\rho$ défini ainsi, la forme product-form survit : $E[N] = \rho/(1-\rho)$.

### Deux obstacles d'implémentation propres à la file

**Publier depuis une transition externe.** Un modèle DEVS atomique ne peut émettre de sortie que via `outputFnc`, laquelle n'est appelée qu'avant une transition **interne**. Or une arrivée est une transition externe. La file n'aurait donc publié sa longueur qu'aux départs, ratant toutes les montées de $n$. La solution est le patron DEVS standard de l'**état transitoire** : sur une arrivée, la file lève un drapeau, se réveille elle-même avec un `timeAdvance` nul, publie la nouvelle longueur pendant ce pas instantané, puis baisse le drapeau et reprend son service intact. L'alternative (reconstruire $N(t)$ dans le runner à partir des arrivées et des départs) aurait fait fuir la sémantique file dans la couche de vérification, ce que l'architecture interdit.

**Le signe porté par le port, pas par le message.** Un client positif et un signal négatif arrivent avec exactement le même payload opaque. Ce qui les distingue est le **port** sur lequel le couplage se termine. Conséquence : les deux sources sont des `PoissonNeuron` ordinaires, réutilisés tels quels, qui ignorent tout des G-networks. Le routage est la responsabilité de l'assemblage, pas de la source. C'est le bénéfice concret de la décision antérieure de garder le `PoissonNeuron` comme source pure non typée, et cette réutilisation n'aurait pas été possible s'il avait été typé pour les spikes. Elle n'aurait pas été possible non plus si le tirage dans `timeAdvance` n'avait pas été corrigé d'abord.

### Mesurer une charge stationnaire : intégrer un escalier

La grandeur validée change de nature. Pour Poisson et MMPP, on comptait des événements. Ici $\rho$ est une charge stationnaire, donc la validation porte sur une **longueur de file moyenne dans le temps**, alors que la couche de vérification ne savait lire que des trains d'événements.

Le point que j'ai dû bien saisir est que $E[N]$ est une moyenne **temporelle**, pas une moyenne d'échantillons. Rester longtemps à $n = 0$ et brièvement à $n = 3$ ne se moyenne pas comme une moyenne arithmétique des valeurs visitées. Il faut pondérer chaque valeur par le temps passé à cette valeur :

$$
E[N] \approx \frac{1}{T}\int_0^T N(t)\,dt.
$$

La couche de vérification a donc gagné une fonction `time_average` qui intègre un signal en escalier, en trois blocs : le préfixe avant le premier enregistrement, les paliers intérieurs, et le suffixe jusqu'à la fin. Cette fonction ignore totalement ce qu'est une file ; elle intègre un signal constant par morceaux, point. Son paramètre `initial_value` est **obligatoire** : la fonction refuse de deviner la valeur du signal avant le premier changement enregistré. Ce refus, qui paraît tatillon, est exactement ce qui garde l'agnosticisme de la couche. La connaissance « ma file démarre vide » reste dans le script d'expérience, qui passe explicitement 0, et ne descend jamais dans la bibliothèque d'analyse.

Un détail méthodologique important est le **warm-up**. La file démarre vide, ce qui n'est pas un tirage de la loi stationnaire mais une valeur particulière qu'on a choisie. Les premières secondes sont un transitoire systématiquement biaisé vers le bas. C'est un **biais**, pas du bruit : allonger la simulation le dilue mais ne l'élimine pas, alors que le retirer explicitement le supprime.

Reste à dimensionner ce qu'on jette, et cent secondes n'est pas un chiffre arbitraire. Le temps de relaxation d'une file est de l'ordre de $1/[(\mu + \lambda^-)(1-\rho)^2]$, soit environ $0.19$ s pour le cas de référence. Cent secondes représentent donc plusieurs centaines de temps de relaxation, pour un coût de 5 % de la fenêtre : généreux et bon marché. La leçon est que ce dimensionnement dépend de $\rho$ et non d'une valeur absolue ; à charge élevée, le même calcul donnerait plusieurs secondes de relaxation et cent secondes deviendrait tout juste confortable.

### De la moyenne à la loi entière

La moyenne $E[N]$ n'est qu'un seul nombre, le premier moment de la loi. Deux lois différentes peuvent partager la même moyenne, donc valider contre $E[N]$ seul est un critère faible. Pour une file de Gelenbe en régime stationnaire, la longueur suit une loi **géométrique** :

$$
P(N = n) = (1 - \rho)\,\rho^n,
$$

dont $E[N] = \rho/(1-\rho)$ découle par sommation. Valider contre la loi entière est strictement plus fort, exactement le même argument que celui qui justifie le KS pour l'équivalence : la moyenne des ISI ne suffisait pas là-bas, il fallait la nature de la mixture ; la moyenne de $N$ ne suffit pas ici, il faut la géométrie.

Côté observation, l'estimateur est la **fraction de temps** passée à chaque longueur, pas la fraction de changements. Une longueur atteinte souvent mais quittée immédiatement pèse presque rien dans une loi stationnaire, définie précisément comme une fraction de temps. Compter les changements donnerait le même poids à un état traversé en 10 ms qu'à un état tenu dix secondes, ce qui gonflerait artificiellement la queue de distribution. La fonction `time_weighted_histogram` fait exactement le même découpage en paliers que `time_average`, mais range chaque durée dans un dictionnaire indexé par la valeur plutôt que de tout sommer dans un accumulateur. On obtient ainsi la loi empirique complète, à comparer directement à la géométrique.

Une relation de cohérence relie les deux fonctions : la moyenne pondérée de l'histogramme doit égaler `time_average` à la précision flottante près, puisque les deux font le même découpage et ne diffèrent que par l'accumulateur. C'est un test d'invariant gratuit, qui vérifie un lien entre deux fonctions sans avoir à calculer la réponse à la main. Il n'était pas encore dans la suite à ce stade, qui n'assertait que la cohérence entre les deux formes **closes** ($\sum_n n\,P(N = n)$ redonne bien $E[N]$). Le pendant empirique a été écrit en semaine 14, où il est asserté sur chaque exécution d'une campagne de file.

### D'une tolérance pragmatique à un vrai critère statistique

Le test d'intégration de la file comparait d'abord la longueur moyenne à $E[N]$ avec une tolérance relative de 15 % sur une graine unique. C'était une bande pragmatique, pas un test statistique, et c'était la seule dette concrètement nommée. Le problème est qu'une tolérance fixe ne distingue pas un modèle correct d'un seuil simplement généreux : elle passe ou échoue sans qu'on sache pourquoi.

La version propre applique la même discipline que le test d'équivalence : **une observation par exécution indépendante**. Chaque graine produit une moyenne temporelle, ces valeurs sont i.i.d. entre graines, et on construit un intervalle de confiance de Student autour de leur moyenne,

$$
\text{IC}_{95} = \bar{\bar N} \pm t_{0.975,\,K-1}\,\frac{s}{\sqrt{K}},
$$

puis on vérifie que la forme close $\rho/(1-\rho)$ tombe dedans. Le Student plutôt que la normale parce que $K$ est petit et que l'écart-type est estimé, pas connu ; utiliser 1.96 sous-estimerait la largeur de l'intervalle. L'intérêt n'est pas d'être « plus strict » au sens naïf, mais de changer la **nature** de l'affirmation : un intervalle construit sur la dispersion observée échoue exactement quand le biais dépasse le bruit, ce qui est la question posée.

Le même dispositif, avec $\lambda^- = 0$, donne le contrôle croisé **M/M/1**. La file de Gelenbe doit alors dégénérer en file classique, de charge $\rho = \lambda^+/\mu$. La valeur validée est réellement différente (0.667 contre 0.5 du cas de référence), donc les deux configurations discriminent entre les deux lectures de $\rho$ : un bug qui ignorerait le canal de destruction passerait le cas de référence et échouerait sur le cas limite. Pour permettre $\lambda^- = 0$, j'ai choisi d'**omettre entièrement** la source négative et son câblage plutôt que d'instancier une source de taux nul. Un taux de Poisson nul n'est pas un processus dégénéré, c'est l'absence de processus ; relâcher l'invariant `rate > 0` pour la commodité d'un appelant aurait affaibli un composant correct. Puisque le signe vit dans le port, « pas de canal de destruction » **est** littéralement « pas de fil vers le port négatif ». La dérivation des flux par étiquette garantit d'ailleurs qu'omettre la source négative ne déplace pas les flux positif et de service : les deux régimes restent directement comparables, ce qu'un test vérifie en comparant l'état initial de la source positive dans les deux configurations.

### Un détail d'API à ne pas deviner

Les noms d'attributs internes de PyPDEVS 2.4.2 ne suivent aucune convention unique, et deux essais successifs sur de mauvaises hypothèses me l'ont rappelé : le couplage entrant d'un port est `inline` en minuscules, l'ensemble des sous-modèles d'un modèle couplé est `component_set` en snake_case. Ni `inLine` ni `componentSet` n'existent. La leçon, cohérente avec la discipline du projet, est d'**inspecter par `dir()` avant d'asserter** plutôt que de deviner. La seconde leçon est plus intéressante que la première : l'assertion structurelle du cas M/M/1 avait d'abord été écrite sur `inline`, un attribut interne non documenté, ce qui couplait inutilement le test à la structure de la librairie. Elle a été remplacée par un comptage de sous-modèles via `component_set`, qui est l'API publique, et la preuve comportementale reste portée par la campagne de conformité M/M/1.

**Difficultés**

- Comprendre le test de Kolmogorov-Smirnov, et surtout son interprétation correcte : la non-réfutation n'est pas une confirmation, et l'échec sur les ISI intra-run était une hypothèse i.i.d. brisée par l'autocorrélation, pas une vraie différence de loi.
- Formuler correctement pourquoi les deux écritures du MMPP divergent : ce n'est pas un ordre de consommation, mais des chemins de dérivation d'étiquettes distincts, ce qui est cohérent avec l'order-independence revendiquée par `rng.py`.
- Distinguer, sur les comptes du MMPP décomposé, un biais systématique (à corriger) d'une sur-dispersion attendue (la signature du modèle, à mettre en valeur), et construire le test discriminant plutôt que de conclure sur une seule graine.
- Comprendre les bases des G-networks au niveau conceptuel et mathématique : pourquoi $\lambda^-$ est au dénominateur, et pourquoi la destruction est un second canal de sortie.
- Saisir qu'une charge stationnaire est une moyenne temporelle, pas d'échantillons, et pourquoi la loi entière est un critère strictement plus fort que la seule moyenne.
- Le patron de l'état transitoire pour publier depuis une transition externe, contrainte propre à DEVS que rien dans les familles précédentes n'avait exigée.

## Semaine 13 (du 27 au 31 juillet 2026) : synthèse architecturale, C4 et UML selon IFT 2255

Objectif d'exploration : traduire l'architecture du projet en diagrammes qui la
prouvent plutôt qu'ils ne l'illustrent. Cela a demandé de réviser une bonne partie
des notions de génie logiciel d'IFT 2255 : le modèle C4 et ses quatre niveaux, la
notation des diagrammes de classes et de séquence, les critères de bonne
conception (cohésion, couplage), les styles architecturaux, et les quatre patrons
de conception du cours. La surprise de la semaine est que dessiner n'est pas
neutre : deux défauts réels du dépôt sont sortis de l'exercice.

### Le modèle C4 : quatre niveaux, et le quatrième est déjà connu

Le modèle C4 décrit un système par quatre vues d'abstraction décroissante : le
**système** dans son contexte (qui l'utilise, de quoi il dépend), les
**conteneurs** (les unités logiques indépendantes, ce qui peut s'exécuter
séparément), les **composants** (les modules à l'intérieur d'un conteneur), et le
**code**. Le point que je n'avais pas intégré avant de réviser : le niveau 4 n'est
pas une notation nouvelle, c'est **le diagramme de classes** de la conception
orientée objet. IFT 2255 établit cette correspondance explicitement, et demande de
savoir passer d'une vue C4 au diagramme de classes et inversement. La notation C4
est volontairement plus simple et **informelle** que UML : elle peut varier d'une
équipe à l'autre, ce qui autorise mes boîtes à porter des couleurs par type
d'élément (personne, système, conteneur, composant, externe) du moment que la
légende est là.

Pour mon projet, chaque niveau répond à une question distincte. Le niveau 1 trace
la **frontière de périmètre** : le méta-formalisme d'Abdelhamid est hors de la
boîte, mon infrastructure est dedans, et c'est précisément ce que ce niveau sert à
dire. Le niveau 2 sépare ce qui s'exécute : le paquet, la suite pytest, le dossier
de figures. Le niveau 3 montre les quatre couches et le sens de chaque flèche. Le
niveau 4 est le diagramme de classes.

> Note rétrospective : le niveau 4 était initialement un diagramme unique de 22
> classes. Après l'ajout du module de campagnes en semaine 14 il en comptait 26, et
> le rendu inline sur GitHub était devenu illisible. Il a été scindé en trois vues
> de classes participantes, chacune répondant à une question (les modèles, l'injection
> du hasard, la vérification et l'orchestration), la vue complète restant disponible
> en fichier éditable. Trois diagrammes qui répondent chacun à une question sont
> d'ailleurs plus proches de ce qu'IFT 2255 appelle un diagramme de classes
> participantes qu'un seul qui montre tout.

### Librairie contre cadriciel : l'inversion de contrôle

IFT 2255 distingue la **librairie** (mon code appelle son code, je garde la
logique de contrôle) du **cadriciel** (son code appelle le mien, je fournis des
morceaux qu'il invoque). La distinction m'a paru académique jusqu'à ce que je
doive étiqueter la flèche vers PythonPDEVS au niveau 1 : NumPy et Matplotlib sont
des librairies, mais PyPDEVS est un cadriciel, et cette inversion de contrôle est
exactement ce qui fait de la pureté de `timeAdvance` une contrainte dure plutôt
qu'un choix de style. Ce n'est pas moi qui décide combien de fois cette fonction
est appelée, c'est le simulateur. Toute la règle du pré-tirage, établie dès la
semaine 11, découle d'un concept d'architecture que je connaissais sans l'avoir
relié.

### L'architecture en couches comme propriété négative

Le style en couches organise l'application en couches ayant chacune un rôle
spécifique, les couches supérieures appelant les inférieures pour obtenir des
services, jamais l'inverse. Ce que la rédaction m'a fait comprendre, c'est que la
revendication intéressante d'un diagramme de couches est **négative** : ce ne sont
pas les flèches présentes qui portent l'argument, ce sont les flèches absentes.
Aucune arête entre `analysis` et `models`, dans aucun sens. Aucune arête sortant
de `rng`. Un diagramme d'architecture honnête se vérifie donc contre les imports
réels, pas contre l'intention : j'ai contrôlé chaque affirmation d'import avant de
la dessiner, ce qui rejoint la distinction vérification / validation (est-ce que
le produit est bien construit ; ici, est-ce que le diagramme dit vrai).

Les critères de bonne conception (forte cohésion, faible couplage, abstraction,
encapsulation) donnent le vocabulaire pour nommer ce que l'architecture fait
déjà : la dérivation par étiquette de `rng.py` est de l'encapsulation (le détail
de dérivation est caché derrière une interface stable), et le fait que
`analysis.py` consomme des listes plates est du couplage minimal. Mais le même
vocabulaire oblige à nommer les défauts : `analysis.py` porte trois
responsabilités (formes closes, estimateurs, figures), donc sa cohésion n'est pas
parfaite. Plutôt que de le cacher, le document d'architecture le déclare comme
dette acceptée avec sa condition de réexamen, ce qui est plus honnête qu'un
découpage cosmétique sur une base gelée.

### Diagramme de classes : les relations portent le sens

La notation UML fixe les visibilités (`+` public, `-` privé), la portée statique
(`{static}` ou soulignement), les classes abstraites (`{abstract}` ou italique),
et les membres écrits `nom : type`. Mais l'essentiel de la révision a porté sur
les **trois relations d'appartenance**, que je confondais partiellement :

- **Composition** (losange plein) : le composant fait partie du composite ; si le
  composite est détruit, ses composants le sont aussi. C'est exactement le lien
  entre un modèle couplé et ses sous-modèles, construits dans son constructeur et
  sans existence en dehors. La multiplicité `0..1` sur la source négative de
  `GQueueExperiment` encode le cas M/M/1 directement dans le diagramme.
- **Agrégation** (losange creux) : appartenance plus faible. Aucun cas dans mon
  code, et le noter est déjà une information.
- **Dépendance** (flèche pointillée) contre **association** (trait plein) : la
  distinction la plus utile de la semaine. Un modèle *dépend* de `RandomStream`
  (il apparaît dans la signature du constructeur et n'est pas retenu) mais est
  *associé* au `Generator` qu'il en dérive (conservé en attribut). Les deux types
  de flèches racontent l'injection de dépendances sans un mot de prose : on
  utilise le flux une fois, on garde le générateur. `MMPPNeuron` en garde deux,
  un par rôle sémantique.

La conception orientée objet ajoute une règle de sélection que j'ignorais : sur un
diagramme de classes participantes, les constructeurs, getters et setters sont
**facultatifs**. Je garde pourtant les constructeurs, et la raison mérite d'être
explicite : `rng: RandomStream` dans une signature est l'endroit exact où
l'injection se produit, et un diagramme qui le masquerait masquerait
l'architecture qu'il documente. L'omission est un choix, pas un oubli, et le
document la déclare.

### Diagramme de séquence : la traçabilité comme contrainte

Les conventions du diagramme de séquence : lignes de vie écrites `objet : Classe`,
messages synchrones en flèche pleine, retours en pointillé, messages numérotés
pour suivre le scénario. Mais la contrainte la plus structurante vient de la
méthode de réalisation des cas d'utilisation : les méthodes qui apparaissent dans
le diagramme de séquence **doivent aussi apparaître dans le diagramme de
classes**. Cette exigence de traçabilité m'a fait découvrir un trou dans mon
propre diagramme : la séquence appelait `exponential(scale)` sur une ligne de vie
`gen`, et la classe `Generator` de NumPy n'existait nulle part dans le diagramme
de classes. Elle y est maintenant, réduite aux deux seules opérations que mon code
appelle. La cohérence entre l'architecture, la conception orientée objet et
l'implémentation n'est pas un slogan : c'est un contrôle mécanique qui trouve des
trous.

Une nuance propre à mon cas : le diagramme de séquence dessine un ordre entre la
transition externe du transducteur et la transition interne du neurone que PDEVS
ne garantit pas (les sorties sont collectées d'abord, puis toutes les transitions
sont déclenchées au même instant). Le diagramme le déclare en note plutôt que de
laisser croire à une garantie. Un diagramme qui affirme plus que ce que le système
promet est un diagramme faux.

### Les patrons de conception : présents, rejetés, non introduits

IFT 2255 couvre quatre patrons : Singleton, Template Method, Stratégie,
Adaptateur. L'exercice type demande de choisir un patron **et de justifier**. La
révision m'a fait comprendre que la justification peut aussi être un rejet, et que
c'est même mon meilleur argument de conception :

- **Template Method** est présent, mais pas écrit par moi : `AtomicDEVS` fixe
  quelles opérations existent, chaque modèle concret fournit les corps, et la
  partie invariante de l'algorithme (l'ordre des appels) vit dans le simulateur.
  C'est le patron distribué de part et d'autre de la frontière du cadriciel.
- **Singleton, rejeté délibérément.** L'implémentation évidente d'une source
  d'aléa est un générateur global au niveau module. C'est un Singleton, et il
  détruit la propriété que le dépôt existe à fournir : l'ordre de construction
  déciderait des tirages, ajouter un modèle perturberait les autres. `RandomStream`
  est la conception inverse. L'absence du patron est la décision d'architecture.
- **Stratégie, non introduite.** Deux tests statistiques dans le même rôle (KS,
  bientôt CvM) seraient le déclencheur naturel. Mais ce qui varie, ce sont deux
  appels à SciPy dans un module de test, pas deux composants : la règle de trois
  n'est pas atteinte, et une Stratégie serait de l'échafaudage autour de six
  lignes.
- **Adaptateur** : absent, rien ne l'appelle.

> Le CvM est arrivé en semaine 14 et le refus a tenu tel quel, ce qui était le
> vrai test de cette prédiction : une décision de conception qui se vérifie quand
> le cas qu'elle anticipait se présente vaut plus qu'une décision jamais éprouvée.

### Ce que dessiner a trouvé

Le résultat le plus concret de la semaine n'est pas les diagrammes, mais ce que
leur construction a fait remonter. Une **fuite de couche** : `discard_warmup`,
fonction pure sur des enregistrements `(t, valeur)`, vit dans le runner de la
file et est réécrite deux fois dans les tests de conformité ; trois copies de six
lignes dans deux couches qui ne devraient pas les porter. Une **incohérence de
dépendances** : `pyproject.toml` déclare `scipy` parmi les dépendances du paquet
alors qu'aucun module sous `src/` ne l'importe.

Les deux ont été résolues en semaine 14, mais pas comme prévu, et l'écart mérite
d'être noté. La fuite de couche a bien été corrigée dans le sens annoncé, en
remontant la fonction dans `analysis.py`, et le déplacement a même révélé une
hypothèse cachée que la fonction transportait avec elle. En revanche mon
correctif annoncé pour `scipy`, le descendre dans l'extra `dev`, est celui qui
**n'a pas** été appliqué : l'arrivée de l'intervalle de confiance a fait entrer
`scipy` dans le paquet pour de bon, donc la déclaration était juste et c'est mon
diagnostic qui était prématuré. La dette s'est résolue par le haut. C'est un
rappel utile : une dette documentée est une hypothèse sur le futur, pas une
ordonnance.

C'est tout de même la meilleure défense de l'exercice : un diagramme qui ne serait
qu'une illustration n'aurait rien trouvé du tout.

**Difficultés**

- Relier des notions connues isolément (inversion de contrôle, pureté des
  fonctions DEVS) en un seul argument : le pré-tirage découle du fait que PyPDEVS
  est un cadriciel et non une librairie.
- La distinction dépendance contre association, et son application à l'injection :
  quel objet est utilisé puis relâché, quel objet est conservé.
- Accepter qu'un diagramme omette, et apprendre à déclarer l'omission plutôt
  qu'à la subir : constructeurs gardés pour montrer l'injection, accesseurs omis,
  auxiliaires de tracé repliés derrière les fonctions qui les composent.
- Formuler un rejet de patron comme une décision de conception à part entière,
  avec le même sérieux qu'une adoption.

## Semaine 14 (du 3 au 7 août 2026) : campagnes multi-graines, budget de précision et deux tests plutôt qu'un

Objectif d'exploration : donner aux trois familles le même dispositif de mesure,
en portant à toutes la discipline multi-graines que seule la file de Gelenbe
appliquait, et ajouter le test de Cramér-von Mises au critère d'équivalence sur
recommandation de mon superviseur. la question qui a commandé la semaine porte sur: **qu'est ce qu'un nombre de graines permet réellement de conclure?**

### Le nombre de graines : une recommandation, et ce qu'elle permet de conclure

Le passage à trente graines par famille vient d'une recommandation de l'équipe,
transmise par Abdelhamid. C'est aussi une valeur
usuelle en analyse de sortie de simulation. J'ai voulu comprendre ce qu'elle
apporte concrètement dans mon cas, moins pour la remettre en question que pour
pouvoir la défendre autrement qu'en citant sa provenance.

La justification qu'on associe spontanément à ce nombre est une règle de pouce
liée au théorème central limite : au-delà d'une trentaine d'observations,
l'approximation normale de la moyenne devient confortable. C'est exact en
général, mais ce n'est pas ce qui opère ici, et le remarquer m'a demandé un
détour. Mon intervalle utilise un quantile de **Student** et non de la normale,
précisément parce que l'écart-type est estimé sur l'échantillon plutôt que connu.
Or Student est construit pour les petits échantillons : il reste valide bien en
dessous de trente. Le nombre est donc bon, mais pas pour la raison que la règle
de pouce suggère.

Ce qui fixe réellement un nombre de réplications est un **budget de précision**.
La question n'est pas « combien de graines rendent le modèle valide », qui n'a
pas de réponse, mais « quelle largeur d'intervalle m'est nécessaire, et combien
de graines me la donnent », qui en a une et se vérifie après coup.

Pour ce projet, la largeur nécessaire se dérive d'une exigence de discrimination.
La file de Gelenbe a trois lectures concurrentes de $\rho$ : la lecture correcte
donne $\mathbb{E}[N] = 0.5000$, la lecture fautive qui soustrairait les négatifs
au numérateur donnerait $0.25$, et le cas limite sans destruction donne $0.6667$.
L'intervalle doit être assez étroit pour séparer ces valeurs sans ambiguïté.
Mesuré à trente graines, il vaut 1.28 % de la valeur prédite, donc elles sont
séparées de plusieurs dizaines de demi-largeurs.

Trente graines est donc à la fois la valeur recommandée par l'équipe et une
valeur que je peux justifier par ce qu'elle produit. Les deux ne se remplacent
pas : la recommandation vient d'une expérience du domaine que je n'ai pas, et la
mesure me permet de répondre à « pourquoi ce nombre » autrement que par « parce
qu'on me l'a dit ».

### L'horizon est souvent un meilleur levier que le nombre de graines

Un résultat qui m'a surpris. La demi-largeur de l'intervalle décroît en
$1/\sqrt{K}$, donc passer de dix à trente graines la divise par 1.7. Mais sur le
MMPP, la dispersion inter-graines est dominée par la **sur-dispersion**, qui
décroît elle en $1/\sqrt{T}$ avec l'horizon de chaque exécution.

J'ai mesuré : à 120 secondes par exécution, la demi-largeur vaut 3.86 % ; à 480
secondes, 1.85 %. Le facteur est de 2.09, soit exactement le $\sqrt{4}$ attendu.
Multiplier l'horizon par quatre a donc fait mieux que tripler le nombre de graines,
pour un coût de calcul comparable.

La leçon générale est qu'augmenter $K$ pour compenser un horizon trop court revient
à traiter le symptôme. Quand la dispersion vient de ce que chaque exécution est
elle-même une mesure bruitée, c'est l'exécution qu'il faut allonger. Quand elle
vient d'une vraie variabilité entre réplications, c'est le nombre de réplications
qui compte. Savoir laquelle des deux domine demande de mesurer, pas de deviner.

### Le Poisson comme point de calibration

Le dispositif à graine unique validait les familles Poisson et MMPP contre
l'intervalle $\mu \pm 1.96\sqrt{\mu}$, qui suppose variance égale à la moyenne.
Mon diagnostic de la semaine 10 avait déjà montré que ce gabarit ne convient pas
au MMPP. En le remplaçant par un intervalle bâti sur la dispersion observée, j'ai
compris quelque chose que je n'avais pas anticipé : **la famille Poisson devient
le point de calibration de tout le dispositif**.

Le raisonnement est le suivant. Pour un Poisson homogène, la variance vaut bien la
moyenne, donc les deux gabarits doivent coïncider. C'est le seul endroit du projet
où je connais la réponse d'avance. Si l'intervalle inter-graines y divergeait de
$\sqrt{\lambda T}$, ce serait la machinerie de campagne qui serait en cause, pas le
modèle. Ils coïncident, donc le désaccord observé sur le MMPP est une information
sur le processus et non un artefact de mesure.

Cela m'a donné deux tests que je n'aurais pas écrits sans y penser : l'écart-type
inter-graines du Poisson doit rester proche de $\sqrt{\lambda T}$, celui du MMPP
doit le dépasser. Le second attrape une faute qu'aucun test de conformité sur la
moyenne ne verrait, à savoir une décomposition qui aurait perdu la modulation :
elle retrouverait $\bar\lambda$ en moyenne tout en déchargeant comme un Poisson
ordinaire. Seule la dispersion sépare les deux.

Sur la figure de conformité, ce contraste est visible sans une phrase
d'explication : la bande du gabarit de Poisson englobe celle de la campagne sur le
panneau Poisson, et se réduit à un mince ruban que la moitié des points dépassent
sur les panneaux MMPP.

### Kolmogorov-Smirnov contre Cramér-von Mises : deux géométries

L'ajout du CvM à côté du KS était une recommandation de mon superviseur, et j'ai
voulu comprendre ce qu'il apporte réellement avant de l'écrire, parce que la
formulation paresseuse (« il est plus précis ») est fausse et se ferait
légitimement contredire.

Les deux tests répondent à la même question mais ne lisent pas la même chose. Le
KS retient l'écart vertical **maximal** entre les deux fonctions de répartition
empiriques,

$$
D = \sup_x |F_1(x) - F_2(x)|,
$$

autrement dit un seul point du support, celui où le désaccord est le pire. Le CvM
**intègre l'écart quadratique** sur tout le support :

$$
W^2 \propto \int (F_1(x) - F_2(x))^2 \, dF(x).
$$

La conséquence est géométrique. Deux courbes qui s'écartent beaucoup en un point
et se recollent partout ailleurs donnent un grand $D$ et une petite aire : le KS
voit, le CvM moins. Deux courbes qui restent légèrement décalées sur tout le
support donnent un $D$ modeste et une aire notable : le CvM voit, le KS moins. La
formulation correcte est donc que le CvM est **plus puissant contre certaines
alternatives**, en particulier les différences diffuses ou situées dans les queues,
et non « plus précis » dans l'absolu.

C'est ce qui m'a fait dessiner la figure d'équivalence comme je l'ai faite : les
deux fonctions de répartition superposées, le crochet vertical qui marque le $D$ que
lit le KS, et l'aire hachurée entre elles qui est ce que le CvM intègre. L'argument
devient visible au lieu d'être asserté.

### Deux statistiques qui n'en font qu'une, et il faut le dire

En lisant les résultats, un détail m'a fait tiquer : le KS rend exactement la même
statistique $D = 0.1333$ sur les comptes et sur les ISI moyens. Une coïncidence
parfaite est toujours suspecte, alors j'ai cherché pourquoi.

L'explication est simple une fois vue. L'ISI moyen d'une exécution vaut
approximativement $T / \text{compte}$, donc les deux échantillons sont presque
déterministiquement liés et ont le même **ordonnancement** entre graines. Or le KS
ne lit que des rangs, donc il ne peut pas les distinguer. Le CvM, qui intègre, les
sépare légèrement (0.0381 contre 0.0372), ce qui confirme le diagnostic.

Ce n'est pas un problème, mais c'est une chose à écrire dans les limites plutôt
qu'à laisser croire : ce sont **deux angles sur une même mesure**, pas deux
confirmations indépendantes. Présenter quatre tests comme quatre preuves séparées
serait une exagération, exactement le genre d'affirmation qu'un examinateur a
raison de mettre à l'épreuve.

### Une p-value élevée ne mesure rien

Point de rigueur que je veux garder pour la soutenance. Mes quatre p-values valent
environ 0.96, ce qui donne envie de dire « les deux écritures sont très
équivalentes ». C'est faux, et pour une raison précise : **sous l'hypothèse nulle,
une p-value est distribuée uniformément sur $[0,1]$**. Obtenir 0.96 est donc un
tirage aussi ordinaire que d'obtenir 0.30. La p-value n'est pas une mesure de
ressemblance, c'est la probabilité d'observer un écart au moins aussi grand si
l'hypothèse nulle est vraie.

La conclusion correcte reste donc celle de la semaine 10, inchangée malgré trente
graines au lieu de dix : aucun des deux tests n'est parvenu à distinguer les deux
écritures, à ce pouvoir et sur ces graines. Ce n'est pas une preuve d'identité.

### Distinguer un biais d'un bruit, à nouveau

Les moyennes du MMPP sont sorties légèrement sous la prédiction, $-2.08$ % à 120
secondes puis $-1.24$ % à 480. Deux valeurs du même signe, ce qui a réveillé le
même soupçon qu'en semaine 10 : la chaîne démarre dans un état fixé plutôt que
tiré selon sa distribution stationnaire $\pi$, et comme elle démarre au repos
(5 Hz), chaque exécution passe ses premières secondes sous-cadencée. Un biais
d'amorçage produirait exactement un déficit négatif décroissant avec l'horizon.

Le test discriminant est le même qu'en semaine 10, mais appliqué à une autre
variable. Si c'est un transitoire d'amorçage, le déficit **absolu** en spikes doit
être à peu près constant quel que soit l'horizon, puisque c'est un nombre fixe de
spikes manqués au début. Si c'est du bruit, il doit varier sans structure.

Mesuré sur trois horizons : $-29.9$ spikes à 120 secondes, $-71.5$ à 480, et
$+108.9$ à 1920. Le déficit n'est pas constant, il **change de signe**, et la
proportion de graines sous la prédiction reste autour de la moitié aux trois
horizons (16, 17 et 13 sur 30). C'est du bruit.

Mon hypothèse était donc fausse, et comprendre pourquoi elle était plausible mais
mal calibrée est instructif : le biais d'amorçage **existe** bel et bien, mais le
temps de séjour moyen au repos est de 2 secondes sur des horizons de 120 secondes
et plus, donc l'effet se compte en quelques spikes contre une dispersion
inter-graines de plusieurs dizaines. La sur-dispersion du MMPP est un bruit
beaucoup plus gros que ce biais. Il reste dans les limites du projet, mais formulé
comme ce qu'il est : mesuré et indétectable à ce budget, pas absent.

La même vérification a été faite sur la file : 18 moyennes sur 30 au-dessus de la
prédiction au cas de référence et 16 sur 30 au cas limite, donc le warm-up de cent
secondes suffit. Un warm-up trop court aurait donné une nette majorité du même
côté.

### Le coût d'une campagne, et pourquoi il n'a pas été un obstacle

J'avais surestimé d'au moins deux ordres de grandeur le temps de calcul d'une
campagne, ce qui m'aurait conduit à des compromis inutiles. Une exécution de file
de 1500 secondes prend 0.09 seconde, et la suite complète, campagnes comprises,
tourne en quelques secondes sur un portable.

La bonne pratique que j'en retire est de **mesurer avant d'arbitrer**. J'ai
chronométré une exécution avant de fixer les horizons et le nombre de graines, ce
qui a supprimé une contrainte que je croyais avoir.

Le point d'ingénierie qui a rendu cela possible est l'usage de fixtures pytest à
portée de **session** plutôt que de module. Les campagnes MMPP servent à la fois le
critère de conformité et le critère d'équivalence, qui vivent dans deux modules
différents ; une portée de module aurait payé chaque campagne deux fois. Avec une
portée de session, les trente exécutions d'une famille sont payées une fois et
partagées par tous les tests qui les lisent. Résultat contre-intuitif mais mesuré :
en ajoutant quatre campagnes et en triplant le nombre de graines, la suite complète
est devenue **plus rapide** qu'avant, parce que l'ancienne version relançait ses
campagnes à chaque test.

### Renverser une décision écrite

Le dernier point de la semaine est de méthode plutôt que de technique. En semaine
12, j'avais écrit qu'il n'y aurait **pas** de cinquième runner en ligne de
commande, au motif que les campagnes assertent une propriété et ne produisent
aucune figure, ce qui en fait des tests et non des expériences.

En semaine 14, j'ai écrit ce cinquième runner. La moitié de l'argument était
devenue fausse : les campagnes produisent maintenant deux figures et un tableau de
résultats, donc le runner satisfait exactement le critère qui justifiait la place
des quatre autres.

Ce qui m'intéresse ici est la forme du renversement. Une décision documentée
énonce des prémisses ; quand l'une d'elles cesse de tenir, la décision doit être
révisée **en nommant laquelle**. C'est très différent d'abandonner une décision en
silence, ce qui laisserait un lecteur incapable de distinguer un changement d'avis
motivé d'un oubli. La même chose vaut pour la dette `scipy` de la semaine 13 : mon
correctif annoncé n'était pas le bon, et le dire vaut mieux que de réécrire
l'histoire.

**Difficultés**

- Comprendre qu'un nombre de réplications se dérive d'une exigence de précision et
  non d'un seuil de validité, et savoir formuler la justification sans s'appuyer
  sur un argument d'autorité.
- Identifier lequel de $K$ ou de l'horizon est le bon levier sur la largeur d'un
  intervalle, ce qui demande de savoir d'où vient la dispersion dominante.
- Formuler correctement l'apport du CvM par rapport au KS : une sensibilité à une
  autre forme d'écart, et non une précision supérieure.
- Repérer que deux statistiques que je croyais complémentaires sont en fait deux
  lectures d'une même mesure, et l'écrire dans les limites plutôt que de laisser
  croire à deux preuves.
- Résister à la lecture naturelle d'une p-value élevée comme mesure de
  ressemblance.
- Rejouer le test discriminant biais contre bruit sur la bonne variable : le
  déficit absolu à travers plusieurs horizons, et non le déficit relatif sur un
  seul.
