# gravitation-newton

Simulation du système solaire en une seule page HTML, construite **uniquement** sur la loi de la
gravitation universelle de Newton. Aucune orbite n'est pré-calculée, aucune ellipse n'est tracée
d'avance : chaque corps n'est qu'une masse, une position et une vitesse, et les trajectoires
émergent de l'intégration des forces, image par image.

👉 **[Voir la simulation en ligne](https://brunofl.github.io/gravitation-newton/)** — ou ouvrir
[`index.html`](index.html) dans un navigateur. Aucune dépendance, aucun serveur, aucun build.

---

## Le modèle physique

### 1. La seule loi utilisée

À chaque pas de temps, l'accélération de chaque corps est la somme des contributions de tous
les autres :

$$\vec{a}_i = \sum_{j \neq i} \frac{G\,m_j}{\|\vec{r}_{ij}\|^2}\,\hat{r}_{ij}$$

Avec 9 corps, cela fait **36 paires** évaluées à chaque pas, en appliquant la troisième loi de
Newton (une paire calculée une fois, appliquée avec des signes opposés). Conséquence importante :
les planètes ne tournent pas autour d'un Soleil fixe, elles **s'attirent aussi entre elles**.
Jupiter perturbe réellement Saturne, et le Soleil oscille autour du barycentre du système.

### 2. Intégration : leapfrog (Verlet des vitesses)

Un intégrateur d'Euler naïf ferait spiraler les planètes vers l'extérieur en quelques dizaines
d'orbites, en injectant de l'énergie à chaque pas. Le schéma retenu est symplectique :

```
v ← v + ½·a·dt        (demi-pas de vitesse)
x ← x + v·dt          (pas de position)
a ← forces(x)         (recalcul complet des 36 paires)
v ← v + ½·a·dt        (second demi-pas)
```

L'énergie totale ne dérive alors quasiment pas, même sur des milliers d'orbites.

### 3. Unités et conditions initiales

Tout est en **SI réel** : mètres, kilogrammes, secondes. `G = 6.67430e-11`, `1 UA = 1.495978707e11 m`.

| Corps | Masse (kg) | Demi-grand axe (m) | Excentricité |
|---|---|---|---|
| Soleil | 1.98892e30 | — | — |
| Mercure | 3.3011e23 | 5.7909e10 | 0.2056 |
| Vénus | 4.8675e24 | 1.08210e11 | 0.0068 |
| Terre | 5.97237e24 | 1.49598e11 | 0.0167 |
| Mars | 6.4171e23 | 2.27956e11 | 0.0934 |
| Jupiter | 1.89819e27 | 7.78479e11 | 0.0489 |
| Saturne | 5.68340e26 | 1.43353e12 | 0.0565 |
| Uranus | 8.68100e25 | 2.87246e12 | 0.0457 |
| Neptune | 1.02413e26 | 4.49506e12 | 0.0113 |

Chaque planète démarre à son périhélie, avec la vitesse donnée par l'équation de vis-viva :

$$v_p = \sqrt{\frac{G M_\odot (1+e)}{a(1-e)}}$$

C'est la **seule** information orbitale fournie : une position et une vitesse initiales.
L'excentricité réelle des orbites en découle naturellement, elle n'est jamais imposée au tracé.

La quantité de mouvement totale est annulée au démarrage (`Σ mᵢvᵢ = 0`), ce qui fixe le barycentre
du système à l'écran plutôt que de laisser l'ensemble dériver.

Les orbites sont traitées dans un plan unique (2D) — les inclinaisons réelles, faibles pour les
huit planètes, sont ignorées. Les longitudes de départ sont approximatives : le but est la
dynamique, pas l'éphéméride.

---

## Vérification numérique

Les périodes affichées dans le panneau de droite ne sont pas des constantes codées en dur : elles
sont **mesurées** par la simulation, en chronométrant l'intervalle entre deux passages au périhélie
(minimum local de la distance au Soleil).

Sur 2 ans simulés avec `dt = 900 s` (70 128 pas) :

| | Mercure | Vénus | Terre |
|---|---|---|---|
| Période mesurée | 0,2408 an | 0,6146 an | 0,9997 an |
| Période réelle | 0,2408 an | 0,6152 an | 1,0000 an |

- **Dérive de l'énergie totale : 2,8 × 10⁻⁸ %.** L'intégrateur est stable.
- La 3ᵉ loi de Kepler (`T² ∝ a³`) n'est jamais utilisée dans le code — elle est *retrouvée*.
- Le Soleil s'est déplacé d'environ 790 000 km sur ces 2 ans, tiré par Jupiter : le mouvement
  du barycentre est bien reproduit.

---

## Interface

| Contrôle | Effet |
|---|---|
| Molette | Zoom centré sur le curseur |
| Glisser | Déplacer la vue |
| Clic sur un corps | Suivre ce corps (recentrage continu) |
| Espace | Pause / reprise |
| Vitesse | Nombre de pas par frame (durée simulée par image) |
| Pas `dt` | Pas d'intégration, de 60 s à 7200 s |
| Traces | Affiche les positions réellement calculées |
| Vecteurs | Vitesse (vert) et accélération (rouge) de chaque planète |
| Système interne / Complet | Cadrage sur les planètes telluriques ou sur les 8 planètes |

Le panneau de droite donne en direct, pour chaque planète : distance au Soleil (UA), vitesse
relative au Soleil (km/s), période mesurée (années), ainsi que le temps écoulé, l'énergie totale
et sa dérive depuis l'instant initial.

### À essayer

Monter `dt` à 7200 s et observer Mercure : sa précession devient visiblement excessive et la dérive
d'énergie grimpe de plusieurs ordres de grandeur. C'est l'erreur de discrétisation qui devient
comparable à la dynamique du corps le plus rapide — la même contrainte que celle qui gouverne le
choix du pas dans les vraies intégrations d'éphémérides.

---

## Structure du code

Tout tient dans [`index.html`](index.html) (403 lignes, sans dépendance) :

| Fonction | Rôle |
|---|---|
| `DATA` | Les données physiques du tableau ci-dessus |
| `initBodies()` | Positions/vitesses initiales, annulation de la quantité de mouvement |
| `computeAccelerations()` | La loi de Newton sur les 36 paires |
| `step(dt)` | Un pas de leapfrog |
| `detectPeriods()` | Chronométrage des passages au périhélie |
| `totalEnergy()` | Énergie cinétique + potentielle, pour le contrôle de dérive |
| `draw()` | Rendu canvas : corps, traces, halo solaire, échelle |
| `loop()` | `requestAnimationFrame` : N pas de physique, puis un rendu |

Physique et rendu sont découplés : le pas d'intégration ne dépend pas du framerate, ce qui rend la
simulation reproductible quelle que soit la machine.

---

## Licence

Voir [LICENSE](LICENSE).
