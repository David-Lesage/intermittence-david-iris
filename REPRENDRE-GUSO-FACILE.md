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
- **Dates anniversaire :** David = **23 août**, Iris = **16 août** (corrigée le 2026-07-13, avant :
  3 septembre). Le seuil 507 h se calcule sur les 12 mois précédents. Modifiable dans le dashboard
  ET dans le profil (champ « Date anniversaire intermittence », stocké sur `state.people[p].anniv`).
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
> **Types :** `concert` · `repet` · **`studio`** (musicien de studio, contrat « chèque-intermittents »,
> conv. coll. **2121 édition phonographique**, PAS un GUSO — Audiens/Congés Spectacles/France Travail).
> Un cachet studio compte **comme un cachet concert : 1 cachet = 12 h** (annexe 10). Le studio est
> **exclu du back-office & du to-do artiste** (pas géré par Des Sons et Des Liens). Le flag `tech` (annexe 8)
> reste orthogonal. Le graphique annuel a 4 catégories via `ficheCat()` : Concert / Répét / Technicien / Studio.
```
{ id, owner:'David'|'Iris'|'both', type:'concert'|'repet'|'studio', num (n° GUSO),
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
  contre l'actualisation FT. Les **dates « possibles »** du mois sont affichées **en grisé** (badge
  🔮, marqueur « +N poss. » sur le mois) mais **non comptées** dans les totaux — pour repérer une date
  encore en « possible » mais déjà connue de FT (à confirmer côté appli).

**⇒ RESTE (backend, reporté) : Phase 5 (emails auto) et Phase 6 (multi-comptes/rôles).**
Détails inchangés ci-dessous (§6-A et §6-D + §7 historique).

---

## 9. 🚀 VISION ÉTENDUE (2026-07-09) — gestionnaire de contrats / « bon plan » / signature

> Demande de David : faire de Guso Facile un vrai **gestionnaire pour musiciens ET structures**
> (profils, éditeur de contrats sur-mesure, évaluateur de « bon plan », négociation, signature).
> Découpage en modules, du **client-side (faisable sans backend)** au **backend requis**.

**Client-side (sans backend) :**
- ✅ **Module 1 — Espace Profil** par artiste (`openProfile`, bouton « 👤 Profil ») : identité/admin
  (régime, n° sécu 🔒, n° GUSO, congés spectacles…), instruments, photo (lien), bio, **« mes
  conditions idéales »** (prix min, visibilité, trajet, distance, logement, loge, scène, ingé son,
  sono), modèles de contrats (liens Drive). Stocké `state.people[p].profile` (chiffré, jamais dans
  le repo public). **LIVRÉ 2026-07-09.**
  - **+ Date anniversaire** modifiable dans le profil (`special:'anniv'` → `state.people[p].anniv`). **2026-07-13.**
  - **+ Coordonnées bancaires (2026-07-13)** : IBAN / banque / BIC + **RIB officiel** en glisser-déposer
    (`_pfRib`, dataURL embarqué, plafond `RIB_MAX` 300 Ko sinon lien Drive `ribLink`) ; bouton « 👁 Afficher »
    (`openDataUrl` → blob). Exposé aussi dans la fiche « Mes artistes » (Module 4) + le « Copier les infos »
    (l'IBAN/BIC servent aux virements ; certaines structures exigent le RIB officiel de la banque).
  - **+ Fiche « Ma structure » (2026-07-13)** : `state.structure` (SIRET, forme, licence spectacle,
    contact, IBAN/banque/BIC + RIB officiel en glisser-déposer, `_stRib`). `openStructure`/`saveStructure`,
    modale `#structBg`, accès via bouton « 🏢 Ma structure » en tête du back-office Des Sons et Des Liens.
- ✅ **Module 2 — Évaluateur de date « bon plan ? »** (`openCond`/`computeCond`, bouton « 🎯 Bon
  plan ? » dans le détail d'un **concert**) : par date, saisir les **conditions réelles** proposées
  par l'orga (`f.cond` : prix, distance, trajet, logement, loge, scène + dimensions, ingé son, sono
  à amener, visibilité, notes ; selects oui/non/« ? » pour distinguer « non » de « non renseigné »)
  → **comparaison en direct** avec les conditions idéales du profil de la personne de l'onglet →
  **verdict/score** 🟢 bon plan (≥75 %) / 🟡 à négocier (45–74 %) / 🔴 à éviter (<45 %) / ℹ️ gris si
  < 3 critères comparables. Liste ✅/❌ avec détail des écarts + **« infos manquantes à demander à
  l'organisateur »** (brique du Module 3). Badge du verdict affiché dans le détail. **LIVRÉ 2026-07-09.**
