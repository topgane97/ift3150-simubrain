---
title: Évaluation & Discussion
---

<style>
    @media screen and (min-width: 76em) {
        .md-sidebar--primary {
            display: none !important;
        }
    }
</style>

# Évaluation

*Dernière mise à jour : semaine 11 (mi-juillet 2026).*

Cette page présente la stratégie de test, les résultats obtenus sur les trois familles de processus, leur interprétation et les limites actuelles.

## Méthodes de validation

La validation opère selon les trois critères de correction posés en hypothèse. Chacun a une nature propre, et le projet les garde volontairement séparés : les tests automatisés vérifient ce qui est déterministe, les expériences valident ce qui est statistique, et le test d'équivalence tranche une question qu'aucun des deux ne pose.

### Critère 1 : les invariants

La suite pytest (122 tests) couvre les invariants structurels et déterministes, sans lancer de simulation dans la grande majorité des cas.

Côté structure : le câblage des sous-modèles, les gardes de validité qui lèvent une exception (`rate > 0` pour le neurone de Poisson ; pour le MMPP et la chaîne de Markov, taux strictement positifs, $Q$ carrée et compatible avec `rates`, lignes de somme nulle, hors-diagonale non négative, taux de sortie strictement positif ; pour la file de Gelenbe, taux de service strictement positif et longueur initiale non négative), le `timeAdvance` infini du Transducer, et l'enregistrement chronologique des couples `(temps, payload)` dans l'horizon de simulation.

Côté déterminisme : la reproductibilité par graine (deux modèles de même graine produisent exactement la même séquence de tirages), la pureté du `timeAdvance` et de l'`outputFnc`, et l'exactitude des fonctions de calcul (`compute_isi`, `theoretical_stats`, `time_average`, `gqueue_mean_length`).

Chaque famille ajoute les invariants propres à sa machine à états :

