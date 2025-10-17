# Swish Utilities - Procédure KCS

Cette procédure décrit comment préparer les fichiers, choisir le type de protection des données (anonymisation, pseudonymisation, masquage), et exécuter Swish Utilities pour traiter des exports ITSM (ServiceNow ou autres). Elle couvre également la prévention et la résolution d’erreurs fréquentes, avec des exemples avant/après.

## Prérequis
* Windows PowerShell
* Python 3.12 installé
* Données en entrée encodées en UTF-8 avec BOM (utf-8-sig)

## Répetoires requis
* Projet : C:\Dev\swish-utils\swish-utilities
* Traitement : C:\Data

## Mise en place de l’environnement
Dans une console PowerShell
```
PS C:\\WINDOWS\\system32> cd C:\\Dev\\swish-utils\\swish-utilities\nPS
C:\\Dev\\swish-utils\\swish-utilities> py -3.12 -m venv .venv\nPS
C:\\Dev\\swish-utils\\swish-utilities> .\\.venv\\Scripts\\Activate
```

## Fichiers nécessaires
Trois fichiers de configuration sont utilisés :
* ```mapping.csv``` – Décrit la méthode appliquée à chaque colonne
* ```important_tokens.txt``` – Liste blanche : éléments à préserver (non masqués)
* ```custom_tokens.txt``` – Liste de mots/expressions à masquer en plus des regex

## Structure du fichier de ```mapping.csv```
Format CSV avec en-tête : column,method,condition
| column  | method | condition |
| ------- | ------ | --------- |
| Premier Groupe assigné   | 1 |
| Groupe d'affectation     | 1 |
| Résolu par               | 1 |
| Affecté à                | 1 |
| Appelant                 | 1 |
| Utilisateur impacté      | 1 |
| Elément de configuration | 1 |
| Référence externe        | 1 |
| Description courte       | 2 |
| Description              | 2 |
| Notes de résolution      | 2 |
| Notes de travail         | 2 |

