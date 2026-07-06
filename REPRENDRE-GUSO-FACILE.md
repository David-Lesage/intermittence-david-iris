# 🎯 REPRENDRE ICI — Guso Facile (app intermittence David & Iris)

> **But de ce fichier** : permettre de reprendre le projet dans une **nouvelle session** avec
> tout le contexte. Lis-le en entier avant d'agir. Dernière mise à jour : **2026-06-24**.
>
> ⚠️ **Prochaine grosse étape = back-office « Des Sons et Des Liens »** (voir §6). David veut
> **valider le plan AVANT toute implémentation** (§7).

---

## 1. C'est quoi l'app

**Guso Facile** = application web qui aide **David Lesage** (et **Iris Chasles**), intermittents
du spectacle, à suivre leurs **507 heures sur 12 mois** (seuil de renouvellement des droits) à
partir de leurs **cachets** (concerts) et **heures de répétition**, déclarés via des **feuillets
GUSO**.

- **En ligne (source de vérité) :** https://david-lesage.github.io/intermittence-david-iris/
- **Code d'accès :** `gusofacile` (déverrouille les données chiffrées).
- **Dates anniversaire :** David = **23 août**, Iris = **3 septembre** (le seuil 507 h se calcule
  sur les 12 mois précédents).
- Contexte humain : ils sont **en stress pour trouver assez de dates cet été** (charge mentale
  administrative énorme — c'est le sens même de l'app : simplifier).

---

## 2. Dépôt, build, déploiement (IMPORTANT)

- **Repo GitHub :** `David-Lesage/intermittence-david-iris` (public), branche `master`.
- **Hébergement :** **GitHub Pages** (site **statique**, pas de backend serveur).
- **📁 Dossier de travail local (canonique, depuis 2026-06-24) :**
  **`/Users/davidlesage/CLAUDE/GUSO FACILE/`** = clone git du dépôt (contient `index.html`, ce doc,
  `favicon.svg`, `og.png`). On édite et on `git push` ICI désormais. Pour une session Claude Code
  dédiée : terminal → `cd "/Users/davidlesage/CLAUDE/GUSO FACILE"` → `claude`.
  ⚠️ `~/CLAUDE/` est synchronisé (iCloud/Drive) : si le `.git` pose souci, mettre la synchro en pause
  pendant les opérations git.
- **Workflow de modif :**
  1. Éditer `index.html` dans `~/CLAUDE/GUSO FACILE/` (tout le code tient dans ce fichier).
  3. Vérifier la syntaxe JS : `node -e "...new Function(scriptBody)..."`.
  4. Tester en local (`python3 -m http.server` + extension Claude-in-Chrome) **sans toucher au
     cloud** (neutraliser `save` pendant les tests : `const realSave=save; window.save=()=>{}` puis
     restaurer + `state.fiches=backup`).
  5. **`git commit` + `git push origin master` SYSTÉMATIQUEMENT** (règle ferme de David, cf.
     mémoire `feedback_guso_deploy`). GitHub Pages redéploie en ~1 min. Proposer **Cmd+Shift+R**.
- ⚠️ Le **fichier source local** `/Users/davidlesage/CLAUDE/INTERMITTENCE/index.html` est une
  **vieille version SANS la synchro Firebase ni les features récentes** — NE PAS l'utiliser comme
  base. La vérité est dans le repo GitHub.

---

## 3. Architecture technique actuelle

- **1 seul fichier** `index.html` : HTML + CSS + JS vanilla (pas de framework, pas de build).
- **Données = 1 objet `state`** : `{ people:{David:{anniv},Iris:{anniv}}, fiches:[...], lastModified }`.
- **Stockage :** `localStorage` (`intermittence_v1`) + **synchro temps réel Firebase Firestore**
  (collection `sync`, doc `state`).
- **Chiffrement :** les données dans Firestore sont **chiffrées AES-GCM** côté client. Clé dérivée
  (PBKDF2-SHA256, 310000 itérations) du code `gusofacile`. ⇒ **Firebase/personne ne peut lire les
  données sans le code.** ⚠️ **Conséquence majeure pour la suite : un backend ne peut PAS lire les
  données chiffrées** sans la clé → impacte les notifications email automatiques (§6-A) et le
  multi-comptes (§6-D).
- **Réconciliation :** last-write-wins via `lastModified` (ISO). `connectSync()` adopte le cloud si
  plus récent. Garde-fou : la graine (seed) ne remplace jamais des données locales existantes.