| Famille | Invariants spécifiques testés |
|---------|-------------------------------|
| **MMPP** | Résolution des deux horloges concurrentes (victoire du spike, victoire de la transition, priorité au spike en cas d'égalité) · décrément de l'horloge de transition plutôt que re-tirage quand un spike gagne · généricité du nombre d'états (une chaîne à trois états simule sans cas particulier) |
| **MarkovChain** | Le taux publié par `outputFnc` est exactement celui de l'état auquel `intTransition` s'engage (cohérence du pré-tirage de destination) |
| **G-queue** | $n \geq 0$ sous rafale de signaux négatifs · l'invariant $n = 0 \iff t_{\text{service}} = \infty$ survit à chaque chemin (départ, destruction, arrivée) · perte silencieuse d'un négatif sur file vide · conservation du résiduel de service lors d'une arrivée · confluence départ-puis-arrivée |

La confluence mérite d'être signalée. PyPDEVS applique par défaut `intTransition` puis `extTransition` quand un départ et une arrivée coïncident. Ce comportement est correct pour la file de Gelenbe, mais le projet ne s'en remet pas au défaut : deux tests l'assertent directement, de sorte qu'un changement de version de la librairie ne pourrait pas modifier silencieusement la sémantique du modèle.

Aucun de ces tests ne vérifie que le taux empirique tombe près de sa valeur théorique. C'est le principe directeur de la stratégie : une assertion portant sur une grandeur aléatoire serait soit instable, soit trivialement vraie. Tester le déterminisme et valider le hasard sont deux activités distinctes, et les mélanger affaiblirait les deux.

### Critère 2 : la conformité à la théorie

La validation statistique est confiée aux expériences en ligne de commande. Dans chaque cas, la prédiction est calculée **à partir des seuls paramètres du modèle**, jamais de sa sortie.

| Famille | Prédiction | Grandeur validée |
|---------|-----------|------------------|
| **Poisson** | $\mathbb{E}[N(T)] = \lambda T$, $\mathbb{E}[\text{ISI}] = 1/\lambda$ | Compte de spikes |
| **MMPP** | $\bar\lambda = \sum_i \pi_i \lambda_i$, avec $\pi$ la stationnaire de la CTMC | Compte de spikes |
| **G-queue** | $\rho = \lambda^+/(\mu + \lambda^-)$, $\mathbb{E}[N] = \rho/(1-\rho)$ | Longueur moyenne de file |

Pour les deux premières familles, l'intervalle de confiance à 95 % sur le compte vient de l'approximation normale $\mu \pm 1.96\sqrt{\mu}$. Pour la file de Gelenbe, la grandeur validée n'est pas un compte d'événements mais une **charge stationnaire**, ce qui a demandé une extension de la couche de vérification décrite plus bas.

### Critère 3 : l'équivalence entre écritures

Le test d'équivalence compare les distributions produites par le MMPP en une brique et le MMPP en plusieurs briques, par un test de Kolmogorov-Smirnov à deux échantillons, sur dix graines indépendantes de 120 secondes chacune.

Deux statistiques sont comparées : le **compte de spikes par graine**, qui teste l'échelle et la signature de sur-dispersion du MMPP, et l'**ISI moyen par graine**, résumé scalaire de la position de la loi des ISI.

Le choix d'agréger **une observation par exécution** plutôt que d'utiliser la séquence des ISI intra-exécution est le point méthodologique central de ce test, et il n'est pas cosmétique. Le KS suppose des échantillons i.i.d. Or la séquence des ISI d'un même run viole cette hypothèse : les longs intervalles se groupent dans l'état lent de la CTMC, donc les ISI sont autocorrélés. Avec environ 1400 ISI corrélés par exécution, la variance de la fonction de répartition empirique est sous-estimée, et le test rejette sur des écarts de 6 à 7 % qui ne signalent aucune différence réelle. C'est un faux positif dû à une hypothèse brisée, pas une découverte. Agréger par graine rend chaque entrée du KS indépendante des autres.

Le coût est explicite : on perd de la résolution sur la **forme** de la loi des ISI. Cette affirmation plus fine est laissée aux figures de comptage cumulé, qui montrent déjà la sur-dispersion partagée par les deux écritures. Baisser le seuil $\alpha$ pour faire passer le test sur les ISI intra-run aurait été du *p-hacking*.

## Extension de la couche de vérification pour la file de Gelenbe

La file de Gelenbe a posé un problème que les deux premières familles n'avaient pas : `analysis.py` ne savait lire que des **trains d'événements**, alors que $\rho$ est une charge stationnaire, donc la validation porte sur une **longueur de file moyenne dans le temps**.

Trois options se présentaient. Reconstruire $N(t)$ à partir des arrivées et des départs aurait fait fuir la sémantique « file d'attente » dans la couche de vérification, ce que l'architecture interdit. Échantillonner périodiquement la longueur aurait introduit un biais de discrétisation et un nouveau modèle à valider.

L'option retenue préserve l'étanchéité des couches. La file publie sa longueur sur un port `length_out` à chaque changement ; le `Transducer` enregistre les couples $(t, n)$ **sans une ligne de modification**, puisqu'il est déjà agnostique au payload ; et `analysis.py` gagne une fonction `time_average` qui intègre un signal en escalier :

$$
\bar{v} = \frac{1}{T}\left[v_{\text{init}} \cdot t_0 + \sum_{i=0}^{n-2} v_i (t_{i+1} - t_i) + v_{n-1}(T - t_{n-1})\right]
$$

Cette fonction ne sait pas qu'il s'agit d'une file. Elle intègre un signal constant par morceaux, point. Son paramètre `initial_value` est **obligatoire** : la fonction ne devine pas la valeur du signal sur $[0, t_0)$. Pour la file de Gelenbe, l'appelant passe 0 ; le fait qu'il doive le dire explicitement rend l'hypothèse visible plutôt que cachée dans une valeur par défaut.

Un obstacle technique a dû être franchi au passage. Un modèle DEVS atomique ne peut pas émettre de sortie depuis une transition externe : `length_out` n'aurait donc publié que les **départs**, ratant toutes les montées de $n$. La solution retenue est le patron DEVS standard de l'état transitoire : sur une arrivée, la file s'auto-réveille avec un `timeAdvance` nul, publie la nouvelle longueur, puis reprend son service en cours. Le service pendant n'est ni consommé ni re-tiré par ce passage, ce qu'un test vérifie explicitement.

## Résultats obtenus

La suite pytest passe entièrement au vert (122 tests), confirmant les invariants structurels et déterministes des modèles, de la couche RNG et de la couche d'analyse.

### Neurone de Poisson homogène

La validation statistique est lancée par :

    python -m simubrain.experiments.run_basic_experiment --rate 20 --duration 60 --seed 42

Pour ce neurone à $\lambda = 20$ Hz simulé sur 60 secondes (seed 42) :

- compte de spikes : 1195 mesurés contre 1200 attendus, dans l'intervalle de confiance à 95% [1132, 1268] ;
- intervalle inter-spikes moyen : 0.0502 s contre 0.0500 s attendu ;
- taux empirique : 19.917 Hz contre 20 Hz attendu.

![Figure de validation pour un neurone de Poisson à 20 Hz sur 60 s, seed 42](images/validation_rate20_seed42.png)

*Validation du neurone de Poisson homogène ($\lambda = 20$ Hz, 60 s, seed 42). En haut : raster des spikes. Au milieu : distribution empirique des ISI superposée à la densité théorique $\text{Exp}(20)$. En bas : comptage cumulé $N(t)$ contre la droite $20 \cdot t$.*

### Neurone MMPP en une brique

La validation est lancée par :

    python -m simubrain.experiments.run_mmpp_experiment --duration 60 --seed 42

Le cas de référence est le neurone à deux états repos / actif : taux $\lambda_0 = 5$ Hz et $\lambda_1 = 40$ Hz, générateur $Q = \begin{pmatrix} -0.5 & 0.5 \\ 2.0 & -2.0 \end{pmatrix}$ (taux de sortie $q_0 = 0.5$/s au repos, $q_1 = 2.0$/s en actif). La distribution stationnaire est $\pi = (0.8, 0.2)$, d'où un taux effectif $\bar\lambda = 0.8 \cdot 5 + 0.2 \cdot 40 = 12$ Hz, soit 720 spikes attendus sur 60 secondes, avec un intervalle de confiance à 95% [667, 773].

Pour ce neurone simulé sur 60 secondes (seed 42) :

- compte de spikes : 739 mesurés contre 720 attendus, dans l'intervalle de confiance à 95% [667, 773] ;
- taux empirique : 12.317 Hz contre 12 Hz attendu ($\bar\lambda$) ;
- intervalle inter-spikes moyen : 0.0808 s contre 0.0833 s attendu ($1/\bar\lambda$).

![Figure de validation pour un neurone MMPP repos / actif sur 60 s, seed 42](images/validation_mmpp_seed42.png)

*Validation du neurone MMPP (repos $\lambda_0 = 5$ Hz, actif $\lambda_1 = 40$ Hz, $\bar\lambda = 12$ Hz, 60 s, seed 42). En haut : raster des spikes, où la signature MMPP est visible à l'oeil (bandes denses à 40 Hz en état actif, zones clairsemées à 5 Hz au repos). Au milieu : distribution empirique des ISI, sans superposition exponentielle puisque la loi est une mixture. En bas : comptage cumulé $N(t)$ contre la droite $\bar\lambda \cdot t$, en escalier (plateaux au repos, montées raides en bouffée).*

### Neurone MMPP en plusieurs briques

La même validation est lancée sur l'écriture décomposée :

    python -m simubrain.experiments.run_decomposed_mmpp_experiment --duration 60 --seed 42

La `MarkovChain` publie les changements de taux, le `ModulatedPoissonNeuron` décharge au taux courant, et le `Transducer` enregistre. Le compte tombe dans le même intervalle de confiance autour de $\bar\lambda T$, ce qui valide le critère 2 pour cette écriture.

![Figure de validation pour le MMPP décomposé sur 60 s, seed 42](images/validation_decomposed_mmpp_seed42.png)

*Validation du MMPP en plusieurs briques (mêmes paramètres, 60 s, seed 42). La trajectoire diffère de celle du monolithe, ce qui est attendu : les deux écritures consomment le hasard dans un ordre différent. La figure est un contrôle visuel du câblage ; la comparaison formelle relève du test d'équivalence.*

### Équivalence entre les deux écritures du MMPP

Sur dix graines indépendantes de 120 secondes, les deux tests de Kolmogorov-Smirnov échouent à rejeter l'hypothèse nulle, tant sur les comptes de spikes par graine que sur les ISI moyens par graine. Aucune différence distributionnelle détectable entre l'écriture en une brique et l'écriture en plusieurs briques.

C'est le résultat central du projet du point de vue méthodologique : il fournit l'instrument qui permet de **tester**, et non seulement d'affirmer, qu'une décomposition est une écriture valide du même modèle.

### File de Gelenbe

La validation est lancée par :

    python -m simubrain.experiments.run_gqueue_experiment --duration 2000 --seed 42

Le cas de référence est une file à un serveur recevant deux flux de Poisson : arrivées positives à $\lambda^+ = 4$ Hz, arrivées négatives à $\lambda^- = 2$ Hz, service exponentiel à $\mu = 10$ Hz. La charge stationnaire prédite est

$$
\rho = \frac{\lambda^+}{\mu + \lambda^-} = \frac{4}{10 + 2} = 0.3333,
$$

d'où une longueur de file moyenne $\mathbb{E}[N] = \rho/(1-\rho) = 0.5$.

Pour cette file simulée sur 2000 secondes (seed 42, transitoire de 100 s écarté) :

- longueur moyenne : 0.4977 mesurée contre 0.5000 attendue, soit une erreur relative de 0.45 % ;
- 18 568 changements de longueur enregistrés, dont 17 619 dans la fenêtre de mesure.

![Figure de validation pour la file de Gelenbe, 20 premières secondes de la fenêtre de mesure, seed 42](images/validation_gqueue_seed42.png)

*Validation de la file de Gelenbe ($\lambda^+ = 4$ Hz, $\lambda^- = 2$ Hz, $\mu = 10$ Hz, 2000 s, seed 42). L'escalier montre les 20 premières secondes de la fenêtre de mesure ; les deux droites horizontales sont les moyennes calculées sur la fenêtre entière.*

## Analyse critique

L'objectif des trois étapes, valider qu'un neurone de Poisson homogène, un neurone à taux modulé par une CTMC, puis une file de Gelenbe exprimés en DEVS reproduisent la théorie, est atteint. Le critère d'équivalence est établi sur le MMPP.

Pour le Poisson, le compte observé est à environ 0.15 écart-type sous la moyenne attendue (l'écart-type vaut $\sqrt{1200} \approx 35$ spikes), donc largement à l'intérieur de l'intervalle de confiance. Les trois métriques pointent dans la même direction, légèrement sous le nominal : ce ne sont pas trois confirmations indépendantes, mais un même déficit de quelques spikes observé sous trois angles, exactement le genre de fluctuation attendue d'un tirage unique. Visuellement, la distribution des ISI épouse la densité exponentielle et le comptage cumulé suit linéairement $\lambda t$ sans dérive.

Pour le MMPP, le compte observé est à environ 0.7 écart-type au-dessus de la moyenne attendue (l'écart-type vaut $\sqrt{720} \approx 27$ spikes), bien à l'intérieur de l'intervalle de confiance. La validation porte ici sur le bon critère : le compte d'un MMPP est asymptotiquement Poisson de moyenne $\bar\lambda T$, donc c'est $\bar\lambda$ et non un taux instantané qui doit être retrouvé. C'est aussi pourquoi le panneau ISI n'a pas de superposition exponentielle : la loi des ISI d'un processus modulé est une mixture sur-dispersée, et un overlay $\text{Exp}(\bar\lambda)$ aurait été trompeur, laissant croire à un défaut là où il n'y en a pas. La moyenne des ISI coïncide bien avec $1/\bar\lambda$, mais c'est une coïncidence de moyenne, pas d'égalité de distribution. La signature en bouffées du raster et l'allure en escalier du comptage cumulé confirment visuellement la modulation.

Pour la file de Gelenbe, l'erreur de 0.45 % sur une fenêtre de 1900 secondes établit la conformité au régime stationnaire. L'allure de l'escalier est cohérente avec la loi géométrique attendue : la file est vide environ deux tiers du temps, monte souvent à 1, rarement à 2, et les pointes à 3 ou 4 sont exceptionnelles, ce qui correspond à $P(N = n) = (1-\rho)\rho^n$ pour $\rho = 1/3$. La moyenne de 0.5 devient visuellement plausible pour un signal qui passe la majorité de son temps à zéro.

Le point conceptuel de cette famille est la place du $\lambda^-$ dans la formule. Un client négatif ne porte aucun travail, il en **détruit**. Il agit donc comme un second canal de départ, ce qui l'inscrit au **dénominateur** de $\rho$, jamais au numérateur. Confondre les deux donnerait $\rho = (\lambda^+ - \lambda^-)/\mu = 0.2$ et une prédiction $\mathbb{E}[N] = 0.25$, deux fois trop basse : l'écart avec l'observé aurait été de 100 %, pas de 0.45 %. La validation discrimine donc effectivement entre les deux lectures.

Le découplage entre la couche de modèles et la couche d'analyse a été mis à l'épreuve à chaque famille, et il a tenu. Pour le MMPP, les panneaux raster et comptage cumulé ont été réutilisés tels quels, et seul le panneau ISI a dû être spécialisé, ce qui a mis au jour une fuite d'abstraction (le docstring promettait à tort la réutilisation intégrale pour toute source future) ; la correction a consisté à dire la vérité sur ce qui se réutilise, plutôt qu'à forcer une réutilisation incorrecte. Pour la file de Gelenbe, l'épreuve était plus sévère puisque la grandeur validée change de nature, et la couche a été étendue par ajout d'une fonction agnostique plutôt que par contamination : `time_average` ignore ce qu'est une file, et le `Transducer` n'a pas bougé d'une ligne pour enregistrer des longueurs plutôt que des spikes.

La réutilisation du `PoissonNeuron` comme source des deux flux de la file de Gelenbe est le second résultat architectural de cette famille. Le signe n'est pas porté par la charge utile mais par le **port d'arrivée** : les deux sources sont des neurones de Poisson ordinaires, qui ignorent tout des G-networks, et c'est le modèle couplé qui décide du sens en câblant l'un sur `positive_in` et l'autre sur `negative_in`. Le routage est ainsi la responsabilité de l'assemblage, pas de la source, ce qui est exactement le découpage que le principe de responsabilité unique demande. Cette réutilisation n'aurait pas été possible si le `PoissonNeuron` avait été typé pour les spikes.

Les résultats portent toutefois sur une seule exécution par modèle pour les critères 1 et 2 : ils établissent la correction sur un cas, pas la robustesse statistique. Seul le critère 3 agrège plusieurs graines.

## Limites du projet

- **Portée des modèles** : les trois familles sont validées. Le cas limite sans signal négatif (M/M/1) reste à simuler comme contrôle croisé, et le méta-formalisme unifié n'est pas du ressort de ce projet : il est conçu par l'équipe, ce projet lui fournit l'infrastructure et les critères.
- **Profondeur statistique** : la conformité à la théorie repose sur une exécution unique par configuration. Une campagne agrégeant plusieurs graines permettrait d'estimer la dispersion réelle plutôt que de constater un seul tirage. L'approximation normale de l'intervalle de confiance n'est fiable que pour de grands comptes attendus ; aux faibles taux ou courtes durées, des quantiles de Poisson exacts seraient préférables.
- **Tolérance du test d'intégration de la file** : le test automatisé compare la longueur moyenne à $\mathbb{E}[N]$ avec une tolérance relative de 15 % sur une graine unique. C'est une bande pragmatique, pas un test statistique ; la version rigoureuse agrégerait plusieurs graines et bâtirait un intervalle de confiance, comme le fait le test d'équivalence. Dette assumée, à résorber si la mesure s'avère instable.
- **Portée du critère d'équivalence** : le test porte sur le compte et l'ISI moyen agrégés par graine, pas sur la forme complète de la loi des ISI. Cette affirmation plus fine n'est étayée que visuellement.
- **Validation de la file par la moyenne seule** : la conformité porte sur $\mathbb{E}[N]$, un seul moment. Comparer l'histogramme pondéré par le temps à la loi géométrique $(1-\rho)\rho^n$ porterait sur la distribution entière et serait un critère strictement plus fort.
- **Portée des tests automatisés** : pytest garantit la correction structurelle et déterministe, mais l'exactitude statistique dépend d'une inspection manuelle des sorties d'expérience.
- **Dépendances externes** : la simulation repose sur PyPDEVS, et la reproductibilité dépend du générateur `default_rng` de numpy, dérivé via `RandomStream`.