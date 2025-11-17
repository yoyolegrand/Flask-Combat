# Flask Combat Mod Design (Forge 47.4.0 / Minecraft 1.20.1)

## 1. Vision
Flask Combat est un mod Forge 1.20.1 (API 47.4.0) centré sur la fabrication et l'utilisation de fioles jetables qui infligent des dégâts magiques instantanés et laissent des zones d'effet temporaires. Chaque fiole peut brûler, geler, électrocuter ou altérer la zone d'impact, créant une boucle de gameplay tactique entre bombardement à distance et contrôle de territoire.

## 2. Arborescence recommandée
```
Flask-Combat/
├── build.gradle.kts            # Script ForgeGradle 6.x (MC 1.20.1 / FG 47.4.0)
├── settings.gradle.kts
├── gradle/                     # Wrapper
├── src/main/java/com/example/flaskcombat/
│   ├── FlaskCombat.java        # Entrée @Mod
│   ├── registry/               # DeferredRegister pour Items, Entities, Attributes, Particles
│   ├── attribute/              # Implémentations custom d'attributs et helpers de calcul
│   ├── item/
│   │   ├── FlaskItem.java      # Classe de base des fioles
│   │   └── types/              # Sous-classes (FireFlaskItem, FrostFlaskItem, etc.)
│   ├── entity/
│   │   └── FlaskProjectile.java# Projectile custom dérivé de ThrowableItemProjectile
│   ├── area/
│   │   ├── FlaskAreaEffect.java# Gestion des zones persistantes et des ticks
│   │   └── AreaScheduler.java  # Gestion serveur des durées
│   ├── data/                   # Gestion des ConfiguredFeature-like JSON pour les flasks
│   └── networking/             # Packets (spawn particules, sync attributs)
├── src/main/resources/
│   ├── META-INF/mods.toml      # Déclaration Forge et dépendances
│   ├── pack.mcmeta
│   ├── assets/flaskcombat/
│   │   ├── lang/en_us.json
│   │   ├── lang/fr_fr.json
│   │   ├── models/item/*.json
│   │   └── textures/item/*.png
│   └── data/flaskcombat/
│       ├── attributes/*.json   # Déclarations de Flask Effectiveness & Duration
│       ├── items/flasks/*.json # Tags et recettes
│       ├── loot_modifiers/*.json
│       └── damage_types/*.json # Damage type "flask_magic"
└── README.md
```

## 3. Composants essentiels
### 3.1 Fiole jetable (`FlaskItem`)
- Consomme l'objet et spawne un `FlaskProjectile`.
- Récupère les attributs du lanceur (Flask Effectiveness, Flask Duration).
- Peut stocker un objet `FlaskProfile` défini via JSON (dégâts de base, statut appliqué, particules, durée de zone par défaut, rayon, tick rate, sons).

### 3.2 Projectile (`FlaskProjectile`)
- Hérite de `ThrowableItemProjectile`.
- À l'impact : calcule les dégâts magiques instantanés et crée la zone persistante.
- Utilise le type de dégâts `flask_magic` pour gérer la réduction d'armure personnalisée.
- Détermine les entités affectées initialement (impact direct + entités dans le rayon immédiat).

### 3.3 Zones persistantes (`FlaskAreaEffect`)
- Entité serveur sans rendu (ou bloc virtuel) qui:
  - Maintient un `AABB` dynamique.
  - Applique des dégâts périodiques et effets de statut.
  - Gère la durée via `AreaScheduler` (tick server).
- Peut déclencher des particules/sons côté client via packets.

### 3.4 Attributs personnalisés
| Nom | Identifiant | Valeur par défaut | Effet |
| --- | ----------- | ----------------- | ----- |
| Flask Effectiveness | `flaskcombat:flask_effectiveness` | 0.0 | Multiplie les dégâts des fioles (instantané + DoT) : `dégâts_final = dégâts_base * (1 + valeur)` |
| Flask Duration | `flaskcombat:flask_duration` | 0.0 | Multiplie les durées (zone + statut) : `durée_finale = durée_base * (1 + valeur)` |

