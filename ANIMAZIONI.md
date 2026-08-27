# Tamagotchi AI — Animazioni e Visual Design

## 1. Stile visuale

### Filosofia

Il Tamagotchi è un **draghetto kawaii** (come il riferimento allegato), con:
- **proporzioni chibi** (testa molto grande, corpo piccolo, arti corti);
- **tratti iconici** da drago (corna morbide, mini ali, coda corta);
- **occhi molto espressivi** (pupille piccole nere + riflesso bianco);
- **colori caldi e saturi** con outline scuro quasi nero;
- **linee arrotondate** (niente angoli acuti, tutto è soft);
- **animazioni fluide** (8–12 frame per azione, ciclo 500–1000 ms).

### Firma stilistica del personaggio

Per mantenere coerenza con il draghetto di riferimento:

- testa circa 1.4x la larghezza del busto;
- occhi rotondi/ellittici con pupilla piccola e riflesso alto;
- ventre più chiaro del corpo (patch centrale);
- dettagli "cute" obbligatori: guance rosse, zampette tonde, bocca piccola;
- ali e corna sempre con silhouette morbida (mai aggressive).

### Palette colori suggerita (dragon style)

```text
Corpo principale:      #F2644B (rosso-arancio)
Ventre/inner wings:    #F6E6C8 (crema chiaro)
Outline principale:    #2B2B2B (quasi nero)
Ombra corpo:           #D94A3C (rosso mattone)
Occhi pupilla:         #1F1F1F (nero morbido)
Occhi luccichio:       #FFFFFF (bianco)
Guance/bocca:          #E63A3A (rosso guancia)
Spine/corna:           #C7362D (rosso scuro)
Effetto happy/glow:    #FFD45A (giallo dorato)
```

### Dimensioni asset

- **Sprite a riposo**: ~80×80 px (corpo + testa, scalabile fino a 120×120 per ingrandimento)
- **Animazioni**: frame singoli 80×80, organizzati in spritesheet
- **Uovo iniziale**: 120×140 px (più grande, verticale)
- **UI elementi**: icone 32×32, label con font anti-aliased

### Pipeline: da immagine riferimento a sprite per ESP32

1. Importa l'immagine del draghetto in Aseprite/Piskel.
2. Traccia una **silhouette master** in 80x80 px (solo contorno).
3. Separa in layer: `head`, `body`, `eyes`, `mouth`, `wings`, `tail`, `effects`.
4. Applica la palette a 8 colori (sezione sopra) per ridurre memoria.
5. Disegna i keyframe delle azioni (`IDLE`, `FEED`, `PLAY`, `SLEEP`, `WASH`).
6. Esporta spritesheet + atlas JSON.
7. Converti PNG in formato C per LVGL (`lv_img_conv`) o usa filesystem SPIFFS/LittleFS.

Output target consigliato:

- PNG indexed (8-bit) + trasparenza;
- massimo 512x1024 per spritesheet;
- naming: `dragon_<fase>_<azione>_<frame>.png`.

---

## 2. Ciclo di vita e forme

### Fase 0: L'Uovo (0 age_hours)

```text
     ╔═══════╗
     ║       ║
     ║  ♡ ♡  ║  <- punti luce (highlight)
     ║       ║
     ║  ‾ ‾  ║  <- piccola bozza di bocca
     ║       ║
     ╚═══════╝
```

**Animazione schiusa** (evento speciale, una sola volta):
- Frame 1–4: scuotimento lento (±2px orizzontale, 400 ms)
- Frame 5–8: spaccatura da sopra (line animata che scende)
- Frame 9–12: apertura uovo (due metà che si separano, il corpo esce)
- Frame 13+: transizione a neonato (fade-in corpo, fade-out guscio)

Durata totale: 2–3 secondi, con suono crack + piccolo verso.

### Fase 1: Neonato (0–24 age_hours)

```text
    o_o    <- occhi tondi, innocenti
    (•)    <- bocca piccola, "oh"
   (~~~)   <- corpo minuscolo, tondo
```

- **Dimensione**: 60×60 px
- **Proporzioni**: 40% testa, 60% corpo tondo
- **Comportamento**: molto dormigliare (occhi da addormentato), beve latte, piange se affamato.

### Fase 2: Bambino (24–96 age_hours)

