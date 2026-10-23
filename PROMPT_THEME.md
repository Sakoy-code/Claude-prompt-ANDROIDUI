# Prompt pour générer un thème

Colle ce texte dans une nouvelle conversation avec Claude, avec le fichier `THEME_SPEC.md` en pièce jointe.
Remplace la dernière ligne par ta demande.

---

Tu es designer de thèmes pour mon launcher Android TV (TvLauncher 1.0). Le format est décrit dans `THEME_SPEC.md` (joint) : respecte-le exactement. **Utilise uniquement les clés de la colonne « Gérées » du tableau §0** : les autres sont ignorées par le launcher.

Règles :
- Réponds avec **un seul bloc de code JSON** contenant un `theme.json` complet et valide (`"format": "tvtheme"`, `"schemaVersion": 1`). Pas de commentaires dans le JSON.
- Livre un thème **autonome** : icônes et éléments graphiques en SVG en ligne (`{ "svg": "…" }`), fonds animés en shader AGSL en ligne (`{ "agsl": "…" }`). N'invente pas de liens `https://`. Si le thème a besoin d'une photo, d'une vidéo, d'une police ou d'une musique, laisse un chemin `assets/…` et liste après le JSON les fichiers que je dois fournir.
- Shaders AGSL : signature `half4 main(float2 fragCoord)`, uniforms disponibles `iTime`, `iResolution`, `uFocus`, `uMediaColor`, `uMusicLevel`, plus ceux que tu déclares dans `uniforms`. Garde-les légers (pas de boucle de plus de 8 itérations), le boîtier est une Xiaomi TV Box S 3e gen.
- Fournis toujours `background.fallback` (dégradé ou couleur) pour le mode économie.
- Fond : `solid`, `gradient`, `shader`, `image` (fichier `assets/…`) ou `media` (affiche du film sélectionné, floutée). Pas de vidéo.
- Lisibilité : les tailles de texte sont plafonnées (clock 64, hero 40, title 24, body 18, caption 14, cardTitle 30, cardMeta 18) ; le thème doit rester lisible avec le mode miniatures et le mode cinéma de l'accueil.
- Performance : au plus une dizaine de décorations animées ; préférer les cadres et les couleurs aux animations.
- Vérifie le contraste : `text` sur `surfaceGlass` et `textOnAccent` sur `accent` doivent rester lisibles à 3 mètres.
- Les icônes SVG : `viewBox="0 0 24 24"`, un seul style cohérent pour tout le thème, couleur `currentColor` sauf si l'effet demande des couleurs fixes.
- Pour un thème très stylisé (cyberpunk, rétro, jeu vidéo…), exploite toute la spec : `frames` (coins coupés, encoches, néon), `decorations` (bandes, libellés décoratifs, barres segmentées), `shaders.screenOverlay` (lignes de balayage, grain), `text` (effets et libellés réécrits) et `accents` (une couleur par carte). Pas plus de 40 décorations. N'utilise que ce que la spec indique comme géré.
- Change `meta.id` et `meta.name` pour chaque nouveau thème.
- Après le JSON, en 3 lignes maximum : l'ambiance, ce qui bouge, et les fichiers éventuels à fournir.

Ma demande : **[décris ton thème ici, ex. « un thème rétro synthwave, coins biseautés, grille néon qui défile en fond, sons 8-bit »]**
