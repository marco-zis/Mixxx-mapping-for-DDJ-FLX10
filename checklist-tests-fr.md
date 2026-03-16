# Checklist de tests — DDJ-FLX10 Prod A / Prod v1.0.4

Document fusionné à partir de :
- `check-list-tests-fr.md`
- `check list tests gb.md`

## Pré-requis / configuration
- [ ] Mixxx 2.3.6+ démarré
- [ ] Mapping DDJ-FLX10 chargé (`Pioneer-DDJ-FLX10.midi.xml` / PROD v1.0)
- [ ] Contrôleur DDJ-FLX10 connecté, alimenté et reconnu
- [ ] Aucune autre sortie MIDI vers le contrôleur (éviter la pollution)
- [ ] Aucune erreur dans la console / le log MIDI au chargement
- [ ] Gains/trim neutres, faders de volume au minimum, crossfader centré

## Section mixer
- [ ] Crossfader fonctionnel
- [ ] Gain master ajustable
- [ ] EQ 3 bandes fonctionnelles (LOW / MID / HIGH)
- [ ] PFL / Cue casque fonctionnel sur 4 canaux
- [ ] HeadMix / Split fonctionnels
- [ ] Vu-mètres verticaux : LEDs suivent correctement le niveau audio sur CH1–CH4

## Section decks (x4)

### Lecture
- [ ] Play / Pause fonctionne sur CH1–CH4 (0x0B)
- [ ] Cue simple fonctionne sur CH1–CH4 (0x0C)
- [ ] CUE + SHIFT = définition du point Cue
- [ ] Chargement de piste fonctionne
- [ ] Changement d’affichage du temps fonctionne (0x3E)
- [ ] Les LEDs Play/Cue restent silencieuses si les outputs sont désactivés dans cette version (pas de pollution sur 0x0B / 0x0C)

### Jog wheels
- [ ] Jog Touch détecté (scratch) (0x36)
- [ ] Pitch bend normal fonctionne (0x21)
- [ ] Scratch mode actif et précis (0x22)
- [ ] Aucune latence, aucun saut, aucun comportement erratique

### Tempo
- [ ] Tempo fader 14-bit fonctionnel sur toute la course (MSB 0x00 + LSB 0x20)
- [ ] Sens correct et retour stable dans Mixxx
- [ ] Tempo range selector fonctionne (0x60)
- [ ] Tempo reset fonctionne (0x41)
- [ ] Beat Sync fonctionne (0x66)
- [ ] Ajustement du rate range via SHIFT + Tempo Reset si prévu côté contrôleur

### Key / Sync / transport avancé
- [ ] Key reset fonctionne (0x64)
- [ ] Key Sync natif fonctionne (0x65)
- [ ] Keylock toggle fonctionne
- [ ] Le bouton Key Sync n’a pas d’effet si SHIFT est maintenu, si cette protection JS est active
- [ ] Reverse toggle fonctionne (0x15)
- [ ] Reverse / Slip Reverse fonctionne selon le design (maintien momentané ou toggle bref)
- [ ] Slip toggle fonctionne (0x40)
- [ ] Quantize toggle fonctionne

### Hotcues
- [ ] Hotcue 1–8 : déclenchement OK
- [ ] Hotcue 1–8 : suppression OK avec SHIFT
- [ ] LEDs hotcues réactives et stables

### Loop
- [ ] Loop In / Out fonctionne (0x10 / 0x11)
- [ ] Reloop / Exit fonctionne
- [ ] Loop Halve fonctionne
- [ ] Loop Double fonctionne
- [ ] Les commandes loop restent réactives avec ou sans SHIFT selon le mapping

### Beatjump
- [ ] Beatjump backward fonctionne (0x19)
- [ ] Beatjump forward fonctionne (0x1A)
- [ ] Taille de saut configurable
- [ ] Aucun doublon ni déclenchement parasite

### Modes / pads / sampler
- [ ] Aucun conflit de mode dans les pads
- [ ] Sampler 1–8 : Play / Stop fonctionne
- [ ] Sampler load fonctionne
- [ ] Sampler PFL fonctionne
- [ ] Sampler sync fonctionne
- [ ] Sampler / FX Pads sans conflit dans cette version

## Section effets
- [ ] FX1–3 Enable / Disable fonctionne (0x50–0x52)
- [ ] Assignation FX par deck fonctionne
- [ ] Wet / Dry mix fonctionnel
- [ ] BeatFX selector fonctionne

## Section browse
- [ ] Browser knob navigation fonctionne (0x40)
- [ ] Chargement de la piste sélectionnée fonctionne
- [ ] Focus navigation fonctionne (0x65 / 0x7A)

## Section SHIFT
- [ ] Shift master fonctionne (0x4B)
- [ ] Shift par deck fonctionne (0x3F)
- [ ] Le modifieur SHIFT est fiable (pas de collage)
- [ ] Les combos SHIFT + boutons sensibles fonctionnent correctement
  - [ ] CUE + SHIFT
  - [ ] Loop Halve / Double
  - [ ] Tempo Reset / Range
  - [ ] Quantize et autres raccourcis définis

## LEDs, feedback et pollution MIDI
- [ ] Feedback LED Keylock / Quantize / Reverse / Sync Key présent et stable
- [ ] Pas de clignotement parasite
- [ ] Pas de spam MIDI sur 0x5B, 0x10 / 0x11, 0x0B / 0x0C
- [ ] Messages MIDI cohérents dans l’ensemble

## Validation technique
- [ ] 0 collision MIDI détectée au chargement
- [ ] Tempo 14-bit géré nativement en XML (pas de handler JS), si attendu
- [ ] Key Sync géré nativement en XML (pas de handler JS), si attendu
- [ ] Lint XML / JS propre
- [ ] Pas d’erreur au démarrage du script

## Performance et stabilité
- [ ] Latence acceptable (< 10 ms)
- [ ] Réactivité des jog wheels satisfaisante
- [ ] Pas de dropouts audibles pendant jog / tempo / loop
- [ ] Stabilité sur longue durée
- [ ] Consommation CPU raisonnable

## En cas d’échec
- [ ] Noter le deck concerné
- [ ] Noter le bouton / la LED concerné(e)
- [ ] Noter le code MIDI observé (status / midino)
- [ ] Préciser le contexte (SHIFT oui / non)
- [ ] Indiquer s’il s’agit d’un problème fonctionnel (contrôle) ou visuel (LED / pollution)

## Rapport de test
Date : ____________

Heure : ____________

Testeur : __________

Version Mixxx : __________

**Statut global** : [ ] PASS  [ ] FAIL

**Issues identifiées** :
1. _________________________
2. _________________________
3. _________________________

**Recommandations** :
__________________________________________________

**Signature** : ____________