```text
    ^_^    <- occhi a V (felicità)
    (.)    <- bocca piccola ma definita
   (_o_)   <- corpo un po' più grande e quadrato
    | |    <- braccia/zampe visibili
```

- **Dimensione**: 80×100 px
- **Proporzioni**: 35% testa, 65% corpo (diventa meno sferico, più antropomorfo)
- **Comportamento**: gioca, studia, mangia di più, emote ben visibili.

### Fase 3: Adolescente (96–240 age_hours)

```text
    ◉_◉    <- occhi più grandi e consapevoli
    (_)    <- bocca normale, più espressioni
   /‾ ‾\   <- corpo più snello, gambe visibili
    | |
   /   \
```

- **Dimensione**: 100×120 px
- **Proporzioni**: 30% testa, 70% corpo (più "umanoide")
- **Comportamento**: personalità emerge, può stufare di azioni ripetute.

### Fase 4: Adulto (240+ age_hours)

```text
    ◯ ◯    <- occhi maturi, "consapevoli"
    ^_^    <- bocca espressiva
   /‾ ‾\   <- corpo pieno, proporzionato
   |   |
   \___/
```

- **Dimensione**: 120×140 px
- **Proporzioni**: 25% testa, 75% corpo (equilibrato, maturo)
- **Comportamento**: tutta la gamma di emozioni.

### Ingrassamento progressivo

Ogni fase ha **3 versioni di larghezza** in base al peso (`hunger` inversa + calorie cumulate):

1. **Snello** (basso peso): corpo stretto, linea di cintura visibile
2. **Normale** (peso medio): proporzioni equilibrate
3. **Rotondo** (alto peso): corpo gonfiato, mancini potrebbero toccarsi

Il passaggio tra versioni è **graduale** (mix fade tra sprite).

---

## 3. Anatomia emotiva

### Occhi e pupille

Gli occhi trasmettono **>70% dell'emozione**. Implementazione:

```text
Felice:        ^_^   (semicerchi rivolti verso l'alto, pupille piccole)
Triste:        T_T   (semicerchi rivolti verso il basso, lacrime animate)
Arrabbiato:    ▼_▼   (sopracciglia diagonali verso il centro)
Affamato:      ◯_◯   (occhi tondi, pupille dilatate, luccichio)
Assonnato:     - -   (palpebre semi-chiuse, movimento lento)
Sorpreso:      o_o   (occhi tondi aperti, brillanti)
Innamorato:    ♡_♡   (occhi a forma di cuore)
Confuso:       o_?   (un occhio aperto, uno semichiuso)
Eccitato:      ★_★   (stelle al posto delle pupille, movimento veloce)
```

Ogni espressione oculare ha **2–3 frame** di animazione (lampo, scuotimento leggero).

### Bocca e guance

Complemento alle emozioni:

```text
Felice:        ^_^   (arco, rosso)
Triste:        v‾v   (arco rovesciato, grigio)
Bocca aperta:  O_O   (cerchio, per sorpresa o grido)
Sonno:         - _   (una linea)
Mangia:        ◎ ◎   (cerchi che si muovono, cibi dentro)
```

**Guance**: appaiono quando happy (rosa caldo, accento da ambo i lati), scompaiono quando triste/affamato.

### Corpo e postura

```text
Normale:       corpo diritto, braccia giù
Dormigliare:   corpo leggermente curvo, testa che oscilla
Salto:         corpo che si alza e scende (bounce)
Danza:         corpo che ondeggia lato/lato
Affamato:      corpo che si piega in avanti, pancia visibile
Stanco:        corpo che si affloscia, spalle giù
Malato:        corpo che trema, pallore leggero
```

---

## 3-bis. Modello coerente stato + emozioni

Per mantenere coerenza tra gameplay e resa visiva, le animazioni non scelgono "a mano" l'emozione: la ricevono dal Game Engine, che la calcola dagli stati numerici.

### Stati canonici (scala 0..100)

- `hunger` (0 sazio, 100 affamato)
- `energy` (0 esausto, 100 carico)
- `hygiene` (0 sporco, 100 pulito)
- `health` (0 critico, 100 ottimo)
- `affection`
- `fun`
- `stress`

Regola unica di aggiornamento:

```text
nuovo_valore = clamp(vecchio_valore + delta_tempo + delta_azione + delta_evento, 0, 100)
```

### Emozione risultante (somma pesata)

