# Changelog

## V53.2 — La partie survit désormais à un rafraîchissement de page
- **Partie locale (`game.js`)** : rafraîchir la page (F5) remettait systématiquement la
  partie à zéro — l'état n'existait qu'en mémoire JavaScript. La partie en cours est
  désormais sauvegardée automatiquement dans `localStorage` après chaque coup et
  restaurée telle quelle au rechargement de la page (position des pions, historique,
  chronomètre, tour en cours — y compris relance automatique de l'IA si c'était son
  tour). Quitter explicitement la partie (bouton « Quitter »/« Nouvelle partie ») efface
  la sauvegarde, pour repartir proprement sur une configuration neuve.
- **Multijoueur en ligne (`online-v39.js`)** : le mécanisme de restauration de session
  (`restoreOnlineSessionIfPossible`, déjà présent dans le code) a été vérifié par un test
  automatisé — il reconnecte bien automatiquement au salon en cours après un
  rafraîchissement, sans ressaisir de code. La partie elle-même n'était donc pas perdue
  en ligne (elle vit dans Firebase, pas dans le navigateur) ; seule la partie locale
  manquait de cette persistance.
- Deux nouveaux tests (`tests/local-persistence-test.js`,
  `tests/online-persistence-test.js`) simulent un vrai rafraîchissement de page (deux
  instances de page séparées partageant le même stockage local / la même base
  Firebase simulée) pour vérifier ce comportement de bout en bout, plutôt que de
  supposer qu'il fonctionne.

## V53.1 — Forfait en multijoueur en ligne
Nouvelle fonctionnalité (n'existait pas auparavant) : quand un joueur quitte une partie
en ligne en cours, cela n'avait aucun effet pour les autres participants — ils ne
voyaient rien, le tour de ce joueur n'était jamais sauté, et le jeu attendait
indéfiniment un coup qui ne viendrait jamais.

- **`forfeitPlayer(uid)`** (`online-v39.js`) : déclare un joueur forfait quand il quitte
  une partie en cours (`leaveRoom`). Son tour est immédiatement sauté si c'était le sien,
  et il est exclu du jeu comme un joueur déjà classé (`isRanked()` étendu pour inclure
  les forfaits — réutilise directement `remainingPlayers()`/`nextActiveUidAfter()`,
  déjà conçus pour ignorer les joueurs classés).
- **Pions grisés** : les pions d'un joueur forfait apparaissent désormais grisés et non
  cliquables sur le plateau (classe `finished-piece`, déjà utilisée en mode local pour
  les joueurs classés — réutilisée pour rester cohérent visuellement).
- **Fin de partie par attrition** : si un ou plusieurs forfaits successifs ne laissent
  plus qu'un seul joueur actif, la partie se termine et ce joueur est déclaré vainqueur
  (`lastRemaining`), exactement comme le prévoyaient déjà les règles du jeu.
- **Classement** : les joueurs forfaits apparaissent toujours en dernière position de la
  liste de classement, distinctement marqués « forfait », quel que soit le moment où ils
  ont quitté la partie (jamais mélangés avec les rangs réels).
- **Règles Firebase assouplies** (`firebase-rules-v37.json`) : la règle d'écriture de
  l'état n'autorisait que le joueur actif ou l'hôte, ce qui aurait bloqué silencieusement
  la déclaration de forfait dans le cas le plus courant (quitter quand ce n'est pas son
  tour). Élargie à tout joueur du salon.
  **⚠️ Action requise : ces règles doivent être recopiées manuellement dans la console
  Firebase (Realtime Database → Règles) du projet — je ne peux pas les déployer à
  distance.**
- Corrigé au passage : `online-v39.html` référençait encore `experience-v44.js`,
  supprimé lors du nettoyage V53 (provoquait un 404 au chargement de la page
  multijoueur ; sans conséquence fonctionnelle car le code gère déjà son absence, mais
  inutile de le charger).
- Nouveau test `tests/online-forfeit-test.js` : vérifie réellement (pas seulement en
  relisant le code) le saut de tour automatique et la fin de partie par attrition sur
  plusieurs forfaits successifs.

## V53 — Édition simplifiée
Nouvelle édition centrée sur les trois modes essentiels : local humains, local
humain(s)/IA, et multijoueur en ligne. Objectif : réduire la surface du projet pour
limiter le risque de régression (voir l'historique ci-dessous, où plusieurs bugs
bloquants sont passés inaperçus d'une version à l'autre à cause de la duplication de
code entre versions parallèles).

**Retiré de cette édition** (fichiers non inclus, restent disponibles dans les archives
précédentes si besoin) :
- Système de mémoire/apprentissage (`experience-v44/45/48`, `experience-v50.js`) et
  Learning Lab (`learning-v41`) — complexité non requise pour les trois modes essentiels.
- Laboratoire multijoueur expérimental non lié depuis la page principale (`online-lab`).
- Anciennes versions multijoueur redondantes (`online-v37`, `online-v38` — simples
  redirections vers `online-v39`, qui reste la version active).
- Fichier `ai-worker.js` autonome, qui n'était jamais chargé par le jeu (le moteur réel
  vit entièrement dans la chaîne `AI_WORKER_CODE` embarquée dans `game.js`) — sa
  présence dans les zips précédents pouvait laisser croire à tort qu'il fallait le
  maintenir en parallèle du code réellement utilisé.
