# Passation — Demiurge

Tu reprends le projet **Demiurge** pour Ple (GitHub `dodz2`). Parle-lui en **français**. Tu n’agis jamais à sa place sans son accord (surtout merge, deploy, fan-out). Lis ce fichier en entier avant de toucher au code.

---

## 1. Une phrase

Demiurge est un **sandbox spatial 3D pixel art** (N-body maison) : le joueur crée, déplace, modifie et détruit des corps, puis observe les conséquences. Inspiration Universe Sandbox, DA pixel art volumétrique, pas un filtre.

**Source de vérité :** `ROADMAP.md`. Toute évolution de périmètre passe par ce fichier **avant** le code. Journal des décisions = **§11**, inliné dans la même PR (un sidecar seul est un **BLOCKER** Reviewer).

---

## 2. Repo et accès

| | |
|---|---|
| Dépôt | **https://github.com/dodz2/demiurge** — **privé** |
| Compte | `dodz2` |
| Branche de vérité | `main` |
| HEAD (29 août 2026, ~21:08 Europe/Paris) | `e56fa60b787cb47af68a23f94c5577fb09042e91` |
| Preview jouable | **https://dodz2.github.io/demiurge/** (public). Le source reste privé ; seule cette page est publique, dans `dodz2/dodz2.github.io` sous `demiurge/` (build Vite avec base `/demiurge/`). |

### Commits squash sur main

- P4 `9161284` — scènes JSON + calibration (21 août 2026)
- P5 `4906759` — PR #1, pipeline pixel art
- P6 `f7475cb` — PR #2, caméra multi-échelles
- P7 `e56fa60` — PR #3, sélection / temps / panneau

Branches feature encore sur le remote (ne plus les prendre comme source de vérité) :

- `feat/phase-5-pixel-art-render`
- `feat/phase-6-multi-scale-camera`
- `feat/phase-7-selection-time-ui` — ne pas rebaser dessus : la tip a parfois un ROADMAP **tronqué**. `main` est intact.

### Outillage (critique)

- **Pas de Cloud Agent Cursor** : SuperGrok n’est pas Cursor Pro. Tout passe par GitHub MCP (`user-Github`) + API Contents.
- **Ne pas git clone** sauf demande explicite de Ple.
- MCP create_or_update_file rate souvent sur les gros fichiers (ROADMAP ~49 Ko, package-lock ~95 Ko). Workaround : PUT Contents API. Un PUT trop petit a **tronqué ROADMAP au blockquote** (NO-GO Ops P7) : vérifier taille et §11 après chaque PUT.
- CLI `gh` pas forcément loggée. Connecteur GitHub de Ple **installé**.
- Self-approve GitHub impossible (auteur = dodz2). Reviewer = COMMENT interne, pas APPROVE.
- **Ne merge que si Ple a dit GO** (ou « une fois que tout est bon »). Méthode : **squash**.
- Langue : français. Commits conventionnels `feat(phase-N):`. Timezone : Europe/Paris.

### Équipe (fan-out autorisé sur Demiurge)

Ple a demandé d’appeler les autres agents à chaque phase.

- **Ops** — checklists GO/NO-GO, ordre, critères de merge
- **Coder** — brief fichiers / API / tests / pièges
- **Reviewer** — grille BLOCKER / SHOULD / HORS SCOPE ; **attendre son verdict avant squash** (P6 a été mergé trop tôt)

Ne lance **pas** de Cloud Agent depuis Coder (STOP déjà nécessaire en P7).

---

## 3. Stack et architecture

TypeScript strict, Vite 8, Vitest 4, Three.js 0.185.x, ESLint + Prettier, CI GitHub Actions (typecheck, lint, test, build, Node 24).

Layout :

- `src/sim` — simulation pure. Zéro import three / DOM / UI. Frontière lint + purity.test.ts
- `src/render` — snapshots vers meshes. RT 480×270 nearest. SCALE unique
- `src/camera` — caméra orbitale custom (pas d’OrbitControls). Sans three
- `src/input` — pointer : orbite vs pick. Picking headless
- `src/ui` — overlay DOM natif (pas de framework) : panneau, barre temps, tooltip, sélection
- `src/scenes` — JSON versionné + loader/saver
- `src/main.ts` — câblage uniquement

Unités sim : distance Mm, masse kg, temps s. G et softening **par scène**. dt jeu = 1/120 s. Intégrateur velocity Verlet. Collisions sphères balayées, résolution par **fusion inélastique** (plus petit id survit).

Flux :

