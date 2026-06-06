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

*Dernière mise à jour : fin de la semaine 5 (début juin 2026).*

Cette page présente la stratégie de test, les résultats obtenus sur le neurone de Poisson homogène, leur interprétation et les limites actuelles.

## Méthodes de validation

La validation opère à deux niveaux que le projet garde volontairement séparés : les tests automatisés vérifient ce qui est déterministe, et l'expérience valide ce qui est statistique.

La suite pytest (19 tests) couvre les invariants structurels et déterministes. Côté structure : le câblage des sous-modèles, la garde `rate > 0` qui lève une exception, le `timeAdvance` infini du Transducer, et l'enregistrement chronologique des couples `(temps, payload)` dans l'horizon de simulation. Côté déterminisme : la reproductibilité par graine (deux modèles de même graine produisent exactement la même séquence de tirages), la positivité de `timeAdvance`, et l'exactitude des fonctions de calcul (`compute_isi`, `theoretical_stats`).

Aucun de ces tests ne vérifie que le taux empirique tombe près de $\lambda$. C'est le principe directeur de la stratégie : une assertion portant sur une grandeur aléatoire serait soit instable, soit trivialement vraie. Tester le déterminisme et valider le hasard sont deux activités distinctes, et les mélanger affaiblirait les deux.

La validation statistique est donc confiée à l'expérience `run_basic_experiment`, qui compare un train simulé aux prédictions du modèle de Poisson homogène : le compte de spikes contre $\lambda \cdot T$, l'intervalle inter-spikes moyen contre $1/\lambda$, et le taux empirique contre $\lambda$. L'intervalle de confiance à 95% sur le compte vient de l'approximation normale $\lambda T \pm 1.96\sqrt{\lambda T}$. Une figure à trois panneaux complète la comparaison numérique par une inspection visuelle.

## Résultats obtenus

La suite pytest passe entièrement au vert (19 tests), confirmant les invariants structurels et déterministes des modèles et de la couche d'analyse.

La validation statistique est lancée par :

    python -m simubrain.experiments.run_basic_experiment --rate 20 --duration 60 --seed 42

Pour ce neurone à $\lambda = 20$ Hz simulé sur 60 secondes (seed 42) :

- compte de spikes : 1190 mesurés contre 1200 attendus, dans l'intervalle de confiance à 95% [1132, 1268] ;
- intervalle inter-spikes moyen : 0.0503 s contre 0.0500 s attendu ;
- taux empirique : 19.833 Hz contre 20 Hz attendu.

![Figure de validation pour un neurone de Poisson à 20 Hz sur 60 s, seed 42](images/validation_rate20_seed42.png)

*Validation du neurone de Poisson homogène ($\lambda = 20$ Hz, 60 s, seed 42). En haut : raster des spikes. Au milieu : distribution empirique des ISI superposée à la densité théorique $\text{Exp}(20)$. En bas : comptage cumulé $N(t)$ contre la droite $20 \cdot t$.*

## Analyse critique

L'objectif de cette étape, valider qu'un neurone de Poisson homogène exprimé en DEVS reproduit la théorie, est atteint. Le compte observé est à environ 0.3 écart-type sous la moyenne attendue (l'écart-type vaut $\sqrt{1200} \approx 35$ spikes), donc largement à l'intérieur de l'intervalle de confiance. Les trois métriques pointent d'ailleurs dans la même direction, légèrement sous le nominal : ce ne sont pas trois confirmations indépendantes, mais un même déficit de quelques spikes observé sous trois angles, ce qui est exactement le genre de fluctuation attendue d'un tirage unique.

Visuellement, la distribution des ISI épouse la densité exponentielle et le comptage cumulé suit linéairement $\lambda t$ sans dérive, deux signatures d'un processus de Poisson homogène. Le découplage entre la couche de modèles et la couche d'analyse paie ici : la même validation s'appliquera aux sources stochastiques futures (MMPP, G-networks) sans modification.

Le résultat porte toutefois sur une seule exécution : il établit la correction du modèle sur un cas, pas sa robustesse statistique. Un écart systématique faible resterait indétectable à ce stade.

## Limites du projet

- Portée du modèle : seul le neurone de Poisson homogène est validé. Les extensions MMPP et G-networks, ainsi que le méta-formalisme unifié, ne sont pas encore implémentées ni évaluées.
- Profondeur statistique : la validation repose sur une exécution unique par configuration. Une campagne agrégeant plusieurs graines permettrait d'estimer la dispersion réelle du compte plutôt que de constater un seul tirage. De plus, l'approximation normale de l'intervalle de confiance n'est fiable que pour de grands comptes attendus ; aux faibles taux ou courtes durées, des quantiles de Poisson exacts seraient préférables.
- Portée des tests automatisés : pytest garantit la correction structurelle et déterministe, mais l'exactitude statistique dépend d'une inspection manuelle des sorties d'expérience.
- Dépendances externes : la simulation repose sur PyPDEVS, et la reproductibilité dépend du générateur `default_rng` de numpy, appelé à évoluer avec le refactor RNG.