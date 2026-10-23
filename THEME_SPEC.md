# Format de thème `.tvtheme` — spécification v1 (launcher 1.0)

Ce document définit le format que le launcher lit pour changer complètement son apparence.
Il sert à la fois de référence pour le code du launcher et de base pour générer des thèmes avec Claude.

---

## 0. Ce que le launcher gère (version 1.0)

Une clé non gérée est **ignorée sans erreur** : le thème s'affiche quand même. Pour un thème qui rend exactement comme prévu, n'utiliser que la colonne « Gérées ».

| Section | Gérées | Ignorées pour l'instant |
|---|---|---|
| `meta` | toutes | — |
| `colors` | toutes (§3.2) | — |
| `typography` | `fonts` (builtin:sans, serif, mono ou `assets/fonts/…`), `styles` : clock, hero, title, body, caption, cardTitle, cardMeta (font, size, weight, letterSpacing, uppercase, italic, color) | `builtin:rounded` (= sans) |
| `progress` | thickness, cardThickness, radius | — |
| `shape` | cornerStyle (rounded, cut), radius.*, borderWidth, borderColor | cornerStyle `squircle` |
| `glass` | enabled, tint, highlight | blur, realtime, grain, saturation |
| `background` | type solid, gradient, shader, image, media ; color, gradient, shader (agsl, uniforms, scale, fps), image, media (dim), overlay (dim, vignette), fallback | type `video`, parallax ; uniforms `uFocus` et `uMusicLevel` restent fixes |
| `motion` | durations, easing, pageTransition (none, fade, slide, slideFade, scale), carouselTransition (crossfade, slide), reduceMotion | stagger, pageTransition `flip`/`zoomBlur` (= scale), carouselTransition `cascade` |
| `focus` | scale, lift, ring, glow | dimOthers, shine, animation |
| `components` | nuvioBadge (label, none) | toutes les autres variantes |
| `effects` | — | toute la section |
| `icons` | apps, appsFallback (original, letter) | ui, style |
| `sounds` | toutes | — |
| `music` | toutes | — |
| `boot` | — | toute la section |
| `frames` | panel, card, cardFocused, pill, pillActive, tile, strip (corners, stroke, strokeGradient, glow, notches, fill) | cadres SVG étirables |
| `decorations` | toutes (§3.16) | contenu `lottie` |
| `shaders` | screenOverlay | cardImage, focusEnter, pageTransition |
| `text` | effects (italic, glow, gradient, uppercase), labels (§3.18) | outline, glitch |
| `accents` | nuvioCards, dockTiles, profiles (fromProfile), strips (spotify, smarttube) | — |

