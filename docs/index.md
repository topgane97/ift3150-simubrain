# SimuBrAIn: Architecture logicielle pour la simulation stochastique à événements discrets

!!! abstract "Projet IFT 3150, Été 2026"

    **Étudiant :** Ryan Chahri, 
    **Superviseur académique :** Eugène Syriani (UdeM), 
    **Expert :** Alexandre Muzy (CNRS), 
    **Encadrant :** Abdelhamid (maîtrise).

## Contexte

SimuBrAIn est un projet de recherche international **UdeM (Canada) / CNRS (France)** dont l'objectif à long terme est de construire des **jumeaux numériques personnalisés du cerveau humain**. Le projet s'articule autour de trois axes complémentaires :

- **Axe 1: Modèles fondés sur la simulation** *(Canada, UdeM)*
- **Axe 2: Modèles fondés sur l'IA** *(France, CNRS)*
- **Axe 3: Modèles hybrides simulation-IA** *(collaboration)*

L'Axe 1, Objectif 2 vise la conception d'une **extension stochastique de DEVS** : un formalisme unique capable de décrire, sous la même écriture, les processus de Poisson, les chaînes de Markov et les files d'attente. Cette conception est menée par Abdelhamid, étudiant à la maîtrise.[^1]

[^1]: L'équipe désigne aussi ce travail par le terme « méta-formalisme », puisqu'un formalisme qui embrasse plusieurs autres formalismes se situe un niveau au-dessus d'eux.

!!! info "Vocabulaire de départ"

    - **Stochastique** : qui comporte du hasard. Un neurone ne décharge pas à intervalles réguliers ; le moment de chaque décharge est tiré au hasard selon une loi de probabilité.
    - **Événement discret** : le temps ne s'écoule pas par petits pas réguliers, il saute d'un événement au suivant. Entre deux décharges, il n'y a rien à calculer.
    - **RNG** *(random number generator, générateur de nombres aléatoires)* : le composant logiciel qui produit le hasard. Il est en réalité déterministe : à partir d'un nombre de départ appelé **graine** *(seed)*, il déroule une suite de nombres qui paraît aléatoire mais qui est toujours la même pour une graine donnée. C'est ce qui rend une simulation stochastique rejouable à l'identique. Un **flux** *(stream)* est une suite issue d'une graine ; deux flux différents doivent être indépendants, pour que le hasard d'un neurone n'influence pas celui d'un autre.
    - **DEVS** : un langage standard pour décrire de tels systèmes. Un modèle DEVS répond à quatre questions : dans quel état suis-je, combien de temps j'y reste, qu'est-ce que j'émets en sortant, et dans quel état je passe ensuite.
    - **Extension stochastique de DEVS** : DEVS décrit bien les systèmes dont le comportement est fixé à l'avance. L'étendre au hasard signifie autoriser les durées et les transitions à être tirées selon des lois de probabilité, et le faire de façon assez générale pour couvrir plusieurs familles de processus aléatoires sous une même écriture.
    - **Modèle atomique / modèle couplé** : un modèle atomique est une brique indivisible. Un modèle couplé est un assemblage de briques reliées par des fils. Un modèle couplé peut lui-même servir de brique.

!!! note "Mon rôle"

    Mon projet ne porte pas sur la conception de l'extension stochastique de DEVS, mais sur une question distincte et préalable : **quelle infrastructure logicielle permet de construire un modèle DEVS stochastique dont on peut prouver qu'il est correct ?**

    Sans réponse à cette question, une extension stochastique de DEVS reste une belle idée qu'on ne sait pas vérifier. Je construis cette infrastructure et je la mets à l'épreuve.

## Problématique

DEVS dit comment **décrire** un système à événements discrets. Il ne dit rien sur la façon d'en **écrire le code** ni sur la façon de **vérifier que ce code est correct** quand le système comporte du hasard.

Écrit naïvement, un modèle stochastique mélange trois choses différentes dans une seule classe :

1. la **logique du modèle** : dans quel état je suis, quand je change d'état, ce que j'émets ;
2. le **tirage au sort** : d'où vient le hasard, quelle graine l'initialise ;
3. la **vérification** : est-ce que ce modèle produit vraiment la bonne loi de probabilité ?

