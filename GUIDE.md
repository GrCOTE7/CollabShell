# My CollabShell — Comprendre le codebase
<!-- no toc -->

- [L'architecture globale en une vue](#larchitecture-globale-en-une-vue)
- [Comment fonctionne la couche déclarative (à maîtriser absolument)](#comment-fonctionne-la-couche-déclarative-à-maîtriser-absolument)
- [Le "Data Flow" complet (le plus important pour contribuer)](#le-data-flow-complet-le-plus-important-pour-contribuer)
- [Le back-end / services (la partie "non-Flet")](#le-back-end--services-la-partie-non-flet)
- [Le design system (pour contribuer joliment)](#le-design-system-pour-contribuer-joliment)
- [Les raccourcis clavier (un "moteur" propre)](#les-raccourcis-clavier-un-moteur-propre)
- [Build multi-plateforme \& CI (pour comprendre "Multi platform")](#build-multi-plateforme--ci-pour-comprendre-multi-platform)
- [Les tests et la qualité](#les-tests-et-la-qualité)
- [Ce qui te manque techniquement pour contribuer (plan d'apprentissage)](#ce-qui-te-manque-techniquement-pour-contribuer-plan-dapprentissage)
- [Par où commencer pour contribuer (recommandations concrètes)](#par-où-commencer-pour-contribuer-recommandations-concrètes)
- [Mon verdict sur tes craintes de "ne pas être à la hauteur"](#mon-verdict-sur-tes-craintes-de-ne-pas-être-à-la-hauteur)
- [💡 Pour aller plus loin](#-pour-aller-plus-loin)

Tu l'as bien identifié : C'est un très beau projet. Voici ce qui est confirmé après exploration :

|         Fait          | Détail                                                                                                                  |
| :-------------------: | ----------------------------------------------------------------------------------------------------------------------- |
| **Langage / version** | Python **3.14** minimum (`requires-python = ">=3.14"`)                                                                  |
|   **Framework UI**    | **Flet ≥ 0.86.5** (avec les extensions `flet-ads`, `flet-cli`, `flet-terminal`)                                         |
|    **SDK backend**    | `google-colab-cli ≥ 0.6.0` (le wrapper Colab officiel), avec **`jupyter-kernel-client <1.0`** (pinning pour compat API) |
| **Gestion de projet** | **uv** (`pyproject.toml` + `uv.lock`), pas de `pip`/`requirements.txt`                                                  |
| **Multi-plateforme**  | Android (APK/AAB), Windows (exe Inno Setup), Linux (.deb/.rpm/.tar.gz) — le tout via **GitHub Actions CI/CD**           |
|       **Tests**       | `pytest` + linting **ruff**                                                                                             |
| **Licence / publié**  | Sur le Google Play Store (`ng.kiri.collabshell`), pub Kiri (Nwokike)                                                    |

**Point clé à retenir** : C'est un projet *conçu pour être contribué*. Il suit **de manière scrupuleuse** les patterns React/Flutter de la nouvelle API Flet déclarative. Chaque fichier a une docstring qui explique *pourquoi* il existe et l'architecture *proven* (ex: *« Follows the proven SpanInsight & DDGS architecture »*, *« Adapted from KTV Player's pattern »*). L'auteur a documenté les ressources manquantes et les contraintes avec un soin rare. C'est exactement le genre de codebase où on apprend énormément en contribuant.

**Collab Shell** est une app à architecture principalement déclarative (React-like), avec des poches d'impératif légitimes et localisées__ — principalement pour les dialogues/overlays (mécanique hors-arbor Flet), le page chrome (NavigationBar/FAB), et quelques effets de bord de lifecycle. Les `.update()` (~37) sont concentrés là, ne servent jamais à piloter le rendu des écrans principaux, et la majorité sont soit des patterns d'overlay standard, soit des rafraîchissements défensifs redondants avec l'observable.
Bref, ces **87% de ces appels touchent des dialogues/overlays/chrome — des couches hors-arbor déclaratif**, ou des effets de bord de lifecycle. C'est le même ratio qu'une vraie app React.



---

## L'architecture globale en une vue

```text
┌───────────────────────────────────────────────────────────────┐
│                        src/main.py                            │
│   AppController : bootstrappe services, restaure préférences  │
│   routing (deep links), lifecycle, puis monte AppShell        │
└──────────────────────────┬────────────────────────────────────┘
                           │ ft.run(main) → page.render()
                           ▼
                   ┌───────────────┐
                   │   AppShell    │  (racine déclarative : ⚙ @ft.component)
                   └──────┬────────┘
                          │ lit via ft.use_context()
   ┌──────────────────────┼───────────────────────────────┐
   ▼                      ▼                               ▼
  Onboarding        SessionScreen                   Dashboard (5 onglets)
(onboarding)      (notebook/terminal)            Home | Notebooks | Terminal
                              │                       | Files | Settings
                              │
   CONTEXTE :  ┌──────────────┴──────────────────────┐
               │  CONTEXTE / ÉTAT (l'ADN du projet)  │
               └───────┬────────────────┬────────────┘
                       │                │
           ┌─────────────▼───┐   ┌──────▼────────┐
           │  core/state     │   │   Services    │
           │  @ft.observable │   │  Colab/OAuth  │
           └─────────────────┘   └───────────────┘
```

**Le cœur du projet = trois "contextes" globaux** définis dans `src/state/` :

1. **`AppStateCtx`** → l'état observable global (créé dans `src/state/__init__.py` à partir de `core/state.state`).
2. **`ControllerMethodsCtx`** → les callbacks de navigation/actions (`navigate_tab`, `open_session`, `toggle_theme`…) exposés à l'UI. **Les composants ne manipulent jamais les services directement** ; ils passent par ces méthodes.
3. **`ServiceCtx`** → les instances de services back-end (`colab`, `storage`, `ad_service`, `page`).

> 💡 **C'est exactement le pattern "Provider" de React.** `ft.create_context(...)` = `React.createContext`, `ft.use_context(...)` = `useContext`.

---

## Comment fonctionne la couche déclarative (à maîtriser absolument)

C'est le sujet le plus important. Flet 0.86 a introduit une API inspirée de React. Voici chaque concept, avec un exemple réel tiré du code.

### a) @ft.component → composant fonctionnelel

```python
@ft.component
def HomeScreen() -> ft.Control:
    # La fonction "décrit" l'UI en fonction de l'état courant.
    # Flet re-rend automatiquement (de façon ciblée) quand un observable change.
    return ft.Column(controls=[...])
```

### b) ft.use_context(...) → lire un contexte globalal

```python
state = ft.use_context(AppStateCtx)              # lit l'état observable
controller = ft.use_context(ControllerMethodsCtx)  # lit les actions
services = ft.use_context(ServiceCtx)            # lit les services
```

Si un état *lu* ici change, le composant est re-rendu.

### c) ft.use_state(...) → état local au composantnt

```python
sessions, set_sessions = ft.use_state([])
is_loading, set_loading = ft.use_state(False)
```

### d) @ft.observable → l'état global réactifif

```python
@ft.observable
class AppState:
    current_tab: int = 0
    active_sessions: list = []
    theme_mode: ft.ThemeMode = ...
```

Quand tu fais `state.current_tab = 2`, **tous les composants qui lisent `state.current_tab` se re-rendent**. C'est le moteur de "surgical re-renders" mentionné dans le README.

### e) ft.on_mounted(...) / ft.use_effect(...) → effets de cycle de vie vie

- `ft.on_mounted(fn)` : exécute `fn` une fois au montage (ex: charger les sessions, enregistrer les raccourcis).
- `ft.use_effect(fn, deps)` : effet réactif qui se rejoue selon les dépendances (utilisé dans `AppShell` pour synchroniser la barre de navigation).

### f) ft.ValueKey(...) → clé de réconciliationon

```python
key=ft.ValueKey(f"session_{state.active_session_name}_{state.session_mode}")
```

Permet à Flet de savoir "quel sous-arbre réutiliser / détruire" quand l'état change (équivalent de `key` React).

---

## Le "Data Flow" complet (le plus important pour contribuer)

Voici le cycle de vie tel que reconstitué.

### E1 – Démarrage** (`AppController.init()`, `main.py`)

1. Configure la page : titre, polices (Outfit/RobotoMono), thèmes clair/sombre, dimensions min.
2. Enregistre des **services système** Flet : `FilePicker`, `Connectivity`.
3. Initialise les services métier : `StorageService` (persistance JSON), `AdService` (AdMob + consentement UMP), `ColabService` (wrapper SDK).
4. Restaure les préférences sauvegardées (`_restore_preferences`).
5. Configure le **routing** (`/home`, `/session?name=...`, `/settings`…) — **idempotent**, sert à la fois aux deep links et à la navigation interne.
6. Monte `AppShell`.

### E2 – Logiciel d'orientation "état gate"** (`AppShell`, déclaratif)

`AppShell` lit l'état et décide quoi rendre :

- pas `app_ready` → écran de chargement
- pas `onboarding_done` → onboarding
- pas `is_authenticated` → écran de connexion
- `active_subview` → plein écran (session / history)
- sinon → le dashboard à 5 onglets

### E3 – Une interaction utilisateur (le flux React)**

```text
Clic utilisateur
   └─► callback (ex: on_click=lambda e: controller.open_session(name, mode))
         └─► AppController.open_session()   [modifie l'état observable]
               state.active_subview = "session"
               state.active_session_name = name
               └─► Flet re-rend AppShell → remonte SessionScreen (déclaration renvoyée)
```

C'est le **paradigme clé** : *« on ne touche jamais l'UI directement ; on modifie l'état observable et le rendu découle de l'état. »*

---

## Le back-end / services (la partie "non-Flet")

### La couche ColabService — src/services/colab/ab/`

C'est un **wrapper asynchrone** autour du SDK `google-colab-cli`. Deux aspects à retenir.

**Lazy loading par sous-modules** : chaque méthode du service importe son implémentation paresseusement :

```python
async def check_auth(self) -> dict:
    from services.colab.auth import check_auth_impl  # import tardif
    return await check_auth_impl(self, ...)
```

C'est fait pour éviter les échecs d'import au démarrage (le SDK `colab_cli` est lourd). À conserver quand tu ajouteras des méthodes.

**Fichiers par domaine** :

- `auth.py` → OAuth2 (`get_auth_url`, `authenticate_oauth2`, `check_auth`, `clear_token`)
- `session_ops.py` → créer/lister/stopper/restarter les sessions
- `execution.py` → exécuter du code Python dans la VM Colab
- `files_ops.py` → upload/download/lister des fichiers
- `terminal_client.py` → connexions WebSocket PTY
- `logs.py`, `vm_ops.py`

**Le keep-alive** est géré **en process** (`_keep_alive_loop` dans `__init__.py`) : itération toutes les 60s, limite 24h, arrêt sur 2 erreurs 4xx consécutives. C'est une partie délicate (l'auteur gère finement les edge cases).

### L'authentification Google OAuth2 — src/services/colab/auth.pyy`

- Génère l'URL d'autorisation via `InstalledAppFlow.from_client_config` avec `remoteredirect`.
- Sauvegarde le **code_verifier** (PKCE) dans un fichier local.
- À la complétion : échange le code contre un token.
- `check_auth` **ne déclenche jamais de prompt terminal interactif** (critique sur mobile) — il relit le token, le refresh si besoin, puis valide auprès d'`oauth2.googleapis.com/tokeninfo`.

### Le stockage — StorageService (src/services/storage_service.py)py`)

- Stockage **JSON plat** (`storage.json`) avec **écriture atomique** (temp + rename) et **debounce de 1s**.
- Sauvegarde de sauvegarde `.bak`.
- Résout le répertoire selon la plateforme via `core/storage_patch.resolve_storage_dir()`.

### Les "patches" — core/storage_patch.py, stdin_hook.py, main.pyin.py`

C'est du **code méta-ingénierie** très intéressant, typique des apps packaging multi-plateforme :

- **`_install_bytes_safe_streams()`** (`main.py`) : wrapper des `sys.stdout/stderr` pour accepter les bytes (sinon `colab_cli` crash sur le runtime packaged).
- **`storage_patch`** : normalise la persistance entre Android et desktop.

---

## Le design system (pour contribuer joliment)

Tout est **centré sur des tokens** (cohérence garantie) :

- **`core/tokens.py`** → *les* valeurs : spacing, typographie, rayons, tailles d'icônes, opacités. **On n'écrit jamais de chiffre "en dur"** dans les composants ; on importe `tokens`.
- **`core/theme.py`** → palette `AppColors` (palette Colab ambre/orange), thèmes clair/sombre `AppTheme`, et helpers adaptatifs `adaptive_glass_bg/border`, `is_light_theme()` (gère le mode SYSTEM via `platform_brightness`).
- **`core/styles.py`** → fabriques de widgets réutilisables : `glass_card`, `solid_card`, `section_header`, `setting_tile`, `hardware_badge`, `status_dot`, `build_banner_ad`.
- **`core/constants.py`** → **toutes** les chaînes texto + options (labels, `ERR_*`, `TIP_*`, listes GPU/TPU…). Aucune chaîne magique dans les vues.

> 💡 **Règle d'or du projet** : quand tu crées une UI, tu assembles `tokens` + `styles` + `constants` + `theme`. Ne réinvente jamais une valeur.

---

## Les raccourcis clavier (un "moteur" propre)

`core/shortcuts.py` + `hooks/use_keyboard_shortcuts.py` :

- `Binding` : une combinaison normalisée (Ctrl == Meta sur macOS).
- `ShortcutRouter` : registre de fournisseurs **à priorité** (les plus récents shadow les anciens → les contextes internes comme Session shadow les globaux).
- sentinelle `SUPPRESS` : un contexte peut "voler" le clavier (cas du terminal).
- Le hook `use_keyboard_shortcuts` sauvegarde/restaure le handler précédent au (dé)montage pour éviter les empilements.

C'est le module le plus **testé** du projet : `tests/hooks/test_use_keyboard_shortcuts.py` et `tests/screens/notebook/test_shortcut_actions.py`.

---

## Build multi-plateforme & CI (pour comprendre "Multi platform")

Tout est dans `.github/workflows/build-all.yml` (déclenché par push de tag `v*`, PR, ou `workflow_dispatch` avec version/build_number) :

- **`prepare-release`** : crée un tag (`vX` ou `build-<sha>` pour les pré-releases) + placeholder GitHub Release.
- **`build-apk-split`** : construit l'**APK split par ABI** (`arm64-v8a`, `armeabi-v7a` via `--split-per-abi`), signé avec keystore décodé depuis un secret, puis **vérifie l'alignement 16 KB** (exigence Google Play) via `scripts/check_elf_alignment.py`.
- **`build-aab`** : AAB pour le Play Store (même vérif 16 KB).
- **`build-windows`** : portable + **installeur Inno Setup** (`.exe`), icône ICO générée avec Pillow.
- **`build-linux`** : bundle + packaging **`.deb` et `.rpm`** (via `fpm`) + `.tar.gz` universel, avec entrées desktop, icônes hicolor, et métadonnées AppStream.

`scripts/check_elf_alignment.py` est un excellent exemple de **script d'ingénierie de release** écrit proprement (analyse ELF `p_align`, gestion 32/64-bit).

---

## Les tests et la qualité

- `tests/conftest.py` : rend `src` importable (layout plat).
- Tests ciblés sur le **shortcut engine** et les actions notebook — pur logique métier, pas de test UI lourd.
- Outils : `pytest` + `ruff` (config dans `pyproject.toml`, `lint.extend-ignore = ["BLE001", "S110"]`).
- Pour lancer : `uv run pytest`, `uv run ruff check .`.

---

## Ce qui te manque techniquement pour contribuer (plan d'apprentissage)

Vu ton niveau (motivé mais modeste), voici les **4 piliers** à consolider, dans l'ordre :

| #   | Compétence                                                                     | Pourquoi                                                                                                                                                                              | Ressources                                                                                            |
| --- | ------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------- |
| 1   | **L'API Flet déclarative 0.86**                                                | C'est la base même du projet. Si tu maîtrises `@ft.component`, `use_state`, `use_context`, `use_effect`, `on_mounted`, `ValueKey`, l'observable → tu peux déjà lire et modifier l'UI. | Docs Flet "State management" (écosystème proche de React). Le code du projet est LE meilleur exemple. |
| 2   | **La programmation asynchrone Python** (`asyncio`, `to_thread`, `create_task`) | Tout le back-end est async : appels réseau, keep-alive, uploads. Omniprésent.                                                                                                         | Tutoriels asyncio Python.                                                                             |
| 3   | **Le pattern React/Flutter (Provider, état, rendu)**                           | Le projet en est une implémentation fidèle. Comprendre React rend le code évident.                                                                                                    | Concevoir mentalement comme React/Flutter.                                                            |
| 4   | **Le build/packaging Flet multi-plateforme** (CI, Inno Setup, fpm)             | Pour toucher aux releases, comprendre `pyproject.toml`/`uv` et le workflow GitHub.                                                                                                    | Docs "flet build".                                                                                    |

### 🔧 Commandes à connaître (pour te mettre à l'aise)

```bash
cd d:\c2\CollabShell
uv sync              # installer les deps
uv run flet run src  # lancer l'app en dev (desktop)
uv run pytest        # lancer les tests
uv run ruff check .  # lint
uv run ruff format . # format
```

---

## Par où commencer pour contribuer (recommandations concrètes)

Pour progresser sans casquer, vise des contributions **encapsulées** :

1. **Choisir un module isolé** : `core/constants.py` (ajouter un label), `core/tokens.py` (un token), un badge coloré, un texte de tip. Peu de risque, forte valeur de cohérence.
2. **Les tests** → c'est le meilleur point d'entrée : ajoute des tests sur des fonctions pures (`shortcuts.normalize_key`, `check_elf_alignment`…). Le projet te montrera exactement les attentes de style.
3. **Améliorer un composant présentational** : un `session_card`, une carte de paramétrage, un "empty state" — tout passe par l'API déclarative et `styles.py`/`tokens.py`.
4. **NE PAS commencer par** : `auth.py`, `terminal_client.py`, le keep-alive, ou le pipeline CI/Android. Ce sont les zones les plus sensibles et pointues.

---

## Mon verdict sur tes craintes de "ne pas être à la hauteur"

Relativise. Trois raisons de te rassurer :

1. **La qualité du projet facilite l'apprentissage.** Le code est *remarquablement bien documenté* — chaque docstring explique le "pourquoi", les contraintes platform, les patterns sources. C'est un codebase *fait pour être* lu et contribué, pas un mur hermétique.
2. **Les patterns sont standard.** Déclaratif React-like, tokens design, services async, providers : ce sont des concepts **du marché**, très documentés en ligne. Tu les réutiliseras dans d'autres projets.
3. **Toute contribution PR est une porte d'entrée.** Un petit fix propre + tests, soumis en bonne et due forme, t'apprendra énormément plus vite que tout cours.

**Conseil sincère** : commence par ouvrir un fichier `.py` du projet dans ton éditeur, lis-le entièrement (les docstrings surtout), et essaie de modifier une valeur de `tokens.py` ou un label de `constants.py` puis relance `uv run flet run src`. Rien qu'à voir le rendu changer grâce à un re-render déclaratif, tu auras fait un premier pas concret.

---

## 💡 Pour aller plus loin

- **Mini-schéma "Data Flow" annoté** pour un cas concret (ex: ouvrir une session).
- **Glossaire des concepts Flet/React** rencontrés dans le code.
- **Préparation de ta première PR** (choisir un module, comprendre le format attendu, rédiger la description).

*N'hésite pas à demander si tu veux que je développe l'un de ces points.*