**Mise en page de l'accueil** (fixe, non modifiable par un thème) : barre du haut, bloc Nuvio (3 cartes), bandeaux Spotify et SmartTube, dock. L'utilisateur peut activer le *mode miniatures* et le *mode cinéma* (le bloc Nuvio s'agrandit quand il est sélectionné et les bandeaux se réduisent) : un thème doit rester lisible dans ces deux états.

**Marges** : le contenu des panneaux, bandeaux et cartes est automatiquement éloigné des grands coins coupés ou rentrants et des encoches de leur cadre. Pas besoin de compenser dans le thème.

## 1. Principes

1. **Tout est optionnel.** Chaque valeur a un défaut. Un thème qui ne définit que 3 couleurs est valide.
2. **Rien ne peut casser le launcher.** Clé inconnue → ignorée. Valeur invalide → valeur par défaut + avertissement dans l'aperçu.
3. **Pas de code exécutable.** Seuls les shaders AGSL (code GPU sandboxé, sans accès réseau ni fichiers) sont acceptés.
4. **Aperçu + confirmation.** Un thème reçu s'applique en aperçu ; sans confirmation sous 15 s, l'ancien revient.
5. **Le mode « économie »** (réglage utilisateur) désactive automatiquement : fond vidéo/shader, flou temps réel, particules, musique. Le thème peut fournir ses propres repli via `fallback`.

---

## 2. Deux façons de livrer un thème

| Forme | Contenu | Quand l'utiliser |
|---|---|---|
| **`theme.json` seul** | Un seul fichier texte. SVG et shaders écrits en ligne, images/musiques par lien `https://`. | Thèmes générés dans un chat : tu copies le texte et tu le colles depuis ton téléphone. |
| **`.tvtheme`** (ZIP) | `theme.json` + dossier `assets/` (images, vidéos, sons, polices…). | Thèmes avec tes propres fichiers (fond photo, musique). |

Structure d'un `.tvtheme` :
```
mon-theme.tvtheme
├── theme.json
└── assets/
    ├── backgrounds/   (jpg, png, webp, mp4)
    ├── icons/         (svg, png, webp)
    ├── sounds/        (ogg, mp3, wav)
    ├── music/         (ogg, mp3)
    ├── fonts/         (ttf, otf)
    ├── shaders/       (agsl)
    └── lottie/        (json)
```

### Référencer une ressource
Partout où une ressource est attendue, on peut écrire :
- `"assets/icons/home.svg"` → fichier dans le ZIP
- `"https://exemple.com/fond.jpg"` → téléchargé une fois et mis en cache
- `{ "svg": "<svg …>…</svg>" }` → SVG en ligne
- `{ "agsl": "…code…" }` → shader en ligne
- `"builtin:nom"` → ressource fournie par le launcher (icônes, sons et shaders de base)

### Référencer une couleur
- `"#RRGGBB"` ou `"#RRGGBBAA"`
- `"{colors.accent}"` → réutiliser une autre couleur du thème
- `"{media.dominant}"`, `"{media.vibrant}"`, `"{media.muted}"` → couleur extraite **en direct** de l'image du média sélectionné (rend le thème dynamique)
- `"{profile.color}"` → couleur du profil Nuvio actif
- Suffixe d'opacité : `"{colors.accent}@40"` = même couleur à 40 %

---

## 3. Structure de `theme.json`

```jsonc
{
  "format": "tvtheme",
  "schemaVersion": 1,
  "meta":        { … },   // §3.1
  "colors":      { … },   // §3.2
  "typography":  { … },   // §3.3
  "shape":       { … },   // §3.4
  "glass":       { … },   // §3.5
  "background":  { … },   // §3.6
  "motion":      { … },   // §3.7
  "focus":       { … },   // §3.8
  "effects":     { … },   // §3.9
  "components":  { … },   // §3.10
  "icons":       { … },   // §3.11
  "sounds":      { … },   // §3.12
  "music":       { … },   // §3.13
  "boot":        { … },   // §3.14
  "frames":      { … },   // §3.15  cadres des panneaux, cartes, boutons
  "decorations": [ … ],   // §3.16  éléments décoratifs ancrés
  "shaders":     { … },   // §3.17  shaders par composant et sur tout l'écran
  "text":        { … },   // §3.18  effets de texte et libellés
  "accents":     { … }    // §3.19  couleurs différentes par élément
}
```

### 3.1 `meta`
| Clé | Type | Défaut | Description |
|---|---|---|---|
| `id` | string `[a-z0-9-]` | requis | Identifiant unique, ex. `verre-depoli` |
| `name` | string | requis | Nom affiché |
| `version` | string | `"1.0"` | Un thème avec le même `id` et une version plus haute remplace l'ancien |
| `author` | string | — | |
| `description` | string | — | |
| `preview` | ressource | capture auto | Vignette dans « Mes thèmes » |
| `mode` | `dark` \| `light` | `dark` | Indicatif, pour les icônes système |

### 3.2 `colors`
| Clé | Rôle |
|---|---|
| `background` | Couleur de fond (sous l'image/shader) |
| `surface` | Panneaux opaques |
| `surfaceGlass` | Teinte des panneaux en verre (avec alpha) |
| `surfaceRaised` | Boutons, pastilles inactives |
| `text` | Texte principal |
| `textMuted` | Texte secondaire |
| `textOnAccent` | Texte sur fond `accent` |
| `accent` | Couleur d'accent (onglet actif, boutons) |
| `focusRing` | Contour de l'élément sélectionné |
| `progressTrack` / `progressFill` | Barres de progression |
| `nuvio` / `spotify` / `smarttube` | Accent propre à chaque bloc (défaut : `accent`) |
| `badge` / `badgeText` | Étiquettes SÉRIE / FILM |
| `danger` / `success` | États |

### 3.3 `typography`
```jsonc
"typography": {
  "fonts": { "display": "assets/fonts/Inter-Bold.ttf", "body": "builtin:sans" },
  "styles": {
    "clock":   { "font": "display", "size": 40, "weight": 600, "letterSpacing": -0.5 },
    "hero":    { "font": "display", "size": 34, "weight": 700 },
    "title":   { "font": "display", "size": 22, "weight": 600 },
    "body":    { "font": "body",    "size": 16, "weight": 400 },
    "caption": { "font": "body",    "size": 13, "weight": 500, "uppercase": true, "letterSpacing": 1.2 }
  }
}
```
Tailles en `dp` pour un écran 1080p. Polices intégrées : `builtin:sans`, `builtin:serif`, `builtin:mono`, `builtin:rounded`.
Police personnalisée : chemin vers un fichier de l'archive `.tvtheme`, ex. `"display": "assets/fonts/Orbitron.ttf"`.

Styles disponibles : `clock`, `hero`, `title`, `body`, `caption`, et pour les cartes Nuvio `cardTitle` (titre) et `cardMeta` (épisode, temps restant). Sans `cardTitle`/`cardMeta`, les cartes reprennent `title`/`body`.
Chaque style accepte aussi `"color"` (couleur ou référence, ex. `"{colors.accent}"`).
Tailles maximales (au-delà, le texte ne tiendrait plus) : clock 64, hero 40, title 24, body 18, caption 14, cardTitle 30, cardMeta 18.

### 3.4 bis `progress` — barres de progression
```jsonc
"progress": {
  "thickness": 3,       // épaisseur des barres (dp, 1–16)
  "cardThickness": 5,   // épaisseur sur les cartes Nuvio (sinon = thickness)
  "radius": 99          // arrondi des extrémités (0 = barre carrée)
}
```
Couleurs : `colors.progressTrack` (fond), `colors.nuvio` / `colors.spotify` / `colors.smarttube` (remplissage de chaque bloc).

### 3.4 `shape`
| Clé | Défaut | Description |
|---|---|---|
| `cornerStyle` | `rounded` | `rounded` \| `cut` (coins biseautés) \| `squircle` |
| `radius.card` | 20 | Cartes Nuvio |
| `radius.panel` | 24 | Grands panneaux |
| `radius.pill` | 999 | Pastilles, onglets |
| `radius.tile` | 16 | Tuiles d'apps |
| `borderWidth` | 1 | Bordure des panneaux |
| `borderColor` | `#FFFFFF1F` | |

### 3.5 `glass` — effet verre dépoli
| Clé | Défaut | Description |
|---|---|---|
| `enabled` | `true` | |
| `blur` | 28 | Rayon du flou (dp) |
| `realtime` | `false` | `true` = floute ce qui bouge derrière (plus coûteux) |
| `tint` | `{colors.surfaceGlass}` | |
| `grain` | 0.04 | Bruit mat, 0–0.2 |
| `highlight` | `0.08` | Reflet clair sur le bord haut |
| `saturation` | 1.2 | Saturation de ce qui est vu à travers |

### 3.6 `background`
```jsonc
"background": {
  "type": "shader",                  // solid | gradient | image | video | shader | media
  "color": "{colors.background}",    // solid
  "gradient": { "angle": 160, "stops": ["#0E1420", "#1A2333"] },
  "image": "assets/backgrounds/fond.webp",
  "video": "assets/backgrounds/boucle.mp4",   // muet, en boucle, 1080p max
  "shader": { "agsl": "…", "uniforms": { "speed": 0.2 } },
  "media": { "blur": 60, "dim": 0.55 },        // type media = image floutée du média sélectionné
  "overlay": { "dim": 0.2, "vignette": 0.3 },
  "parallax": 0.02,                  // décalage léger quand le focus se déplace
  "fallback": { "type": "gradient" } // utilisé en mode économie
}
```
Types gérés : `solid`, `gradient`, `shader`, `image` (fichier de l'archive ou lien) et `media` (image du titre Nuvio sélectionné, floutée, assombrie de `media.dim`). `video` n'est pas encore géré (repli utilisé).

**Uniforms fournis aux shaders** : `iTime` (s), `iResolution` (px), `uFocus` (position 0–1 de l'élément sélectionné), `uMediaColor` (couleur du média sélectionné), `uMusicLevel` (volume de la musique en cours, 0–1).
Signature attendue : `half4 main(float2 fragCoord)`.

### 3.7 `motion`
```jsonc
"motion": {
  "durations": { "fast": 120, "base": 240, "slow": 420 },    // ms
  "easing": { "standard": "cubic-bezier(0.2,0,0,1)", "emphasized": "spring(0.75,380)" },
  "pageTransition": "slideFade",     // none | fade | slide | slideFade | scale | zoomBlur | flip
  "carouselTransition": "crossfade", // changement de profil : none | crossfade | slide | cascade
  "stagger": 40,                     // décalage d'apparition entre éléments (ms)
  "reduceMotion": false
}
```
Easing : `linear`, `cubic-bezier(a,b,c,d)`, ou `spring(dampingRatio, stiffness)`.

### 3.8 `focus` — élément sélectionné à la télécommande
| Clé | Défaut | Description |
|---|---|---|
| `scale` | 1.06 | Agrandissement |
| `lift` | 6 | Élévation (ombre) |
| `ring` | `{ "width": 2, "color": "{colors.focusRing}", "offset": 3 }` | |
| `glow` | `{ "radius": 0, "color": "{media.vibrant}@60" }` | Halo |
| `dimOthers` | 0 | Assombrit les autres éléments (0–1) |
| `animation` | `{ "easing": "emphasized" }` | |
| `shine` | `false` | Reflet qui balaie la carte au focus |

### 3.9 `effects` — non géré pour l'instant
```jsonc
"effects": {
  "particles": { "type": "snow", "density": 0.3, "color": "#FFFFFF80", "speed": 0.5 }, // none|snow|stars|dust|bubbles|rain
  "clockAnimation": "flip",          // none | fade | flip | roll
  "progressAnimation": "shimmer"     // none | shimmer | pulse
}
```

### 3.10 `components` — variantes de forme (seul `nuvioBadge` est géré)
Le thème change la forme des blocs en choisissant parmi des variantes (la disposition générale de l'accueil reste la même).

| Composant | Variantes |
|---|---|
| `nuvioCard` | `backdrop` (image horizontale) \| `poster` (affiche verticale) \| `minimal` (titre + barre) |
| `nuvioBadge` | `label` \| `dot` \| `none` |
| `profileTabs` | `pills` \| `avatars` \| `underline` |
| `progressBar` | `line` \| `thick` \| `segmented` \| `glow` |
| `mediaStrip` (Spotify/SmartTube) | `compact` \| `card` \| `vinyl` \| `waveform` |
| `button` | `filled` \| `outlined` \| `ghost` \| `bracket` (`[ Texte ]`) |

Note sur `waveform` : Android ne laisse pas une app lire le son d'une autre de façon fiable. L'onde est donc **simulée** : elle bouge pendant la lecture, se fige en pause, et sa forme suit la progression du morceau. Elle ne réagit pas au vrai son.
| `dock` | `tiles` \| `icons` \| `iconsLabeled` |
| `topNav` | `pills` \| `underline` \| `iconsOnly` |
| `statusChips` | `chips` \| `icons` \| `hidden` |

### 3.11 `icons`
```jsonc
"icons": {
  "style": { "mask": "squircle", "tint": null },   // mask : none|circle|squircle|rounded ; tint = recolore les icônes
  "apps": {
    "tv.twitch.android.app": { "svg": "<svg …/>" },
    "com.spotify.tv.android": "assets/icons/spotify.png"
  },
  "appsFallback": "original",                       // original | monochrome | tinted | letter
  "ui": {
    "home": { "svg": "…" }, "apps": "…", "settings": "…",
    "wifi": "…", "bluetooth": "…", "weather": "…",
    "play": "…", "pause": "…", "next": "…", "previous": "…",
    "resume": "…", "search": "…", "back": "…", "phone": "…"
  }
}
```
Icône manquante → bannière ou icône d'origine de l'app (`appsFallback: "letter"` affiche l'initiale).
Les icônes SVG sont teintées avec `colors.text` (dessine-les en `currentColor`) ; les PNG/WebP gardent leurs couleurs.
État actuel du launcher : `icons.apps` et `appsFallback` sont gérés ; `icons.ui` et `style` pas encore.

### 3.12 `sounds`
```jsonc
"sounds": {
  "volume": 0.6,
  "focus":    "assets/sounds/tick.ogg",
  "select":   "builtin:click-soft",
  "back":     "…",
  "page":     "…",
  "error":    "…",
  "themeApplied": "…"
}
```
Sons courts (< 1 s), au format WAV, OGG ou MP3.
Sons intégrés (synthétisés par le launcher, aucun fichier à fournir) : `builtin:tick-soft`, `builtin:tick`, `builtin:click-soft`, `builtin:click`, `builtin:swoosh`, `builtin:swoosh-back`, `builtin:blip`, `builtin:error`, `builtin:chime`.
L'utilisateur peut couper les sons dans Réglages → Sons.

### 3.13 `music` — musique de fond
```jsonc
"music": {
  "tracks": ["assets/music/ambiance.ogg", "https://…/piste2.mp3"],
  "volume": 0.25,
  "shuffle": true,
  "fadeMs": 1500,
  "onlyOnHome": true,          // coupée hors du launcher
  "pauseWhenMediaPlays": true  // coupée si Spotify ou autre joue
}
```
Désactivée par défaut côté utilisateur (Réglages → Sons), même si le thème en fournit. Pistes : fichiers de l'archive (`assets/music/…`) ou liens `https://`.

### 3.14 `boot` — non géré pour l'instant
```jsonc
"boot": {
  "animation": "assets/lottie/intro.json",  // Lottie, 3 s max
  "sound": "…",
  "skipAfterFirstLaunchToday": true
}
```

### 3.15 `frames` — cadres
Remplace la forme simple d'un composant par un vrai cadre dessiné.
Cibles gérées : `panel` (grands panneaux), `card` (cartes Nuvio), `cardFocused` (carte sélectionnée), `pill` (pastilles, onglets), `pillActive` (pastille sélectionnée), `tile` (tuiles du dock), `strip` (bandeaux Spotify et SmartTube).
```jsonc
"frames": {
  "panel": {
    "corners": { "tl": "cut:18", "tr": "cut:18", "br": "cut:32", "bl": "round:8" }, // round:N | cut:N | notch:N | square ; ou "cut:14" pour les 4
    "stroke": { "width": 2, "color": "#3A3F4A", "dash": [6, 4], "double": { "gap": 4, "width": 1 } },
    "strokeGradient": { "angle": 0, "stops": ["#FFE600", "#FF2A6D"] },  // remplace la couleur du contour
    "glow": { "radius": 12, "color": "#FFE60066" },                       // néon permanent
    "notches": [ { "edge": "top", "at": 0.5, "length": 160, "depth": 8 } ], // encoches : top | right | bottom | left
    "fill": "{colors.surfaceGlass}"
  },
  "card": { "corners": "cut:14", "stroke": { "width": 2, "color": "{item.accent}" }, "glow": { "radius": 14, "color": "{item.accent}@50" } }
}
```
`{item.accent}` = couleur de l'élément (voir §3.19), avec suffixe d'opacité possible (`{item.accent}@50`).
Tailles en dp. Le néon est dessiné par contours successifs (pas de flou GPU) ; il est coupé en mode économie.
Non géré pour l'instant : cadres SVG étirables.

### 3.16 `decorations` — éléments décoratifs
Éléments posés **par-dessus** l'interface, jamais sélectionnables. 40 au maximum.
```jsonc
"decorations": [
  {
    "anchor": "screen.left",           // {ancre}.{bord}
    "offset": [0, 0],                  // décalage en dp (négatif = vers la gauche / le haut)
    "size": [12, "fill"],              // nombre (dp), "fill" (taille de l'ancre) ou "40%"
    "content": { "svg": "<svg …>…</svg>" },
    "repeat": "y",                     // none | x | y : motif répété (tuile carrée)
    "animation": { "type": "scroll", "speed": 14 },  // none | scroll | blink | pulse | flicker | rotate
    "opacity": 0.9
  },
  { "anchor": "nuvioPanel.topRight", "offset": [-24, 8], "content": { "text": "DATA_SCTM", "style": "caption", "color": "#FF2A6D" }, "animation": { "type": "flicker" } },
  { "anchor": "nuvioPanel.top", "size": ["30%", 4], "content": { "segments": ["#FFE600", "#FF2A6D", "#3A3F4A"] }, "animation": { "type": "pulse", "period": 2600 } }
]
```
Ancres : `screen`, `topBar`, `nuvioPanel`, `nuvioCard`, `profileTabs`, `spotifyStrip`, `smarttubeStrip`, `dock`, `dockTile`.
Bords : `topLeft`, `top`, `topRight`, `left`, `center`, `right`, `bottomLeft`, `bottom`, `bottomRight` (la décoration est placée à l'intérieur du bloc, le long de ce bord).
Une ancre répétée (`nuvioCard`, `dockTile`) place la décoration sur **chaque** exemplaire ; `{item.accent}` n'y est pas disponible (utiliser une couleur).
Contenu : `svg` (écrit dans le thème), `image` (fichier de l'archive ou lien), `text` (+ `style` : caption, body, title, hero ; + `color`), `segments` (barre de couleurs).
Performance : les animations de décor tournent à 15 images/s et se mettent en pause pendant la navigation (chaque image redessine tout l'écran). Préférer peu de décorations animées.
Animations : `speed` en dp/s pour scroll, tours/min ÷ 6 pour rotate ; `period` en ms pour blink et pulse. Toutes coupées en mode économie.

### 3.17 `shaders` — effet sur tout l'écran
```jsonc
"shaders": {
  "screenOverlay": { "agsl": "…lignes de balayage…", "opacity": 0.5, "uniforms": { } }
}
```
Shader AGSL dessiné par-dessus toute l'interface, en demi-résolution. Uniforms : `iTime`, `iResolution` (de la couche réduite). Doit renvoyer une couleur avec transparence (ex. `half4(0, 0, 0, 0.2)` sur une ligne sur deux).
Non géré pour l'instant : shaders par composant (`cardImage`, `focusEnter`, `pageTransition`).

### 3.18 `text` — effets de texte et libellés
```jsonc
"text": {
  "effects": {
    "clock": { "italic": true, "glow": { "radius": 10, "color": "#FFE600AA" } },
    "hero":  { "gradient": ["#FFFFFF", "#FFE600"], "uppercase": true }
  },
  "labels": {
    "nuvioKicker": "REPRENDRE SUR NUVIO //",
    "clockSubtitle": "NIGHT CITY LOCAL TIME"
  }
}
```
Effets par style (`clock`, `hero`, `title`, `body`, `caption`, `cardTitle`, `cardMeta`) : `italic`, `glow` (lueur, coupée en mode économie), `gradient` (dégradé horizontal), `uppercase`.

Libellés modifiables (valeur par défaut entre parenthèses) :
| Clé | Par défaut | Variables |
|---|---|---|
| `nuvioKicker` | Reprendre sur Nuvio | `{profile}` |
| `seriesBadge` | Série | `{title}` |
| `movieBadge` | Film | `{title}` |
| `spotifyKicker` | Spotify · {state} | `{state}` (en lecture / en pause) |
| `smarttubeKicker` | SmartTube · {state} | `{state}` (en lecture / reprendre) |
| `resume` | Reprendre | |
| `open` | Ouvrir | |
| `allApps` | Toutes les apps | |
| `clockSubtitle` | (vide) | ligne affichée au-dessus de la date |

### 3.19 `accents` — couleurs par élément
```jsonc
"accents": {
  "nuvioCards": ["#B967FF", "#FFE600", "#05D9E8"],  // 1re, 2e, 3e carte
  "profiles":   "fromProfile",                      // ou une liste de couleurs
  "dockTiles":  "fromAppIcon",                      // couleur extraite de l'icône, ou une liste
  "strips":     { "spotify": "#1EF2A0", "smarttube": "#FF2A6D" }
}
```
Dans `frames`, `{item.accent}` vaut la couleur de l'élément concerné (carte, tuile, bandeau).
`profiles: "fromProfile"` colore l'onglet de profil sélectionné avec la couleur du profil Nuvio.

---

## 4. Limites

| Élément | Limite |
|---|---|
| Fichier `.tvtheme` | 80 Mo |
| `theme.json` seul | 1 Mo |
| Images | 3840×2160 max (redimensionnées en 1080p) |
| Vidéo de fond | MP4 H.264/H.265, 1080p, 60 s, sans son |
| Musique | 10 pistes, 15 Mo chacune |
| Shaders | 1 fond + 1 effet ; coupés si l'image tombe sous 45 fps |
| SVG | Pas de scripts, pas de liens externes |

---

## 5. Transfert depuis le téléphone

1. Réglages → Thèmes → le launcher affiche un QR code (adresse locale + code à usage unique, valable 10 min).
2. Le téléphone ouvre une page servie par la TV, sur le réseau local uniquement.
3. Trois options : **déposer un fichier** `.tvtheme`/`.json`, **coller le texte** JSON, ou **coller un lien**.
4. La TV valide, affiche les avertissements, applique en aperçu → confirmation à la télécommande ou sur le téléphone.