Ce mélange produit trois problèmes concrets.

!!! warning "Trois problèmes"

    **1. Le hasard est soudé au modèle.** Si chaque modèle crée lui-même son générateur aléatoire, l'ordre dans lequel on construit les modèles change les résultats. Ajouter un neurone à un réseau modifie les décharges de tous les autres. Une expérience cesse alors d'être rejouable, et deux modèles ne peuvent plus être testés isolément.

    **2. La vérification est enfermée dans le modèle.** Si le modèle calcule lui-même ses propres statistiques, ce code de vérification ne peut servir à aucun autre modèle. Il faut le réécrire à chaque nouveau type de neurone.

    **3. Il n'y a rien à quoi comparer la sortie.** Un test logiciel ordinaire compare un résultat obtenu à un résultat attendu : « la fonction doit rendre 4 ». Un modèle stochastique n'a pas de résultat attendu, il rend une valeur différente à chaque exécution. La question « ce modèle est-il correct ? » n'a donc pas de réponse par oui ou non toute prête.

Et une question qui commande toutes les autres. Un même processus peut s'écrire de deux façons : **en une seule brique** qui fait tout, ou **en plusieurs briques reliées** qui font la même chose ensemble. La seconde forme est celle dont un projet à l'échelle du cerveau a besoin, parce qu'elle seule permet d'assembler et de réutiliser. Mais alors :

> **Comment démontrer que la version en plusieurs briques fait vraiment la même chose que la version en une seule ?**

Tant qu'on ne sait pas répondre à ça, dire d'une extension stochastique de DEVS qu'elle est composable est une affirmation qu'on ne peut pas vérifier.

## Hypothèse

!!! tip "Mon hypothèse de travail"

    Un modèle DEVS stochastique devient rejouable, testable et assemblable si l'on impose deux choses :

    1. **Séparer le code en trois couches étanches** : les modèles DEVS d'un côté, la source de hasard de l'autre, la vérification statistique en troisième. Chaque couche ignore les deux autres autant que possible.

    2. **Remplacer le test « résultat attendu = résultat obtenu », qui n'existe pas ici, par trois critères de correction de nature différente.**

### Les trois critères de correction

Puisqu'on ne peut pas comparer une sortie aléatoire à une valeur fixe, on vérifie trois choses distinctes, de la plus stricte à la plus faible.

| Niveau | Nom | Question posée | Nature |
|:------:|-----|----------------|--------|
| 1 | **Invariants** | Le modèle respecte-t-il ses propres règles, quelle que soit la valeur tirée ? | Certitude |
| 2 | **Conformité à la théorie** | La sortie observée correspond-elle à la formule mathématique calculée à l'avance ? | Statistique |
| 3 | **Équivalence** | Deux écritures différentes du même processus sont-elles indiscernables ? | Statistique |

**Niveau 1 Invariants.** Ce sont des propriétés vraies à chaque exécution, indépendamment du hasard. Exemples : une file d'attente ne contient jamais un nombre négatif de clients ; deux exécutions lancées avec la même graine donnent exactement le même résultat ; un taux de décharge est toujours strictement positif. Ces propriétés se testent comme du code ordinaire, sans même lancer de simulation.

**Niveau 2 Conformité à la théorie.** On calcule à l'avance, à partir des seuls paramètres du modèle, la valeur que la théorie prédit. On lance ensuite la simulation et on vérifie que ce qu'on observe tombe dans l'intervalle de confiance autour de cette prédiction. Le point crucial : **le calcul théorique ne regarde jamais la sortie de la simulation**, sinon il ne prédirait plus rien, il décrirait.

**Niveau 3 Équivalence.** On écrit le même processus de deux façons, on lance les deux, et on vérifie qu'aucun test statistique ne parvient à les distinguer.