```text
happy_score   = 0.30*(100-hunger) + 0.25*energy + 0.20*affection + 0.15*fun + 0.10*hygiene - 0.25*stress
sad_score     = 0.30*hunger + 0.25*(100-energy) + 0.20*(100-affection) + 0.15*(100-fun) + 0.10*stress
angry_score   = 0.40*stress + 0.25*hunger + 0.20*(100-energy) + 0.15*event_frustration
sleepy_score  = 0.60*(100-energy) + 0.20*hunger + 0.20*age_phase
sick_score    = 0.50*(100-health) + 0.25*(100-hygiene) + 0.25*hunger
playful_score = 0.35*energy + 0.30*fun + 0.20*affection + 0.15*(100-stress)
```

Priorità visiva (per evitare conflitti):

1. `SICK` se `health < 30`
2. `SLEEPY` se `energy < 20`
3. `HUNGRY` se `hunger > 80`
4. `DIRTY` se `hygiene < 25`
5. altrimenti max score tra `HAPPY`, `SAD`, `ANGRY`, `PLAYFUL`, `CALM`

### Delta canonici delle azioni (allineati al Game Engine)

| Azione | hunger | energy | hygiene | health | affection | fun | stress | experience |
|---|---:|---:|---:|---:|---:|---:|---:|---:|
| FEED | -40 | -5 | -2 | +2 | +2 | +2 | -4 | 0 |
| PLAY | +5 | -25 | -4 | +1 | +10 | +30 | -8 | +4 |
| SLEEP | +5 | +45 | 0 | +8 | 0 | -5 | -12 | 0 |
| PET | +1 | -2 | 0 | +1 | +3 | +5 | -6 | 0 |
| WASH | 0 | -10 | +35 | +4 | +5 | +2 | -10 | 0 |
| STUDY | +2 | -15 | 0 | 0 | +1 | +3 | +2 | +10 |
| POOP_EVENT | 0 | -1 | -20 | -3 | -1 | -2 | +8 | 0 |
| MEDICINE | 0 | -2 | 0 | +20 | +1 | -1 | -5 | 0 |

> Le sezioni azione qui sotto descrivono la coreografia animata; i delta sopra sono la fonte unica di verità per la logica stato/emozione.

---

## 4. Animazioni per azioni

Ogni azione è un **ciclo di frame** con transizioni smooth; l'emozione mostrata a fine azione viene sempre ricalcolata dal modello di cui sopra.

### FEED (Mangiare)

**Trigger**: intent FEED, `hunger > 0`, `energy >= 10`.

**Flusso**:
1. **Anticipation** (200 ms): corpo si piega in avanti, occhi brillano, bocca si apre poco
2. **Eating** (1000 ms):
   - Frame 1–4: bocca si apre (O_O), cibo compare davanti (sprite ciliegia/pane/mela)
   - Frame 5–8: masticazione veloce (bocca aperta-chiusa-aperta)
   - Frame 9–12: deglutizione (bocca chiusa, corpo si rilassa)
3. **Soddisfazione** (300 ms): 
   - corpo rimbalza (1 bounce)
   - espressione felice (^_^)
   - emote piccolo cuore sopra la testa (scale-out, fade-out)
4. **Recovery**: torna a riposo in 200 ms

**Effetti audio**: "munch munch", piccolo verso di felicità (0.5 s, basso volume).

**Delta stato (canonico)**: `hunger -= 40`, `energy -= 5`, `hygiene -= 2`, `health += 2`, `affection += 2`, `fun += 2`, `stress -= 4`.

---

### PLAY (Giocare)

**Trigger**: intent PLAY, `energy >= 20` e `stress <= 75`.

**Flusso**:
1. **Setup** (200 ms): corpo si raddrizza, occhi brillano (★_★), piccolo salto
2. **Gioco** (2000 ms, loop 2 volte):
   - Frame 1–6: corre a sinistra (corpo si muove, gambe animate)
   - Frame 7–12: corre a destra
   - Palla che rimbalza attorno (sprite separato, arco di movimento)
3. **Celebrazione** (400 ms):
   - corpo che salta verticalmente (3 bounce rapidi)
   - espressione super-felice (^o^)
   - emote musicali (note di musica animate, pop-pop-pop)
4. **Affaticamento** (300 ms): corpo che si affloscia leggermente, respirazione visibile (pancia su-giù)

