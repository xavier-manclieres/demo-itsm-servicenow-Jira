# Rapport Power BI

![Rapport sur données fictives](rapport%20Power%20BI%202026-10-10%20104811.png)

Le rapport lit la réponse de WF3 (`GET /webhook/kpi`). Pour le construire sans dépendre d'une instance ITSM, on part de `exemple-kpi.json` : une réponse de WF3 sur **données fictives** (10 candidats problèmes et 14 REX sur 8 semaines).

| Fichier | Rôle |
|---|---|
| `exemple-airtable.json` | Enregistrements Airtable fictifs, tels que n8n les lit |
| `exemple-kpi.json` | Réponse de WF3 sur ces enregistrements, produite par `npm run exemple:kpi` |
| `rapport-itsm.pbip` + dossiers `.Report` et `.SemanticModel` | Le rapport, en projet Power BI (fichiers texte) |

`exemple-kpi.json` n'est pas écrit à la main : le script exécute le vrai nœud « Calculer les indicateurs » de WF3. Un test échoue s'il n'est plus à jour, si bien que le rapport est toujours construit sur la forme exacte de la réponse réelle.

## 1. Charger les données (Power Query)

Dans Power BI Desktop : *Accueil › Obtenir des données › Requête vide*, puis *Accueil › Éditeur avancé*. Coller le code, valider, puis renommer la requête (clic droit › Renommer). Recommencer pour chacune des quatre requêtes.

**`KPI`** : la source. Clic droit › décocher *Activer le chargement*, car les trois autres requêtes en dépendent.

```powerquery
let
    // Construction : fichier d'exemple du dépôt (adapter le chemin)
    Source = Json.Document(File.Contents("C:\chemin\vers\demo-itsm-servicenow\powerbi\exemple-kpi.json"))
    // Données réelles : remplacer la ligne ci-dessus par
    // Source = Json.Document(Web.Contents("https://<votre-n8n>/webhook/kpi"))
in
    Source
```

**`Synthese`** : une ligne d'indicateurs.

```powerquery
let
    Lignes = Table.FromRecords({KPI[synthese]}),
    Typee = Table.TransformColumnTypes(Lignes, {
        {"problemes_proposes", Int64.Type}, {"problemes_a_trier", Int64.Type},
        {"problemes_valides", Int64.Type}, {"problemes_rejetes", Int64.Type},
        {"problemes_crees", Int64.Type}, {"taux_validation_pct", type number},
        {"incidents_regroupes", Int64.Type}, {"rex_rediges", Int64.Type},
        {"rex_a_relire", Int64.Type}, {"rex_publies", Int64.Type},
        {"delai_relecture_moyen_h", type number}, {"mttr_p1_min", type number}
    }),
    Libelle = Table.RenameColumns(Typee, {{"delai_relecture_moyen_h", "Délai de relecture (h)"}})
in
    Libelle
```

**`Problemes`** : un candidat par ligne.

```powerquery
let
    Lignes = Table.FromRecords(KPI[problemes],
        {"signature", "ci", "nb_incidents", "statut", "probleme", "propose_le"}, MissingField.UseNull),
    Typee = Table.TransformColumnTypes(Lignes, {{"nb_incidents", Int64.Type}, {"propose_le", type datetimezone}}),
    Jour = Table.AddColumn(Typee, "jour", each Date.From(DateTimeZone.ToLocal([propose_le])), type date)
in
    Jour
```

**`REX`** : un brouillon par ligne, avec sa semaine.

```powerquery
let
    Lignes = Table.FromRecords(KPI[rex],
        {"incident", "titre", "duree_min", "statut", "redige_le"}, MissingField.UseNull),
    Typee = Table.TransformColumnTypes(Lignes, {{"duree_min", type number}, {"redige_le", type datetimezone}}),
    Jour = Table.AddColumn(Typee, "jour", each Date.From(DateTimeZone.ToLocal([redige_le])), type date),
    Semaine = Table.AddColumn(Jour, "semaine", each Date.StartOfWeek([jour], Day.Monday), type date)
in
    Semaine
```

Terminer par *Accueil › Fermer et appliquer*. Les listes vides ne posent pas de problème : les colonnes sont nommées d'avance.

## 2. Mesures (DAX)

Dans la vue Données, sélectionner la table, puis *Nouvelle mesure*. Contrairement aux valeurs de `Synthese`, calculées une fois pour toutes par WF3, ces mesures réagissent aux filtres : par CI, par semaine, etc.