Signification des méthodes :
* Pseudonymisation (hash déterministe) : la valeur entière est remplacée par un identifiant pseudonymisé (même entrée → même sortie).
* Masquage de texte libre : seules les occurrences d’informations sensibles (emails, téléphones, IP, grands nombres…) sont remplacées par des marqueurs (<#M>, <#P>, <#I>, <#>).
* Drop : suppression de la colonne.
* Filtrage conditionnel : conserve uniquement les lignes dont la valeur est dans la condition.

## Structure du fichier ```important_tokens.txt```

```
@altimance.fr
@e.altimance.fr
@scc.com
@fr.scc.com
Active Directory
PowerShell
```

## Choisir : anonymisation, pseudonymisation ou masquage
* Anonymisation (méthode 1 si configurée ainsi) : remplacement irréversible et non ré-identifiable (ex. valeur générique).
* Pseudonymisation (méthode 1 avec hash/HMAC) : remplacement déterministe, ré-identifiable via un secret ou une table de correspondance.
* Masquage (méthode 2) : conservation du texte, mais caviardage des occurrences sensibles par des tokens (<#M>, <#P>, etc.).

## Exemples avant / après

Exemple 1 – Pseudonymisation de colonnes identifiantes

```
Avant: 
Affecté à: "Jean Dupont" 
Appelant: "Julie PELERIN"
```
``` 
Après (method=1): 
Affecté à: 9f2e2a... (SHA-256) 
Appelant: 6b1c44... (SHA-256)
```

Exemple 2 – Masquage de champs texte libres

```
Avant (Description): 
"Contact: julie.pelerin@scc.com, tél: +33 6 12 34 56 78, IP: 10.10.2.45"
```
``` 
Après (method=2): 
"Contact: <#M>, tél: <#P>, IP: <#I>"
```

Exemple 3 – Préservation via important_tokens.txt

```
Avant (Notes): 
"Envoyer le résumé à @altimance.fr et au groupe Active Directory"
```
``` 
Après (method=2 + important_tokens): 
"Envoyer le résumé à @altimance.fr et au groupe Active Directory" (non masqués)
```

## Préparer les CSV (qualité des données)

* Encodage recommandé : UTF-8 avec BOM ("utf-8-sig").
* Délimiteurs : veiller aux virgules dans les textes ; si présentes, entourer le champ de guillemets."
* Lignes irrégulières : utiliser un pré-nettoyage tolérant si nécessaire (engine="python", on_bad_lines).

### Pré-nettoyage tolérant (optionnel)
```
import pandas as pd 

src = r"C:\Data\to_mask\incclients.csv" 
dst = r"C:\Data\to_mask\incclients_clean.csv" 

df = pd.read_csv(src, dtype=str, encoding='utf-8-sig', engine='python', on_bad_lines='warn')

df.to_csv(dst, index=False, encoding='utf-8-sig') 
print(f'Nettoyage OK -> {dst} (lignes: {len(df)})') 

$tempPy = Join-Path $env:TEMP 'sanitize_inc.py' Set-Content -Path $tempPy -Value 
$code -Encoding UTF8 python $tempP
```

## Exécution du masquage – commandes types
Masquer un fichier unique :
```
python .\run.py --mask ` 
    --input_dir "C:\Data\to_mask" ` 
    --input_file "incclients.csv" ` 
    --output_dir "C:\Data\masked" ` 
    --output_format "csv" ` 
    --mapping_path ".\mapping_inc.csv" ` 
    --important_token_file ".\important_tokens.txt" ` 
    --csv_chunk_size 200000 ` 
    --set_dtype ` 
    --input_encoding "utf-8-sig"
```

## Masquer tous les fichiers du dossier (retirer --input_file)

```
python .\run.py --mask ` 
    --input_dir "C:\Data\to_mask" ` 
    --output_dir "C:\Data\masked" ` 
    --output_format "csv" ` 
    --mapping_path ".\mapping_inc.csv" ` 
    --important_token_file ".\important_tokens.txt" ` 
    --csv_chunk_size 200000 ` 
    --set_dtype ` 
    --input_encoding "utf-8-sig"
```

## Contrôles qualité après traitement

* Vérifier que les colonnes pseudonymisées (method=1) ressemblent à des hashes (64 caractères hexadécimaux).
* Vérifier l’absence d’emails/téléphones bruts dans les champs de texte (méthode 2)

```
$csv = Import-Csv "C:\Data\masked\incclients_processed.csv" 
$cols = 'Affecté à','Appelant','Utilisateur impacté' 
$regex = '^[a-f0-9]{64}$' 
foreach ($c in $cols) { $bad = $csv | Where-Object { $_.$c -and ($_.$c -notmatch $regex) };"{0}: NOK={1}" -f $c, $bad.Count }
```

## Dépannage (Troubleshooting)
### Erreur : Expected X fields in line N, saw

Cause : Lignes CSV mal formées (virgules non-quotées, guillemets non fermés).
Correctifs :
* Pré-nettoyage (engine="python", on_bad_lines)
* Corriger manuellement la ligne signalée. 
* Adapter le parseur côté run.py pour une lecture plus tolérante.

```
pd.read_csv(path, engine='python', on_bad_lines='warn', dtype=str, encoding='utf-8-sig')
```

### Erreur : can't open file '...gen_mapping_from_csv.py'

Cause : Script absent ou chemin invalide.
Correctifs : 
* Vérifier la présence du fichier dans SwishPack ou commenter/retirer l’étape de génération automatique des mappings.

## FAQ

```
Quelle différence entre anonymisation et pseudonymisation ?
Anonymisation : remplacement irréversible sans possibilité de ré-identification. Pseudonymisation : remplacement déterministe (ex. hash/HMAC) permettant de conserver la corrélation et une ré-identification contrôlée (via un secret ou une table de correspondance).
```

```
À quoi sert important_tokens.txt ?
À préserver certains termes (domaines, groupes, acronymes) lors du masquage de texte (method=2).
```

```
Comment choisir les colonnes à traiter en method=1 ?
Cibler les identifiants personnels ou fonctionnels (noms, prénoms, groupes, numéros de ticket, identifiants techniques) pour conserver la corrélation sans exposer la donnée brute.
```

```
Puis-je exécuter le traitement sur tout un dossier ?
Oui, retirez --input_file et conservez --input_dir.
```

```
Que faire si mon CSV contient des accents mal lus ?
Forcer --input_encoding "utf-8-sig" et vérifier que le fichier a bien un BOM UTF-8.
```

## Annexes

### Template : ```mapping_inc.csv```
| column  | method | condition |
| ------- | ------ | --------- |
| Numéro                   | 1 |
| Premier Groupe assigné   | 1 |
| Groupe d'affectation     | 1 |
| Résolu par               | 1 |
| Affecté à                | 1 |
| Appelant                 | 1 |
| Utilisateur impacté      | 1 |
| Elément de configuration | 1 |
| Référence externe        | 1 |
| Description courte       | 2 |
| Description              | 2 |
| Notes de résolution      | 2 |
| Notes de travail         | 2 |


### Exemple : ```important_tokens.txt```
```
@altimance.fr
@e.altimance.fr
@fr.scc.com
@scc.com
Active Directory
PowerShell
```