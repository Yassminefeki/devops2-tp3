# TP3 — Réponses

**Binôme :** NOM Prénom — NOM Prénom

> Répondez sous chaque question. Les mesures (durées, pourcentages) sont celles de **votre** machine.

## Partie 1 — Pyramide des tests

**Q1.**
- `tests/unit` : niveau unitaire. Outil : Vitest. La stack Docker n'est pas nécessaire pour les exécuter ; ils testent des fonctions pures sans réseau ni navigateur. Le service `unit` est simplement lancé dans un conteneur pour standardiser l'environnement.
- `tests/component` : niveau de composant. Outil : Vitest + Testing Library + jsdom. Pas besoin de la stack applicative, seulement du navigateur simulé.
- `tests/integration` : niveau d'intégration. Outil : Vitest, exécuté en environnement Node. Oui, il faut la stack `strapi` + `postgres` active, car le test appelle l'API réelle de Strapi.
- `tests/e2e` : niveau de bout en bout. Outil : Playwright. Oui, il faut la stack complète (`frontend`, `strapi`, `postgres`) pour lancer le site réel dans un vrai navigateur.

**Q2.** La pyramide est large à la base et étroite au sommet pour deux raisons principales :
- coût et vitesse : les tests unitaires sont les moins chers à lancer et les plus rapides à répéter ; ils peuvent être exécutés à chaque commit ;
- stabilité et diagnostic : ils vérifient des petits morceaux isolés, donc l'échec est plus facile à localiser. À l'inverse, les tests E2E sont plus coûteux, plus lents et plus sensibles à l'environnement, donc on n'en met qu'un petit nombre, sur les scénarios critiques.

## Partie 2 — Tests unitaires

**Q3.**
Avant la correction, le test unitaire qui échoue est celui du format d'une heure pleine : `formatDuration(60)`.
Le message de Vitest est du type :

```text
Expected: "1 h"
Received: "1 h 0"
```

On exécute 11 tests dans `tests/unit/cours.test.ts` au total ; 1 échec. `Expected` = valeur attendue par le test, définie par l'assertion ; `Received` = valeur réellement renvoyée par la fonction testée.

**Q4.** Le bug venait de `formatDuration` : quand `hours > 0` et `minutes == 0`, la fonction ajoutait encore les minutes (la variable `rest` valait 0) et produisait `"1 h 0"` au lieu de `"1 h"`. La correction ajoute une condition explicite :

```ts
return rest === 0 ? `${hours} h` : `${hours} h ${rest}`;
```

Ainsi, si le reste est nul, on ne concatène pas `0` à l'affichage.

**Q5.** Avant l'ajout des deux tests, la couverture de `lib/cours.ts` est :
- lignes : 100 %
- branches : 100 %
- fonctions : 85,71 %
- instructions : 94,73 %

D'après le rapport HTML, il n'y avait pas de ligne ou de branche non couverte sur le fichier `cours.ts`, mais il restait une fonction anonyme ou un chemin interne non exercé dans `filterCours` / `normalize`, ce qui explique le pourcentage inférieur des fonctions. En pratique, le fichier est presque entièrement couvert, mais pas totalement au niveau des fonctions.

**Q6.** Après les deux tests ajoutés, la couverture de `lib/cours.ts` reste pratiquement identique sur le rapport :
- lignes : 100 %
- branches : 100 %
- fonctions : 85,71 %
- instructions : 94,73 %

Les tests ajoutés couvrent bien le cas `filterCours([], ...)` (liste vide) et le cas combinant `query` + `level` (`"git"` + `"debutant"`). Ils touchent un chemin qu'on ne testait pas avant, surtout la clause `needle !== ""` avec un filtre de niveau actif. Il reste néanmoins un chemin de fonction non couvert, notamment une fonction anonyme ou un callback interne, ce qui explique que le taux de fonctions ne passe pas à 100 %.

**Q7.** La couverture de lignes mesure le nombre de lignes exécutées ; la couverture de branches mesure si les différentes alternatives d'un `if`, d'un ternaire ou d'un `&&` ont bien été parcourues. Un code à 100 % sur les lignes n'est pas forcément sans bug : il peut avoir des chemins logiques non testés, des cas limites, ou des mauvais assertions. Dans ce TP, le bug `formatDuration(60)` est un bon exemple : la ligne de retour était exécutée, mais le branchement `rest === 0` n'avait pas été testé, donc la mauvaise sortie `"1 h 0"` passait inaperçue.