- `game.orig.js` (fichier de comparaison utilisé pendant le développement du Path
  Planner, sans utilité en production).

**Ajouté :**
- `tests/integration-test.js` — nouveau test qui fait *réellement tourner* une partie
  (démarrage, coup côté page, exécution directe du Worker IA réel) dans un DOM simulé
  (jsdom), au lieu de simplement vérifier la présence de texte dans le code. C'est le
  test qui, s'il avait existé plus tôt, aurait attrapé dès la première exécution les bugs
  `INITIAL_POSITIONS` et `AI_OPENING_DISTINCT_PIECES` décrits ci-dessous dans
  l'historique V51.1/V51.2.
- `package.json` (jsdom en dépendance de développement, uniquement pour les tests —
  aucune dépendance n'est chargée par le jeu lui-même, qui reste 100 % statique).

**Inchangé (déjà en place depuis V51.2, vérifié à nouveau ici) :** Victory Planner à
paliers (5/7/9 demi-coups en duel, 3/4/5 en multijoueur), Path Planner, protection XSS
(`escapeHtml`), protection anti-injection de formule CSV, correctif `roomCode` du
multijoueur en ligne.

## V51.2 — Correctif critique : la partie ne démarrait plus
- **Bug bloquant (déjà présent dans le zip v51 initial, avant même mes correctifs v51.1) :** les déclarations `const INITIAL_POSITIONS = {...}` et `const SYMMETRIC_MOVES = parseMoveTable(MOVES_TEXT);` (thread principal, utilisées par `createPieces()` et par la validation des coups côté page) avaient été supprimées par erreur — probablement lors d'un remplacement automatique trop large pendant un patch v50/v51 antérieur. Résultat : cliquer sur « Lancer la partie » levait `ReferenceError: Can't find variable: INITIAL_POSITIONS` et bloquait toute partie, en local comme en ligne. Restauré à l'identique de la v48/v50 (clés `YELLOW/RED/BLUE/BLACK`, cohérentes avec `COLOR_KEYS`). Vérifié par exécution réelle de `createPieces()` (28 pièces générées, positions correctes) et par `parseMoveTable(MOVES_TEXT)` (table de déplacements valide), pas seulement par relecture du code.
- Cette régression n'affectait pas le Worker IA (qui a sa propre copie interne de `SYMMETRIC_MOVES`, reçue via `postMessage`), ce qui explique qu'elle soit passée inaperçue pendant la revue du moteur de recherche en v51.1 : le jeu ne pouvait plus du tout démarrer, donc l'IA n'avait simplement jamais l'occasion de jouer.

## V51.1 — Corrections d'efficacité de l'IA
- **Bug critique corrigé (préexistant, hérité de la v50, indépendant du patch Path Planner) :** dans le Worker IA, `openingMovePool` référençait la constante globale `AI_OPENING_DISTINCT_PIECES`, absente de la portée isolée du Worker. À **chaque tour normal** (sans victoire/blocage immédiat à jouer), le Worker levait une `ReferenceError` silencieusement rattrapée, et le jeu retombait systématiquement sur l'IA de repli à 1 coup au lieu du Victory Planner/Path Planner. Corrigé (lecture de `CONFIG.openingDistinctPieces`), et vérifié par exécution réelle du moteur dans un bac à sable Node (pas seulement une vérification syntaxique).
- **Bug de double application de coup corrigé dans `tacticalMovePool` :** pour chaque coup candidat, `staticMoveScore` était appelé alors que ce même coup était déjà temporairement appliqué sur le plateau, ce qui provoquait un second déplacement vers la même case (« double bascule » de la face RECTO/VERSO) et faussait l'évaluation de position pour **chaque candidat, à chaque nœud de recherche**. `staticMoveScore` est désormais appelé avant toute application temporaire.
- **Complexité quadratique supprimée :** `tacticalMovePool` recalculait `countImmediateWinningMoves` (coûteux, lui-même en O(n)) pour chaque coup candidat — un coût O(n²) redondant avec la détection des coups gagnants déjà garantie par ailleurs. Supprimé. Le motif gagnant « avant » était également recalculé identiquement pour chaque candidat ; il est désormais calculé une seule fois. Le Path Planner et le Victory Planner disposent ainsi de plus de marge de calcul pour la même profondeur, à budget de temps égal.
- **Régression du budget de réflexion corrigée :** `AI_MAX_TURN_MS`/`AI_WORKER_DEADLINE_MS` étaient retombés à 7000/6500 ms (au lieu des ~10 s prévus), avec le garde-fou principal qui coupait 200 ms **avant** même le délai interne du Worker. Portés à 10000/10500 ms, avec une marge de sécurité correcte.
- **Config du Worker complétée :** les profondeurs/limites tactiques et critiques (`tacticalDepthTwoPlayers`, `criticalCandidateLimitMulti`, etc.) n'étaient jamais transmises par le thread principal — le Worker fonctionnait uniquement grâce à des valeurs de repli codées en dur. Elles sont désormais explicitement envoyées depuis `game.js`.
- **Régression de sécurité corrigée (hors périmètre IA) :** la protection contre l'injection de formule CSV (`csvCell`, ajoutée en v48.2) avait disparu dans cette branche ; rétablie.
- Correctifs déjà présents dans cette branche et vérifiés intacts : XSS (`escapeHtml`), bug `currentRoomCode` du multijoueur.
- Aucune régression : `tests/smoke-test.js` et `tests/v51-path-planner-test.js` passent ; le moteur a été exécuté réellement (pas seulement vérifié syntaxiquement) sur un plateau synthétique multi-pièces pour confirmer que `openingMovePool` et `tacticalMovePool` s'exécutent sans erreur.

## V51 — Path Planner stratégique
- navigation Dijkstra sur les déplacements légaux ;
- recherche de la meilleure porte d’entrée du groupe central ;
- détection et coût des barrières adverses ;
- mémoire locale des itinéraires échoués et des retours ;
- score de contournement fondé sur le coût réel, les alternatives et la progression ;
- Victory Planner et mémoire hybride V50 conservés.

# Changelog

## v51.0.0

- Ajout de `VR50Memory` et migration automatique de la mémoire v48.
- La mémoire ne choisit plus directement le coup avant le Victory Planner.
- Transmission des statistiques de la position courante au Web Worker IA.
- Intégration des résultats appris dans l'ordre des coups et le départage des scores.
- Pondération prudente par moyenne, variance, nombre d'essais et confiance.
- Influence mémoire plafonnée pour empêcher un ancien mauvais apprentissage de masquer une victoire ou une défense forcée.
- Ajout du compteur « Coups fiables (3+ essais) ».
- Préparation des profils IA stable/candidate et de la promotion Champion.
- Conservation des correctifs multijoueur de la v48.3 et du Victory Planner v49.
