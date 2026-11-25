# Le moteur CPU dans le portfolio (Emscripten)

## Comment la page lance le moteur

`index.html` charge `demo/renderer.js` une seule fois, puis appelle `createCpuEngine()` deux fois :

| Instance | Canvas | argv[1] | Lancement |
|---|---|---|---|
| La théière | `#theiere` (en-tête) | `Assets/teapot.cfg` | au chargement de la page |
| La démo « 1992 » | `#demo` (section Moteur de rendu CPU) | `Assets/demo.cfg` | au clic sur le bouton (autorise le son) |

`argv[2]` est le sélecteur du canvas : `main` le met dans `AppConfig::canvasSelector`, et `Window::create` le donne à SDL
(`SDL_HINT_EMSCRIPTEN_CANVAS_SELECTOR` et `SDL_HINT_EMSCRIPTEN_KEYBOARD_ELEMENT`) avant de créer la fenêtre.

La page remet aussi son propre titre d'onglet quand SDL écrit « CPU Engine » dedans.

## Mettre à jour le site après une modification du moteur

1. Dans le dossier du moteur : `cmake --build build-web`
2. Copier `bin/renderer.js`, `bin/renderer.wasm` et `bin/renderer.data` dans `demo/` à la racine du site
3. Tester depuis la racine du site : `python -m http.server 8000`, puis http://localhost:8000
   (outils du navigateur ouverts, onglet Réseau, « Disable Cache » coché)

## Ce qui est déjà fait dans le moteur

- `main` lit `argv[1]` (fichier de démo) et `argv[2]` (sélecteur de canvas)
- `Application::run()` découpé : `tick()` + `emscripten_set_main_loop_arg` sur le web
- `music = none` dans un `.cfg` : pas de son, `analyze()` renvoie un silence
- `updateTitle()` désactivé sur le web
- Build web : port SDL3 (`-sUSE_SDL=3`), `-sMODULARIZE=1 -sEXPORT_NAME=createCpuEngine`,
  `--preload-file Assets@/Assets`, sortie `renderer.*`
- Mono-thread sur le web : `hardware_concurrency()` vaut 1, le ThreadPool travaille sur le thread principal

## Reste à faire

- **Échap sur le web** : aujourd'hui il arrête définitivement le moteur (`emscripten_cancel_main_loop`).
  Les visiteurs appuient sur Échap par réflexe : l'ignorer sous `__EMSCRIPTEN__`.
- **F1 (caméra libre)** : dans le navigateur, F1 ouvre l'aide. Si tu veux ce mode sur le web, prends une autre touche.
- **Pause hors de l'écran (facultatif)** : la page appelle `_cpu_pause()` / `_cpu_resume()` si elles existent :
  ```cpp
  #ifdef __EMSCRIPTEN__
  extern "C" EMSCRIPTEN_KEEPALIVE void cpu_pause()  { emscripten_pause_main_loop(); }
  extern "C" EMSCRIPTEN_KEEPALIVE void cpu_resume() { emscripten_resume_main_loop(); }
  #endif
  ```
  Au retour, `dt` est déjà limité à 0,1 s dans `tick()` : pas de saut d'animation.
