# Intervertir deux layouts (.szs) sans casser la logique

> Note : le corps de `al::initLayoutActor` n'est pas encore décompilé. Ce qui concerne le nommage
> interne du `.szs` est déduit des chaînes du binaire (`data/data_strings.csv`) et des signatures.

## 1. Chargement d'un layout

```cpp
al::initLayoutActor(this, info, "CounterLife", nullptr);   // archive, suffixe
```

- Chemin : `LayoutData/<Nom>.szs` (chaînes `"LayoutData/%s"`, `"%s.szs"`).
  La variante `initLayoutActorLocalized` passe par `LocalizedData/<Langue>/...`
  (`lib/al/Library/File/FileUtil.cpp`).
- Chargement via `ResourceSystem::findOrCreateResource` (`lib/al/Project/Resource/ResourceSystem.cpp`) :
  mis en cache **par nom**, dans la catégorie (tas) définie par `SystemData/ResourceSystem.szs`
  → `ResourceCategoryTable`, sinon dans la catégorie courante (`"シーン"`).
- Contenu (déduit) :

```
<Nom>.szs
 ├─ layout.lyarc
 │   ├─ blyt/<Nom>.bflyt            ("%s.bflyt")
 │   ├─ blyt/<Parts>.bflyt          (sous-layouts)
 │   ├─ anim/<Nom>_<Action>.bflan   ("*%s_*.bflan", "%s_%s.bflan")
 │   └─ timg/__Combined.bntx
 └─ byml de config éventuels (InitActor, InitMainGroup, InitSound…)
```

- Textes : `MessageData/LayoutMessage` → `LayoutMsg/<Nom>.msbt`, indexés par nom de layout.
- Sous-parties : `al::initLayoutPartsActor(child, parent, info, "ParList00")` — `"ParList00"` est le nom
  d'un pane Parts du `.bflyt` parent, pas une archive. Les sous-parties viennent donc du même `.szs`.

## 2. Contrat implicite code ↔ `.szs`

Exemple (`src/Layout/CounterLifeCtrl.cpp`) :

| Appel | Exigence |
|---|---|
| `getPaneLocalTrans(mCounterLife, "All")` | pane `All` |
| `setPaneStringFormat(..., "TxtLife", ...)` | pane texte `TxtLife` |
| `startAction(mCounterLifeUp, "Break", "Life")` | groupe `Life` + anim `<Nom>_Break` sur ce groupe |
| `getActionFrameMax(counter, "Gauge", "Gauge")` | groupe `Gauge` + anim `Gauge` |
| `startAction(this, "Appear")` | anim `Appear` sur le groupe principal |

Comportement en cas d'absence (`lib/al/Library/Layout/LayoutActionFunction.cpp`) :

- `getActionFrameMax`, `startFreezeAction`, `startActionAtRandomFrame` déréférencent le groupe sans
  vérification → crash si le groupe manque.
- `startAction` appelle directement `LayoutActionKeeper::startAction` → anim absente = crash probable.
- `isActionEnd` renvoie `true` si le groupe n'existe pas → les nerves Appear/Wait/End s'enchaînent
  instantanément.
- Seuls `tryStartAction` / `isExistAction` sont sûrs.

**Règle : le layout de remplacement doit contenir au minimum tous les panes, groupes, actions et
panes Parts utilisés par le code de l'autre.**

## 3. Méthode A — échange de fichiers (romfs / LayeredFS)

Renommer `A.szs` ↔ `B.szs` ne suffit pas :

1. Ouvrir les deux `.szs` (ex. Switch Toolbox) et leur `layout.lyarc`.
2. Renommer `blyt/B.bflyt` → `blyt/A.bflyt` et `anim/B_*.bflan` → `anim/A_*.bflan`
   (ne pas renommer les bflyt de parts).
3. Recompresser en `A.szs` ; faire l'inverse pour `B.szs`.
4. Vérifier le contrat (section 2) : relever les chaînes de la classe
   (`grep -n '"' src/Layout/<Classe>.cpp`) et ajouter les panes / groupes / `.bflan` vides manquants.
5. Échanger aussi `LayoutMsg/A.msbt` ↔ `LayoutMsg/B.msbt` dans `MessageData/LayoutMessage.szs`.
6. Mémoire : catégories différentes ou tailles très différentes → risque de débordement du tas.

## 4. Méthode B — redirection dans le code

Recompilation : changer la chaîne dans le constructeur.

Hook global (exlaunch) sur `al::initLayoutActor` (et `initLayoutActorLocalized`) :

```cpp
HOOK_DEFINE_TRAMPOLINE(InitLayoutSwap) {
    static void Callback(al::LayoutActor* actor, const al::LayoutInitInfo& info,
                         const char* arc, const char* suffix) {
        const char* msg = arc;
        if (al::isEqualString(arc, "A")) arc = "B";
        else if (al::isEqualString(arc, "B")) arc = "A";

        if (msg != arc)  // conserver les textes d'origine
            return al::initLayoutActorUseOtherMessage(actor, info, arc, msg, suffix);
        Orig(actor, info, arc, suffix);
    }
};
```

Attention à une éventuelle réentrance si `initLayoutActorUseOtherMessage` appelle `initLayoutActor`
en interne (ajouter un drapeau si besoin). Les sous-parties suivent automatiquement. Le contrat de la
section 2 reste obligatoire.

## 5. Animations

- Action = `.bflan` `<Layout>_<Action>`, appliquée aux groupes du bflyt (`LayoutPaneGroup`).
- Il faut le même nom d'action sur le même groupe ; le contenu et la durée peuvent différer
  (les nerves attendent `isActionEnd`).
- Exception : jauges (`startFreezeGaugeAction`, `getActionFrameMax`) utilisent la frame max comme
  échelle.
- Anim manquante → un `.bflan` vide sur le bon groupe évite le crash.

## Cas trivial

`CounterLife`, `CounterLifeKids` et `CounterLifeUp` sont créés par la même classe `CounterLife`
(`src/Layout/CounterLifeCtrl.cpp`) : même contrat, échange sans risque. Bon premier test.
