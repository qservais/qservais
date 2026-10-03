# Outil de gestion budgétaire personnel — Design

- **Date :** 2026-10-03
- **Statut :** validé en brainstorming, en attente de relecture de la spec
- **Révision 2 (2026-10-03) :** import des **relevés PDF** (pas de CSV
  disponible), carte de crédit Mastercard ajoutée comme 3ᵉ compte, règles
  avec condition de montant — après analyse de relevés réels
- **Point de départ :** classeur « v2 » (plan prévisionnel) livré le 2026-10-03

> Ce document ne contient volontairement aucune donnée financière personnelle
> (montants, noms, IBAN). Les chiffres réels vivent dans un fichier de données
> exclu de Git (voir § 9).

---

## 1. Contexte

La v2 est un **plan prévisionnel** fiable : onglets `Paramètres`, `Lignes`
(postes récurrents avec début/fin/changement de prix), `Ponctuels`
(montants exceptionnels), `Plan` (27 mois calculés), `Tableau de bord`,
`Analyse`, `Guide`. Elle ne connaît rien du **réel** : au bout de quelques
mois, les soldes affichés divergent du compte, et « Vie quotidienne » reste
une boîte noire.

## 2. Objectif

Transformer le plan en **outil de gestion** utilisé chaque semaine/mois :

1. suivre **chaque dépense réelle**, par ligne du budget, sur trois comptes
   (Belfius compte à vue = compte salaire, Belfius Mastercard = carte de
   crédit, Revolut = 2ᵉ compte perso) + espèces ;
