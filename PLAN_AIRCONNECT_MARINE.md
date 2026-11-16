# Banc Marine — partie AirConnect : plan d'implémentation (validé)

> Plan validé avec l'utilisateur le 16/11 (chat téléphone), en continuité de la session
> « Plan d'action banc Marine Rconnect » et du fichier local `docs/PLAN_RCONNECT_2026-11-16.md`.
> **Ce document fait foi en cas de différence** avec ce fichier (décisions plus récentes).
> Projet : `hvac-marine-bench` — flow `flow/bancMarine.json`. Travailler sur la branche de dev habituelle.
> Règle générale : **ajouter sans casser**. Après chaque lot, vérifier que tout le reste fonctionne encore.

---

## 0. Principes (décisions prises)

1. **Deux mondes seulement** sur le Marine, décidés par `machine_type` :
   - `CREW` → CREW (déjà en service)
   - `SMART`, `PAX`, `AUREA` → **AirConnect** (automate Loytec IO-591, OPC-UA)
   - **L'AirSmart n'existe pas sur le Marine.** `SMART` est un *nom de type de machine*, pas l'AirSmart.
   - Un seul test partout : `mt === 'CREW'` → branche CREW, sinon → branche AirConnect. Jamais de 3ᵉ branche.
2. **Cases génériques `has_*`** : une case dit seulement « ce test se fait / ne se fait pas ».
   Elle ne dit jamais si c'est du CREW ou de l'AirConnect : c'est le **type de machine** qui le dit.
   → plus aucune colonne `crew_has_*` / `rc_has_*`.
3. **Un seul questionnaire** pour tous les types (CREW compris), filtré par **type puis famille**.
4. **Une seule source de vérité** : le formulaire (cases affichées) et l'exécution (`_naConditions`)
   lisent la **même** config. Plus aucune condition sur un nom de type en dur dans `_naConditions`.
5. Pas de liste d'exclusions : les filtres excluent déjà, et l'opérateur décoche le reste.

---

## Lot 1 — Formulaire (onglet *Formulaire*)

### 1.1 Nouveau nœud function « Config questionnaire »
Familles réelles dans le code : `CTIA`, `CVIA`, `CPIA` (`marine.familles`).
**La famille est rangée sous chaque type** (CTIA pour CREW ≠ CTIA pour AUREA) :

```js
const config = {
  CREW: {
    tests: ["has_res_therm_secu", "has_moteur_vanne", "has_motovC", "has_lon", "has_dali"],
    retirer: { CTIA: [], CVIA: [], CPIA: [] }   // l'utilisateur remplira lui-même (ex. résistance)
  },
  PAX:   { tests: [/* TOUS les tests */], retirer: { CTIA: [], CVIA: [], CPIA: [] } },
  AUREA: { tests: [/* TOUS les tests */], retirer: { CTIA: [], CVIA: [], CPIA: [] } },
  SMART: { tests: [/* TOUS les tests */], retirer: { CTIA: [], CVIA: [], CPIA: [] } },
};
```

