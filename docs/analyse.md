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

### Graine, PRNG et reproductibilité

Une **graine** (*seed*) est le point de départ déterministe d'un générateur
pseudo-aléatoire : une même graine produit toujours la même séquence, donc des
résultats reproductibles par d'autres chercheurs (c'est un concept qu'Abdelhamid avait
souligné pour la reproduction d'expériences). Intuition via un LCG jouet
$x_{n+1} = (5x_n + 3) \bmod 16$, en partant de $7$ : $6, 1, 8, 11, 10, \dots$

Le travail sur **MRG32k3a** (L'Écuyer) généralise cette idée : période très longue et
**sous-flux** (*substreams*) indépendants, ce qui donne à la fois le parallélisme et
la reproductibilité, avec un sous-flux par neurone. Trajectoire d'implémentation
envisagée, où la couche RNG reste échangeable en un seul endroit :

```python
self.rng = np.random.default_rng(seed)        # aujourd'hui (NumPy)
self.rng = LEcuyerRNG.substream(neuron_id)    # plus tard (backend MRG32k3a)
```

**Difficultés**

- Le concept de graine et de PRNG est revenu à plusieurs sessions : clarifié par le LCG jouet, mais encore à consolider du côté MRG32k3a et des sous-flux.
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

**Difficultés**

- S'assurer de passer $1/\lambda$ (et non $\lambda$) à NumPy, dont la convention `scale` est la moyenne.
- Le retour d'état sous forme de dictionnaire conservé dans l'état du modèle.
- Bien saisir le rôle du `Transducer` et ce qu'un modèle couplé signifie au sens DEVS.
- Côté expérience, cerner quoi afficher et quelles données sont réellement importantes à analyser.
- Généraliser le seed en classe me donne du fil à retordre, mais m'amène à réfléchir davantage comme un ingénieur logiciel, ce qui est positif.