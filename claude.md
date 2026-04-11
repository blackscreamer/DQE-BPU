# DQE / BPU — Application de Devis BTP
## Référence technique complète — `claude.md`

---

## 1. Vue d'ensemble

Application web **standalone** (fichier HTML unique, aucun serveur requis) pour créer des devis de construction au format DQE (Détail Quantitatif Estimatif) ou BPU (Bordereau des Prix Unitaires) selon la norme algérienne.

- **Fichier** : `devis_app.html`
- **Dépendance externe** : SheetJS (`xlsx.full.min.js`) via CDN pour l'export Excel
- **Police** : Times New Roman 11pt, texte noir
- **Devise** : DA (Dinar Algérien), format `fr-DZ`

---

## 2. Structure des données

Toutes les données sont dans le tableau JavaScript `rows[]`. Chaque ligne est un objet avec `type` et `id` (UID unique).

### Types de lignes

| type | Description | Champs spécifiques |
|------|-------------|-------------------|
| `chap` | Chapitre | `desig`, `collapsed` |
| `sub` | Sous-chapitre | `desig`, `level` (1/2/3), `collapsed` |
| `art` | Article | `desig`, `unite`, `qty`, `pu` |
| `subart` | Sous-article (a, b, c…) | `desig`, `unite`, `qty`, `pu` |
| `blank` | Ligne description | `desig` |

### Variables globales

```js
let rows  = [];     // tableau des lignes
let mode  = 'DQE';  // 'DQE' | 'BPU'
let nid   = 1;      // compteur pour uid()
let selId = null;   // id de la ligne sélectionnée
let C     = {...};  // palette de couleurs (objet)
```

---

## 3. Numérotation

| Type | Format | Règle |
|------|--------|-------|
| Chapitre | `I`, `II`, `III`… | Chiffres romains, indépendants |
| Sous-chapitre | `1`, `2`, `3`… | **Continus** à travers tous les chapitres, jamais remis à zéro |
| Article | `1.01`, `1.02`, `2.01`… | Numéro de sous-chapitre + `.` + séquence 2 chiffres, remis à 0 par sous-chapitre |
| Sous-article | `a)`, `b)`, `c)`… | Lettres minuscules, remises à 0 par article parent |

Fonction : `buildNums()` → retourne `{nums[], letters[]}`

---

## 4. Hiérarchie des totaux

```
Chapitre I
  Sous-chapitre 1
    Article 1.01            → qty × pu = montant
    Article 1.02
      a) sous-article        → qty × pu  (article parent : colonnes U/Q/PU/MHT vides)
      b) sous-article
  [Total Sous-chapitre 1]   → somme articles + sous-articles
  Sous-chapitre 2
    ...
  [Total Sous-chapitre 2]
[Total Chapitre I]          → somme de tous les articles du chapitre
...
[TOTAL GÉNÉRAL HT]
[TVA (19%)]
[TOTAL TTC]
```

**Règle importante** : si un article (`art`) a des sous-articles (`subart`) qui le suivent immédiatement, les colonnes U / Quantité / P.U. / Montant de l'article parent sont **laissées vides** (fonction `artHasSubarts(id)`).

**Règle totaux chapitre/sous-chapitre** : la ligne titre (`chap`, `sub`) a sa dernière `<td>` **vide** (aucun id, aucune valeur). Les totaux apparaissent uniquement dans les lignes séparées `rts`/`rtc` générées par `subTotRow()`/`chapTotRow()` à la fin de chaque groupe.

Fonction : `computeTotals()` → retourne `{chapT:{id:total}, subT:{id:total}}`
- Chaque `subT[id]` = somme des art du sous-chapitre (les subart appartenant à un art parent ne sont **pas** recomptés)
- Les sous-chapitres imbriqués remontent vers les parents via un `subStack`
- `grandTotal()` → total général HT sans double-comptage

---

## 5. Couleurs — objet `C`

```js
C = {
  chapBg, chapFg,   // Chapitre (défaut: rouge #c00000 / blanc)
  sub1Bg, sub1Fg,   // Sous-chap niv.1 (défaut: jaune / noir)
  sub2Bg, sub2Fg,   // Sous-chap niv.2 (défaut: bleu clair)
  sub3Bg, sub3Fg,   // Sous-chap niv.3 (défaut: vert clair)
  artBg,  artFg,    // Article (défaut: blanc / noir)
  saBg,   saFg,     // Sous-article (défaut: #f5f5f5 / noir)
  tsBg,   tsFg,     // Total sous-chapitre (défaut: vert clair)
  tcBg,   tcFg,     // Total chapitre (défaut: jaune)
  gtBg,   gtFg,     // Total général (défaut: jaune)
}
```

Les couleurs sont sauvegardées avec le projet JSON (`v:5`).

**Important** : pour que le texte soit visible sur fond coloré, les inputs utilisent :
```css
color: ${fg} !important;
-webkit-text-fill-color: ${fg};
caret-color: ${fg};
```

---

## 6. Comportement de l'interface

### Sélection
- Clic sur une ligne → `selectRow(id, trEl, event)` → met `selId`, ajoute classe `sel-row`, positionne le panneau flottant
- Clic en dehors → `clearSelection()` → masque le panneau

### Panneau flottant (`#float-ctrl`)
- **Position** : `fixed`, calculée à droite de la ligne sélectionnée
- **Boutons** : ⠿ Déplacer (drag), ⧉ Dupliquer, ✕ Supprimer
- **Non imprimé** (`display:none` en `@media print`)

