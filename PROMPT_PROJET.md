# Prompt pour recréer TvLauncher de zéro

À coller dans Claude Code (sur un PC sur le même Wi-Fi que la box), avec en pièces jointes
`themes/THEME_SPEC.md` et `themes/PROMPT_THEME.md`. Remplace les éléments entre `<…>`.

---

Je veux un launcher Android TV personnalisé, codé de zéro, qui remplace complètement l'accueil Google TV de ma box. Travaille sur le vrai appareil, étape par étape, et vérifie chaque changement par une capture d'écran prise sur la TV (`adb exec-out screencap -p`). Pose-moi les questions qui dépendent de mes goûts, avec des propositions. Réponds en français.

## Matériel et contraintes

- Box : <Xiaomi TV Box S 3e gen>, Google TV, Android 14 (API 34), 2 Go de RAM, écran 1080p à 60 Hz. Lis tout de même `getprop` pour confirmer.
- Le PC est sous Windows avec Android Studio installé (SDK et JDK `…\Android Studio\jbr`). Mets le projet dans un **chemin court** (ex. `C:\Users\<moi>\TvLauncher`) : les chemins longs cassent Gradle sous Windows. Utilise le Gradle wrapper.
- La performance est prioritaire : cible 60 images/s, mesure avec `dumpsys gfxinfo <pkg> framestats` (temps UI, rendu et GPU séparés) avant et après chaque effet.

## Pile technique