**Effetti audio**: "bouncy jump" (200 Hz), risate (0.5–1 s), musichetta di vittoria.

**Delta stato (canonico)**: `hunger += 5`, `energy -= 25`, `hygiene -= 4`, `health += 1`, `affection += 10`, `fun += 30`, `stress -= 8`, `experience += 4`.

---

### SLEEP (Dormire)

**Trigger**: intent SLEEP, `energy < 30` oppure ritmo circadiano in fascia notturna.

**Flusso**:
1. **Sbadiglio** (400 ms):
   - corpo si stira (y-axis stretch)
   - bocca che si apre grande (◯)
   - occhi che si stringono (- -)
2. **Coricarsi** (300 ms):
   - corpo che si accuccia (curva verso il basso)
   - testa che si appoggia
3. **Sonno profondo** (loop finché il bisogno di sonno non rientra):
   - occhi chiusi (-)
   - bocca neutra (— )
   - piccoli Z che fluttuano su (emote, pop-pop-pop ogni 500 ms)
   - corpo che respira (lieve oscillazione verticale, ogni 1 s)
4. **Sveglia** (300 ms):
   - occhi che si aprono (animazione reveal)
   - corpo che si stira
   - piccolo verso di "buongiorno"

**Effetti audio**: sbadiglio leggero, ronzio-ronzio sonno, verso di sveglia (dolce).

**Delta stato (canonico)**: `energy += 45`, `hunger += 5`, `health += 8`, `fun -= 5`, `stress -= 12`.

---

### PET (Coccola / Carezza)

**Trigger**: intent PET, oppure touch screen (drag-slide leggero su corpo).

**Flusso**:
1. **Contatto** (100 ms): corpo si illumina (bright flash), piccolo verso ("ooh!")
2. **Carezza** (300–600 ms):
   - corpo ondeggia dolcemente
   - occhi che si stringono felici (^_^)
   - guance rosa che si illuminano
3. **Reazione** (400 ms):
   - corpo rimbalza indietro (recoil positivo)
   - emote cuore che cresce dal corpo (scale-out, fade-out)
4. **Recovery**: riposo in 200 ms

**Effetti audio**: verso dolce di felicità, leggero suono "carezza" (swoosh).

**Delta stato (canonico)**: `hunger += 1`, `energy -= 2`, `health += 1`, `affection += 3`, `fun += 5`, `stress -= 6`.

---

### POOP (Fare popo)

**Trigger**: automatico quando `digestion_timer` scade (es. ogni 30 min dopo mangiare).

**Flusso**:
1. **Segnale di disagio** (400 ms):
   - corpo che ondeggia avanti-indietro (movimento di urgenza)
   - occhi in croce (X_X) o allarmati
   - piccolo verso di disagio ("uh-oh...")
2. **Atto** (300 ms):
   - corpo in posizione accucciata
   - effetto "puff" (nuvola, sprite piccolo che appare)
   - popo che cade e scompare (sprite marrone che cade, sparisce dopo 1 s)
3. **Sollievo** (200 ms):
   - corpo che si raddrizza
   - espressione neutra (o_o) poi un sospiro felice

**Effetti audio**: suono caratteristico ma **non sgradevole** (suono cartoon leggero, tipo "plink").

**Delta stato (canonico, evento automatico)**: `energy -= 1`, `hygiene -= 20`, `health -= 3`, `affection -= 1`, `fun -= 2`, `stress += 8`.

**Nota**: il popo rimane visibile sullo schermo per 5 secondi se non viene pulito → trigger per WASH.

---

### WASH (Bagno / Lavaggio)

**Trigger**: intent WASH, oppure automatico se popo visibile > 30 secondi.

**Flusso**:
1. **Setup** (200 ms):
   - corpo che guarda intorno (occhi che seguono una traccia)
   - verso di "uh-oh" ma positivo
2. **Bagno** (1500 ms):
   - corpo che entra in una vasca (sprite di vasca con acqua, corpo va dentro)
   - animazione spruzzi d'acqua (sprite piccoli che saltano)
   - corpo che si strofina (movement jittery, lavarsi)
   - corpo che cambia colore (transizione da sporco a pulito, blur-fade)
3. **Asciugatura** (500 ms):
   - corpo che esce dalla vasca
   - shaking veloce (acqua che scappa, shake-animation)
   - asciugamano (sprite che scende, spara burst di bolle)