### Insertion
- Toute nouvelle ligne est insérée **après la ligne sélectionnée** via `insertIdx()`
- Si rien n'est sélectionné → ajout en fin de liste

### Raccourcis clavier
- `+` (touche plus, hors input) → ajoute un article après la sélection

### Unité par défaut
- `lastUniteBefore(insertIdx())` → hérite de l'unité de la ligne précédente (art ou subart)

### Drag & drop
- Poignée `⠿` dans le panneau flottant
- Drop sur n'importe quelle ligne du tableau
- Un `chap` emporte tout son bloc (sous-chapitres + articles)

---

## 7. Modes DQE / BPU

| Colonne | DQE | BPU |
|---------|-----|-----|
| Quantité | ✓ visible | masquée (`visibility:hidden`) |
| Montant | `qty × pu` | `pu` seul |

---

## 8. Sauvegarde / Chargement

Format JSON (version `v:5`) :

```json
{
  "v": 5,
  "l1": "Titre ligne 1",
  "l2": "Sous-titre ligne 2",
  "l3": "Sous-titre ligne 3",
  "mode": "DQE",
  "tva": "19",
  "rows": [...],
  "nid": 42,
  "C": { "chapBg": "#c00000", ... }
}
```

---

## 9. Export Excel (SheetJS)

- Reproduit la même structure hiérarchique avec les totaux injectés aux bons endroits
- Colonnes : N° | Désignation | U | [Quantité] | Prix U HT | Montant HT
- Lignes supplémentaires en bas : Total HT, TVA, Total TTC
- Largeurs de colonnes définies dans `ws['!cols']`

---

## 10. Impression / Print

```css
@media print {
  /* Masquer : toolbar, footer, panneau flottant, color picker */
  /* Forcer les couleurs de fond à l'impression */
  * { -webkit-print-color-adjust: exact !important;
      print-color-adjust: exact !important; }
}
```

---

## 11. Fonctions principales

| Fonction | Rôle |
|----------|------|
| `render()` | Reconstruit tout le HTML du tableau |
| `recalc()` | Met à jour les totaux sans re-render |
| `buildNums()` | Calcule la numérotation de toutes les lignes |
| `buildVisibility()` | Détermine les lignes masquées (collapse) |
| `computeTotals()` | Calcule chapT et subT |
| `grandTotal()` | Total général HT sans double-comptage (art parent compte ses subart) |
| `artTotal(r)` | Retourne le montant unitaire d'un art/subart (qty×pu ou pu seul en BPU) |
| `artHasSubarts(id)` | Vrai si l'art a des subart qui le suivent immédiatement |
| `artParentTotal(r)` | Somme des subart d'un art parent |
| `lastUniteBefore(idx)` | Unité héritée pour nouvelle ligne (remonte jusqu'au premier art/subart précédent) |
| `insertIdx()` | Index d'insertion (après sélection) |
| `selectRow(id,tr,e)` | Sélectionne une ligne |
| `addRow(type,level)` | Ajoute une ligne |
| `delSelected()` | Supprime la ligne sélectionnée |
| `dupSelected()` | Duplique la ligne sélectionnée |
| `toggleCollapse(id,e)` | Réduit/développe chap ou sub |
| `fitTotalCol()` | Ajuste la largeur de la colonne Montant |
| `saveJSON()` | Exporte le projet en `.json` |
| `loadJSON(inp)` | Charge un projet `.json` |
| `doExport()` | Exporte en `.xlsx` |
| `applyColors()` | Lit le color picker et re-render |

---

## 12. Structure HTML du tableau

```
<table id="tbl">
  <colgroup>
    <col class="cn">  70px   N°
    <col class="cd">  auto   Désignation
    <col class="cu">  46px   Unité
    <col class="cq">  82px   Quantité
    <col class="cp">  92px   Prix U
    <col class="ct">  auto   Montant (auto-fit)
  </colgroup>
  <thead>...</thead>
  <tbody id="body">
    <!-- Généré par render() -->
    <!-- Ordre : row, [sub-total après chaque sub], [chap-total après chaque chap] -->
    <!-- Après le dernier groupe : TOTAL HT + TVA + TTC -->
  </tbody>
</table>
```

---

## 13. Points d'attention

- **Chapitre/sous-chapitre** : la ligne titre a `colspan="4"` (désignation) + une `<td>` vide en fin — **aucun id ni valeur de total dans la ligne titre**
- **Article avec sous-articles** : toujours appeler `artHasSubarts()` avant de render les colonnes U/Q/PU/Montant ; si vrai → cellules vides + `artParentTotal()` pour le montant
- **Double-comptage** : `computeTotals()` et `grandTotal()` ignorent les `subart` dont le parent `art` est déjà compté
- **Couleurs sur inputs** : toujours utiliser `-webkit-text-fill-color` + `caret-color` + `!important` (sinon le texte est invisible sur fond coloré)
- **Numérotation sous-chapitre** : globale et continue, ne pas remettre à 0 lors d'un nouveau chapitre
- **Totaux** : lignes séparées (`rts`, `rtc`) APRÈS le groupe, puis `rgt` / `rtva` / `rttc` en bas du tableau
- **Impression** : `print-color-adjust: exact` indispensable pour préserver les fonds colorés ; TVA et TTC sont dans le tableau (pas dans le footer div) → visibles à l'impression automatiquement