- ✅ **Module 3 — Organisateur & suivi de négociation** (`openOrga`, bouton « 🏢 Organisateur » à côté
  de « 🎯 Bon plan ? » dans le détail d'un **concert**) : par date, saisir la **structure qui emploie**
  (`f.orga` : structure, contact, téléphone, email, adresse, SIRET, forme juridique, n° licence
  spectacle, notes) + l'**état de la négociation** (`f.nego` : statut 📞 contact / 🤝 négociation /
  📤 contrat envoyé / ✅ signé / ❌ annulé, et **échéance de signature**). **Badges** dans le détail
  (statut + « ⏳ à signer avant le JJ mois » en jaune ≤ 7 j, « ⚠️ échéance dépassée » en rouge) et
  **mini-marqueur d'échéance** dans le pied des cartes de la liste. Cœur = bloc **« ❔ Infos
  manquantes »** qui fusionne les 5 champs orga clés vides (structure, contact, email, SIRET, forme)
  + les critères « ? » du Module 2 (`computeCond`), et bouton **« 📋 Copier la demande d'infos »** qui
  génère un message poli prêt à envoyer (mail/WhatsApp) listant exactement ces manques, signé du
  prénom de l'artiste. ⚠️ Ne partage **jamais** de lien de l'app / deep-link (code d'accès commun) —
  partage externe sécurisé = Phase 6/backend. **LIVRÉ 2026-07-09.**
- ✅ **Module 4 — Section structure « 👥 Mes artistes »** (`boArtistsHtml`/`openArtistCard`, en haut
  du back-office) : une carte par personne de `state.people` (photo → fallback 🎤, nom complet
  `FULL_NAMES`, qualification, instruments) → **fiche administrative** dépliée (état civil,
  **n° sécu masqué par défaut** + bascule 👁, n° GUSO, congés spectacles, adresse, tél, email,
  régime/qualification), bouton **📋 Copier les infos** (bloc texte complet pour DPAE/GUSO, sécu en
  clair) + **👤 Ouvrir le profil complet** (`openProfile`) + retour liste sans fermer le back-office.
  Cloisonnement (Myriam ≠ David) = Phase 6. **LIVRÉ 2026-07-09.**
- **Module 5 — Signature de l'artiste** : canvas pour dessiner sa signature (petit dataURL),
  intégrable aux contrats.

**Backend requis (Phase 5/6, à cadrer/budgéter à part) :**
- **Module 6 — Lien à l'organisateur** : l'orga remplit SA partie à distance (contact/SIRET/signe)
  via un lien → un tiers écrit des données **sans le code commun** ⇒ backend.
- **Module 7 — Signature électronique 2 parties + génération du contrat PDF signé** ⇒ backend + stockage.
- **Module 8 — Partage sécurisé/cloisonné** (vrais comptes) = Phase 6.
- **Emails auto** (rappels DPAE/GUSO/paiement) = Phase 5.

**Données sensibles :** le **n° sécu** (et le Google Sheet fourni par David :
`docs.google.com/spreadsheets/d/1hJkZECxTeBoxU1yasma0AHq7rinK6vBQrXqEaRGEOII`) vont dans la **LIVE
data chiffrée** (Firestore), **jamais** dans `ENC_SEED`/le repo public. David a demandé de lire le
Sheet pour pré-remplir les profils (à faire dans un flux contrôlé, écriture dans les données live).

**Contrainte technique récurrente :** tout `state` tient dans **1 document Firestore (~1 Mo max)** →
photos/PDF = **liens Drive** (pas de fichiers en base64), pas de vrai upload sans backend.

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

## 9-bis. Ajouts 2026-07-14 (session 4 tâches)
- **A. Dépôt de facture par date** (`factureBlock`/`factPick`/`factDrop`/`factRemove`/`factSetUrl`/`openFacture`, cap `FACT_MAX`=350 Ko) : dans `openDetail`, si aucune facture accessible → zone glisser-déposer « 🧾 Facture manquante » (PDF/image, `f.factureFile={name,type,data}`) + champ « coller un lien Drive » (`f.factureUrl`) ; sinon bouton « Ouvrir la facture » (+ « 🗑 Retirer » si non-officielle). Exclu des dates studio.
- **B. Back-office DPAE inclut les dates possibles** : la catégorie DPAE liste aussi les dates `hypo` **à venir** (badge « 🔮 possible », couleur `--hypo`) pour anticiper ; GUSO/Factures restent aux dates confirmées.
- **C. Retour à l'écran d'origine** (`openDetail(id,from)` + `_detailFrom`) : `closeDetail()` ré-ouvre le back-office si origine `backoffice` (rafraîchi), le back-office + fiche artiste si `artist:<p>`, sinon retour liste. Lignes BO appellent `openDetail(id,'backoffice')` sans fermer le BO.
- **D. Écran d'accueil** (`#welcomeOverlay`, `openWelcome`/`chooseSpace`) : affiché par `startApp()` (sauf deep-link `#date=`), 2 cartes « 👤 Espace artiste » / « 🏢 Espace structure », mémorise `localStorage.gf_space`, bouton « 🏠 Accueil » dans l'en-tête.

## 9-ter. Ajouts 2026-07-14 (session UX dashboard/détail/accueil)
- **1. Célébration des 507 h atteintes** : dans `render()`, `won = totalH>=THRESHOLD` → `.hero.celebrate` (CSS). Jauge `#gaugeArc` passe en `var(--ok)` + glow SVG (filtre `#gaugeGlow` feGaussianBlur/feMerge défini dans le `<svg>` du hero) + anim `gaugePulse`. Bloc normal `#heroNormal` masqué, `#heroWin` (« 🎉 Objectif 507 h atteint ! » + `#heroWinSub` marge) affiché. SVG festif `#heroCelebrate` (trophée + 3 étincelles `.spark`) à droite du hero, masqué < 640 px. Ligne anniversaire devient « 📅 Date anniversaire : … · dans X jours ». Sinon dashboard normal inchangé.
- **2. Clarté confirmé vs projection** : hero étiqueté `.hero-tag` « confirmé » ; ligne « Il me reste X à trouver ». Bloc projection : titre « 🔮 Si tes dates possibles se confirment », dernière cellule « resterait à trouver », `#projFoot` reformulé en réduction (« Tes N date(s) possible(s) feraient passer le reste à trouver de X h à Y h. »).
- **3. Fenêtre de détail** (`openDetail`) : croix `.bo-x` ✕ en haut à droite du `#modalBox` (`.modal{position:relative}`) ; boutons d'action tous en `.sm` sur une seule ligne (labels raccourcis « 🔗 Lien », « 🔮 Potentielle », « 📄 GUSO ») — `.modal{max-width:620px}` ; label « 🎼 Répétition » (raccourci) + `.kv .k{white-space:nowrap}` ; paddings/marges réduits pour limiter le scroll.
- **4. Accueil + guide génériques** : sous-titre carte « Espace artiste » sur 2 lignes (« Suivi des heures, dates et démarches. » / « Une vision claire. »), sans « David & Iris ». `GUIDE.artiste` réécrit en « toi/ton prénom » sans nommer David/Iris.

## 9-quater. Optimisation NAVIGATION MOBILE (2026-07-14)
Ergonomie/responsive uniquement (aucune feature nouvelle), tout sous media queries → desktop inchangé (vérifié à 1280 px : 6 boutons d'en-tête en ligne, bouton inline « Ajouter une date », pas de FAB).
- **En-tête compact** (`@media max-width:640px`) : l'en-tête n'empile plus 6 boutons. `.hdr-tools` contient 🏠 Accueil (toujours visible) + un bouton `.hdr-menu-btn` « ☰ Menu ». Les 5 actions secondaires (Guide, Récap, Profil, Sauvegarder, Importer) sont dans `.hdr-actions#hdrActions`, panneau déroulant absolu (`left:0`, `width:min(240px,100vw-32px)`) ouvert par `toggleHdrMenu(ev)` (classe `.open`). Fermeture auto via listener `document.click` (sauf clic sur le bouton menu). Boutons du menu en pleine largeur, ≥44 px.
- **Onglets tactiles** (`≤560px`) : `.tab` min-height 48 px, flex centré, texte 13 px.
- **FAB « ➕ »** : bouton flottant fixe bas-droite (`.fab`, 60×60, z-index 130), affiché ≤640 px ; l'ancien `.action-bar` inline est masqué en mobile ; `.wrap{padding-bottom:88px}` pour ne rien cacher. FAB masqué quand une modale est ouverte : `body:has(.modal-bg.show) .fab{display:none}`.
- **Modales plein écran** (`≤560px`) : `.modal{max-width:100%;width:100%;min-height:100vh;border-radius:0}`. Croix `.bo-x` en `position:fixed` haut-droite, 44×44, fond `--panel2` (toujours visible). Formulaire d'ajout `form.fiche` en 1 colonne, `.add-head` sticky. `.kv` et `.ba-tab` en 1 colonne.
- **Zéro débordement horizontal** : `html,body{overflow-x:hidden}` + `#yearChart svg{width:100%}`. Vérifié : `scrollWidth==innerWidth` (375) sur dashboard, menu ouvert, modales détail/add/back-office/accueil.

## 9-quinquies. Module « 🔍 Vérifier & bilan » (2026-07-14)
Contrôle de cohérence de la période en cours + bilan imprimable. Aucune régression sur l'existant.
- **`runCheck(person)`** : sur `currentPeriod(person)` et les dates CONFIRMÉES (non hypo, via `fichesFor(p,'period')`), remonte les anomalies par catégorie. Alertes (sur dates passées `start<today0()`, type≠studio) : `guso-num` (n° GUSO vide), `guso-feuillet` (`steps.guso=false`), `dpae` (`dpaeDone(f,pp)=false` pour chaque `dpaePersons`), `paiement` (`steps.paiement=false`) ; `warn` (f.warn non vide) toujours. Infos (non bloquant) : `studio` (hors circuit GUSO), `actu` (`steps.actu`). Retour `{ok, issues:[{niveau:'alerte'|'info',cat,date,lieu,msg,id}], stats:{heures,cachets,repetH,brut,nbDates,prouvees,aTamponner}, period, person}`. (David ≈521 h, 18 dates, 11 alertes ; Iris ≈412 h, 13 dates.)
- **UI** : modale `#bilanBg` (style `recap-modal`, croix `.bo-x`, toggle David/Iris `blTab*`). `openBilan(person)`/`closeBilan(ev)`, `buildBilan(person)` → verdict (🟢 « Tout est en ordre » ou 🟡 « N points à régler » liste cliquable → `closeBilan();openDetail(id)`), récap `.bl-recap`/`.bl-stat` (heures/507, cachets, répét, brut, dates, prouvées, à tamponner), liste `.bl-row` de toutes les dates + statut `_blDateStatus(f,person)`. CSS `.bl-*` ajouté après `.r-hero .bo-sub`.
- **Accès** : bouton « 🔍 Vérifier & bilan » dans `#hdrActions` (après Récap mensuel) + bouton `.bl-hero-btn` (vert) dans `#heroWin` (visible quand `.hero.celebrate` actif).
- **Impression** : `printBilan()` → `window.open` d'un HTML autoportant (styles inline, `@page`, titre fenêtre « Bilan Guso Facile ») construit par `buildBilanPrint(person)` : titre « Bilan intermittence — <Prénom> », période, verdict, points à régler, tableau récap, tableau des dates+statut, rappels infos ; `window.print()` après 400 ms. Fallback alert si pop-up bloqué.
- **Testé** (port 8000, `pushRemote`/`save` neutralisés) : runCheck stats cohérentes, modale verdict+récap+liste OK, clic anomalie ouvre la date (`modalBg`), toggle Iris OK, print HTML (9 Ko, titre correct), mobile 375 px plein écran sans débordement (grille stats en 2 colonnes).

## 9-sexies. Accueil SVG + encouragements + libellés virement + log coches (2026-07-14)
- **1. Accueil épuré + 2 SVG maison** : suppression de l'affichage « Dernier espace utilisé » (l'élément `#welcomeHint` a été retiré ; `openWelcome()` ne l'écrit plus, mais `localStorage.gf_space` reste mémorisé en interne). Carte 1 renommée « **Espace artistes** » (pluriel). Les 2 pictos emoji sont remplacés par des **SVG inline** (`.wc-svg`, viewBox `0 0 120 84`, `max-width:60%` → responsives) : carte artistes = 3 silhouettes (chanteur+micro `--accent`, guitariste `--accent2`/`--iris`, danseur `--ok`) ; carte structure = personnage souriant multi-bras `--accent2` tenant 4 feuilles GUSO/contrats (`--accent`/`--iris`/`--ok`/`--warn`).
- **2. Encouragements dashboard artiste** : constante `ENCOURAGEMENTS` (15 phrases : foi/responsabilisation/entraide). Bandeau `#encourageBox` (`.encourage`, ✨ + texte italique `--muted` + bouton `↻` `pickEncouragement()`) inséré juste sous le hero, avant `#proofStrip`. `pickEncouragement()` tire une phrase au hasard dans `#encourageText` ; appelé une fois par chargement depuis `render()` (seulement si le texte est vide → stable entre re-rendus), re-tirable via `↻`.
- **3. Libellés cases virement** (clés `paiement`/`recu` inchangées) : `paiement` → 💸 short « Facture réglée », label « Facture Des Sons et Des Liens → Résonances réglée ». `recu` → 🏦 short « Salaire reçu », label « Salaire net reçu par l'artiste (via GUSO) ». Se propagent partout via `STEPS` (liste `renderTodo`, détail `stepChip`, toast). La catégorie back-office « Factures / virements » (tableau séparé ligne ~1470) est inchangée.
- **4. Log léger des coches** : variable globale `_space` (init depuis `localStorage.gf_space`, défaut `'artiste'`, mise à jour par `chooseSpace`), helper `spaceLabel()` (« espace Artiste »/« espace Structure »). Dans `toggleStep`, chaque coche/décoche pousse `{date:isoLocal(today0()), text:'☑️/⬜ <short> cochée/décochée · <espace>'}` dans `f.notes` (affiché dans « 📝 Historique des corrections » d'`openDetail`). DPAE nominative → short « DPAE Iris » / « DPAE <owner> ». (Le vrai « qui » fiable = Phase 6 comptes/login.)
- **Testé** (port 8000, `pushRemote`/`save` neutralisés, 375 px) : accueil sans « dernier espace », « Espace artistes », 2 SVG affichés et responsives ; phrase d'encouragement visible et variable (14 distinctes /40 tirages) ; libellés « Facture réglée »/« Salaire reçu » ; `toggleStep` ajoute bien une note horodatée avec l'espace (« ☑️ Facture réglée cochée · espace Structure », « ⬜ DPAE David décochée · espace Structure ») — notes de test nettoyées ensuite. Aucune erreur console au reload.

## 9-septies. Header réorganisé + 2 badges clarifiés + bloc « À faire maintenant » (2026-07-14)
- **1. Header réordonné** (`.hdr-tools`) : ordre `🏠 Accueil · 🔔 À faire (badge) · 📅 Récap mensuel · 🔍 Bilan · 👤 Profil · ❓ Guide · ⋯`. Les actions NAV/SUIVI (À faire, Récap, Bilan, Profil, Guide) sont dans `#hdrActions` (inline desktop / dropdown ☰ mobile). Les actions DONNÉES (Sauvegarder/Importer) sont reléguées dans un menu secondaire `⋯` desktop (`#hdrMore`, `toggleHdrMore()`/`closeHdrMore()`) et, sur mobile, dans le bas du menu ☰ via un sous-bloc `.hdr-data-mob` séparé par un filet (le `⋯` et `#hdrMore` sont masqués sur mobile). `closeHdrMenu()` extrait en fonction. Bouton « 🔔 À faire » = `focusTodo()` (ferme les menus, scrolle vers `#todoNow` + flash), avec `<span id="todoBadge">` (compteur, `hidden` si 0).
- **2. Badge bouclier** (`#proofStrip`, 1er `.pb`) : désormais piloté par `totalH>=507` (et non plus `allCoherent`). Sous 507 h → « 🛡️ Droits à sécuriser · reste X h » (warn, X=`remaining`) ; à 507 h+ → « 🛡️ Droits sécurisés ✓ » (ok/vert). `data-tip` : « Tu sécurises tes droits d'intermittent en atteignant 507 h sur 12 mois. »
- **2bis. Badge « à vérifier »** : rendu **cliquable** (`.pb.warn.clickable`, `onclick="openFirstWarn()"`), libellé « ⚠️ N incohérence(s) à vérifier », `data-tip` explicatif. `openFirstWarn()` ouvre `openDetail(window._firstWarnId)` (1ʳᵉ fiche `f.warn` de la période, id stocké dans `window._firstWarnId` en fin de bloc proofStrip). Masqué si N=0.
- **3. Bloc `#todoNow`** (`renderTodoNow(p)`, appelé en tête de `render()`) : inséré juste après le hero, avant `#encourageBox`. Liste les pièces manquantes de la personne courante triées par urgence (clé `sk`) : incohérences `f.warn` (sk -3, « à vérifier »), puis dates PASSÉES de la période (feuillet GUSO manquant / DPAE en retard / facture-salaire en attente → sk -2/-1, « en retard »), puis DPAE des dates À VENIR (sk = J-X ; badge « J-X », rouge si ≤3 j). Chaque ligne cliquable → `openDetail(id)`. Compteur repris dans `#todoBadge`. Si vide → `.allok` « ✅ Rien en attente, tout est carré ». CSS `.todonow`/`.tn-*` (responsive, labels ellipsés desktop, wrap mobile).
- **Testé** (port 8000, neutralisé, David 521 h + Iris 412 h, desktop 1280 ET mobile 375) : header réordonné desktop avec badge « À faire 8 » ; menu ☰ mobile = nav + filet + Sauvegarder/Importer (0 débordement, scrollWidth=375) ; `⋯` desktop OK (Sauvegarder/Importer) ; bouclier « Droits sécurisés ✓ » (David) / « Droits à sécuriser · reste 95 h » (Iris) ; « ⚠️ 2 incohérences à vérifier » clique → détail ouvert ; `#todoNow` trie incohérences → en retard → J-X (test date future injectée à J-2 → « DPAE à faire … J-2 » urgent, retirée ensuite ; RDV réel à J-3 visible). Rechargé en fin.

## 9-octies. AJOUTER de nouveaux artistes — espace PARTAGÉ (2026-07-14)
> Objectif : ne plus être câblé pour 2 artistes en dur (David/Iris) mais pouvoir en **ajouter** d'autres, chacun avec son onglet + suivi 507 h complet. ⚠️ Ce n'est PAS le cloisonnement par compte (tout le monde voit tout) — le **vrai cloisonnement / login = Phase 6** (backend, cf. §6-D / §7 Phase 6).
- **Onglets dynamiques** : `#tabs` n'a plus les `.tab` David/Iris en dur — `renderTabs()` génère un `.tab[data-p]` par `Object.keys(state.people)` (dot coloré via `personColor(p)`), + un `.tab.tab-addart` « ➕ Artiste » (→ `openAddArtist()`), + `#boTab` (back-office). Appelé dans `startApp()` ET en tête de `render()` (reste synchro après sync/import distant). `setPerson` inchangé (toggle `.tab.active` par `data-p`), accent via `personColor(p)`.
- **`personColor(p)`** : David=`var(--david)`, Iris=`var(--iris)`, les autres tournent sur `PERSON_PALETTE` (orange, vert, violet…). `fullName(p)` lit désormais `state.people[p].fullName` (fallback `FULL_NAMES` puis clé).
- **Modale « ➕ Ajouter un artiste »** (`#addArtistBg`) : Nom complet (obligatoire) + Date anniversaire intermittence. `submitAddArtist()` dérive une clé/prénom unique (`artistKey()`, gère accents + doublons → `Marius`, `Marius2`…), crée `state.people[cle]={anniv, profile:{}, fullName}`, `save()` + `renderTabs()` + `renderOwnerOptions()` + `setPerson(cle)` + toast.
- **Toggles profil/récap/bilan** : plus de boutons David/Iris en dur — conteneurs `#profileToggle`/`#recapToggle`/`#bilanToggle` remplis par `renderPersonToggle(id, active, fn)` (1 `.r-tab` par artiste). `openProfile/openRecap/openBilan` appellent ce helper.
- **Select propriétaire** (`#f_owner`) : `renderOwnerOptions()` = tous les artistes (owner=clé, propriétaires SOLO) + « 👥 David et Iris (partagée) » (value `both`). Les nouveaux artistes n'ont JAMAIS `both`.
- **⚠️ Correctif clé** : `belongsTo(f,p)` = `f.owner===p || (f.owner==='both' && (p==='David'||p==='Iris'))`. Avant, `f.owner==='both'` était traité comme « appartient à tout le monde » → un nouvel artiste héritait des dates partagées David+Iris. Corrigé dans `fichesFor`, `avgRate`, `renderTodoNow` (`mine`), `buildRecap` (`involves`). `both` reste STRICTEMENT David+Iris. Cas spéciaux `repetIris`/`dpaeIris`/DPAE nominative inchangés.
- **Testé** (port 8000, neutralisé, desktop + mobile 375) : baseline David 521 h / Iris 412 h ; ajout « Marius Test » (anniv 2026-03-15) → onglet orange + dashboard **0/507** cohérent ; date SOLO (owner=Marius, 2 cachets=24 h) → Marius 24 h, **David 521 / Iris 412 INCHANGÉS**, date absente des listes David/Iris ; une date `both` compte toujours pour David ET Iris mais **PAS** pour Marius ; Marius présent dans « 👥 Mes artistes » et dans les 3 toggles ; onglets + modale OK en 375. Artiste test retiré de `state`, rien persisté (save neutralisé), page rechargée en fin.
- **Non fait** (volontaire) : suppression d'artiste (optionnelle, non essentielle) — à ajouter plus tard avec garde-fou « aucune date + confirmation forte ». Cloisonnement réel = **Phase 6**.

## 9-nonies. Page PROMO autonome `presentation.html` (2026-07-14)
> Page vitrine/marketing autoportante (tout inline, thème sombre, palette de l'app), 2 publics : artistes + structures. NE touche PAS `index.html`.
- Fichier `presentation.html` à la racine → en ligne sur `https://david-lesage.github.io/intermittence-david-iris/presentation.html`.
- Sections : Hero (logo dégradé + accroche + boutons « Ouvrir l'app » → `index.html` / « Voir les fonctionnalités » → ancre), Le problème (chips 507 h/DPAE/GUSO/factures/France Travail), Pour les artistes (👤), Pour les structures (🛠️), Fonctionnalités clés (grille 8 tuiles), Comment ça marche (3 étapes), CTA + « créé avec amour ❤️ par David Lesage » + footer.
- Testé desktop + mobile 375 : aucun débordement horizontal, liens OK, rendu propre.

## 9-decies. Carte du monde des dates + km parcourus (2026-07-15)
> Vue carte plein écran (Leaflet + OpenStreetMap), géocodage AUTO via Nominatim, kilomètres parcourus depuis le domicile de l'artiste. Tout dans `index.html`.
- **Bouton** « 🗺️ Carte » dans `#hdrActions` (à côté de 👤 Profil) → `openMap()`.
- **Modale** `#mapBg` (`.map-modal`, style `.modal-bg`/`recap-modal`, croix `.bo-x` → `closeMap()`). Contenu : sélecteur de plage `#mapRange`, panneau km `#mapKm`, barre de progression `#mapProgress`, carte `#mapCanvas`, message adresse-vide `#mapNoAddr`, liste « non localisés » `#mapUnloc`.
- **Leaflet paresseux** : `loadLeaflet()` injecte CSS+JS depuis `unpkg.com/leaflet@1.9.4` à la 1re ouverture seulement (Promise mémoïsée `_mapLeaflet`). En cas d'échec réseau → message clair « Impossible de charger la carte (connexion internet ?) ». Tuiles OSM standard.
- **Géocodage Nominatim** : `_geocode(q)` (fetch `nominatim.openstreetmap.org/search?format=json&limit=1&q=`). `_placeQuery(f)` = `f.place` sinon `f.lieu`, + « , France » si aucun pays détecté. Boucle SÉQUENTIELLE avec `_sleep(1100)` entre requêtes (respect politique ~1 req/s). **Cache** : `f.geo={lat,lng,q}` sur la fiche (re-géocodé seulement si le lieu a changé ; les échecs sont cachés `{lat:null,fail:true}` pour ne pas re-tenter) + `state.people[p].profile.homeGeo={lat,lng,q}` pour le domicile. `save()` après géocodage → persistance du cache.
- **Domicile** : `_homeAddress(p)` lit `profile.home_address` OU `profile.adresse`. **Adresse vide** → carte masquée + message « 📍 Renseigne ton adresse dans ton profil… » + bouton « 👤 Ouvrir le profil » (`openProfile`).
- **Marqueurs** : 🏠 domicile (divIcon emoji) + un pastille colorée par date (`ficheCat` ; `var(--hypo)` violet pour les possibles/hypo). Popup = titre + date (`fmt`) + ville + cachets + statut possible + km (aller / A/R). `fitBounds` englobe tout (maxZoom 11).
- **Km** : `haversineKm(a,b)` (formule de haversine locale, R=6371 km). Panneau « ≈ X km parcourus (aller-retour) » = somme des allers ×2 + nombre de dates localisées / non localisées. Km par date dans le popup.
- **Plage de dates** : boutons « 📆 Période en cours » (`currentPeriod`) / « 🗓️ Année civile » / « ∞ Tout » (`setMapRange` → `_refreshMap`, `_fichesInRange`).
- **CSS** : bloc « Carte des dates » dans le `<style>` principal (`.map-canvas` hauteur 52vh, `.map-pin`, `.map-home-icon`, `.map-km`, `.map-progress`, `.map-unloc-d`…).
- **Testé** (port 8000, `pushRemote` neutralisé pendant le test puis rétabli, espace artiste, David) avec RÉSEAU EXTERNE FONCTIONNEL : bouton ouvre la vue ; Leaflet + tuiles OSM chargés ; géocodage Nominatim réel (barre de progression 4/18 → …) ; **7 dates localisées, 11 non localisées, ≈ 3 050 km (A/R)** ; marqueur 🏠 à Paris + pastilles colorées ; `fitBounds` OK ; popup vérifié (titre/date/ville/km) ; cache persisté (`f.geo` sur 18 fiches + `homeGeo`) ; haversine validé (Paris→Lyon = 391 km) ; filtres période/année/tout comptent 18/17/38 ; **cas adresse vide** → message + bouton profil, canvas masqué ; croix ferme ; **375 px sans débordement horizontal** ; aucune erreur console. (Les 11 « non localisés » = lieux étrangers/vagues type « E-Motion Ibiza », « …Hongrie » → géocodage échoue proprement et sont listés.)

## 9-undecies. Carte v2 — lieux à renseigner + lieux supposés + relier chronologiquement (2026-07-15)
> 3 améliorations de la carte demandées par David, tout dans `index.html`. La liste passive « non localisés » (`.map-unloc-d`) est remplacée par 2 panneaux ACTIFS + 1 toggle.
- **1) 📍 Lieux à renseigner** (panneau `#mapTodo`, classe `.map-todo`) : chaque date NON localisée (géocodage échoué / pas de lieu) est listée avec un **mini-champ « ville / lieu » + bouton ✓** (Entrée aussi). Saisie → `_mapQuickGeo(id,val)` géocode, pose le marqueur, cache `f.geo={lat,lng,q,manual:true,mq}` et `save()` + `_refreshMap()`. `toast` de succès/échec. Un `f.geo.manual` n'est **jamais re-géocodé** (protégé dans le filtre `todo`).
- **2) ⚠️ Lieux supposés — à confirmer** (panneau `#mapGuess`, classe `.map-guess`) : quand `f.place` est vide OU échoue, on **tente un géocodage sur le TITRE `f.lieu`** (sans « , France » forcé, pour trouver l'étranger type Ibiza/Budapest). Si trouvé → `f.geo.guessed=true` + `mq` = requête utilisée. Marqueurs supposés **visuellement distincts** (`.map-pin.guess` : contour pointillé, opacité réduite, « ? » en surimpression + « ⚠️ supposé » dans le popup). Chaque supposé a **« ✓ Confirmer »** (`_mapConfirmGuess` → `guessed:false, confirmed:true`, marqueur plein, persiste) et **« ✏️ Corriger »** (`_mapCorrectGuess` déplie un mini-champ ville → `_mapQuickGeo`, re-géocode en manuel). Le panneau km indique « (dont N supposée(s) ≈ X km A/R) ».
- **3) 🔗 Relier les dates (chronologique)** (toggle `#mapOpts`, `toggleMapConnect` / `_mapConnect`) : trace une **polyline Leaflet** reliant les marqueurs triés par `f.start`, encadrée par le 🏠 domicile aux deux extrémités (visualise la « tournée » + cohérent avec le km A/R). Style sobre : couleur `--accent2` (résolue via `getComputedStyle`), `weight:2.5`, `opacity:.55`. Désactivable ; la polyline est redessinée à chaque `_refreshMap` (layer nettoyé).
- **Cache/clé** : `f.geo.q` reste `_placeQuery(f)` comme clé stable (pas de re-géocodage inutile) ; `mq` = requête réellement trouvée (affichage) ; `guessed`/`manual`/`confirmed` = flags de statut. Politique Nominatim respectée (`_sleep(1100)`, rien de déjà caché n'est re-tenté).
- **Testé** (port 8000, `pushRemote` neutralisé, espace artiste, David, réseau externe OK) avec 3 fiches de test injectées puis PURGÉES : « Festival Budapest Hongrie » (place vide) → **marqueur supposé** à Budapest (47.51,19.05) + panneau « à confirmer » ; **« ✓ Confirmer »** → `guessed:false,confirmed:true`, panneau vidé ✅ ; « E-Motion Ibiza » (échec titre) → panneau **« à renseigner »**, saisie « Ibiza, Espagne » → `_mapQuickGeo` place le marqueur (38.97,1.42), `manual:true`, retiré du panneau ✅ ; « Soirée privée XYZ » (place+titre échouent) → reste « à renseigner » ✅ ; **toggle 🔗** → 1 polyline 🏠 Paris → Ibiza → Budapest → 🏠 (ordre chrono 10 août → 20 août) VUE à l'écran ✅ ; panneau km « ≈ 4 686 km (A/R) · 2 dates localisées · 1 à renseigner » ; **375 px sans débordement** ; données de test supprimées (retour à 64 fiches, `pushRemote` neutralisé → cloud NON pollué).

## 9-duodecies. Module « 🎪 Préparation de tournée » (1ère brique) (2026-07-15)
> Nouvel outil pour développer/relancer, tout dans `index.html`. Bouton **« 🎪 Tournée »** dans `#hdrActions` → modale `#tourBg` (plein écran mobile, croix) avec `renderPersonToggle('tourToggle',...,'openTour')` + 3 onglets internes (`.tour-tab` / `tourSetTab`). `openTour(person)` / `closeTour`. `_tourPerson`, `_tourTab`, `_tourEditId`, `_tourFilterYear`, `_tourSearch`, `_tourMailContactKey`, `_tourMailTpl`.
- **A) 📇 Carnet de contacts** (`renderTourContacts`) : `tourContacts(p)` **agrège les `f.orga` des dates de l'artiste** (via `belongsTo`) **+ contacts manuels** `state.people[p].contacts=[{id,structure,contact,email,telephone,notes,manual:true}]`, dédoublonnés par `_tourKey(structure,email)` (clé = structure lowercased, sinon email). Un contact manuel voit ses champs vides complétés par l'orga. Chaque carte : structure, référent, email, tél, notes, badge « auto »/« ajouté », + **historique** `tourHistory(p,c)` (dates confirmées matchant la clé, récent→ancien) avec **dernière fois** en évidence (⭐, clic → `openDetail`). Actions : **✉️ Relancer** (`tourRelance(key)` → onglet Mails, template relance, contact présélectionné), **✏️ Éditer** (`tourEditContact`/`tourContactForm`/`tourSaveContact`), **🗑️ Retirer** (manuels uniquement, `tourRemoveContact`). `save()` à chaque ajout/édition.
- **B) 📜 Concerts passés** (`renderTourPast`) : dates PASSÉES confirmées de l'artiste (`start < today0()`, `!hypo`), récent→ancien. Filtre **année** (`_tourFilterYear`) + **recherche** lieu/place/organisateur (`_tourSearch`). Ligne `.tour-pastrow` : date, lieu/titre, ville, cachets, 🏢 organisateur. Clic → `closeTour();openDetail(f.id)`.
- **C) ✉️ Mails types** (`renderTourMails`, `TOUR_TEMPLATES`) : 4 modèles FR (**relance** / **présentation** / **festival** / **remerciement**). Sélecteur de modèle + sélecteur de contact (liste du carnet) → `tourFillMail()` pré-remplit objet + corps avec **artiste** (`fullName(p)`), **contact/structure**, et pour la relance/remerciement la **dernière date jouée ensemble** (« On a joué ensemble à {lieu} le {date}… », `tourLastGig`). Objet + corps **éditables** avant copie. **📋 Copier le mail** (`tourCopyMail`, pattern `copyDateLink` : `navigator.clipboard.writeText` + fallback `prompt` + toast).
- **CSS** : `.tour-tabs/.tour-tab`, `.tour-card` (badge `.tc-tag`/`.tc-tag.man`, `.tc-meta`), `.tour-hist`, `.tc-actions`, `.tour-search`, `.tour-pastrow`, `.tour-mailbox` — style sombre réutilisant `--panel`/`--line`/`--accent2`.
- **Testé** (port 8000, `pushRemote`+`save` neutralisés, espace artiste, David = 38 dates dont 1 avec `f.orga` « HARMONIE » / Cédric Moulié) : bouton ouvre la modale ✅ ; Contacts agrège HARMONIE (badge « auto », historique 1 date, ⭐ dernière fois « Studio — Album Marina — 17 août 2026 ») ✅ ; **ajout manuel** « Festival Les Nuits Claires / Marie Dupont » → 2 contacts, 1 stocké dans `contacts` ✅ ; Concerts passés = 35 lignes, années 2026/2025/2024, filtre « paris » → 9, année 2025 → 15 ✅ ; Mails → « Relancer » sur HARMONIE génère l'objet « On se reprogramme une date ? — David Lesage » + corps pré-rempli avec « On a joué ensemble à Studio — Album Marina le 17 août 2026 » ; **📋 Copier** met bien objet+corps au presse-papier (intercepté) ✅ ; **375 px sans débordement** sur les 3 onglets ; aucune erreur console ; page **rechargée** en fin de test (données de test purgées, cloud NON pollué).