Kotlin + Jetpack Compose (foundation uniquement, pas de tv-material), minSdk 31, targetSdk 35, AGP 8.13, Kotlin 2.2, kotlinx-serialization, Coil 3 (+ coil-svg, okhttp), ZXing core (QR codes). Une seule activité `singleTask`, filtre HOME + LEANBACK_LAUNCHER, `<queries>` pour LEANBACK_LAUNCHER, LAUNCHER, `spotify:` et les paquets Nuvio/Spotify. Retour ne ferme jamais le launcher (il revient à l'accueil).

## Connexion ADB

- Débogage sans fil : `adb pair` avec le code, `adb connect`, puis `adb tcpip 5555` pour un port fixe. L'IP de la box et le port du débogage sans fil peuvent changer : retrouve-les avec `adb mdns services`.
- Sur Android 14, si `adb install` bloque, passe par `pm install`.
- Avant toute commande qui change l'écran, vérifie que je ne regarde pas quelque chose (`dumpsys activity activities | grep topResumedActivity`, `dumpsys media_session`). Après tes tests, remets l'écran, le thème et les réglages comme je les avais.

## Remplacer l'accueil Google TV — IMPORTANT

- **Ne désactive JAMAIS** `com.google.android.apps.tv.launcherx` ni `com.google.android.tungsten.setupwraith` : sur cette box, setupwraith prend la touche Home et l'écran reste noir ; `set-home-activity` répond « Success » sans effet.
- Solution qui marche : un **service d'accessibilité** du launcher, dans son **propre processus** (`android:process=":home"`) pour survivre à un plantage de l'UI :
  - `flagRequestFilterKeyEvents` + `canRequestFilterKeyEvents` : `onKeyEvent` intercepte `KEYCODE_HOME`, le consomme et ouvre le launcher (réglable) ;
  - secours : sur `TYPE_WINDOW_STATE_CHANGED` du paquet launcherx, rouvrir le launcher (max 1/s) ;
  - dans `onServiceConnected`, vérifier `rootInActiveWindow` (démarrage de la box, mise à jour de l'app) ;
  - activation par `settings put secure enabled_accessibility_services` en **conservant** les services existants.
- Les appuis simulés par `adb shell input keyevent 3` ne passent pas par l'accessibilité : teste la touche Home avec la **vraie télécommande** et un journal.
- Vérifie qu'aucun autre service d'accessibilité (ex. **Button Mapper**) ne relance Google TV ou un ancien launcher sur Home (`logcat` : « START … launcherx … from uid »). Demande-moi avant de le désactiver.

## Accueil (disposition validée)

Fond selon le thème. De haut en bas :
1. **Barre du haut** : heure + date (sous-titre optionnel du thème), onglets en pastilles Accueil / Apps / Réglages, à droite statut Wi-Fi (rafraîchi : au démarrage le Wi-Fi arrive après le launcher), Bluetooth, paramètres Android.
2. **Bloc Nuvio** : avatar + nom du profil, onglets de profil avec photos (masqués s'il n'y a qu'un profil), 3 cartes horizontales (image de fond, étiquette SÉRIE/FILM, titre, « S2 · É4 — nom de l'épisode · reste 23 min » seulement sur la carte sélectionnée, barre de progression). La carte sélectionnée est plus large et plus haute (animation). Mise en colonne : étiquette en haut, textes en bas, jamais de chevauchement.
   - **Mode miniatures** (réglable) : quand la sélection n'est pas sur Nuvio, l'en-tête disparaît et les cartes s'agrandissent.
   - **Mode cinéma** (réglable) : quand la sélection est sur Nuvio, la carte sélectionnée s'élargit encore, Spotify/SmartTube deviennent une fine barre (pochette, titre, boutons) et le dock s'efface.
3. **Bandeaux Spotify et SmartTube** côte à côte (chacun masquable) : pochette/miniature avec fondu + zoom au changement, titre défilant s'il est trop long, petit égaliseur animé pendant la lecture, artiste et temps séparés (le temps reste toujours lisible). Spotify : précédent / lecture-pause / suivant. SmartTube : « Reprendre ».
4. **Dock** : les apps choisies (pas de tuile « Toutes les apps »). Touches 1–9 = apps du dock, Menu = page Apps.

N'ajoute **pas** de rangée de playlists ni de bandeau Spotify qui s'agrandit : ça redimensionne Nuvio et `LazyRow` dans un `Row` en `IntrinsicSize` fait planter Compose.

## Nuvio (Nuvio Sync, backend Supabase)

- Serveur `https://api.nuvio.tv` (secours `https://api-two.nuvioapp.space`). La clé publique « anon » n'est pas dans le code source : **demande-moi l'autorisation**, puis lis-la dans l'APK de Nuvio installé (`pm path com.nuvio.tv`, `adb pull`, chercher `eyJhbGciOiJIUzI1NiIs` dans les `.dex`), et supprime la copie de l'APK ensuite. En-têtes : `apikey` + `Authorization: Bearer <jeton>`.
- Connexion par QR code (comme Nuvio TV) : RPC `start_device_login_session` (p_device_nonce, p_redirect_base_url `https://nuvio.tv/link`, p_device_type `tv`, p_device_name) → afficher `verification_uri_complete` en QR + `user_code` ; sonder `poll_tv_login_session` (p_code = device_code, p_device_nonce) ; si `approved`, échanger via `POST /functions/v1/tv-logins-exchange` {code, device_nonce} ; renouvellement `POST /auth/v1/token?grant_type=refresh_token`. Nouveau code automatiquement à l'expiration.
- Données par profil : `sync_pull_profiles` (profile_index, name, avatar_color_hex, avatar_url, avatar_id), `sync_pull_watch_progress` (p_profile_id → content_id, content_type, video_id, season, episode, position, duration, last_watched), `sync_pull_watched_items` (épisodes vus).
- Avatars : `avatar_url`, sinon `get_avatar_catalog` (`/storage/v1/object/public/avatars/<storage_path>`), sinon `get_member_profile_avatar_catalog` (téléchargement authentifié `/storage/v1/object/authenticated/membership-profile-avatars/<path>`, mis en cache).
- Reprises : progression entre 2 % et 92 %, la plus récente par titre, 3 maximum. **Épisode suivant** : si le dernier épisode vu d'une série (vue < 30 jours) est terminé, proposer le suivant non vu et déjà sorti (Cinemeta `videos[].released`). Horodatages parfois en secondes, parfois en millisecondes : normaliser.
- Métadonnées : Cinemeta `https://v3-cinemeta.strem.io/meta/<type>/<id>.json` (titre, background, poster, logo, noms d'épisodes), cache disque 7 jours.
- Lancement : `Intent(ACTION_VIEW)` vers `com.nuvio.tv/com.nuvio.tv.MainActivity`, extras `contentId`, `contentType`, `videoId` (ou `id:saison:épisode`), `name`, `poster`, `backdrop`, `logo`, `season`, `episode`, `episodeTitle`, `launchMode=stream`, flags `NEW_TASK | CLEAR_TASK`. Nuvio affiche son choix de profil (je dois désactiver « se souvenir du dernier profil » dans Nuvio). Il n'existe pas d'extra pour choisir le profil.
- Rafraîchir à chaque retour sur le launcher. Réglage : profils affichés (au moins un).

## Lecture en direct (Spotify, SmartTube) via les sessions média

- `NotificationListenerService` vide + `MediaSessionManager.getActiveSessions`, autorisé par `adb shell cmd notification allow_listener <pkg>/<service>`.
- Pièges vérifiés :
  - Spotify met son **logo** dans `METADATA_KEY_DISPLAY_ICON` : ne jamais l'utiliser comme pochette ; garder la dernière pochette tant que le titre ne change pas.
  - SmartTube rouvert par un lien publie des métadonnées vides, puis `duration = -1` et `position = 0` à la fermeture : garder titre, durée et position précédents.
  - Quand la lecture passe sur un autre appareil (Spotify Connect) la session disparaît ou se fige : figer la position ; considérer « en lecture » seulement si la position extrapolée < durée + 5 s.
  - Si la session n'est plus active, envoyer les commandes en touches média (`AudioManager.dispatchMediaKeyEvent`).
- Mémoriser sur disque la dernière lecture Spotify et SmartTube (pochette réduite comprise) pour l'afficher après un redémarrage.
- SmartTube : `METADATA_KEY_MEDIA_ID` = identifiant YouTube ; miniature `https://i.ytimg.com/vi/<id>/mqdefault.jpg` ; reprise par `https://www.youtube.com/watch?v=<id>&t=<s>s` avec le paquet `org.smarttube.stable`.
- Historique Spotify (optionnel) : API Web en OAuth PKCE avec mon propre Client ID (mode développement : propriétaire Premium obligatoire, adresse de retour `http://127.0.0.1:8888/callback`).

## Page Apps

Grille de toutes les apps (bannière TV sinon icône), tri « mon ordre / A-Z / récentes / masquées », mise à jour automatique à l'installation ou la désinstallation (BroadcastReceiver), icônes du dock chargées en premier et en parallèle. Appui long → menu : ajouter/retirer du dock, déplacer, masquer, infos, désinstaller, **Annuler**. Le menu s'ouvre touche OK encore enfoncée : **ignorer OK jusqu'à son relâchement**, sinon la première action se déclenche seule. Mode déplacement : flèches = déplacer, OK = valider, Retour = **annuler** (restaurer l'ordre) ; la sélection reste sur l'app déplacée.

## Réglages

Menu latéral : Thèmes, Accueil, Comptes (Nuvio, Spotify), Télécommande, Réseau (type, réseau, IP, signal en mots), Bluetooth (appareils associés ; `BLUETOOTH_CONNECT` accordé par ADB), Sons, Performance (mode économie, mémoire), Système Android (raccourcis vers les paramètres), À propos (version, appareil, procédure de retour à Google TV). Navigation encadrée (`focusGroup` + `focusProperties`) : droite = premier élément du contenu, gauche = retour sur la section, pas de saut en diagonale. Contenus défilants avec une petite marge pour que l'élément agrandi ne soit pas coupé.

## Thèmes

- Format décrit dans `THEME_SPEC.md` (joint) : implémente exactement ce qui y est marqué « géré » dans le tableau §0, et mets ce tableau à jour à la fin. `PROMPT_THEME.md` (joint) sert à générer des thèmes avec Claude.
- Lecture tolérante : section invalide = valeurs par défaut + avertissement, jamais de plantage. Tailles de texte plafonnées. Encoches plus longues que le bord : les raccourcir (sinon plantage). Marges intérieures calculées selon les coins coupés/encoches du cadre (`frameInset`).
- Envoi depuis le téléphone : petit serveur HTTP local ouvert uniquement pendant l'écran QR, code à usage unique dans l'URL, page web (fichier `.json`/`.tvtheme`, texte collé, lien). Archives ZIP : refuser les chemins `..`, gérer les dossiers écrits avec `\` (archives Windows) et un dossier racine. Aperçu 15 s avec « Garder » / « Annuler », retour automatique sinon. Supprimer un thème reçu par appui long.
- Sons intégrés synthétisés en WAV au premier lancement (`builtin:tick`, `click`, `swoosh`, `chime`…), musique de fond coupée par défaut, seulement sur l'accueil, en pause si une autre app joue (attention : dans `onCompletionListener`, appeler le `release()` du lecteur de musique et non celui du `MediaPlayer`).
- Livre 3 thèmes : « Verre dépoli » (défaut, sombre, verre mat, fond shader qui prend la couleur du titre sélectionné), « Néon nuit », « Neon City » (démo cyberpunk : coins coupés, néon par carte, bandes de danger, lignes de balayage).

## Leçons de performance (à appliquer dès le départ)

- Shader de fond calculé en demi-résolution dans une couche réduite puis agrandie, 30 i/s max ; fonds fixes (dégradé, vignettage) dessinés une fois dans une image en cache.
- Pas de `clip` par panneau ; reflet du verre limité au tiers haut.
- Animation de sélection uniquement dans `graphicsLayer` / `drawWithContent` (aucune recomposition) ; la couleur du média passe par une lambda lue au dessin.
- Contours et néons des cadres précalculés en image ; néon = contours successifs, pas de flou GPU.
- Décorations animées à 15 i/s, en pause pendant 1,5 s après chaque touche de la télécommande.
- Un `RuntimeShader` ne peut pas être dessiné dans une image en mémoire (rendu logiciel) : le dessiner directement, en demi-résolution.
- Interrupteurs de diagnostic par ADB (`--es perf "nobg,noglass,noframes,nodecor,nooverlay"`) pour mesurer le coût de chaque effet. Mode économie qui coupe les effets lourds.

## Livraison

- Signature de publication avec une clé fixe **hors du dépôt** (chemin dans `local.properties`, repli sur la clé de debug) et versionName 1.0.0.
- `outils/installer.ps1 <ip:port>` (installation, autorisations média/Bluetooth/localisation, service d'accessibilité en conservant les autres) et `outils/retour-google-tv.ps1 <ip:port> [-Desinstaller]`, en UTF-8 avec BOM.
- README : fonctions, compilation, installation, retour à Google TV, signature, procédure pour fermer ADB (désactiver le débogage sans fil et USB, révoquer les autorisations, redémarrer, vérifier avec `Test-NetConnection <ip> -Port 5555`).
- Git dès le début, avec des commits par étape ; `captures/`, `*.apk`, `local.properties` ignorés (les captures montrent mon historique).
- Test final : **redémarrer la box** et vérifier que le launcher s'ouvre seul, puis appui Home réel.

## Ordre de travail

1. Vérifier l'environnement, créer le projet, moteur de thèmes minimal + accueil avec données de démo, installer, mesurer.
2. Nuvio (connexion, reprises, épisode suivant, avatars, lancement).
3. Spotify et SmartTube en direct.
4. Envoi de thèmes par QR.
5. Apps (dock, ordre, masquer) et Réglages complets.
6. Thèmes avancés (fonds image/média flouté, icônes, sons, musique, cadres, décorations, effets de texte, accents) + thème Neon City + mesures.
7. Remplacement de l'accueil (accessibilité), finitions, signature, scripts, README, test de redémarrage.