!!! danger "Le point délicat : deux sens de « pareil »"

    On pourrait croire que deux modèles équivalents doivent produire **exactement la même suite d'événements**. C'est faux, et c'est le piège central du projet.

    Les deux versions dérivent leurs générateurs le long de chemins d'étiquettes différents : la version en une brique tire dans un seul flux, la version en plusieurs briques tire dans deux flux séparés, chacun identifié par son propre nom. Leurs suites d'événements **diffèrent nécessairement**. Exiger qu'elles soient identiques déclarerait fausse toute décomposition, y compris les correctes.

    Le bon critère n'est pas « même suite » mais **« même loi de probabilité »**. Deux dés honnêtes ne donnent pas la même suite de résultats ; ce sont pourtant le même dé. C'est cette égalité-là qu'il faut tester, et elle se teste par comparaison de distributions.

    **Ma sous-hypothèse :** si un test statistique n'arrive pas à distinguer la version en une brique de sa version en plusieurs briques, alors la décomposition est **une écriture valide du même modèle**. C'est précisément le critère dont l'extension stochastique de DEVS a besoin pour justifier qu'il est composable. 

## Solution

L'infrastructure est mise à l'épreuve sur **trois familles de processus** de complexité croissante, chacune traitée selon le même motif.

!!! info "Les trois familles"

    - **Poisson** : un neurone qui décharge au hasard à un rythme constant. Le cas de base.
    - **MMPP** *(Markov-Modulated Poisson Process)* : un neurone dont le rythme n'est plus constant mais commandé par un mécanisme caché qui bascule entre régimes, par exemple repos (5 Hz) et actif (40 Hz). Deux sources de hasard imbriquées.
    - **G-networks** *(Gelenbe)* : des files d'attente où circulent des signaux positifs (qui ajoutent du travail) et négatifs (qui en retirent). C'est la métaphore de l'excitation et de l'inhibition entre neurones.

### Les trois couches

| Couche | Rôle | Ce qu'elle garantit |
|--------|------|---------------------|
| **Hasard** | Distribue les flux aléatoires, à partir d'une seule graine | Chaque flux est identifié par un **nom**, pas par un numéro d'ordre. Ajouter un neurone au réseau ne dérange aucun autre neurone. |
| **Modèles DEVS** | La logique de simulation, briques atomiques et assemblages | Le hasard est **reçu de l'extérieur**, jamais créé à l'intérieur. Un modèle déclare son besoin, il ne se sert pas tout seul. |
| **Vérification** | Compare l'observé au théorique, trace les figures | **N'importe rien du simulateur.** Elle ne reçoit qu'une liste de temps. Elle peut donc observer n'importe quel modèle, présent ou futur, sans modification. |

Les dépendances vont toujours dans le même sens : les modèles reçoivent du hasard, la vérification reçoit des résultats. Aucune flèche ne remonte.

### Les quatre briques d'une expérience

Chaque famille de processus est montée selon le même schéma, hérité de la théorie de la simulation (Zeigler) :

| Brique | Rôle | Exemple |
|--------|------|---------|
| **Source** | Le modèle qu'on veut étudier. Il produit des événements. | Un neurone qui décharge |
| **Sonde** | Un modèle passif qui écoute et note la date de chaque événement reçu. Il ne produit rien lui-même. | Le transducteur |
| **Assemblage** | Le modèle couplé qui relie la sortie de la source à l'entrée de la sonde. | Un fil entre les deux |
| **Expérience** | Le programme qui fixe les paramètres et la graine, lance la simulation pour une durée donnée, relit ce que la sonde a noté, et compare au théorique. | Un script en ligne de commande |

Deux propriétés rendent ce schéma réutilisable. La **sonde ignore ce qu'elle observe** : elle note des couples (date, contenu) sans savoir s'il s'agit de décharges, de clients ou d'autre chose. Elle sert donc pour les trois familles sans modification. Et **la source ignore qu'elle est observée** : brancher une sonde ne change rien à son comportement.

Séparer la source de la sonde a un autre effet : le modèle sous test ne mesure pas ses propres résultats. C'est ce qui rend possible la couche de vérification indépendante.

### Le motif de validation

Les trois familles partagent le même dispositif de mesure : seule la source change, la sonde, l'assemblage et l'expérience sont identiques. Chaque famille est validée par les critères 1 et 2 : ses invariants, puis la conformité de sa sortie à la formule théorique.