- UI / Input envoie des **commandes** à la Simulation (seule proprio de l’état)
- Simulation envoie des **snapshots** lecture seule vers Render / UI
- Simulation envoie des **événements** (collisions) vers Render. Les FX = P10 ; aujourd’hui on drain sans FX

Une PR qui importe du rendu/DOM dans `src/sim` est rejetée. En P5–P7 on a exigé **src/sim 0 diff** (SHAs blob identiques vs main).

Rendu pixel art (ne pas casser) :

- Render target 480×270, NearestFilter, generateMipmaps false, setPixelRatio(1)
- Upscale = blit nearest letterbox 16:9, pas de shader pixelate, pas d’upscale CSS
- SCALE = 1 uniquement dans `src/render/scale.ts` (1 unité scène = 1 Mm). Meshes via toScenePosition / toSceneRadius. Jamais mesh.position pour follow ou picking
- Une seule PerspectiveCamera. aspect toujours RT_ASPECT, jamais innerWidth/innerHeight
- Fond spatial généré à l’intérieur du RT

Scène par défaut : `src/scenes/data/systeme-simple.json` — étoile Heliandre id 1 + planètes (Braise id 2, etc.), G calibré, orbites 45 / 75 / 105 Mm.

---

## 4. Où on s’est arrêté

**Phases 0 à 7 sont mergées sur main.** Prochaine phase produit : **P8 — outil de création de corps**. Ne pas commencer P8 (ni P9–P11, ni les SHOULD) sans **GO explicite de Ple**.

### P0–P4

Socle npm / TypeScript / Vite / CI. Simulation pure, gravité N-body, collisions/fusions + file d’événements, scènes JSON, round-trip, calibration gameplay. Demos headless phase 1 à 4.

### P5 — pipeline pixel art (PR #1, squash `4906759`)