- Type seul sélectionné → on affiche `tests` (on n'attend pas la famille).
- Type + famille → `tests` **moins** `retirer[famille]`.
- Listes de départ : CREW = sa liste actuelle, rien de retiré. PAX / AUREA / SMART = tous les tests, rien de retiré.
  L'utilisateur affinera les listes à la main ensuite → **le format doit rester simple et lisible.**
- Déclenché par un inject « au démarrage » et envoyé au template (un ui_template ne lit pas le contexte
  du flow directement) **et** stocké en contexte global pour `Init Sequence`.

### 1.2 Template `Reference + Load Article`
- `smartQ` + `crewQ` → **un seul** questionnaire (ex. `questionnaireQ`). Simple liste JS, aucune migration due au renommage.
- `ng-repeat` sur `marine.testsVisibles()` (double filtre ci-dessus).
- **Une case qui disparaît (changement de type/famille) est décochée automatiquement.**
- Les 2 blocs `ng-if="marine.machine_type === 'SMART'"` (Composition + Configuration réseau ETH)
  → « type renseigné et ≠ CREW ».
- Supprimer le bloc PAX/AUREA « Questionnaire {{machine_type}} — à définir ».
- Ajouter les cases des **10 tests récupérés du Terrestre** (lot 7) et les cases génériques `has_lon`, `has_dali`.
- Ajouter toutes les nouvelles clés dans `_allBoolKeys` (sinon pas enregistrées).

### 1.3 Champ « Nombre de ventilateurs »
- Champ numérique avec flèches haut/bas, défaut **1** → colonne existante `nb_motovP`
  (aujourd'hui seulement alimentée par l'import Excel, jamais demandée au formulaire).
- Maximum : **à définir plus tard** (pas d'info pour l'instant).

### 1.4 Colonnes génériques + migration CREW
| ancienne colonne | nouvelle colonne générique |
|---|---|
| `crew_has_electric_heater` | `has_res_therm_secu` |
| `crew_has_valve_actuator` | `has_moteur_vanne` |
| `crew_has_fan` | `has_motovC` |
| `crew_has_lon` | `has_lon` (nouvelle) |
| `crew_has_dali` | `has_dali` |

- Écrire le script SQL de migration des articles existants (peu d'articles, 4 CREW en base de dev).
- ⚠ **Sur le banc de production, la migration sera faite avec Claude, prudemment** : prévoir le script, ne pas l'exécuter seul en prod.
- Mettre à jour tous les lecteurs des anciennes clés : `Guard TEST_RES` (`has_res_therm_secu || crew_has_electric_heater`),
  `ready_si` de `Test Def` (`crew_has_valve_actuator`), Resolve Fan/Res/Vanne Target, etc. → grep `crew_has_`.
- Nouvelles colonnes dans `marine_articles` + `Upsert marine_articles`, `Fill Form Fields`, `Valid Form`,
  `Apply Test Config`, `Build Expected Columns`, import Excel (`Normaliser et valider`, `Écrire l'article`).

---

## Lot 2 — Gestionnaire (onglet *SEQUENCEUR_TESTS*)

- `Init Sequence` / `_naConditions` : un test est NA si sa case n'est pas cochée **ou** s'il n'est pas dans
  la liste visible (type + famille) de la config du lot 1.
  → supprimer `TEST_ETH/FW/BCK: _mt !== 'SMART'`, `TEST_MENAGE/TEST_DND: _mt !== 'AUREA'`, `TEST_LON: !_m.crew_has_lon`.
- Supprimer la ligne fantôme `TEST_LPAD` (`_naConditions` + `lpad` dans `Init Test Params`) : aucun test derrière.
- `Détection État Automate` : `mt !== 'SMART'` → `mt === 'CREW'` (aujourd'hui PAX/AUREA ne sont pas détectés).
- `Test Def` : déclarer les nouveaux tests (`order`, `timeout_ms`) ; corriger le commentaire périmé « rSMART = IO-591 ».
- `TEST_DISCO` : **pas de case**, toujours actif en AirConnect, jamais en CREW.

## Lot 3 — Socle séquenceur
- `rejouer_avant` dans `Test Def` + reprise élargie dans `Action RETESTER`.
- `PROLONGER_TIMEOUT` dans `Sequence Engine`.
- Clés `Init Test Params` : V6V, DISCO, FW, BCK, ETH, CLEAR, VPN, BACNET.
- `Init Test Params` : sorties **FAN1, FAN2, FAN3** définies en dur (le Marine n'avait que FAN1).

## Lot 4 — Gestion du 230 V
| | CREW | AirConnect |
|---|---|---|
| coupure sur KO | automatique | interdite |
| bouton « Couper 230 V » (menu KO) | non | oui |
| après Réessayer | DO4 + IO6 | DO4 puis attente automate (ping) |

- Porter du Terrestre les 17 nœuds du groupe KO (`coupe230_*`, `reprise_*`, boucle d'attente, `li_reprise_auto`)
  + les 5 nœuds `Reprise COM` (`ft_reco_*`) + `diag_fortest.py`.
- `reprise_ping_timeout_ms: 180000` (**3 min**, clé à part), surveillance toutes les 5 s jusqu'au timeout.
- Ouvrir `relais230_guard` à l'AirConnect.
- ⛔ Ne pas toucher `ko_do4_off_fn` ni `retry_do4_on_fn` (leurs gardes CREW sont des protections).

## Lot 5 — `TEST_DISCO` seul
Onglet (77 nœuds) + `scripts_LDISCO/` + `setup_automate.py` + `empreinte_automate.py`. Valider de bout en bout avant la suite.

## Lot 6 — Branches AirConnect des tests existants
Le `else` des comparaisons `=== 'CREW'` existantes **est** la branche AirConnect : il suffit de le remplir.

| test | à faire |
|---|---|
| `TEST_FORTEST` | rien (machine à part) |
| `TEST_VANNE` | l'IP (User Registers `Cooling_Valve_S`/`_M`) |
| `TEST_RES` | l'IP + paire `Electric_Heater_S`/`_M` + `sonde2` |
| `TEST_FAN` | IP + **tr/min** + **N instances en parallèle** selon `nb_motovP` (FAN1..FANn des `Init Test Params`) |
| `TEST_RELAIS_230V` | 2 vraies branches : relais (CREW) / ping (AirConnect) |
| V6V | ajouter `V6V_1`→`FORTEST_1` et `V6V_2`→`FORTEST_2` à côté du ForTest |

⛔ Ne **pas** porter les 27 nœuds `rSMART`/Modbus du Terrestre (10 FAN, 11 RES, 6 VANNE).
Règle : on ne duplique pas un gros programme pour changer une IP.

## Lot 7 — Les 10 onglets du Terrestre
`TEST_FW` · `TEST_BCK` · `TEST_ETH` · `TEST_SONDE_AIR` · `TEST_SONDE_EAU` · `TEST_V6V` · `TEST_CLEAR` · `TEST_VPN` · `TEST_BACNET` · `TEST_IP`
(288 nœuds), un onglet à la fois, + les 9 scripts manquants + les **4 correctifs du 16/11** portés avec le V6V.
- `send_backup.py` / `send_firmware.py` : prendre la version **Terrestre**.
- Pas de serveur DHCP ; scripts réseau seulement ceux utiles à l'onglet admin.
- Colonnes `ip_finale_*` à créer en base Marine (calées sur le Terrestre).

## Lot 8 — Onglet ADMINISTRATION
Porter l'onglet (29 nœuds + `dashboard/page_admin.html`) : `TEST_FAN1`/`FAN2` → `TEST_FAN`, ajouter `TEST_RES`
et les tests Marine, grisage lu sur le questionnaire.

### 8.1 `TEST_RES` en admin
- « Commencer » sur la bulle RES : **d'abord ventilos ON** (étape propre à l'onglet admin, mêmes sorties que
  `Init Test Params`), ventilos **en continu** pendant tout le test, puis lancement de `TEST_RES` **inchangé**.
- Pas d'obligation de lancer FAN avant ; le test FAN reste indépendant (il s'arrête à son résultat).
- À vérifier : reprendre le même mécanisme qui fait tourner les ventilos pendant RES en séquence (FAN → RES).
- **Sécurité résistance** : réutiliser la surveillance + le pop-up de refroidissement forcé + la barre de
  température existants. ⚠ Le bouton du pop-up arrête la séquence : en admin il n'y a pas de séquence →
  il doit fonctionner sans (refroidissement puis retour à l'état normal).

### 8.2 Resets séparés (admin uniquement)
- Le nœud `Reset résistance + ventilos + vanne` **reste inchangé** pour Home / séquence.
- En admin, 3 resets séparés : `Admin – Reset résistance`, `Admin – Reset ventilos`, `Admin – Reset vanne`.
- Chaque test ne reset que ce qu'il utilise : RES → résistance **puis** ventilos · FAN → ventilos · VANNE → vanne.
- Avant de couper les ventilos : ne pas couper si un autre test admin qui en a besoin (FAN ou RES) est en cours.
- La résistance est toujours coupée en premier.

---

## Mis de côté (plus tard)
- Nombre maximum de ventilateurs.
- Choix de la sortie de chaque ventilateur dans le formulaire (pour l'instant FAN1..FAN3 en dur).
- Contenu réel des listes type/famille : rempli à la main par l'utilisateur.
- Terrestre : choix « tr/min ou booléen » par article.

## Tests à faire après implémentation
- 4 types × 3 familles + « type sans famille » : bonnes cases affichées.
- Changer de famille après avoir coché → cases cachées décochées.
- Charger un article existant (avant et après migration).
- Une séquence CREW complète : **rien ne doit avoir changé**.
- Une séquence AirConnect (PAX, AUREA, SMART).
- Admin : RES seul, RES + VANNE en parallèle, RES + FAN en parallèle.