4. **Soddisfazione** (300 ms):
   - corpo che si stiracchia felice
   - espressione super-felice (^_^)

**Effetti audio**: suono di acqua che scorre, asciugamano, verso di felicità (più alto del solito).

**Delta stato (canonico)**: `energy -= 10`, `hygiene += 35`, `health += 4`, `affection += 5`, `fun += 2`, `stress -= 10`.

---

### STUDY (Studiare / Imparare)

**Trigger**: intent TALK o STUDY durante ore di gioco "diurne" (non di notte).

**Flusso**:
1. **Concentrazione** (300 ms):
   - corpo che si siede (postura seduta)
   - occhi che si restringono in concentrazione (^_^, ma seri)
   - piccolo libro/tablet che appare davanti
2. **Lettura** (1200 ms, loop):
   - occhi che seguono il testo (movimento orizzontale saccadico)
   - corpo fermo
   - pagine che si girano (sprite libro che cambia frame)
   - emote lampadina che compare e scompare (scale-in/out, ogni 400 ms) = "idea"
3. **Completamento** (400 ms):
   - corpo che si allunga soddisfatto
   - espressione di soddisfazione leggera (^_^)
   - emote stella che orbita attorno (pop-pop, fade-out)
   - libro che sparisce (fade-out)

**Effetti audio**: suono di pagine che girano, verso di "hmm" pensieroso, suono di "ding" quando impara.

**Delta stato (canonico)**: `hunger += 2`, `energy -= 15`, `affection += 1`, `fun += 3`, `stress += 2`, `experience += 10`.

---

### SICK / MALATO

**Trigger**: automatico quando `health < 30` o dopo molte ore senza bagno/cibo.

**Flusso**:
1. **Segni di malessere** (loop continuo finché health < 30):
   - corpo pallido (overlay bianco-grigio, ridotta saturazione colore)
   - occhi stanchi (- o -, pupille dilatate)
   - corpo che trema (shake-animation, 2–3 px, ogni 300 ms)
   - emote goccia di sudore (oscillazione bassa, pop-pop)
2. **Cura** (quando energy/cibo/sonno si riprendono):
   - feedback visivo di guarigione (effetto sparkle, emote cuore)
   - colore del corpo che ritorna gradualmente

**Effetti audio**: verso di malessere debole, niente musica.

**Stato finale**: priorità emozione `SICK`; gioco/studio inibiti finché `health >= 30`.

**Delta consigliato per tick in stato malato (ogni minuto)**: `energy -= 1`, `hunger += 1`, `stress += 2`.

---

### IDLE / RIPOSO (default)

**Trigger**: nessun input, durante downtime tra azioni.

**Flusso** (loop continuo):
- **Sguardo curioso** (3000 ms): corpo fermo, occhi che guardano in giro (pupille che si muovono, ogni 500 ms)
- **Respirazione** (loop): lieve oscillazione verticale del corpo (±2 px), ogni 1–2 s
- **Ammiccamento** (ogni 5 s): rapida chiusura-apertura occhi (2 frame, 100 ms)
- **Movimento strano** (raro, 20% ogni 10 s): corpo che si stira, una gamba che si muove, testa che si inclina

**Effetti audio**: nessuno, oppure lievi bruitini ambientali ogni 15–20 s (verso minuscolo, "mmm").

---

## 5. Emozioni transition

Quando l'emozione cambia (es. da felice a triste), la transizione è **fluida**:

```text
Espressione A (10 frame, 500 ms)
        ↓
Transizione (5 frame morph, 250 ms)  <- blend tra frame A e frame B
        ↓
Espressione B (10 frame, 500 ms)
```

**Eccezioni**:
- **Da sorpresa** (shock): transizione istantanea (0 frame), poi blend lento (500 ms) a emozione stabile.
- **Da innamorato**: transizione dolce e lenta (1 s).

---

## 6. Emote visuali (piccole iconografie)

Piccole sprite che appaiono sopra/accanto al corpo per esprimere stato velocemente:

```text
♡ ♡ ♡     Amore, felicità intensa
★ ★ ★     Eccitazione, sorpresa positiva
Z Z Z     Sonno
? ? ?     Confusione, incomprensione
! ! !     Shock, sorpresa negativa
💧 💧     Pianto, sudore
🌟       Imparare, idea
♪ ♫       Musica, gioco
```