Ces attributs sont enregistrés via `DeferredRegister<Attribute>` et liés à l'équipement via modifiers (armures, talismans, potions).

## 4. Formules de calcul
### 4.1 Dégâts instantanés
```
baseDamage = profile.impactDamage (ex: 4.0 pour Fire Flask)
effectiveness = attacker.getAttributeValue(FLASK_EFFECTIVENESS)
impactDamage = baseDamage * (1.0 + effectiveness)
```

### 4.2 Durée de zone
```
baseDuration = profile.areaDurationTicks (ex: 200 ticks = 10s)
durationBonus = attacker.getAttributeValue(FLASK_DURATION)
areaDuration = baseDuration * (1.0 + durationBonus)
```

### 4.3 Dégâts périodiques
```
baseTickDamage = profile.dotDamage (ex: 2.0 chaque 0.5s)
tickInterval = profile.tickInterval (10 ticks)
effectiveness = attacker.getAttributeValue(FLASK_EFFECTIVENESS)
areaTickDamage = baseTickDamage * (1.0 + effectiveness)
```

### 4.4 Durées d'effet de statut
Toutes les durées appliquées par la fiole (feu, gel, poison, etc.) sont multipliées par `(1.0 + Flask Duration)`.

## 5. Exemple : Fire Flask
| Composant | Valeur | Notes |
| --- | --- | --- |
| `impactDamage` | 4.0 | Dégâts magiques immédiats |
| `status` | Fire (10s) | Durée ajustée par Flask Duration |
| `areaDuration` | 200 ticks (10s) | Zone de flammes |
| `areaRadius` | 3 blocs | Peut être data-driven |
| `areaTickDamage` | 2.0 | Appliqué toutes les 10 ticks (0.5s) |
| `areaStatus` | Fire (4s) | Tant que l'entité reste dans la zone |
| `particles` | `minecraft:flame` + `minecraft:small_flame` |
| `soundImpact` | `minecraft:item.bottle.throw` + custom |

## 6. Contenu data-driven
- **`data/flaskcombat/flasks/*.json`** : profils de fiole (fire, frost, toxic, arcane, void...).
- **Tags d'items** (`data/.../tags/items/ingredients/*.json`) : ingrédients de craft.
- **Recettes** (`data/.../recipes/flasks/*.json`) : shapeless, alchemy.
- **Damage type** (`data/.../damage_type/flask_magic.json`) : définit bypass/resistance.
- **Loot modifiers** : ajoute des flasks aux coffres.

## 7. Crafting & progression
1. **Atelier d'alchimie** (bloc custom) : interface à 3 slots (flasque vide + essence + catalyseur).
2. **Système de qualité** : niveau Commun/Rare/Epique augmente `impactDamage` et `areaDuration`.
3. **Talismans & armures** : fournissent les attributs Flask Effectiveness & Duration.

## 8. Checklist technique Forge 47.4.0
- `mods.toml` : `loaderVersion = "[47,)"`, `license`, `displayTest`. Dependances: Forge, Minecraft 1.20.1.
- `pack.mcmeta` : `pack_format` 18 (1.20.1).
- Utiliser `DeferredRegister` pour tous les contenus.
- Coté client : `ParticleRenderers.register`, `ItemProperties.register` pour animations.
- Tests : `RunClient`, `RunData` pour générer assets.

## 9. Roadmap
1. **MVP** : Implémenter Fire Flask + attributs custom.
2. **Étendre** : Ajouter Frost, Toxic, Arcane flasks.
3. **QoL** : HUD pour stacks de flasks, tooltips dynamiques, compat modded attributes.
4. **Balancing** : Config serveur (gains attributs max, cooldown global, friendly fire toggle).

## 10. Outils recommandés
- IntelliJ IDEA + Gradle.
- Data generators Forge pour recettes/models.
- `runData` pour générer assets.
- `CurseGradle` si publication CurseForge.

Cette spécification couvre les dossiers et systèmes nécessaires pour construire le mod Flask Combat conformément aux contraintes Forge 47.4.0 / MC 1.20.1.
