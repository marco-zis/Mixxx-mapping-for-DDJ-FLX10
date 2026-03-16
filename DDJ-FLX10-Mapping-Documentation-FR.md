# Pioneer DDJ-FLX10 — Documentation technique mise à jour (audit XML/JS)

## 1. Objet du document

Cette documentation décrit **l’état réel des fichiers fournis** à partir de l’analyse conjointe du XML, du JavaScript et de la documentation d’origine.  
Elle documente donc le mapping **tel qu’il est actuellement déclaré**, y compris les incohérences détectées entre le XML et le JS.

## 2. Sources analysées

- `Pioneer-DDJ-FLX10.midi.xml`
- `Pioneer-DDJ-FLX10-scripts.js`
- `DDJ-FLX10-Mapping-v1.0-Documentation.md`

## 3. Identité du mapping

| Champ | Valeur observée |
|---|---|
| Nom XML | `DDJ-FLX10 PROD` |
| Auteur | `Marc Zischka` |
| Compatibilité déclarée | `Mixxx 2.3.6+` |
| Version dans le XML | `v1.0` |
| Version dans l’en-tête JS | `v0.2` |
| Statut de cette documentation | révision documentaire basée sur l’état analysé |
| Date de révision | 2026-03-16 |

> Remarque importante : les numéros de version internes ne sont pas homogènes entre les fichiers.  
> Cette documentation emploie donc la notion d’**état analysé** plutôt qu’une version unique théorique.

## 4. Statistiques d’audit

| Élément | Valeur |
|---|---|
| Entrées XML (`<control>`) | 373 |
| Sorties XML (`<output>`) | 28 |
| Bindings JS déclarés dans le XML | 65 |
| Fonctions JS déclarées | 38 |
| Fonctions JS effectivement appelées par le XML | 12 |
| Bindings XML orphelins (script-binding sans fonction JS correspondante) | 3 |
| Collisions MIDI détectées (`status` + `midino`) | 10 |

## 5. Architecture actuelle

### 5.1 Répartition XML / JS

Le mapping suit globalement cette logique :

- **XML** pour les contrôles Mixxx natifs simples : play, cue, hotcues, EQ, volume, PFL, keylock, slip, reloop, navigation, UI, effets, samplers.
- **JS** pour les comportements multi-états ou dépendants d’un contexte :
  - gestion du **SHIFT**,
  - **tempo 14-bit** via MSB/LSB,
  - **jog touch / scratch / bend**,
  - **reverse** avec mode slip au maintien et toggle au SHIFT,
  - bascule **temps restant / temps écoulé**,
  - cycle du **rate range**.

### 5.2 Liste blanche JS réellement utilisée par le XML

Ce sont les seules fonctions JS effectivement appelées par le XML et résolues dans le script :

1. `shiftHandler`
2. `wheelTouch`
3. `wheelTurn`
4. `rate_msb`
5. `rate_lsb`
6. `LoopHalveShift`
7. `LoopDoubleShift`
8. `reverse`
9. `TimeTypeChange`
10. `rate_reset`
11. `RangeSelector`
12. `syncKeyHandler`

### 5.3 Bindings XML orphelins

Les bindings suivants sont déclarés en `script-binding` dans le XML, mais **aucune fonction JS correspondante n’existe** dans le fichier JavaScript :

- `beatjump_forward`
- `beatjump_backward`
- `beatsync`

En l’état, ces trois entrées sont incohérentes et doivent être considérées comme **à vérifier / à corriger**.

### 5.4 Fonctions JS présentes mais non appelées depuis le XML

#### Fonctions de cycle de vie ou d’assistance interne
- `init`
- `shutdown`
- `_getDeckFromGroup`
- `_updateRate`
- `_updateTimeMode`
- `sensitivityMinimizer`
- `sensitivityMaximizer`
- `updateLEDs`

#### Handlers legacy ou alternatifs non utilisés par le XML actuel
- `shift`
- `loopIn`
- `loopOut`
- `browseNavigation`
- `syncKey`
- `cue`
- `play`
- `Play`
- `pfl`
- `loopOutButton`
- `loopInButton`
- `reloop`
- `sync`
- `playPause`
- `cueButton`
- `loopManual`
- `autoLoop`
- `reverseHandler`

## 6. Résumé fonctionnel du mapping

### 6.1 Commandes deck communes (Decks 1 à 4)