- **Auto-reconnexion :** le code est mémorisé (`intermittence_pw`) pour ne pas re-déverrouiller.
- **Rafraîchissement auto à la date du jour** (`refreshIfNewDay`, visibilitychange + intervalle).

### Modèle d'une fiche (date)
```
{ id, owner:'David'|'Iris'|'both', type:'concert'|'repet', num (n° GUSO),
  start, end, lieu (titre/objet), place (salle/ville),
  repetStart, repetEnd, cachets, repet (h répét David), repetIris (h répét Iris si owner='both'),
  brut, tech (bool, technicien annexe 8), hypo (bool, date "possible" non validée),
  steps:{dpae,guso,paiement,recu,actu} (étapes admin cochables),
  warn (texte d'incohérence "à vérifier", '' si OK),
  notes:[{date,text}] (historique des corrections),
  pdf (chemin/lien éventuel du feuillet) }
```
- **`owner:'both'`** = date partagée David+Iris : apparaît dans les 2 onglets, compte pour chacun.
  Heures par personne via **`ficheHours(f, personne)`** et **`repetFor(f, personne)`** (Iris peut
  avoir `repetIris` ≠ `repet`).
- **`hypo:true`** = date **possible/projetée** (compte dans la projection, pas dans les heures
  validées).

### Tables de liens (par n° GUSO → id fichier Google Drive)
- **`GUSO_DRIVE`** : n° GUSO → feuillet GUSO (PDF) dans Drive. ~56 dates reliées.
- **`FACTURE_DRIVE`** : n° GUSO → facture (PDF) dans Drive. Factures **2026 reliées** (pas encore
  2024/2025).
- **`FACTURE_INFO`** : n° GUSO → `{s:salaire net, c:charges, a:frais admin, t:total facture, who:"David + Iris"}`.
  Sert au rapprochement comptable affiché sur chaque date.

### Fichiers Google Drive (montés localement)
- **Drive Résonances** (monté) : `/Users/davidlesage/Library/CloudStorage/GoogleDrive-david.lesage@resonancesproductions.org/Drive partagés/1 - RESONANCES PRODUCTIONS/`
  - `0 - DEPENSES/<année>/<N - MOIS>/` : factures GUSO (Des Sons et Des Liens → Résonances),
    nommage `<montant>€ - Facture - Resonances Production - DD-MM-YYYY.pdf`. **2026 trié par mois.**
  - `9 - INTERMITTENCE - GUSO/GUSO CLASSÉ (par date)/DAVID|IRIS/` : feuillets GUSO.
  - `1 - FACTURES CLIENTS/` : **factures Résonances → organisateur** (budget total, déplacements,
    multi-musiciens) — **PAS encore exploitées dans l'app** (cf. §5 reste à faire).
- **`pdftotext`** (poppler) et **`pypdf`** dispo en local pour extraire le texte des PDF.
- Pour récupérer un **id Drive** : MCP `search_files` (titre exact). Pour relier un feuillet : le
  copier dans le Drive monté → l'id se récupère via le MCP.

---

## 4. Le flux financier réel (à connaître absolument)

3 acteurs :
1. **Résonances Productions** (asso) : facture le **CLIENT/organisateur** de la date (budget total,
   déplacements, plusieurs musiciens). → dossier Drive `1 - FACTURES CLIENTS`.