## Partie 3 — Tests de composant

**Q8.** `CoursCard.test.tsx` vérifie le rendu réel du composant React : le titre affiché comme lien vers `/cours/...`, le niveau, la durée formatée, et la liste des mots-clés. Un test unitaire de `formatDuration` ne vérifie que la fonction pure, sans rendre de JSX ni simuler l'interface. Le navigateur est simulé par `jsdom`, configuré dans `frontend/vitest.config.mts` avec `test.environment: "jsdom"` et le fichier `tests/setup.ts`.

**Q9.** On cherche par rôle ou par texte parce que c'est la manière la plus proche du comportement utilisateur : on teste ce qu'un vrai visiteur voit et clique, sans dépendre de classes CSS ou de la structure DOM interne. Cela rend les tests plus robustes et plus lisibles ; un changement de style ou de structure ne cassera pas le test si l'accessibilité reste bonne.

**Q10.** Il n'y a pas de test de composant pour `app/page.tsx` parce que c'est une page serveur qui compose plusieurs éléments, récupère les données via `getCours()` et intègre la liste de cours. Ce niveau de test est plutôt couvert par les tests d'intégration (API Strapi) et les tests E2E (parcours complet du navigateur).

## Partie 4 — Tests d'intégration

**Q11.** Les tests d'intégration vérifient que l'API Strapi répond bien avec les 6 cours du seed, avec les bons champs (`documentId`, `title`, `session`, `duration`, `level`, `topics`), dans le bon ordre (`[1,2,3,4,5,6]`), et que la récupération d'une fiche via `getCoursById()` fonctionne. Ce qui n'est vérifié par aucun test unitaire ou de composant est donc le contrat réel avec le backend.

**Q12.** Si Strapi est arrêté, `docker compose run --rm --no-deps integration` échoue avec une erreur réseau de type `fetch failed`, `ECONNREFUSED` ou `Strapi a répondu...` selon le moment exact. Les tests unitaires passent toujours parce qu'ils n'appellent pas l'API et qu'ils testent uniquement des fonctions pures. `--no-deps` sert à empêcher Docker Compose de relancer automatiquement les services dépendants de `integration` (ici Strapi) avant l'exécution des tests ; on veut tester l'état réel sans démarrage implicite.