| MIDI | Contrôle logique | Implémentation | Remarques |
|---|---|---|---|
| `0x00` + `0x20` (CC) | Tempo fader MSB/LSB | JS | via `rate_msb` / `rate_lsb` |
| `0x0B` | Play | XML natif | `play` |
| `0x0C` | Cue | XML natif | `cue_default` |
| `0x10` | Loop In | XML natif | `loop_in` |
| `0x11` | Loop Out | XML natif | `loop_out` |
| `0x14` | Reloop | XML natif | `reloop_toggle` |
| `0x15` | Reverse | JS | maintien = slip reverse, SHIFT = toggle reverse |
| `0x1C` | Keylock | XML natif | `keylock` |
| `0x35` | Quantize | XML natif | `quantize` |
| `0x36` | Jog touch | JS | active / désactive le scratch |
| `0x3E` | Affichage temps | JS | bascule `show_seconds_elapsed` |
| `0x3F` / `0x4B` | Shift | JS | 4 boutons deck + 1 bouton master, état SHIFT global |
| `0x40` | Slip | XML natif | `slip_enabled` |
| `0x41` | Tempo default | XML natif | `rate_set_default` |
| `0x46` | Tempo reset | JS | remet aussi en cohérence l’état interne du fader |
| `0x54` | PFL | XML natif | `pfl` |
| `0x58` | Sync enabled | XML natif | `sync_enabled` |
| `0x60` | Rate range | JS | cycle entre plusieurs plages |
| `0x64` | Key reset | XML natif | `reset_key` |
| `0x65` | Key sync | JS | handler dédié, voir point de vigilance sur le nom du contrôle |
| `0x66` | Beat sync | XML incohérent | déclaré en `script-binding`, sans fonction JS |
| `0x70` / `0x71` | Rewind / Forward | XML natif | `back` / `fwd` |
| `0x4C` / `0x4D` | Loop halve / double | JS | codes SHIFT dédiés |

#### Particularités deck par deck

- **Load track**
  - Deck 1 : `0x96 / 0x46`
  - Deck 2 : `0x96 / 0x47`
  - Deck 3 : `0x96 / 0x48`
  - Deck 4 : `0x96 / 0x49`

- **Orientation**
  - Decks 1 et 3 : `0x16`
  - Decks 2 et 4 : `0x18`

- **Beatjump**
  - déclaré sur `0x19` et `0x1A`,
  - mais les decks 2 à 4 réutilisent à tort le `status 0x90`,
  - ce qui crée une collision avec le deck 1.

### 6.2 Jog wheels

Le script implémente :

- **touch** via `wheelTouch` :
  - `value 0x7F` → `engine.scratchEnable(...)`
  - `value 0x00` → `engine.scratchDisable(...)`

- **rotation** via `wheelTurn` :
  - si Mixxx est en scratch → `engine.scratchTick(...)`
  - sinon → `engine.setValue(group, "jog", ...)` pour le bend

Le XML déclare deux entrées de rotation (`0x21` et `0x22`) mais elles appellent la **même** fonction JS.  
Le comportement effectif dépend donc surtout de l’état scratch activé par le touch, pas du numéro MIDI seul.

### 6.3 Tempo

#### Implémenté
- Fader tempo 14-bit par deck via JS (`rate_msb` + `rate_lsb`)
- Reset natif Mixxx sur `0x41` (`rate_set_default`)
- Reset JS supplémentaire sur `0x46` (`rate_reset`)
- Sélecteur de plage via `RangeSelector` sur `0x60`

#### Point technique
Le JS combine bien MSB + LSB, mais applique ensuite une échelle interne fixe de **±8 %** dans `_updateRate`.  
Le handler `RangeSelector` incrémente `rateRange` de manière numérique avec un modulo 4 ; la validité exacte de cette logique dépend de la façon dont Mixxx interprète cette valeur.  
La logique JS de calcul du fader restant indépendante de cette plage, il faut considérer tout le bloc tempo comme **à valider en test réel**.

### 6.4 Transport et lecture

Par deck :

- `play`
- `cue_default`
- `slip_enabled`
- `reverse` (JS)
- `back` / `fwd`
- `sync_enabled`
- `reset_key`
- `keylock`
- `quantize`
- `LoadSelectedTrack`

### 6.5 Hot Cues

Le mapping déclare **8 hot cues par deck** :

- **activation** : notes `0x00` à `0x07`
- **clear** : mêmes notes sur le statut suivant

Répartition par deck :

- Deck 1 : `0x97` / `0x98`
- Deck 2 : `0x99` / `0x9A`
- Deck 3 : `0x9B` / `0x9C`
- Deck 4 : `0x9D` / `0x9E`

### 6.6 Loops