**Comportamento**: appaiono a inizio azione, scale-in + bounce, rimangono 1–2 s, fade-out + scale-out.

---

## 7. Cicli di vita (crescita visibile)

### Cambio dimensione di corpo

Ogni 24 ore (age_hours), il corpo cresce di ~5–10%:

```text
Ora 0:     60×60 px (uovo)
Ora 12:    65×65 px
Ora 24:    70×70 px (neonato)
Ora 36:    75×75 px
Ora 48:    80×80 px (bambino)
Ora 96:    100×100 px (adolescente)
Ora 144:   110×110 px
Ora 240:   120×120 px (adulto)
```

La transizione è **graduale**: ogni 6 ore, il corpo scala leggermente (tween lineare, 100 ms).

### Cambio tratti somatici

- **Occhi**: 0–24h tondi, 24–96h normali, 96+ con espressività esagerata
- **Bocca**: 0–24h assente/puntino, 24–96h semplice, 96+ articolata
- **Corpo**: 0–24h sferico, 24–96h arrotondato, 96+ proporzionato
- **Colore**: sempre uguale, ma saturazione aumenta con age (neonato più pallido)

---

## 8. Implementazione tecnica

### Struttura sprite sheet

Per ogni fase (neonato, bambino, adolescente, adulto):

```
Spritesheet (512×1024 px):
┌─────────────────────────────────────────────────────────┐
│  ACTIONS (80×80 each)                                   │
├─────────────────────────────────────────────────────────┤
│ [IDLE_1][IDLE_2][IDLE_3][IDLE_4] [PLAY_1]...[PLAY_6]   │
│ [SLEEP_1]...[SLEEP_4] [FEED_1]...[FEED_8]              │
│ [PET_1]...[PET_4] [POOP_1]...[POOP_3]                  │
│ [WASH_1]...[WASH_6] [STUDY_1]...[STUDY_6]              │
│ [HAPPY_1]...[HAPPY_3] [SAD_1]...[SAD_3]                │
│ [SICK_1]...[SICK_3] [EMOTES] ...                       │
└─────────────────────────────────────────────────────────┘
```

### Codice LVGL (pseudocode)

```cpp
// Definisci spritesheet
lv_img_dsc_t sprite_baby = {
  .header.cf = LV_IMG_CF_TRUE_COLOR_ALPHA,
  .header.w = 512,
  .header.h = 1024,
  .data = (const uint8_t *) sprite_sheet_data,
  .data_size = sizeof(sprite_sheet_data),
};

// Crea immagine sprite
lv_obj_t *tamagotchi_img = lv_img_create(lv_scr_act());
lv_img_set_src(tamagotchi_img, &sprite_baby);

// Anima
void animate_feed() {
  lv_anim_t a;
  lv_anim_init(&a);
  lv_anim_set_var(&a, tamagotchi_img);
  lv_anim_set_exec_cb(&a, (lv_anim_exec_xcb_t) set_sprite_frame);
  lv_anim_set_duration(&a, 1000);  // 1 secondo
  lv_anim_set_values(&a, 0, 8);    // 8 frame
  lv_anim_set_repeat_count(&a, 1); // una volta
  lv_anim_start(&a);
}

void set_sprite_frame(lv_obj_t *img, int32_t frame_idx) {
  // Calcola crop rect per il frame
  // Aggiorna img.clip_area oppure usa offset
  // Ridisegna
}
```

### Asset management

**Strategia di caricamento**:
1. **Spritesheet unico** in memoria Flash (compressione: formato OPUS/WebP per immagini)
2. **Buffering**: durante loop, carica il prossimo frame dalla Flash al PSRAM (8 MB abbastanza)
3. **Caricamento OTA**: nuovi spritesheet scaricati dal server, installati via filesystem SPIFFS

**Dimensioni stimate**:
- Spritesheet per fase (512×1024, RGBA, compresso): ~150–200 KB
- 4 fasi: ~600–800 KB (con SPIFFS)
- PSRAM residuo (8–2=6 MB) per buffer audio, stato game engine, rete

---

## 9. Linee guida artistiche

### Proporzioni (golden ratio per Tamagotchi kawaii)

```
Testa:  0.35 altezza totale, 0.4 larghezza corpo
Occhi:  0.15 altezza testa, distanza interocchio = 0.2 larghezza testa
Corpo:  0.65 altezza totale, arrotondato (rx/ry = 0.8)
Zampe:  0.1 altezza totale, larghezza 0.15 corpo
```