**Q13.** Un test d'intégration en échec n'indique pas forcément un bug dans le code applicatif. Deux autres causes possibles sont :
- le backend n'est pas démarré ou pas encore prêt (`strapi`/`postgres` non healthy) ;
- une erreur de configuration ou de réseau (`STRAPI_URL`, variables d'environnement, port, DNS interne Docker, base de données inaccessible).

## Partie 5 — Tests de bout en bout

**Q14.** Le scénario qui échoue est : "la fiche du cours d'introduction affiche sa durée". La capture d'écran et la trace montrent que le texte de la durée est affiché comme `1 h 0` au lieu de `1 h`, donc la page est correctement rendue mais avec le mauvais format. C'est le même bug que Q3 : le test unitaire a détecté le mauvais calcul dans `formatDuration`, puis le test E2E le reproduit à travers le parcours utilisateur. Il échoue même après la correction de la partie 2 parce que le frontend Docker a été construit avant la correction et contenait encore le code buggué ; il faut reconstruire l'image (`docker compose up -d --build frontend`) pour l'appliquer.

**Q15.** Le test unitaire permet de trouver la cause le plus vite, car il isole directement la fonction concernée et donne un message clair (`Expected`/`Received`). Le test E2E prouve que l'utilisateur est touché, puisqu'il reproduit le comportement visible dans le navigateur sur la vraie page.

**Q16.**

| Niveau | Nombre de tests | Durée d'exécution | Besoin de la stack ? |
|---|---:|---:|---|
| Unitaire + composant | 19 | 1,71 s | Non |
| Intégration | 4 | 457 ms | Oui, Strapi + Postgres |
| E2E | 4 | 3,1 s | Oui, frontend + Strapi + Postgres |

**Q17.** Un test flaky est un test instable : il passe ou échoue sans qu'il y ait forcément un vrai changement fonctionnel. Playwright réduit ce risque avec :
- l'attente automatique des éléments/conditions (auto-wait),
- les assertions réessayées (`toHaveCount`, `toBeVisible`, `toHaveURL`, etc.),
- la capture de trace/screenshot en cas d'échec pour diagnostiquer les ralentissements, la présence d'éléments, ou les états transitoires.

## Partie 6 — SonarQube

**Q18.**

```properties
# Identifiant unique du projet dans SonarQube
sonar.projectKey=tp3-cours
sonar.projectName=TP3 - Cours DevOps

# Code applicatif à analyser
sonar.sources=app,components,lib

# Code des tests
sonar.tests=tests

# Rapport de couverture Vitest (format lcov)
sonar.javascript.lcov.reportPaths=coverage/lcov.info

# Fichiers ignorés
sonar.exclusions=node_modules/**,.next/**,coverage/**,playwright-report/**,test-results/**

# Fichiers exclus du calcul de couverture
sonar.coverage.exclusions=app/**,lib/api.ts,components/CoursList.tsx,components/DevOpsLoop.tsx,*.config.*

sonar.sourceEncoding=UTF-8

# Attendre le verdict du Quality Gate
sonar.qualitygate.wait=true
```

Explications :
- `sonar.projectKey` : identifiant unique du projet dans SonarQube ;
- `sonar.projectName` : libellé visible dans l'interface ;
- `sonar.sources` : dossiers sources à analyser ;
- `sonar.tests` : dossier des tests pour les distinguer du code applicatif ;
- `sonar.javascript.lcov.reportPaths` : chemin du rapport de couverture généré par Vitest ;
- `sonar.exclusions` : on ignore les dossiers lourds ou non utiles à l'analyse (`node_modules`, build Next, coverage, résultats Playwright) ;
- `sonar.coverage.exclusions` : on exclut `app/**` et `lib/api.ts` car les pages Next ne sont pas ciblées par la couverture unitaire et `api.ts` est testé en intégration, pas en unité ; `components/CoursList.tsx` et `components/DevOpsLoop.tsx` sont couverts par E2E, pas par Vitest.

**Q19.** Avec le Quality Gate TP3, le statut au moment rouge est `FAILED` : le gate échoue parce que `Coverage < 90` et `Vulnerabilities > 0`. Le problème majeur est la vulnérabilité “hard-coded password” dans `lib/api.ts`. Après correction, le gate passe au vert. En final, le rapport Sonar montre :
- bugs : 0
- vulnérabilités : 0
- security hotspots : 0
- code smells : 9
- couverture : 98,0 %
- duplication : 5,6 %

**Q20.**
- Bug : défaut fonctionnel réel. Exemple ici : `formatDuration(60)` renvoyait `"1 h 0"` ; c'est un mauvais comportement du code.
- Vulnerability : faille de sécurité exploitable. Exemple ici : mot de passe codé en dur dans `lib/api.ts` (`typescript:S2068`).
- Security hotspot : point sensible qui n'est pas forcément exploité maintenant, mais qui mérite revue humaine (par exemple un secret, une gestion de token ou une entrée utilisateur critique).
- Code smell : problème de qualité, lisibilité ou maintenance. Exemple : complexité cognitive de `CoursList.tsx`, imports inutilisés, ou conditions `if` trop imbriquées.

**Q21.** La règle concernée est `typescript:S2068` : “Review this potentially hard-coded password.” Le risque est qu'un secret codé en dur soit récupéré dans le dépôt, les logs de CI, les forks, ou l'historique Git, ce qui permettrait de se connecter à l'API Strapi. La correction consiste à lire le mot de passe depuis l'environnement :

```ts
const API_PASSWORD = process.env.STRAPI_CLIENT_PASSWORD ?? "";
```

Puis dans `docker-compose.yml` :

```yaml
environment:
  STRAPI_URL: http://strapi:1337
  STRAPI_CLIENT_PASSWORD: ${STRAPI_CLIENT_PASSWORD}
```

Et enfin dans `.env` / `.env.example` :

```env
STRAPI_CLIENT_PASSWORD=change-moi
```

Le mot de passe vient donc du fichier `.env`, est injecté par Docker dans le conteneur `frontend`, puis lu par `process.env.STRAPI_CLIENT_PASSWORD` dans le code. Un secret dans le code est dangereux même dans un dépôt privé, car les clones, les copies de travail, les sauvegardes, les logs, les tickets, les captures d'écran ou l'historique Git peuvent le diffuser.

**Q22.** Les code smells corrigés dans `CoursList.tsx` sont principalement :
- `typescript:S1128` — severity `MINOR` — imports inutilisés. Le nouveau code n'importe que ce qu'il utilise vraiment.
- `typescript:S6660` — severity `MAJOR` — `if` utilisé comme unique instruction dans un `else`. Le nouveau code remplace ces branches par des fonctions plus simples (`describeEmpty`, `describeResults`) et un rendu direct.
- `typescript:S3776` — severity `CRITICAL` — complexité cognitive trop élevée. La refonte extrait les calculs formatés dans des fonctions dédiées, réduisant la complexité du composant.
- `typescript:S6759` — severity `MINOR` — props de composant non marqués en lecture seule. Voir la refonte du composant et l'utilisation de fonctions purement déclaratives, plus faciles à lire et à maintenir.

Le nouveau code règle le problème en factorisant la logique dans des helpers, en simplifiant les branches, et en retirant les imports superflus.

**Q23.** Le code dupliqué se trouve dans les blocs de logique répétée autour du formatage d'un message de résultat ou dans des portions de logique UI qui se répètent entre composants de rendu. Pour le supprimer, on extrait une fonction ou un petit helper partagé (`describeEmpty`, `describeResults`), ou on factorise un composant plus générique pour éviter la duplication de logique métier/affichage.

**Q24.** La dette technique correspond au coût de correction des problèmes de qualité logicielle (bugs, vulnérabilités, code smells, duplication, complexité). SonarQube l'exprime en effort de correction, souvent en minutes ou en jours. Dans le rapport, la dette est visible sous forme d'"effort" / "technical debt"; avant la correction, elle était élevée à cause de la vulnérabilité, de la complexité et des smells. Après les corrections, elle diminue sensiblement ; l'indicateur Sonar reflète bien la réduction du coût de maintenance.

**Q25.** "New Code" dans SonarQube correspond au code modifié depuis la dernière analyse ou depuis la base de code cible (branche, commit de référence). Les conditions du gate TP3 concernent le nouveau code par défaut, puis le gate ajoute des conditions sur le code global. L'intérêt est de ne pas bloquer une PR à cause d'un legacy ancien ; on veut surtout empêcher qu'un nouveau changement apporte de la dette ou des vulnérabilités. Sonar recommande de juger le nouveau code pour encourager l'amélioration continue sans rendre le projet impossible à faire évoluer.

**Q26.** Avec `sonar.qualitygate.wait=true`, le scanner n'arrête pas simplement l'analyse : il attend le verdict du Quality Gate, puis, si celui-ci est rouge, il renvoie une erreur et le processus se termine avec un code non nul. C'est très utile en CI, car le pipeline est alors stoppé avant le déploiement ou le merge, ce qui évite d'intégrer du code non conforme aux règles de qualité.

## Partie 7 — Synthèse

**Q27.** Un pipeline CI cohérent pour ce projet doit suivre l'ordre suivant :
1. `build` / compilation du projet ;
2. tests unitaires + composants ;
3. tests d'intégration contre Strapi ;
4. tests E2E sur le site réel ;
5. analyse SonarQube avec couverture ;
6. contrôle du Quality Gate ;
7. éventuellement déploiement ou merge si tout est vert.

Justification :
- on fail-fast sur le build et les tests les plus rapides ;
- les tests unitaires sont les moins coûteux et détectent les erreurs locales rapidement ;
- les tests d'intégration vérifient ensuite la cohérence avec Strapi ;
- les tests E2E valident le parcours utilisateur réel, mais ils sont plus lents ;
- SonarQube vient enfin pour analyser la qualité, la sécurité et la couverture, et bloquer le merge si le gate est rouge.

Les étapes qui doivent bloquer le merge d'une PR sont : les tests unitaires, les tests d'intégration, les tests E2E et la validation Sonar/Quality Gate. Une PR ne doit pas être mergée tant qu'un test est rouge ou qu'un gate est non conforme.