2. **Des Sons et Des Liens** (asso avec **licence de spectacle**) : facture **Résonances** pour
   réaliser les **fiches GUSO** des artistes, et prend une **commission** (les "frais
   administratifs", ~5%). → dossier Drive `0 - DEPENSES` (factures déjà reliées dans l'app).
3. **L'artiste** (David/Iris) : reçoit le **salaire** (net) via le GUSO.

- **Coût employeur d'un cachet de 12 h ≈ 189 € MINIMUM** (salaire net + charges + 5% frais admin),
  confirmé par facture Résonances du 06/06/2026. Les cachets réels peuvent être plus élevés.
- **Selon la trésorerie de Résonances**, il y a plus ou moins de répétitions payées (ça dépend du
  flux). Christophe paie souvent **sous 24 h** dès réception facture+GUSO (sauf trésorerie basse).
- Les **retards de paiement** viennent surtout des **organisateurs** (côté Résonances).

---

## 5. État des features (DÉJÀ FAIT ✅)

- Synchro temps réel chiffrée + indicateur "Toutes les données sont sauvegardées".
- Onglets **David / Iris**, sélecteur de personne.
- **Dashboard refondu** : hero en gros « J'ai effectué X h sur 507 h · reste X h · dans X jours »,
  jauge, rythme « ≈ X h/mois », **plan mois par mois** (au prorata des jours réels restants, tient
  compte des dates possibles).
- **À facturer** recalé sur **189 €/cachet minimum (dont 5% frais admin)**.
- **Date partagée** `owner:'both'` + **heures de répét par personne**.
- **Feuillets GUSO** ouvrables (Drive) : bouton « 📄 Ouvrir le GUSO ».
- **Factures** reliées + bouton « 🧾 Ouvrir la facture ».
- **Rapprochement comptable** sur chaque date : salaire brut **+** montant facturé détaillé
  (salaire net + charges + frais admin), mention multi-musiciens (« couvre David + Iris »).
- **Correction d'incohérence** : modifier une date avec un « ⚠️ à vérifier » efface l'avertissement
  et ajoute une **note d'historique** datée (`f.notes`).
- **Alertes agenda** Google (calendrier « Iris & David ») : 4 rappels mensuels récurrents
  (juin→sept) « heures à trouver ». ⚠️ Chiffres figés → à **régénérer** quand les données changent.
- Ajout/modif/suppression de dates, dates possibles, étapes admin cochables, export/import JSON,
  graphique annuel, infobulles partout, niveaux non concernés.

### RESTE À FAIRE (hors grosse étape) — non commencé

> ✅ **FAIT (2026-06-24) — DPAE nominative par personne sur les dates partagées** (remarque Des Sons
> et Des Liens). Sur une date `owner:'both'`, la DPAE est désormais **dissociée** : `steps.dpae` =
> David, `steps.dpaeIris` = Iris (helpers `dpaeDone(f,personne)` / `dpaeKey` / `dpaePersons`). Le
> **détail** affiche **2 cases** (« DPAE David » / « DPAE Iris ») ; le **back-office** liste la DPAE
> manquante **par personne** (1 ligne par personne, dédoublonnée par groupe+personne) tandis que GUSO
> et facture restent **1 ligne par date** ; le **regroupement ±7 j** propage la DPAE **par personne**.
> Migration douce (`ensureDpaeIris` : Iris hérite de l'état existant). **Reste lié = Phase 5** : le
> **dépôt des 2 PDF** séparés (DPAE David / DPAE Iris) nécessite le backend.

1. **(petit) Auto-remplir la date de fin = date de début** à la saisie (en général 1 seule date par
   GUSO) — gain de temps.
2. **(petit) Inverser** le bloc « À facturer aux organisateurs » et le bloc « Projection avec tes
   dates possibles » (ordre dans le dashboard).
3. **(moyen) Badge de validation/cohérence GUSO** : vérifier que le **feuillet GUSO existe** pour
   chaque date ET qu'il est **cohérent** avec cachets/heures annoncés (surtout sur les dates
   projetées, car Des Sons et Des Liens peut faire des erreurs). Badge « tout est cohérent » quand
   actif. Dans le **bandeau du haut** : afficher les heures **prouvées par GUSO tamponné** vs les
   heures **pas encore tamponnées** → ce qu'il reste à faire côté Des Sons et Des Liens.
4. **Relier 2024/2025** (factures + feuillets) — coûteux en tokens (lire/croiser bcp de PDF).
5. **Factures CLIENTS** (Résonances → organisateur) : budget organisateur + **frais de déplacement**
   + détail multi-musiciens. Dossier `1 - FACTURES CLIENTS`. Permettrait le rapprochement complet.

---

## 6. 🚀 PROCHAINE GROSSE ÉTAPE — Back-office « Des Sons et Des Liens »

> Source : transcription d'une conversation orale David ↔ **Christophe** (Des Sons et Des Liens,
> + **Myriam** intéressée). Besoins extraits et structurés ci-dessous.

### A. Notifications email automatiques (cycle de vie d'une date)
1. **J-3/J-4 avant** une date à déclarer → email à **Des Sons et Des Liens** : « DPAE à faire »,
   avec **lien cliquable direct vers LA date** dans Guso Facile. Christophe fait la DPAE puis
   **coche « DPAE faite »** dans l'app.
2. **J+1 (lendemain de la date)** → email auto : « la date a eu lieu → éditer le **GUSO** + la
   **facture** », avec lien vers la date. Christophe édite les 2 docs et les **upload dans l'app**.