Le critère 3, l'équivalence, ne s'applique pas partout, et c'est délibéré :

| Famille | Écriture | Pourquoi |
|---------|----------|----------|
| **Poisson** | Une seule brique | Il n'y a rien à décomposer : une source, un seul flux de hasard |
| **MMPP** | Les deux, puis confrontées | Deux sources de hasard imbriquées : c'est le premier cas où la décomposition est un vrai choix, donc le premier où elle demande une preuve |
| **G-networks** | Plusieurs briques d'emblée | L'assemblage est l'objet même du modèle ; un monolithe n'aurait aucun sens pour un réseau |

Le MMPP est donc le **banc d'essai du critère d'équivalence** : le plus simple des cas où décomposer est possible, et où l'on peut vérifier que la décomposition ne trahit pas le modèle. La méthode y est mise à l'épreuve une fois, sérieusement, plutôt que trois fois pour la forme.

**Je n'ai pas construit trois modèles : j'ai construit une infrastructure et une méthode de vérification, éprouvées sur trois familles de complexité croissante.**


## Impact à long terme

- **Pour SimuBrAIn.** L'extension stochastique de DEVS conçu par l'équipe hérite d'un harnais de vérification et d'une gestion du hasard déjà éprouvés sur trois familles de processus. Surtout, il hérite du critère d'équivalence : l'instrument qui permet de tester, et non seulement d'affirmer, qu'il se décompose correctement.

- **Pour le passage à l'échelle.** Le cerveau compte environ \(10^{11}\) neurones. Identifier les flux de hasard par nom plutôt que par ordre d'arrivée est ce qui rend possible de rejouer un neurone précis sans rejouer tout le réseau, et d'ajouter un neurone sans invalider les autres. Ce n'est pas un détail technique, c'est un prérequis d'architecture.

- **Au-delà des neurosciences.** La séparation en trois couches et les trois critères de correction s'appliquent à tout système à événements discrets comportant du hasard : réseaux informatiques, épidémiologie, files d'attente industrielles.

!!! note "Ce que ce travail n'est pas"

    Le test d'équivalence est un **critère empirique réfutable**, pas une preuve mathématique. Ne pas parvenir à distinguer deux modèles n'est pas la même chose que démontrer qu'ils sont identiques : c'est un échec à les distinguer, répété sur plusieurs graines. 

## Validation et évaluation

- **Invariants** : tests automatisés déterministes sur la pureté des fonctions, le format des messages échangés, le rejet des paramètres invalides, et la reproductibilité à graine fixe. Ces tests ne lancent aucune simulation.
- **Conformité à la théorie** : comparaison du taux mesuré au taux calculé à l'avance. Pour le MMPP, le taux effectif \(\bar\lambda = \sum_i \pi_i \lambda_i\), moyenne des taux pondérée par le temps passé dans chaque régime. Pour la file de Gelenbe, la charge \(\rho = \lambda^+ / (\mu + \lambda^-)\).
- **Équivalence** : test de Kolmogorov-Smirnov comparant les distributions produites par la version en une brique et la version en plusieurs briques, sur plusieurs graines indépendantes.
- **Reproductibilité** : dépôt GitHub avec README, tests et expériences relançables à l'identique depuis une seule graine.

## Méthodologie

Le projet avance famille par famille, sur des processus de complexité croissante, en réutilisant le même dispositif de mesure et en renforçant l'infrastructure à mesure que les modèles l'exigent :

| Étape | Famille | Contenu |
|:-----:|---------|---------|
| 1 | **Poisson** | Source à rythme constant, sonde, assemblage, expérience · comparaison observé vs théorique |
| 2 | **Infrastructure RNG** | Extraction de la couche de hasard et de la couche de vérification hors des modèles |
| 3 | **MMPP** | Source en une brique, source en plusieurs briques, test d'équivalence |
| 4 | **G-networks** | File de Gelenbe en plusieurs briques, comparaison à la formule de Gelenbe |

Travail en **Python** avec **PyPDEVS**, sous **Git/GitHub**, avec rencontres hebdomadaires de supervision.

## Échéancier

!!! info "Suivi détaillé"
    Le suivi complet est disponible dans la page [Suivi de projet](suivi.md).