2. comparer **réel vs prévu** et savoir **combien il reste à dépenser** ;
3. **recaler** le plan sur le solde réel à chaque clôture mensuelle ;
4. suivre des **objectifs** (fonds d'urgence, projets datés) ;
5. suivre le **patrimoine** (valeur nette) mois après mois.

### Décisions prises avec l'utilisateur

| Sujet | Décision |
|---|---|
| Usage | Suivi détaillé (chaque transaction) |
| Saisie | Import des **relevés PDF** **et** saisie mobile |
| Plateforme | Google Sheets |
| Comptes | Belfius compte à vue, Belfius Mastercard (crédit), Revolut (2ᵉ compte **perso**), Espèces |
| Format d'import | **PDF** des relevés officiels (aucun export CSV accessible) ; contrôle par les soldes |
| Long terme | Objectifs/projets + Patrimoine |
| Approche | **B** — formules visibles + Apps Script pour l'import et la clôture |
| Catégories | **Un seul niveau** : une transaction est rangée dans une ligne du budget |
| Recalage | Le plan repart du dernier solde réel clôturé |
| Mois | **Mois civil (1 → 31)**, salaire compté le mois où il est reçu |
| Mobile | Fusion automatique saisie mobile ↔ ligne bancaire |
| Code | Versionné dans `qservais/qservais`, données perso hors Git |

### Hors périmètre (YAGNI)

- Scénarios « et si… » (copie de fichier suffit pour l'instant).
- Horizon glissant automatique (le plan garde une fenêtre fixe, réglable
  via « Premier mois du plan »).
- Sous-catégories à deux niveaux.
- Appli web dédiée, synchronisation bancaire automatique (PSD2/API).
- Création de règles depuis le Journal en un clic.
- Import CSV (aucun export CSV accessible à l'utilisateur ; à ajouter plus
  tard comme second format si besoin).

## 3. Architecture des onglets

| Onglet | Rôle | Remplissage |
|---|---|---|
| `Tableau de bord` | Mois en cours (réel vs prévu, reste à dépenser), **trésorerie nette** (comptes − encours carte), alertes, objectifs, valeur nette | Formules (+ sélecteur de mois) |
| `Plan` | Projection mensuelle, recalée sur le réel | Formules |
| `Réel vs Prévu` | Par mois × ligne : prévu, réel, écart, % consommé, reste ; historique 12 mois | Formules |
| `Journal` | Toutes les transactions réelles | Script (import, formulaire) + corrections manuelles |
| `Règles` | Mots-clés → ligne du budget ; motifs de virements internes | Utilisateur |
| `Objectifs` | Fonds d'urgence et projets | Utilisateur (cible, date, poche) + formules |
| `Patrimoine` | Bloc « État actuel » + historique des clôtures | Utilisateur + script de clôture |
| `Lignes` | Postes du budget (existant) | Utilisateur |
| `Ponctuels` | Montants exceptionnels prévus (existant) | Utilisateur |
| `Paramètres` | Réglages (existant + nouveaux) | Utilisateur |
| `Réponses formulaire` | Arrivée brute du Google Form | Google Forms |
| `Guide` | Installation + routine mensuelle | Statique |

L'onglet `Analyse` de la v2 est retiré (constat ponctuel, déjà traité).

## 4. Modèle de données

### 4.1 `Lignes` (inchangé + compléments)

Colonnes v2 conservées : Section (Revenu/Dépense/Épargne), Libellé (unique),
Catégorie, Montant, Fréquence (Mensuel/Annuel), Mois, Début, Fin, Nouveau
montant, À partir de, Indexé, Note, Contrôle.

Changements :
- capacité du `Plan` relevée de 10/32/6 à **20 revenus / 60 dépenses /
  15 épargnes** (constantes du générateur, modifiables) ;
- le menu « Ligne du budget » du Journal et des Règles propose, en plus des
  libellés de `Lignes`, deux valeurs techniques absentes de `Lignes` :
  `À classer` et `Virement interne` (exclues des totaux).

### 4.2 `Journal`

| Col | Champ | Détail |
|---|---|---|
| A | Date | date de l'opération (comptabilisation) |
| B | Mois | `=DATE(YEAR(A),MONTH(A),1)` — mois civil |
| C | Compte | `Belfius` / `Mastercard` / `Revolut` / `Espèces` |
| D | Libellé | libellé bancaire normalisé (ou note mobile) |
| E | Montant | signé : **négatif = sortie**, positif = entrée |
| F | Ligne du budget | menu déroulant sur `Lignes` (+ lignes techniques) |
| G | Source | `Import` / `Mobile` |
| H | Statut | `OK` / `Provisoire` (saisie mobile non encore rapprochée) |
| I | Note | libre |
| J | Empreinte | clé de dédoublonnage (écrite par le script, masquée) |

Volume attendu : ~150 lignes/mois → ~2 000/an ; formules en `SUMIFS` sur
colonnes ouvertes.

### 4.3 `Règles`

| Col | Champ | Détail |
|---|---|---|
| A | Priorité | entier ; la plus petite gagne |
| B | Compte | vide = tous |
| C | Le libellé contient | texte, insensible à la casse et aux accents |
| D | Sens | vide / `Sortie` / `Entrée` |
| E | Montant min | vide = pas de borne (valeur absolue) |
| F | Montant max | vide = pas de borne (valeur absolue) |
| G | Ligne du budget | menu ; peut être `Virement interne` |

La condition de montant est nécessaire : certains flux récurrents se
reconnaissent à leur montant plus qu'à leur libellé (ex. de nombreux petits
virements entrants de même montant sans mot-clé constant).

Exemples livrés (génériques) : supermarchés → Courses, stations-service →
Carburant, virement vers son propre IBAN → Virement interne, retrait
distributeur → Virement interne, prélèvement « RELEVE MASTERCARD » sur le
compte à vue → Virement interne (remboursement de la carte), recharge
Revolut payée par carte → Virement interne.

### 4.4 `Patrimoine`

- **État actuel** (jaune) : Poste | Type | Valeur. Types : `Liquidités`,
  `Épargne`, `Investissement`, `Immobilier`, `Dette` (valeur saisie positive,
  comptée en négatif). Postes par défaut : Belfius, Revolut, Espèces, Mastercard (encours, en Dette),
  Épargne SOS, ETF, Crypto, Maison (valeur estimée), Prêt maison (capital
  restant), Prêt perso (restant), Autres dettes.
- **Historique** (rempli par la clôture) : Mois | Poste | Type | Valeur.

### 4.5 `Objectifs`

Objectif | Type (`Fonds d'urgence` / `Projet`) | Montant cible | Date cible |
Poche liée (poste du Patrimoine) | Actuel | Manque | Mois restants |
Versement mensuel nécessaire | Versement prévu (ligne Épargne de même nom
dans le Plan, mois en cours) | Statut | Progression (barre texte `REPT("█")`).

### 4.6 `Paramètres` (nouveaux)

- Mois de couverture du fonds d'urgence (défaut 6) ;
- Seuil d'alerte « % consommé » (défaut 80 %) ;
- Fenêtre de rapprochement mobile ↔ banque (défaut ± 3 jours) ;
- Comptes : nom, type (`Compte à vue` / `Carte de crédit` / `Espèces`),
  IBAN ou 4 derniers chiffres de carte (servent à reconnaître les virements
  internes). Ces identifiants vivent dans le classeur de l'utilisateur,
  jamais dans le dépôt.

## 5. Flux

### 5.1 Import d'un relevé PDF — `Budget → Importer un relevé`

1. Dialogue HTML avec sélecteur de fichier **PDF** (un ou plusieurs
   fichiers ; fait sur PC ou téléphone).
2. **Extraction du texte dans le navigateur** avec pdf.js (chargé depuis
   cdnjs dans le dialogue) : chaque page → lignes reconstituées à partir des
   positions (y puis x) des fragments de texte, colonnes conservées grâce
   à l'abscisse. Seul ce texte structuré est envoyé au script ; le PDF n'est
   pas stocké.
3. **Détection du type de relevé** par des marqueurs d'en-tête :
   - *Belfius compte à vue* : lignes d'opération « NNNN JJ-MM-AAAA (VAL.
     JJ-MM-AAAA) ±montant » suivies de lignes de description ; solde
     d'ouverture « SOLDE AU … » et solde de clôture en fin de relevé ;
     en-têtes/pieds de page répétés à ignorer ;
   - *Belfius Mastercard* (« ÉTAT DES DÉPENSES ») : en-tête avec date de
     clôture, date de débit, période « Transactions du … au … » ; lignes
     « JJ/MM JJ/MM DESCRIPTION LIEU PAYS montant EUR ± » (année déduite de
     la période ; `+` = remboursement/crédit), ligne de détail facultative
     « (Via Apple Pay) » ; ligne « Total … EUR − » ; en-têtes répétés à
     chaque page ;
   - *Revolut* : tableau « Date de valeur / Date de réception / Description
     / Argent sortant / Argent entrant / Solde », dates « 7 sept. 2026 »,
     montants « 9,99€ », lignes « ID de transaction », « De : », « À : »
     rattachées à l'opération ; le sens se déduit de la colonne (abscisse)
     et est confirmé par la variation du solde.
   Type inconnu → message explicite, rien n'est écrit.