3. **Quand GUSO + facture déposés** → email à **Résonances (David)** : « il faut payer »
   (trésorerie OK → paiement souvent < 24 h, sans attendre).
4. *(Futur)* connexion **banque** : vérifie solde, vérifie si l'organisateur a payé, relances.

⚠️ **Contrainte technique forte** : l'app est **statique (GitHub Pages, pas de backend)** ET les
données sont **chiffrées** (un serveur ne peut pas les lire sans le code). Les emails automatiques
**exigent une infra serveur** (ex. Firebase Functions + scheduler, ou Supabase Edge Functions +
cron — le **dev externe** gère déjà Supabase/Stripe sur Neotone Studio, possible réutilisation) ET
une **stratégie pour les données** (stocker des métadonnées de planification non chiffrées, ou
donner la clé au backend, ou backend qui ne voit que le strict nécessaire). **À cadrer.**

### B. 3ᵉ onglet / Dashboard « Des Sons et Des Liens » (back-office to-do)
- En plus de David / Iris : un onglet **tableau de bord pour Christophe/Myriam**.
- Vue transversale **« ce qu'il reste à faire »** : toutes les **DPAE à faire**, tous les **GUSO à
  éditer**, toutes les **factures à éditer** pour Résonances — calculable depuis les `steps` admin
  existants (dpae/guso/paiement…) sur toutes les fiches.
- ✅ **Faisable côté client** (pas de backend) — bonne 1ʳᵉ brique.

### C. Regroupement DPAE / dates liées (fenêtre 7 jours)
- Une **seule DPAE** peut couvrir **plusieurs dates dans ±7 jours** (ex. 3 concerts/semaine +
  répètes). Une **fiche GUSO peut regrouper plusieurs dates** (3 lieux, **1 seul salaire / 1 seule
  charge**) — regroupement **MANUEL**.
- **Mécanisme demandé** : à la déclaration d'une DPAE (ou création de date), le logiciel **détecte
  les autres dates dans ±7 jours** et propose un **pop-up** : « Veux-tu relier cette DPAE à ces
  dates ? » avec **cases à cocher**. L'utilisateur choisit lesquelles relier.
- Notes : DPAE = promesse d'embauche (nom/prénom/date naissance, **pas de lieu**) ; **15 jours**
  pour déclarer après une date faite. Modèle possible : champ `dpaeGroupId` reliant des fiches.
- ✅ **Faisable côté client.**

### D. Multi-comptes / rôles (VISION FUTURE — gros chantier)
- **Login/mot de passe individuel par artiste**.
- Dashboard **Des Sons et Des Liens** = accès à **tous leurs artistes**.
- Chaque artiste : accès **à ses dates uniquement**.
- **Cloisonnement** : David ↔ Iris se voient mutuellement (cas spécial), mais **Myriam séparée**
  (David ne voit pas Myriam et inversement).
- Vision long terme : devenir une **boîte de prod** (mutualiser organisateurs/plans).
- ⚠️ **Re-architecture MAJEURE** : aujourd'hui = **1 doc chiffré partagé, 1 code commun**. Le
  multi-utilisateur impose une **vraie auth + isolation des données par compte + rôles**. Le dev
  externe utilise déjà **Supabase Auth** (Neotone Studio) → piste de réutilisation. **Très gros.**

---

## 7bis. ✅✅ FAIT (2026-06-24) — Phases 0 à 4 livrées et EN LIGNE

- **Phase 0** ✅ date de fin auto = date début · blocs Projection/À facturer inversés.
- **Phase 1** ✅ bandeau preuve GUSO (heures prouvées/tamponné vs à tamponner) + badge cohérence.
- **Phase 2** ✅ 3ᵉ onglet « 🛠️ Des Sons et Des Liens » (`openBackoffice`) : to-do transversal
  DPAE / feuillets GUSO / factures-virements, tous artistes (clic → ouvre la date).
- **Phase 3** ✅ deep-links `#date=<id>` (ouvre la fiche, même après synchro cloud) + bouton
  « 🔗 Copier le lien » dans le détail.
- **Phase 4** ✅ regroupement DPAE : à l'ajout d'une date confirmée, pop-up détectant les dates
  dans ±7 j (`maybeDpaeGroup`/`applyDpaeGroup`/`dpaeGroupId`) ; cocher la DPAE d'une date groupée
  coche tout le groupe ; back-office dédoublonne les DPAE liées (marqueur « 🔗 +N »).