#### Contrôles natifs
- `loop_in`
- `loop_out`
- `reloop_toggle`

#### Contrôles JS
- `LoopHalveShift` sur `0x4C`
- `LoopDoubleShift` sur `0x4D`
- `reverse` sur `0x15`

Le JS contient aussi d’anciens handlers (`loopIn`, `loopOut`, `loopManual`, `autoLoop`, `reloop`) mais ils ne sont plus appelés par le XML actuel.

### 6.7 Mixer, gain, volume et EQ

#### Master
- `gain` : `0xB6 / 0x08`
- `crossfader` : `0xB6 / 0x1F` + `0xB6 / 0x3F` (14-bit)
- `headMix` : `0xB6 / 0x0C` + `0xB6 / 0x2C` (14-bit)
- `headGain` : `0xB6 / 0x0D` + `0xB6 / 0x2D` (14-bit)

#### Par deck
- `pregain` MSB : `0x04`
- `pregain` LSB : `0x24`
- `volume` MSB : `0x13`
- `volume` LSB : `0x33` **uniquement sur les decks 1 et 2**
- EQ High : `0x07` + `0x27`
- EQ Mid : `0x0B` + `0x2B`
- EQ Low : `0x0F` + `0x2F`

#### Point de vigilance sur la précision
- Les **EQ** sont correctement déclarés en 14-bit.
- Le **crossfader** et le **headphone mix/gain** sont correctement déclarés en 14-bit.
- Le **pregain** possède un LSB (`0x24`) sur les 4 decks, mais le MSB (`0x04`) est déclaré en `normal` et non en `fourteen-bit-msb`.
- Le **volume fader** possède un LSB (`0x33`) seulement sur les decks 1 et 2.

### 6.8 Quick Effects

- Activation QuickEffect deck :
  - Deck 1 : `0x97 / 0x16`
  - Deck 2 : `0x99 / 0x16`
  - Deck 3 : `0x9B / 0x16`
  - Deck 4 : `0x9D / 0x16`

- Knob `super1` :
  - Deck 1 : `0xB6 / 0x17` (7-bit)
  - Deck 2 : `0xB6 / 0x18` + `0xB6 / 0x38` (14-bit)
  - Deck 3 : `0xB6 / 0x19` + `0xB6 / 0x39` (14-bit)
  - Deck 4 : `0xB6 / 0x1A` + `0xB6 / 0x3A` (14-bit)

### 6.9 Effects Units

Le XML déclare :

- **4 effect units**
- assignation vers les decks via `group_[ChannelX]_enable`
- assignation vers le **Master**
- assignation vers le **Headphone**
- activation des **slots d’effet 1 à 3** sur les notes `0x50`, `0x51`, `0x52`

#### Remarque
Certaines assignations utilisent aussi `0x44`, `0x45`, `0x46`, `0x47` pour le Headphone, ce qui s’ajoute aux boutons UI / load / micro déjà présents ailleurs.  
Cela mérite une validation terrain.

### 6.10 Navigation, bibliothèque et UI

#### Bibliothèque
- Browse knob : `0xB6 / 0x40` → `MoveVertical`
- Focus gauche : `0x96 / 0x65` → `MoveFocusBackward`
- Focus droite : `0x96 / 0x7A` → `MoveFocusForward`
- Maximize library : `0x97 / 0x40`

#### UI
- Show mixer : `0x97 / 0x43`
- Show waveforms : `0x97 / 0x41`
- Show 4 decks : `0x97 / 0x42`
- Show microphone : `0x97 / 0x46`
- Show samplers :
  - `0x90..0x93 / 0x22`
  - `0x97 / 0x45`
- Show effect rack :
  - `[EffectRack1] show` sur `0x90 / 0x1E`, `0x91 / 0x1E`, `0x97 / 0x44`
  - `[Skin] show_effectrack` aussi sur `0x91 / 0x1E`
- Show 4 effect units :
  - `0x97 / 0x17`
  - `0x99 / 0x17`
  - `0x9B / 0x17`
  - `0x9D / 0x17`
  - plus `0x92 / 0x1E` et `0x93 / 0x1E`

### 6.11 Samplers

Le mapping couvre les **8 samplers**, mais pas de manière totalement symétrique.

#### Samplers 1 à 4
- `cue_gotoandplay` : `0x30` à `0x33`
- `start_stop` : `0x34` à `0x37`
- autre couche `cue_gotoandplay` : `0x38` à `0x3B`
- autre couche `start_stop` : `0x3C` à `0x3F`
- `beatsync` : `0x70` à `0x73`
- `pfl` : `0x74` à `0x77`