4. **Normalisation** : date, compte, libellé (texte de l'opération, sans
   références techniques), montant signé (négatif = sortie), identifiant
   bancaire si présent (n° d'opération Belfius, ID Revolut).
5. **Contrôle des montants (garde-fou principal)**, refus de l'import avec
   l'écart constaté s'il échoue :
   - compte à vue et Revolut : solde d'ouverture + somme des opérations =
     solde de clôture, **au centime** ;
   - Mastercard (pas de solde sur le relevé) : somme des opérations =
     ligne « Total », au centime.
   Ce contrôle détecte toute opération manquée ou mal lue.
   **Contrôle croisé** (avertissement, non bloquant) : le total d'un relevé
   Mastercard doit correspondre à un prélèvement « RELEVE MASTERCARD » du
   même montant sur le compte à vue autour de la date de débit.
6. **Empreinte** = identifiant bancaire s'il existe ; sinon compte | date |
   montant | libellé normalisé | rang d'occurrence. Toute empreinte déjà
   présente dans le Journal est ignorée → réimporter un relevé ou des
   relevés qui se chevauchent est sans danger.
7. **Rapprochement mobile** : pour chaque nouvelle ligne bancaire, si une
   ligne `Provisoire` existe avec même compte, même montant et date à
   ± N jours → la ligne bancaire reprend la ligne du budget et la note de la
   saisie mobile, la saisie provisoire est supprimée. Une saisie mobile ne
   peut être rapprochée qu'une fois (la plus proche en date).
8. **Catégorisation** via `Règles` (priorité croissante, premier match) ;
   sinon `À classer`.
9. **Écriture atomique** : toutes les lignes sont préparées en mémoire puis
   écrites en un seul appel ; en cas d'erreur, rien n'est écrit.
10. **Bilan** : « N lues · solde vérifié ✓ · N ajoutées · N doublons
    ignorés · N rapprochées avec le mobile · N à classer ». Le solde de
    clôture lu est proposé pour l'« État actuel » du Patrimoine.

**Carte de crédit et double comptage.** Les achats faits avec la
Mastercard sont importés depuis le relevé Mastercard (compte
`Mastercard`). Le prélèvement mensuel du relevé sur le compte à vue est un
`Virement interne`, de même qu'une recharge Revolut payée avec la carte et
les virements entre ses propres comptes. Ainsi chaque dépense est comptée
une seule fois, au moment de l'achat. L'encours de la carte apparaît comme
`Dette` dans le Patrimoine.

Les recharges Revolut payées par la carte apparaissent sous deux formes :
côté Mastercard « Revolut … (Via Apple Pay) », côté Revolut « Recharge sur
Apple Pay via *NNNN » où NNNN est le **numéro de compte de l'appareil**
Apple Pay, différent des 4 derniers chiffres imprimés sur la carte. Les
règles de virement interne reposent donc sur les libellés (« Revolut »,
« Recharge sur Apple Pay », virement vers/depuis son propre IBAN), pas sur
le numéro de carte.

Les trois formats ont été observés sur des relevés réels (structure
seulement). Les fixtures de test sont des extraits **anonymisés et
réécrits avec des valeurs fictives** (structure identique, aucune donnée
réelle).

### 5.2 Saisie mobile — Google Form

- Formulaire réservé aux **sorties** (dépenses et versements d'épargne) ;
  les revenus arrivent par l'import.
- Champs : Montant (positif), Ligne du budget (lignes Dépense + Épargne),
  Compte (Belfius / Mastercard / Revolut / Espèces), Note. Date = horodatage.
- Déclencheur `onFormSubmit` : ajoute une ligne au Journal, montant négatif,
  Source `Mobile`, Statut `Provisoire` (comptes bancaires) ou `OK` (Espèces).
- `Budget → Mettre à jour le formulaire` : resynchronise la liste des lignes
  du budget (Dépense + Épargne) dans le formulaire.

### 5.3 Clôture — `Budget → Clôturer le mois`

1. Demande le mois à clôturer (défaut : mois précédent). Les soldes de
   clôture lus lors des imports du mois pré-remplissent l'« État actuel »
   (modifiables).
2. Avertit (sans bloquer) s'il reste des lignes `À classer` ou `Provisoire`
   sur ce mois.
3. Si le mois existe déjà dans l'historique → confirmation avant écrasement.
4. Copie le bloc « État actuel » dans l'historique avec ce mois.

### 5.4 Installation — `Budget → Installer`

Crée le Google Form, le lie au classeur (onglet `Réponses formulaire`),
installe le déclencheur `onFormSubmit`, protège les onglets calculés
(avertissement seulement). Idempotent : relancer ne crée pas de doublon.

## 6. Calculs

### 6.1 Convention de mois

Mois civil partout. Dans `Lignes`, le salaire démarre au mois où il est
reçu (changement par rapport à la v2, qui le décalait d'un mois). L'écart de
la projection v2 → v3 est uniquement dû à ce décalage et doit être chiffré
lors de la vérification.

### 6.2 Réel par ligne et par mois

- Dépense : `= −SUMIFS(Journal!E, Journal!F, ligne, Journal!B, mois)`
- Revenu : `= SUMIFS(…)`
- Épargne : virements vers les poches d'épargne/investissement rangés sur la
  ligne Épargne correspondante (signe comme une dépense).
- `Virement interne` et `À classer` sont exclus des totaux ; `À classer`
  est compté à part (compteur + alerte).

### 6.3 `Réel vs Prévu`

Pour le mois choisi et chaque ligne : Prévu (cellule du `Plan`), Réel,
Écart, % consommé, Reste = Prévu − Réel. Mise en forme : orange ≥ seuil
d'alerte, rouge > 100 %. Bloc historique : réel des 12 derniers mois +
moyenne des 3 derniers mois clôturés à côté du prévu.

### 6.4 Recalage du `Plan`

Nouvelles lignes : « Solde réel (clôture) » = somme des postes de type
`Liquidités` de l'historique pour ce mois (vide si non clôturé) et
« Écart réel − prévu ».

`Début du mois m` = solde réel clôturé de m−1 s'il existe, sinon solde fin
prévu de m−1 (premier mois : solde de départ des Paramètres).

### 6.5 Point bas avant salaire

En mois civil, le solde fin de mois masque le creux avant le salaire.
Nouvelle ligne du `Plan` : « Point bas avant salaire » = Début du mois −
Total dépenses − Total épargne (revenus ignorés : estimation prudente).
Les alertes (`Juste`/`NÉGATIF`), le « point le plus bas » et la « dépense
extra possible » du tableau de bord utilisent cette ligne.

### 6.6 Objectifs

- Actuel = dernière valeur de la poche liée dans l'historique du Patrimoine.
- Fonds d'urgence : cible = N mois × moyenne des dépenses réelles des
  6 derniers mois clôturés (repli sur le prévu s'il y a moins de 3 mois
  clôturés).
- Versement nécessaire = Manque / Mois restants (garde-fou si 0 mois).
- Statut : `Atteint` si Actuel ≥ cible ; `À l'heure` si versement prévu ≥
  nécessaire ; sinon `En retard`.

### 6.7 Trésorerie nette

Trésorerie nette = Σ soldes `Liquidités` − encours de la carte de crédit
(dernière valeur connue : État actuel du Patrimoine). Affichée sur le
tableau de bord à côté du solde des comptes : un solde de compte élevé
financé par la carte n'apparaît plus comme de l'argent disponible.

### 6.8 Patrimoine

Valeur nette = Σ actifs − Σ dettes (dont l'encours de la carte de
crédit), par mois clôturé. Plus-value
ETF/crypto = valeur − total versé réel (cumul du Journal sur la ligne
Épargne correspondante). Graphique d'évolution sur le tableau de bord.

## 7. Apps Script

```
apps-script/
  appsscript.json
  Menu.gs           onOpen : menu Budget
  Install.gs        installation idempotente (Form, trigger, protections)
  ImportDialog.html sélecteur de PDF + extraction du texte avec pdf.js
  Import.gs         orchestration : lit le Journal/Règles, appelle core, écrit
  Close.gs          clôture mensuelle
  Form.gs           onFormSubmit, synchro de la liste du formulaire
  core.js           logique pure (aucun appel Google) :
                    detectStatement, parseBelfiusAccount,
                    parseBelfiusMastercard, parseRevolut, checkBalances,
                    fingerprint, matchRule, matchMobile, normalizeText
```

`core.js` est valide à la fois comme fichier Apps Script et comme module
Node (`if (typeof module !== 'undefined') module.exports = …`) pour être
testé hors Google.

## 8. Gestion des erreurs

| Situation | Comportement |
|---|---|
| PDF de type inconnu ou illisible (scan sans texte) | Message, aucun import |
| Somme des opérations ≠ écart des soldes | Import refusé, écart affiché |
| Erreur pendant l'import | Rien n'est écrit (écriture unique en fin de traitement) |
| Devise ≠ EUR | Ligne importée en `À classer`, devise en note |
| Mois déjà clôturé | Confirmation avant écrasement |
| `À classer`/`Provisoire` restants à la clôture | Avertissement, non bloquant |
| Ligne du Journal pointant vers une ligne inexistante | Signalée par la colonne Contrôle + compteur du tableau de bord |
| Saisie dans un onglet calculé | Avertissement de protection |
| Division par zéro (objectifs, % consommé) | `IFERROR` → 0 ou vide |

## 9. Dépôt, données et livraison

```
budget/
  generator/build_workbook.py   génère le .xlsx à partir d'un fichier de données
  data/exemple.json             valeurs fictives (versionné)
  data/budget.json              données réelles (.gitignore)
  apps-script/…                 § 7
  tests/
    core.test.js                tests Node (node:test) des fonctions pures
    fixtures/                   textes de relevés réécrits avec des valeurs fictives
    test_workbook.py            vérifications du classeur généré
  dist/                         .xlsx générés (.gitignore)
  README.md                     installation + routine mensuelle
```

Livrables à l'utilisateur : le `.xlsx` (à importer dans Google Sheets),
les fichiers Apps Script à coller, le guide pas-à-pas.

Formules : uniquement des fonctions compatibles Excel 2007 / LibreOffice
(SUMIFS, INDEX/MATCH, SUMPRODUCT, IFERROR, `_xlfn.MINIFS`) afin que le
classeur soit vérifiable automatiquement et reste utilisable dans Excel
(hors import/clôture).

## 10. Reprise de la v2

- `Lignes`, `Ponctuels`, `Paramètres` repris tels quels, sauf :
  - Salaire : début au mois de réception (§ 6.1) ;
  - « Vie quotidienne » découpée en Courses / Carburant / Restos & sorties /
    Shopping / Santé, même total qu'en v2 (répartition initiale indicative,
    à recalibrer après 2-3 mois de réel).
- `Transactions` (v2 d'origine) : ses entrées sont déjà dans `Ponctuels`.

## 11. Vérification et critères d'acceptation

1. Recalcul LibreOffice : **0 erreur de formule**.
2. Projection sans aucune clôture = projection v2, à l'écart près du
   décalage du salaire (§ 6.1), écart chiffré et expliqué.
3. `core.test.js` couvre : détection du type de relevé, parsing Belfius
   compte à vue, Mastercard et Revolut (fixtures : descriptions sur
   plusieurs lignes, en-têtes de page répétés, montants ≥ 1 000 avec
   séparateur de milliers), contrôle des soldes (cas juste et cas d'une
   opération manquante → refus), empreinte et dédoublonnage (réimport d'un
   relevé qui chevauche), contrôle du « Total » Mastercard et contrôle
   croisé avec le prélèvement, virements internes (remboursement
   Mastercard, recharge Revolut par carte, chaîne carte → Revolut →
   compte à vue), conditions de montant des règles, priorité
   des règles, rapprochement mobile (cas : match, hors fenêtre, deux
   candidats, espèces jamais rapprochées).
4. Validation sur les **vrais relevés de l'utilisateur, en local
   uniquement** (jamais versionnés) : solde vérifié au centime pour chacun.
5. Jeu d'essai : un mois fictif d'environ 100 transactions + une clôture →
   vérification manuelle de `Réel vs Prévu`, du recalage du `Plan`, des
   `Objectifs` et du `Patrimoine`.
6. Aucune donnée personnelle dans les fichiers versionnés.