**Autres livrés (2026-07-06) :**
- ✅ **DPAE nominative par personne** sur les dates partagées (voir §5).
- ✅ **Récap mensuel « rapprochement France Travail »** (`openRecap`/`buildRecap`, bouton
  « 📅 Récap mensuel ») : par personne (David/Iris), regroupe les dates **confirmées par n° GUSO**
  (1 déclaration = 1 ligne, comme l'écran « Mes activités » de FT) puis **par mois**. Chaque ligne :
  **date/période · nb cachets · heures répét · brut**. Total par mois + total global. Sert à pointer
  contre l'actualisation FT. ⚠️ N'affiche **que les dates confirmées** (pas les « possibles ») — une
  date encore en « possible » mais déjà connue de FT n'apparaît donc pas (à confirmer côté appli).

**⇒ RESTE (backend, reporté) : Phase 5 (emails auto) et Phase 6 (multi-comptes/rôles).**
Détails inchangés ci-dessous (§6-A et §6-D + §7 historique).

---

## 7. ✅ PLAN VALIDÉ PAR DAVID (2026-06-24)

> **DÉCISION : faire « Tout le client d'abord » = Phases 0→4 ci-dessous (côté client, SANS
> backend).** Les Phases 5 (emails/backend) et 6 (multi-comptes) sont **reportées** (à cadrer
> séparément, après — priorité Neotone Studio). ⇒ La prochaine session peut **démarrer
> directement l'implémentation des Phases 0→4**, par petits incréments **déployés un par un**
> (commit+push après chaque), sans tout enchaîner d'un coup. Tester sans polluer le cloud.

**Ordre d'implémentation conseillé (Phases 0→4) :**
1. **Phase 0** — auto-date-fin (= date début à la saisie) + inverser les blocs « À facturer » ⇄
   « Projection dates possibles ».
2. **Phase 1** — badge cohérence GUSO + bandeau « heures prouvées (GUSO tamponné) vs à tamponner ».
3. **Phase 3** — deep-links `#date=<id>` (ouvre la fiche) — petit, réutilisable, à faire tôt.
4. **Phase 2** — 3ᵉ onglet « Des Sons et Des Liens » (to-do DPAE / GUSO / factures).
5. **Phase 4** — regroupement DPAE : pop-up détection des dates dans ±7 jours + `dpaeGroupId`.

**Découpage en phases (rappel, du moins cher / sans backend au plus lourd) :**

- **Phase 0 — petits réglages (rapide, client) :** auto-date-fin, inversion des 2 blocs dashboard.
- **Phase 1 — badge cohérence GUSO + bandeau "prouvé vs à tamponner" (client) :** §5.3.
- **Phase 2 — Onglet « Des Sons et Des Liens » (dashboard back-office to-do) (client) :** §6-B.
- **Phase 3 — Deep-links vers une date (`#date=<id>` ouvre la fiche) (client) :** brique
  réutilisable tout de suite et indispensable pour les futurs emails (§6-A).
- **Phase 4 — Regroupement DPAE / pop-up ±7 jours (client) :** §6-C.
- **Phase 5 — Notifications email automatiques (BACKEND requis) :** §6-A. Nécessite infra serveur +
  stratégie données chiffrées + sans doute le **dev externe**. Décision/budget à part.
- **Phase 6 — Multi-comptes / rôles (BACKEND, re-archi majeure) :** §6-D. Gros projet, à cadrer
  avec le dev externe (Supabase). Probablement après Neotone Studio.

**Recommandation :** faire **Phases 0→4 côté client** (rapides, vrai gain immédiat pour Christophe,
sans coût d'infra), PUIS cadrer sérieusement Phases 5–6 (backend) séparément.

⚠️ **PRIORITÉ GLOBALE de David : Neotone Studio + site vitrine d'abord** (budget tokens limité). Donc
même validé, avancer par petits incréments déployés, sans tout enchaîner d'un coup.

---

## 8. Mémoire / préférences clés
- **Toujours déployer en ligne après chaque modif** (commit+push), sans attendre la demande.
- **Tester sans jamais polluer le cloud** (neutraliser `save`, restaurer le state).
- Communication : explications longues en chat, **choix courts/cliquables**.
- Style : pour l'esthétique/UX proposer des variantes ; pour le structurel, trancher puis exécuter.
- Données = sacrées : ne jamais risquer un écrasement ; vérifier local **et** cloud en cas de doute.