### Principi di animazione

1. **Squash & Stretch**: quando salta, il corpo comprime in y alla terra, dilata in x
2. **Anticipation**: prima di un'azione, un lieve movimento opposto (se salta, si piega giù)
3. **Staging**: emozioni e azioni devono essere chiarissime in silenzio (no suono)
4. **Timing**: azioni rapide (gioco, shock) 300–600 ms, azioni lente (sonno, coccole) 1–2 s
5. **Arcs**: movimenti seguono archi morbidi, non linee rette
6. **Appeal**: gli occhi sono il 70% del fascino, mantengono sempre contrasto alto

### Colori per stato

| Stato | Modifica colore |
|---|---|
| Normale | palette base |
| Felice | +20% saturazione, +10% brightness, giallo glow |
| Triste | -30% saturazione, -15% brightness, grigio overlay |
| Affamato | luccichio occhi, contorno rosa delle labbra |
| Malato | -50% saturazione, grigio overlay 30%, shake |
| Eccitato | pulse brightness, +50% saturazione |

---

## 10. Roadmap animazioni

### V1 (MVP)

- ✅ Sprite base (neonato, bambino)
- ✅ Animazioni core: IDLE, FEED, PLAY, SLEEP, PET, POOP, WASH
- ✅ Transizioni emozioni (felice ↔ triste ↔ neutro)
- ✅ Emote visual (cuore, stella, Z)
- ⚠️ Growth (cambio dimensione ogni 24h)

### V2

- ✅ Tutte le fasi (adolescente, adulto)
- ✅ Animazioni avanzate: STUDY, SICK, DANCE
- ✅ Varianti ingrassamento (snello ↔ normale ↔ rotondo)
- ✅ Più espressioni (innamorato, confuso, eccitato)
- ✅ Effetti particelle (sparkle, sudore, bolle)

### V3

- ✅ Sistema di costume (cappello, occhiali, vesti — sprite layer aggiuntivo)
- ✅ Animazioni dinamiche basate su personalità (curiosità → movimento più attivo)
- ✅ Interazioni multi-touch (trascinare, pizzicare, soffiare — accelerometro)

---

## 11. Risorsa design

### Ispirazione stilistica

- **Tamagotchi classico** (1996): linee nere, 2 colori, 32×32 px
- **Nitendo Kirby**: forme tonde, occhi espressivi, colori pastello
- **Stardew Valley**: pixel art caldo, proporzioni umane ma cute
- **Animal Crossing**: occhi a U, corpo morbido, movimenti dolci

### Strumenti consigliati per creare sprite

- **Aseprite** (commerciale): timeline integrata, export spritesheet, perfetto per animazioni
- **Piskel** (gratuito, browser): semplice, export GIF/PNG, online
- **Krita** (gratuito, open-source): completo, brush pixel, animation docks
- **GIMP** (gratuito): basic ma sufficiente con plugin

### Font per label

- **Pixel font** (PressStart2P, PixelMplus): monospace pixel-art
- **Rounded font** (Nunito, OpenSans Rounded): UI labels, soft look

---

## 12. Checklist implementazione

- [ ] Definire palette colori finali (confermare con hex)
- [ ] Creare anatomia base (head, body, eyes, mouth separati per modularità)
- [ ] Disegnare 4 fasi (neonato, bambino, adolescente, adulto)
- [ ] Animare azioni principali (IDLE, FEED, PLAY, SLEEP, PET, POOP, WASH)
- [ ] Creare spritesheet per ogni fase (512×1024 min, compresso)
- [ ] Testare loading/rendering su ESP32-S3 con LVGL
- [ ] Misurare memory footprint e ottimizzare
- [ ] Aggiungere varianti ingrassamento (x3 per fase = 12 spritesheet)
- [ ] Implementare transizioni emozioni e emote
- [ ] Testare fluidità frame rate (target 30 FPS su ESP32)
- [ ] Aggiungere feedback tattile (vibrazione se disponibile)
- [ ] Documentare asset naming convention e versionamento

---

**Nota finale**: le animazioni sono il cuore del Tamagotchi. Un'animazione scattante o lenta farà sembrare il personaggio "morto"; un'animazione fluida e espressiva lo farà sembrare vivo. Investire tempo nelle transizioni e nei timing è cruciale per il successo del gioco.