### Plan prévisionnel

| Période | Activités | Livrable / Jalon |
|---------|-----------|------------------|
| **Semaine 1**<br>4 → 8 mai | Setup du site web du cours · environnement conda + PyPDEVS · dépôt GitHub | Environnement opérationnel |
| **Semaines 2–4**<br>11 → 29 mai | Familiarisation PyPDEVS (neurone Poisson, transducer, modèle couplé) · validation taux empirique vs théorique · extension queue-based (file recevant les spikes) | Neurone de Poisson, transducer, modèle couplé et File recevant les spikes |
| **Semaines 5–8**<br>1ᵉʳ → 26 juin | Comprendre Markov et MMPP (*Markov-Modulated Poisson Process*) · comprendre les G-networks de Gelenbe · implémenter un de ces modèles + valider | Markov/MMPP et/ou G-networks de Gelenbe et Mise en commun I-II |
| **Semaines 9–12**<br>29 juin → 24 juillet | Implémentation du neurone stochastique via l'extension stochastique de DEVS · benchmarks de performance | neurone stochastique via l'extension stochastique de DEVS, benchmarks de performance et Mise en commun III |
| **Semaine 13**<br>27 → 31 juillet | Foire : kiosque de démonstration · finalisation du dépôt (README, exemples reproductibles) | Démo + repo finalisé |
| **Semaines 14–15**<br>3 → 14 août | Rédaction du rapport final · présentation finale (25 min) | Rapport + présentation |

### Plan révisé

| Période | Activités | Livrable / Jalon |
|---------|-----------|------------------|
| **Semaines 1–4**<br>4 → 29 mai | Environnement, dépôt, familiarisation PyPDEVS · neurone de Poisson, sonde d'observation, assemblage couplé · tests d'invariants | Famille Poisson : modèles atomiques et couplé |
| **Semaine 5**<br>1ᵉʳ → 5 juin | Couche de vérification indépendante du simulateur · comparaison observé vs théorique du neurone de Poisson | Couche de vérification · validation Poisson |
| **Semaines 6–7**<br>8 → 19 juin | Couche de hasard : flux dérivés par nom, indépendants de l'ordre · étude de Markov DEVS et de Gelenbe | Couche de hasard injectée · architecture en trois couches établie |
| **Semaine 8**<br>22 → 26 juin | Neurone MMPP en une brique (deux horloges concurrentes, générique en nombre de régimes) · invariants · conformité au taux effectif | Famille MMPP : version en une brique validée |
| **Semaines 9–10**<br>29 juin → 10 juillet | Version MMPP en plusieurs briques (mécanisme de commutation + source à rythme variable) · **test d'équivalence de Kolmogorov-Smirnov** entre les deux versions | **Critère de niveau 3 : équivalence entre écritures** |
| **Semaine 11**<br>13 → 17 juillet | File de Gelenbe en plusieurs briques (arrivées positives et négatives, service) · invariants de file · expérience couplée | Famille G-networks : file de Gelenbe assemblée |
| **Semaine 12**<br>20 → 24 juillet | Conformité de la file de Gelenbe à \(\rho = \lambda^+/(\mu + \lambda^-)\) · cas limite sans signal négatif (M/M/1) | **Trois familles validées · code gelé** |
| **Semaine 13**<br>27 → 31 juillet | Diagrammes UML (classes en couches,C4) · finalisation du dépôt (README, exemples reproductibles) et début du rapport final · Foire : kiosque de démonstration | Synthèse architecturale · démo · dépôt finalisé | 
| **Semaine 14**<br>3 → 7 août | Continuer la rédaction du rapport final (problème, hypothèse, solution, impact) et début de la préparation à la présentation finale | **Rapport final remis le 7 août** |
| **Semaine 15**<br>10 → 14 août | Préparation et présentation finale (25 min) | Présentation finale |

## Navigation du site

- **[Suivi](suivi.md)**: Journal de bord et avancement
- **[Journal de bord](analyse.md)**: Analyse et recherche
- **[Réalisation](realisation.md)**: Implémentation
- **[Évaluation](evaluation.md)**: Résultats et validation