Three.js confiné à src/render. PixelArtRenderer consomme getSnapshot(). Palette par catégorie (star #FFD166, planet #5B8CDE, asteroid #A8A29E) + visualColor. Caméra encore statique +Z, lookAt origine. L’étoile paraissait hors-centre (les planètes tirent l’étoile). Corrigé en P6 par le follow id 1.

### P6 — caméra multi-échelles (PR #2, squash `f7475cb`)

Module src/camera sans Three.js. Interdit OrbitControls / TrackballControls.

- Sphérique yaw / pitch / distance / target / panOffset
- Amortissement exponentiel λ = 12 s⁻¹, dt **frame**, jamais dt sim
- Zoom exponentiel k = 0.001
- Pan et déplacement monde proportionnels à la distance
- followTarget ancre lookAt sur toScenePosition(snapshot.position) + panOffset. Jamais mesh.position. Id absent/fusionné → unfollow propre
- Défaut : étoile id 1, distance 260, dès la frame 0
- near = max(1e-4, distance × 0.02), far = distance × 80. Pas de floating origin, pas de logarithmicDepthBuffer
- Renderer : applyView(pose) seulement, aspect = RT_ASPECT
- Pointer/wheel : même mapPointerToLetterbox que le blit ; barres noires ignorées
- Test headless follow 300 s (orbite synthétique + Braise / systeme-simple), dérive < 1e-9
- Constantes : DEFAULT_DISTANCE 260, DEFAULT_FOLLOW_ID 1, MAX_DISTANCE 2000, ORBIT_RADIANS_PER_PIXEL 0.005, PITCH_LIMIT ≈ 85°, CAMERA_FOV 50

Fichiers : src/camera (constants, math, orbital-camera, input, index, tests), src/render/letterbox.ts, pixel-renderer.ts, main.ts, README, docs/journal-phase-6.md.

### P7 — sélection, temps, panneau (PR #3, squash `e56fa60`) — dernier livrable

Livrable : « je sélectionne → je lis → j’accélère → j’observe ».

- **P7-1** Picking headless `src/input/picking.ts` : rayon depuis CameraPose, NDC x = nx×2−1, y = 1−ny×2, sphère toScenePosition + toSceneRadius(visualRadius). Pas de Raycaster / mesh. Highlight PixelArtRenderer.setSelection(id) emissive Lambert 0x222222 → 0x886622, sans recréer le mesh.
- **Clic vs drag** Seuil 5 px letterbox (CLICK_MOVE_THRESHOLD_PX). Coordinateur src/input/pointer.ts. Clic gauche < 5 px = pick (ou clear si miss). ≥ 5 px = orbite P6. Droit / milieu / Maj = pan. Molette = zoom. Letterbox miss = ignore.
- **P7-2** src/ui/info-panel.ts : nom, catégorie, masse, rayons, |v|, distance = |position| depuis l’origine en Mm. Snapshot only.
- **P7-3** src/ui/time-controls.ts : Pause / ×1 / ×10 / ×100. loop.update(elapsed × timeScale, world.step). **Même dt** (1/120). Pause = 0 sous-pas. camera.update(snapshot, elapsed) = dt **frame** réel. Espace = pause, 1/2/3 = warps. maxStepsPerUpdate 4096 (préexistant).
- **P7-4** Tooltip nom, pointer-events none. Hover ≠ sélection.
- **P7-5** src/ui/selection.ts : id disparu/fusionné → null, panneau vide, pas de resélection auto du survivant, pas de throw.
- **Follow** inchangé au pick (étoile P6). Clic vide clear sélection/highlight, pas le follow.

Tests au merge : **112 / 17 fichiers** (85 P0–P6 + 27 P7). src/sim 0 diff vs f7475cb. CI verte.

HUD : `DEMIURGE - phase 7 · {fps} fps · ×{scale}`. Barre temps / panneau : pointer-events auto. index.html : #info-panel, #time-bar, #tooltip.

---

## 5. Contrôles joueur (état actuel)

- Clic gauche sans glisser : sélectionner / désélectionner
- Glisser gauche : orbite amortie
- Molette : zoom exponentiel
- Droit / milieu / Maj+gauche : pan proportionnel à la distance
- Barres noires 16:9 ignorées
- Barre Pause / ×1 / ×10 / ×100
- Survol : nom

Local : npm install puis npm run dev (Vite, souvent localhost:5173). Ou la preview Pages ci-dessus.

---

## 6. Backlog SHOULD (pas bloquant, ne pas toucher sans GO Ple)

Issu des reviews P6/P7 (Ops / Coder / Reviewer) :

1. **Micro-orbite** — orbit() part dès le premier pointermove, même si maxMovement < 5 px. Un clic tremblé = orbite et pick. Fix : bufferer l’orbite jusqu’au seuil. Coder + Reviewer d’accord, pas un BLOCKER.
2. applyView force CAMERA_FOV / RT_ASPECT alors que le picking lit pose.fov / pose.aspect. Alignés aujourd’hui ; demain ça diverge = pick ≠ pixels.
3. Sidecars redondants avec §11 : docs/journal-phase-6-roadmap-rows.md, docs/journal-phase-7-roadmap-rows.md. Supprimables après GO.
4. Vec3 tripliqué : src/sim/vec3.ts, src/camera/math.ts, src/input/picking.ts. NIT, ne pas fusionner les modules à la légère (frontière sim).
5. Hitch à ×100 : ~192 pas/frame à 60 fps OK ; un freeze ~0,34 s tape maxStepsPerUpdate 4096 et jette du temps simulé.
6. Pas de test picking aux zooms extrêmes (P7-1 « tous les zooms »). Les tests couvrent NDC / nearest / miss / visualRadius / t<0 / letterbox.
7. Capture manuelle P6 jamais faite : zoom planète → système → planète (critère visuel ROADMAP).
8. Ligne journal P6 « Pas de picking (P7) » périmée.
9. isPointerClick surtout utilisé dans les tests.
10. CAMERA_NEAR / CAMERA_FAR restent des défauts constructeur ; applyView les écrase chaque frame.

---

## 7. Ce qu’il reste (ROADMAP) — ne pas anticiper

Graphe : P8 → P9 → P10 → P11. Rien hors phase courante. Idée séduisante → section Post-MVP du ROADMAP, pas du code.

### Phase 8 — création de corps (prochaine)

Geste fondateur : placer + drag de vitesse. Livrable : un novice crée en 2 min une planète qui orbite.

- P8-1 Barre 3 presets (étoile / planète / astéroïde), masses/rayons documentés
- P8-2 Clic = position sur un plan orbital
- P8-3 Drag = direction + amplitude, flèche vectorielle pixelisée
- P8-4 Release = commande atomique spawn, vitesse = drag à epsilon près
- P8-5 Échap + aperçu visuel ; rien de corrompu après cancel

L’aperçu pendant le drag est purement visuel. Le corps spawné devient sélectionnable. Ne pas muter la sim depuis l’UI autrement que par commande. Hors scope : presets exotiques, température, placement relatif.

**Piège P7 :** le clic gauche sert déjà au pick et à l’orbite. P8 devra un **mode outil** (barre) pour que clic+drag = spawn, sans casser pick/orbite. Trancher avec Ops/Coder dans le ROADMAP §11 avant de coder.

### Phase 9 — édition et suppression

Slider log masse, rayon physique (+ visuel cohérent), facteur de vitesse, suppression du sélectionné. Chaque edit = commande atomique. Éditer masse/rayon ne change jamais la vitesse tout seul. Guards > 0. Suppression propre pendant une collision. Pas d’undo, pas de multi-corps, pas de température.

### Phase 10 — feedback visuel

Consommer enfin drainEvents() : impact pixelisé + secousse caméra proportionnelle à l’énergie, traînées (côté rendu), vecteurs vitesse toggle, skins pixel 2–3 par catégorie, lumière depuis l’étoile. Aucun FX ne mute la sim. Pas de sons, débris physiques, anneaux, atmosphères.

### Phase 11 — polish et DoD MVP

Scène d’ouverture calibrée, build prod, plafond de corps, aide 3–5 lignes, playtest scénario ROADMAP §2.1, run 30 min ×100. Cocher toute la section 7 du ROADMAP.

Post-MVP (ne pas négocier maintenant) : floating origin, Barnes-Hut, prédiction d’orbites, fragmentation, audio, éditeur, save UI, etc.

---

## 8. Processus à reproduire (P5–P7)

1. Ple dit GO Phase N.
2. Brief Ops (checklist, ordre, DoD merge), Coder (fichiers, API, tests, pièges), Reviewer (grille BLOCKER). Fan-out OK sur ce projet.
3. Branche feat/phase-N-… depuis main actuel. Peu de commits (P6/P7 ont dérivé à 13–17 commits bruyants ; le squash sauve l’historique).
4. Implémenter uniquement PN-x. Préserver P5–P7. src/sim 0 diff sauf besoin réel P8+ (alors commandes atomiques, pas d’écriture Body depuis l’UI).
5. Tests headless sans WebGL. Garder les 112 existants verts. typecheck + lint + test + build.
6. README (statut + contrôles) + ROADMAP §11 **inliné** (vérifier que le fichier n’est pas tronqué : plusieurs centaines de lignes, table P0…PN). Journal docs/journal-phase-N.md en plus, pas à la place.
7. PR vers main, body FR, « Ne pas merger » le temps des reviews.
8. CI verte + GO Ops + GO Reviewer (0 BLOCKER) → squash merge. Ple a validé ce rythme.
9. Ping Ops / Coder / Reviewer avec le lien PR puis avec le SHA squash. Pas de phase suivante tant que Ple n’a pas dit GO.
10. Preview Pages : rebuild Vite avec base /demiurge/ et pousser dist/ vers dodz2/dodz2.github.io chemin demiurge/ si Ple veut un lien cliquable.

BLOCKERS types Reviewer : casser l’orbite P6, UI qui mute la sim, ×100 qui change les trajectoires, sélection fantôme après fusion, RT/SCALE/sim cassés, pick dans le letterbox, Raycaster Three, panneau ≠ snapshot, tests rouges, §11 absent ou ROADMAP tronqué, hors-scope (P8 dans une PR P7, etc.).

---

## 9. Fichiers chauds pour P8

À lire en premier sur main @ e56fa60 :

- ROADMAP.md §6 Phase 8 + §11 (fin de table)
- src/main.ts — boucle : timeScale → step → snapshot → selection → sync → highlight → camera.update(frame dt) → applyView → render → drainEvents
- src/input/pointer.ts, picking.ts, src/camera/input.ts — conflit clic
- src/ui/* — overlay, pointer-events
- src/sim/world.ts — addBody existe déjà ; l’UI ne l’appelle pas encore
- src/render/pixel-renderer.ts — setSelection, applyView
- src/scenes/data/systeme-simple.json

Commandes : npm run dev, test, lint, typecheck, build.

---

## 10. Ce que tu dis à Ple en arrivant

Les phases 0–7 sont sur main (`e56fa60`). Il peut jouer ici : https://dodz2.github.io/demiurge/

La suite ROADMAP est **P8 création de corps**. Tu n’y touches pas tant qu’il n’a pas dit GO. Les SHOULD (buffer 5 px, Vec3, captures, sidecars) sont parkés.

Première action : attendre le GO P8, relire ROADMAP.md Phase 8, brief Ops/Coder/Reviewer, puis une PR propre avec §11 inliné et src/sim touché seulement via addBody (commande), jamais en contournant le monde.