```dax
Taux de validation =
DIVIDE (
    CALCULATE ( COUNTROWS ( Problemes ), Problemes[statut] IN { "Validé", "Créé" } ),
    CALCULATE ( COUNTROWS ( Problemes ), Problemes[statut] IN { "Validé", "Créé", "Rejeté" } )
)

Incidents regroupés = SUM ( Problemes[nb_incidents] )

REX publiés = CALCULATE ( COUNTROWS ( REX ), REX[statut] = "Publié" )

REX à relire = CALCULATE ( COUNTROWS ( REX ), REX[statut] = "À relire" )

MTTR P1 (min) = AVERAGE ( REX[duree_min] )
```

Mettre *Taux de validation* au format pourcentage. Si Power BI refuse les virgules, c'est qu'il attend les séparateurs français : remplacer `,` par `;` dans les formules, ou choisir les séparateurs standard dans *Fichier › Options et paramètres › Options › Paramètres régionaux*.

Le délai de relecture vient directement de `Synthese[Délai de relecture (h)]` (colonne renommée dans la requête `Synthese`, pour que le libellé de la carte soit propre) : le calculer demande la date de publication, que WF3 n'expose pas ligne par ligne.

## 3. Mise en page

Une page, « Vue d'ensemble », lue de haut en bas : les chiffres, puis où ça se concentre, puis ce qui attend une action.

1. **Bandeau** : un seul visuel *Carte* à six valeurs : Taux de validation, Incidents regroupés, REX publiés, REX à relire, MTTR P1 (min) et `Synthese[Délai de relecture (h)]`. Titre du visuel désactivé.
2. **Où se concentrent les incidents récurrents** : *graphique à barres empilées*. Axe Y `Problemes[ci]`, axe X `Problemes[nb_incidents]`, légende `Problemes[statut]`, trié par nombre décroissant.
3. **REX rédigés par semaine** : *histogramme empilé*. Axe X `REX[semaine]` (type catégoriel), axe Y nombre de `REX[titre]`, légende `REX[statut]`.
4. **File de relecture** : *tableau* `REX[titre]`, `REX[redige_le]`, `REX[duree_min]`, `REX[statut]`, avec un filtre de visuel `REX[statut]` = « À relire » (3 lignes avec l'exemple). Dans les propriétés du champ `duree_min`, choisir *Ne pas synthétiser*.

Cliquer sur une ligne du tableau filtre toute la page (« filtre croisé ») : recliquer dessus, ou dans une zone vide, pour l'annuler. Avant de choisir un type de visuel, cliquer dans une zone vide : sinon Power BI transforme le visuel sélectionné.

Garder une couleur par statut dans tous les visuels, par exemple Créé et Publié en vert, Validé en bleu, À trier et À relire en gris, Rejeté en rouge clair.

## 4. Valeurs attendues avec l'exemple

| Indicateur | Valeur |
|---|---|
| Taux de validation | 75 % |
| Incidents regroupés | 54 |
| REX publiés / à relire | 10 / 3 |
| MTTR P1 | 97,1 min |
| Délai de relecture | 12,5 h |
| CI les plus touchés | ERP-PROD (9), DB-ORA-01 (8), NAS-FICHIERS (7) |
| REX par semaine | 1, 1, 2, 1, 2, 2, 2, 3 (du 17 août au 5 octobre) |

Si le rapport affiche ces valeurs, les requêtes et les mesures sont justes. Les premières valeurs sont aussi vérifiées par les tests du dépôt.

## 5. Enregistrer dans le dépôt

*Fichier › Enregistrer sous*, type **Fichiers de projet Power BI (*.pbip)**, dans ce dossier `powerbi/`, sous le nom `rapport-itsm`. Power BI crée des fichiers texte (`rapport-itsm.Report/`, `rapport-itsm.SemanticModel/`), qui se commitent et se relisent par diff comme le reste du dépôt. Le cache local est exclu par le `.gitignore` que Power BI ajoute. Ne pas enregistrer en `.pbix`, un format binaire ignoré par Git ici.

La requête `KPI` contient le chemin du fichier d'exemple sur la machine d'origine (`X:\Projets-GIT\…`). Après un clonage, l'adapter : *Accueil › Transformer les données › KPI › Éditeur avancé*, puis corriger le chemin de `File.Contents`. Les fichiers `.pbi\cache.abf` et `.pbi\localSettings.json` sont exclus par le `.gitignore` du dépôt.

## 6. Passer aux données réelles

1. Dans n8n, créer un identifiant **Basic Auth** dédié à Power BI (utilisateur et mot de passe longs), l'associer au nœud « Demande de Power BI » de WF3, puis activer WF3.
2. Dans la requête `KPI`, remplacer la ligne `Source` par la version `Web.Contents` (adresse de votre n8n).
3. À la première actualisation, Power BI demande comment se connecter : choisir **De base** et saisir l'utilisateur et le mot de passe. Power BI les garde dans son propre coffre : ils n'apparaissent pas dans les fichiers du projet, qui peuvent donc être commités sans risque.