#### Samplers 5 à 8
- `cue_gotoandplay` : `0x30` à `0x33` sur `status 0x99`
- `start_stop` : `0x34` à `0x37` sur `status 0x99`
- `beatsync` : `0x70` à `0x73`
- `pfl` : `0x74` à `0x77`

#### Cas particulier
- **Sampler 3** possède un `pregain` 14-bit sur `0x03` + `0x23`.

### 6.12 Enregistrement

- `toggle_recording` : `0x97 / 0x47`

## 7. Sorties / LEDs

Le XML déclare **28 sorties**, soit **7 par deck** :

- `cue_indicator`
- `play_indicator`
- `pfl`
- `quantize`
- `reverse`
- `keylock`
- `sync_key`

##### Notes
- Les LEDs Play/Cue/Quantize/Reverse/Keylock/PFL sont bien prévues par le XML.
- Le JS possède aussi une fonction `updateLEDs`, mais elle n’est utilisée qu’en interne.
- Le handler `syncKeyHandler` écrit `key_sync`, alors que les sorties XML écoutent `sync_key`.  
  Cette divergence de nommage doit être vérifiée dans Mixxx.

## 8. Incohérences et collisions détectées

### 8.1 Collisions MIDI (`status` + `midino`)

| `status` | `midino` | Collision détectée |
|---|---|---|
| `0x90` | `0x19` | beatjump backward déclaré 4 fois pour les decks 1 à 4 |
| `0x90` | `0x1A` | beatjump forward déclaré 4 fois pour les decks 1 à 4 |
| `0x97` | `0x13` | Effect Unit 4 assign CH1 / Sampler 3 start_stop |
| `0x99` | `0x13` | Effect Unit 4 assign CH2 / Sampler 7 start_stop |
| `0x9B` | `0x13` | Effect Unit 4 assign CH3 / Sampler 3 start_stop |
| `0x91` | `0x1E` | `EffectRack1.show` / `Skin.show_effectrack` |
| `0x90..0x93` | `0x22` | duplication `show_samplers` entre `[Samplers]` et `[Skin]` |

### 8.2 Incohérences fonctionnelles

1. `beatjump_forward`, `beatjump_backward` et `beatsync` sont en `script-binding` sans fonction JS.
2. Les decks 2 à 4 utilisent le mauvais `status` pour le beatjump (`0x90` au lieu d’un canal dédié).
3. `syncKeyHandler` écrit `key_sync`, alors que les outputs XML suivent `sync_key`.
4. `pregain` mélange déclaration 7-bit et LSB 14-bit.
5. `volume` n’a un LSB que sur les decks 1 et 2.
6. `QuickEffect super1` est 7-bit sur le deck 1 mais 14-bit sur les decks 2 à 4.
7. Les versions internes XML / JS / documentation ne correspondent pas entre elles.

## 9. Recommandations de nettoyage

1. Remplacer les `script-binding` orphelins (`beatjump_forward`, `beatjump_backward`, `beatsync`) par des bindings XML natifs, ou ajouter les fonctions JS manquantes.
2. Corriger les statuts MIDI du beatjump pour les decks 2 à 4.
3. Harmoniser `key_sync` / `sync_key`.
4. Normaliser la gestion 14-bit de `pregain`, `volume` et `QuickEffect`.
5. Supprimer ou documenter explicitement les doublons UI (`show_samplers`, `show_effectrack`).
6. Purger les handlers JS legacy non utilisés pour simplifier la maintenance.

## 10. Plan de test recommandé

### Tests de base
- Play / Cue / PFL / Keylock / Quantize
- Volume, crossfader, pregain, EQ
- Browse et chargement de piste
- Activation QuickEffect et Effect Units

### Tests spécifiques JS
- SHIFT global
- Tempo 14-bit (MSB + LSB)
- Rate reset
- Rate range
- Jog touch / scratch / bend
- Reverse maintien / reverse SHIFT

### Tests de cohérence à surveiller
- Beatjump sur decks 2 à 4
- Beat sync sur les 4 decks
- LED de key sync
- Précision réelle du volume deck 3 et 4
- Conflits éventuels entre samplers et assignations FX

---

**Conclusion**  
Le mapping couvre déjà une grande partie du DDJ-FLX10, mais l’audit met en évidence un état **mixte** :  
une base XML conséquente, un noyau JS utile pour les cas complexes, et plusieurs reliquats ou collisions à nettoyer pour obtenir une documentation et un comportement parfaitement cohérents.
