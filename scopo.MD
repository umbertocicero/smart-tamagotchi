# Tamagotchi AI — Progetto

## Scopo

Realizzare un piccolo animale virtuale fisico, portatile e interattivo che:

- ascolta l'utente tramite microfono;
- risponde vocalmente;
- visualizza espressioni e animazioni su un display touch;
- possiede bisogni e stato interno da vero Tamagotchi;
- sviluppa una personalità;
- conserva ricordi e interazioni nel tempo;
- utilizza un piccolo modello AI open-source eseguito su un server online;
- punta a un costo operativo software pari a 0 €/mese.

L'obiettivo non è creare semplicemente un chatbot, ma una creatura digitale persistente con cui l'utente sviluppa una relazione nel tempo.

**Principio architetturale fondamentale**: l'AI non è il cervello del Tamagotchi. Il cervello è il **Game Engine**, che gestisce bisogni, stato, emozioni e regole del gioco in autonomia, con un proprio "clock" interno. L'AI è solo l'**interprete della voce** dell'utente: ascolta e traduce ciò che viene detto in una intenzione (*intent*) predefinita, che il Game Engine poi esegue. Questo rende il modello AI necessario molto più semplice, veloce ed economico.

---

## 1. Hardware

### Scheda principale — Waveshare ESP32-S3-Touch-LCD-2

Caratteristiche principali:

- ESP32-S3R8
- 8 MB PSRAM
- 16 MB Flash
- display IPS touch 2" 240×320
- touch capacitivo
- Wi-Fi
- Bluetooth
- IMU a 6 assi
- microSD
- gestione batteria Li-Po
- connettore batteria MX1.25

La scheda costituisce il cuore del dispositivo.

### Componenti aggiuntivi

| Componente | Funzione | Quantità | Costo indicativo |
|---|---|---:|---:|
| INMP441 | Microfono MEMS I2S | 1 | ~€0,80 |
| MAX98357A | Amplificatore audio I2S | 1 | ~€0,80 |
| Speaker 4 Ω / 3 W | Riproduzione voce | 1 | ~€2–5 |
| Li-Po 3,7 V ~1000 mAh | Alimentazione | 1 | ~€7 |
| Waveshare ESP32-S3-Touch-LCD-2 | MCU + display + touch + IMU | 1 | ~€29 |

Costo hardware iniziale stimato: circa **€40–43**.

> Verificare sempre connettore, polarità e dimensioni della batteria prima dell'acquisto.

### Componenti non necessari nella prima versione

- Camera OV5640
- microSD
- pulsanti fisici
- accelerometro esterno
- modulo TP4056
- DAC esterno
- LED RGB

---

## 2. Architettura hardware

```text
                    ┌─────────────────────────┐
                    │   TAMAGOTCHI AI         │
                    │                         │
                    │ ESP32-S3R8              │
                    │ 8 MB PSRAM / 16 MB      │
                    │                         │
                    │ LCD Touch 2"            │
                    │ IMU                     │
                    │ Wi-Fi / Bluetooth       │
                    └───────┬─────────┬───────┘
                            │         │
                          I2S       I2S
                            │         │
                       ┌────▼───┐ ┌──▼────────┐
                       │ INMP441│ │ MAX98357A │
                       │ MIC    │ │ AMP       │
                       └────────┘ └────┬──────┘
                                       │
                                    Speaker

                         USB-C / Li-Po 3.7 V
```

---

## 3. Architettura software

L'intelligenza è organizzata su **due livelli** (architettura tiered), in modo che il dispositivo resti utilizzabile anche senza rete e il server intervenga solo quando serve davvero:

- **Tier 0 — On-device (ESP32-S3, offline)**: il Game Engine gira sempre a bordo. Per la voce, l'ESP32 esegue un riconoscimento comandi leggero (ESP-SR) e un **NLU-lite** basato su matching/distanza tra parole per mappare le frasi su un set ristretto di intent. Il feedback audio usa clip pre-registrate. Questo copre l'80% delle interazioni quotidiane (nutrire, giocare, accarezzare, dormire, stato) con latenza minima e costo zero.
- **Tier 1 — Server (online, opzionale)**: quando la frase non viene riconosciuta a bordo o serve una conversazione più ricca, l'audio viene inviato al server che esegue STT accurato (es. Whisper), un classificatore/LLM più potente e TTS neurale.

Il Game Engine resta l'unico "cervello": entrambi i livelli si limitano a produrre un `intent` che il Game Engine esegue.

```text
            ┌──────────────────────────── ESP32-S3 (Tier 0) ────────────────────────────┐
            │                                                                            │
  VOCE ──►  │  I2S mic ─► Wake word (ESP-SR) ─► Command reco ─► NLU-lite (word-distance)  │
            │                                        │ riconosciuto?                     │
            │                             ┌──────────┴──────────┐                         │
            │                          SÌ │                     │ NO / bassa confidenza   │
            │                             ▼                     ▼                         │
            │                        GAME ENGINE           (se online)                    │
            │                             │                     │                         │
            │                     clip audio locale            └──► SERVER (Tier 1) ──┐    │
            └─────────────────────────────┼──────────────────────────────────────────┼────┘
                                           ▼                                          ▼
                                    DISPLAY / SPEAKER                    STT ► NLU/LLM ► TTS ► intent
```

### Architettura corretta (Game Engine al centro)

```text
                         TAMAGOTCHI
                             │
                             ▼
                     ┌─────────────┐
                     │ GAME ENGINE │
                     │             │
                     │ hunger      │
                     │ thirst      │
                     │ energy      │
                     │ mood        │
                     │ sleep       │
                     │ age         │
                     │ needs       │
                     └──────┬──────┘
                            │ decide cosa fare
                            ▼
                     ┌─────────────┐
                     │   DISPLAY   │
                     │             │
                     │ animazioni  │
                     │ emozioni    │
                     │ segnali     │
                     └─────────────┘

BAMBINO
   │
   │ "Hai fame?" / "Mangia" / "Giochiamo?"
   ▼
┌─────────────┐
│     AI      │
│ interpreta  │
│ intenzione  │
└──────┬──────┘
       │
       ▼
  INTENT (JSON)  ──────────────►  GAME ENGINE
```

**Principio fondamentale**: il Game Engine deve rimanere separato dal modello AI.

Il modello AI decide **solo**:

- qual è l'intenzione (`intent`) espressa dalla frase dell'utente;
- eventuali parametri semplici associati (es. nome scelto, oggetto indicato).

Il Game Engine gestisce **tutto il resto**:

- `hunger`, `energy`, `mood`, `health`;
- `age`, `experience`, crescita, evoluzione;
- emozioni e animazioni;
- regole del gioco.

### Flusso end-to-end (Tier 1 — fallback server)

Usato solo quando il Tier 0 non risolve a bordo (linguaggio libero, risposta parlata nuova):

```text
                  TAMAGOTCHI
                      │
                      │ Wi-Fi
                      ▼
              ┌─────────────────┐
              │   SERVER ONLINE │
              │                 │
              │ Speech-to-Text  │
              │        ↓        │
              │  Classificatore │
              │     intent      │
              │        ↓        │
              │  Game Engine    │
              │        ↓        │
              │ Text-to-Speech  │
              │   (opzionale)   │
              └────────┬────────┘
                       │
                       ▼
                    ESP32
                       │
                    Speaker
```

---

## 4. AI — interprete delle intenzioni

### Requisiti

Il modello deve essere:

- open-source / con licenza compatibile con il progetto;
- molto piccolo;
- eseguibile su CPU;
- quantizzabile;
- installabile su un server proprio;
- senza API a pagamento.

### Un compito molto più semplice del previsto

Grazie alla separazione tra AI e Game Engine, non serve un modello "intelligente" in senso ampio: serve un modello capace di **classificare bene 10–20 intenzioni predefinite**, ricevute in linguaggio naturale (italiano) via voce.

Esempi di intent:

`FEED` · `PLAY` · `SLEEP` · `WAKE` · `PET` · `CHECK_STATUS` · `CHECK_HUNGER` · `CHECK_MOOD` · `GREET` · `STOP` · `YES` · `NO`

Possibili estensioni future: `TALK` · `SING` · `DANCE` · `WASH` · `GIVE_ITEM`.

Esempi di traduzione frase → intent:

| Frase del bambino | Intent restituito |
|---|---|
| "Pippo, vuoi mangiare?" | `{ "intent": "CHECK_HUNGER" }` |
| "Dai, mangia!" | `{ "intent": "FEED" }` |
| "Giochiamo?" | `{ "intent": "PLAY" }` |
| "Come stai?" | `{ "intent": "CHECK_STATUS" }` |

È poi il Game Engine a decidere cosa succede realmente (es. se `hunger > 80` risponde in un modo, altrimenti in un altro).

### Modello LLM (opzionale)

Se in futuro serve una conversazione più ricca (non solo intent), si può affiancare un Tiny LLM:

- modello iniziale da valutare: **Qwen2.5-0.5B-Instruct** o equivalente;
- quantizzato (es. 4-bit) per ridurre RAM e consumo CPU;
- non gestisce la logica del gioco: riceve un contesto compatto e produce risposte brevi.

Il LLM resta comunque **opzionale**: un semplice modello speech-intent (STT → classificatore di intenzione) può essere sufficiente da solo, con tre vantaggi:

- latenza molto più bassa;
- modello molto più piccolo;
- server molto meno costoso.

---

## 4-bis. STT, TTS e NLU possono girare sull'ESP32-S3?

L'ESP32-S3R8 (8 MB PSRAM, 16 MB Flash, vector instructions) **non** può eseguire STT/TTS/NLU "full", ma può eseguire versioni **lite** sufficienti per il gioco. Sintesi di fattibilità:

| Funzione | On-device (Tier 0) | Server (Tier 1) | Note |
|---|---|---|---|
| **Wake word** | ✅ Sì — **ESP-SR / WakeNet** | — | Rilevamento di una parola di attivazione (es. "Pippo") a basso consumo. |
| **STT comandi** | ⚠️ Parziale — **ESP-SR / MultiNet** | ✅ Whisper | MultiNet riconosce un set fisso di comandi brevi (decine). Italiano non nativo: si usano comandi fonetici brevi o si addestra un vocabolario ridotto. |
| **STT dettato libero** | ❌ No | ✅ Whisper (base/small) | Trascrizione di frasi arbitrarie: solo lato server. |
| **NLU** | ✅ Sì — **NLU-lite** (vedi §4-ter) | ✅ classificatore / Tiny LLM | On-device: keyword + distanza tra parole su intent predefiniti. |
| **TTS** | ⚠️ Limitato — clip pre-registrate / TTS concatenativo | ✅ TTS neurale | A bordo conviene riprodurre campioni audio pre-registrati (WAV/OPUS in Flash). Sintesi neurale in tempo reale: solo lato server. |

**Conclusione**: il percorso vocale essenziale (wake word → comando → intent → reazione + clip audio) è realizzabile **interamente a bordo**, offline. Il server serve solo per il linguaggio libero, risposte parlate "nuove" e conversazione avanzata.

---

## 4-ter. NLU-lite on-device (matching per distanza tra parole)

Invece di un modello linguistico, il Tier 0 usa un **classificatore a regole fuzzy**: un dizionario di parole-chiave per ogni intent e un confronto per similarità che tollera errori di STT, plurali e varianti.

### Lessico intent → parole chiave (esempio)

```json
{
  "FEED":         ["mangia", "mangiare", "pappa", "cibo", "fame", "nutri"],
  "PLAY":         ["gioca", "giochiamo", "partita", "giocare", "divertiamoci"],
  "SLEEP":        ["dormi", "nanna", "sonno", "riposa", "buonanotte"],
  "PET":          ["coccola", "carezza", "accarezza", "bravo"],
  "CHECK_STATUS": ["come", "stai", "stato", "tutto", "bene"],
  "GREET":        ["ciao", "buongiorno", "ehi", "salve"],
  "YES":          ["si", "certo", "ok", "va", "bene"],
  "NO":           ["no", "basta", "stop", "fermati"]
}
```

### Algoritmo

```text
1. NORMALIZZA il testo STT:
   - minuscolo, rimozione accenti/punteggiatura
   - rimozione stopword ("il", "la", "un", "di", "dai"...)
   - tokenizzazione in parole

2. Per ogni TOKEN e per ogni KEYWORD del lessico:
   - calcola la distanza di Levenshtein (edit distance)
   - match se distanza <= soglia (es. <= 1 per parole corte, <= 2 per lunghe)
     → tollera "mangja", "manga", "mangiaa" ecc. da STT imperfetto

3. PUNTEGGIO per intent = somma dei match pesati
   (match esatto = 1.0, match fuzzy = 0.6)

4. SELEZIONE:
   - intent con punteggio massimo, se >= soglia_confidenza → esegui
   - altrimenti:
       online?  → inoltra audio/testo al SERVER (Tier 1)
       offline? → intent = ASK_REPEAT ("non ho capito", animazione dubbio)
```

### Perché funziona bene qui

- Il dominio è **chiuso** (10–20 intent): non serve capire il linguaggio, solo distinguere poche azioni.
- L'**edit distance** assorbe gli errori tipici dello STT su comandi brevi.
- È **deterministico, minuscolo e velocissimo**: poche KB di lessico, nessuna inferenza pesante, gira in pochi ms sull'ESP32-S3.
- Degrada con eleganza: se non è sicuro, chiede di ripetere (offline) o passa al server (online).

> In pratica il vocabolario può essere versionato sul server e scaricato via OTA, così si aggiungono intent/sinonimi senza ricompilare il firmware.

---

## 5. Memoria e personalità

Ogni Tamagotchi possiede un profilo persistente. Esempio:

```json
{
  "name": "Pippo",
  "age_days": 12,
   "state": {
    "hunger": 42,
    "energy": 76,
    "hygiene": 81,
    "health": 88,
    "affection": 91,
    "fun": 63,
    "stress": 22,
    "experience": 37

   "emotion": {
      "current": "HAPPY",
      "scores": {
         "happy": 78,
         "sad": 21,
         "angry": 16,
         "sleepy": 19,
         "sick": 8,
         "playful": 69
      }
   },

  "personality": {
    "curiosity": 87,
    "affection": 91,
    "humor": 65,
    "independence": 34
  },

  "memories": [
    "L'utente ha scelto il nome Pippo",
    "Gli piace giocare",
    "Ieri ha mangiato una mela"
  ]
}
```

La memoria viene salvata sul server e utilizzata per costruire il contesto del modello. Questo permette a un modello molto piccolo di apparire molto più coerente e personale.

---

## 6. Interazione vocale

Il flusso è **on-device-first**: si prova prima a risolvere tutto a bordo (veloce e offline), e solo in caso di incertezza si passa al server.

```text
UTENTE PARLA
     ↓
Wake word (ESP-SR)              ← attiva l'ascolto (es. "Pippo")
     ↓
ESP32 registra audio (I2S)
     ↓
Command reco + NLU-lite (a bordo)
     ↓
  riconosciuto con confidenza?
     ├── SÌ ─────────────► Game Engine ─► reazione visiva + clip audio locale
     │
     └── NO / conversazione ricca:
              ├── online  → Wi-Fi → STT server → NLU/LLM → Game Engine → TTS → Speaker
              └── offline → intent ASK_REPEAT (chiede di ripetere)
```

Il sistema deve privilegiare risposte brevi per ridurre latenza e consumo. Le reazioni base (mangiare, giocare, dormire, coccole, stato) devono restare **istantanee e locali**; il server è riservato al linguaggio libero e alle risposte parlate "nuove".

---

## 7. Interazione touch

Il display touch permette di evitare inizialmente i pulsanti fisici.

Possibili interazioni:

- accarezzare;
- nutrire;
- giocare;
- dormire;
- parlare;
- visualizzare statistiche;
- aprire inventario;
- visualizzare ricordi.

### Le emozioni sono principalmente visive

Non tutto deve passare dalla voce: il personaggio comunica soprattutto tramite il display, che è a tutti gli effetti il suo volto.

- **Espressioni**: felice, triste, arrabbiato, assonnato, affamato, spaventato, innamorato, eccitato.
- **Animazioni**: saltella, trema, dorme, mangia, corre, si gira, guarda il bambino, si arrabbia.

Il display 240×320 diventa quindi il vero volto del personaggio; la voce è solo un canale aggiuntivo.

---

## 8. Stato del Tamagotchi

Parametri iniziali:

| Parametro | Descrizione |
|---|---|
| `hunger` | aumenta nel tempo (0 = sazio, 100 = affamato) |
| `energy` | diminuisce con attività e tempo (0 = esausto, 100 = pieno) |
| `hygiene` | diminuisce nel tempo/eventi sporco (0 = sporco, 100 = pulito) |
| `health` | influenzata da fame, sonno, igiene e cura |
| `affection` | cresce con interazioni positive, cala con trascuratezza |
| `fun` | cresce con gioco, cala con inattività |
| `stress` | cresce con bisogni critici, cala con cura e routine |
| `experience` | cresce nel tempo e con attività (studio/gioco) |
| `age` | aumenta automaticamente |
| `personality` | evolve in base alle esperienze |
| `memory` | conserva eventi importanti |

### Modello unificato stato + emozioni

Tutti gli stati sono normalizzati su scala `0..100` e aggiornati con regole additive:

```text
nuovo_valore = clamp(vecchio_valore + delta_tempo + delta_azione + delta_evento, 0, 100)
```

Le emozioni non sono valori indipendenti "hardcoded": sono il risultato della somma pesata degli stati correnti.

```text
happy_score   = 0.30*(100-hunger) + 0.25*energy + 0.20*affection + 0.15*fun + 0.10*hygiene - 0.25*stress
sad_score     = 0.30*hunger + 0.25*(100-energy) + 0.20*(100-affection) + 0.15*(100-fun) + 0.10*stress
angry_score   = 0.40*stress + 0.25*hunger + 0.20*(100-energy) + 0.15*event_frustration
sleepy_score  = 0.60*(100-energy) + 0.20*hunger + 0.20*age_phase
sick_score    = 0.50*(100-health) + 0.25*(100-hygiene) + 0.25*hunger
playful_score = 0.35*energy + 0.30*fun + 0.20*affection + 0.15*(100-stress)
```

L'emozione visualizzata è quella con score massimo, con una logica di priorità per evitare conflitti:

1. `SICK` (se `health < 30`)
2. `SLEEPY` (se `energy < 20`)
3. `HUNGRY` (se `hunger > 80`)
4. `DIRTY` (se `hygiene < 25`)
5. altrimenti massimo tra `HAPPY`, `SAD`, `ANGRY`, `PLAYFUL`, `CALM`

Questo rende coerente la pipeline: bisogni -> stati numerici -> emozione risultante -> animazione.

### Delta canonici delle azioni (coerenti con animazioni)

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

> Nota: tutti i delta sono cumulativi e poi clampati in `0..100`; la stessa logica vale sia online sia offline.

### Il Tamagotchi vive autonomamente

Caratteristica fondamentale: il bambino non deve interagire perché il gioco funzioni. Il Game Engine continua a funzionare grazie a un clock interno:

```text
TIME
 │
 ├── ogni minuto
 │     ├── hunger aumenta
 │     ├── energy cambia
 │     └── mood cambia
 │
 ├── ogni ora
 │     └── bisogni aggiornati
 │
 └── eventi
       ├── hunger critico
       ├── sonno
       ├── tristezza
       └── richiesta attenzione
```

Esempio di sequenza autonoma:

1. **08:00** — Pippo è felice.
2. **12:00** — Fame al 60%: Pippo mostra un'animazione di fame.
3. **14:00** — Fame all'85%: Pippo diventa triste e visualizza un segnale di richiesta attenzione.
4. Il bambino chiede: *"Vuoi mangiare?"* → AI restituisce `CHECK_HUNGER` → Game Engine valuta `hunger > 80` → risposta `YES` → Pippo mostra fame sul display.
5. Il bambino dice: *"Mangia!"* → AI restituisce `FEED` → Game Engine applica i delta canonici (`hunger -= 40`, `energy -= 5`, `affection += 2`, `stress -= 4`) → il display ricalcola l'emozione risultante e mostra Pippo più sereno.

---

## 9. Personalità evolutiva

La personalità non deve essere completamente casuale. Esempio:

```text
Curiosità       87
Affetto         91
Indipendenza    34
Umorismo        65
Timidezza       20
Energia         78
```

Le interazioni dell'utente modificano lentamente questi valori. Il Tamagotchi può quindi sviluppare comportamenti differenti, ad esempio:

- utente che gioca molto → creatura più giocosa;
- utente che parla spesso → maggiore affetto;
- utente che trascura il personaggio → comportamento più triste;
- molte interazioni → maggiore familiarità.

---

## 10. Offline mode

Il dispositivo deve continuare a funzionare anche se il server non è raggiungibile. In assenza di connessione:

- il Game Engine continua a funzionare;
- fame/energia/umore continuano ad aggiornarsi;
- **la voce continua a funzionare** per i comandi base tramite wake word + NLU-lite on-device;
- il feedback audio usa clip pre-registrate in Flash;
- vengono registrati gli eventi;
- vengono utilizzate risposte locali predefinite;
- gli eventi vengono sincronizzati quando torna la connessione.

```text
                 SERVER OFFLINE
                       │
                       ▼
                 ESP32 continua
                 a funzionare
                       │
                       ▼
                 eventi locali
                       │
                 connessione OK
                       │
                       ▼
                   SYNC SERVER
```

---

## 11. Server

**Obiettivo**: ospitare il sistema AI senza costi mensili.

Prima opzione da valutare: **Oracle Cloud Free Tier**.

Possibile struttura:

```text
┌───────────────────────────────────────┐
│         SERVER ONLINE                 │
│                                        │
│ Linux VM                              │
│                                        │
│ API Backend                           │
│ Classificatore intent / Tiny LLM      │
│ STT                                    │
│ TTS                                    │
│ Database                              │
└───────────────────────────────────────┘
```

Per il primo prototipo si può utilizzare anche un PC locale per sviluppare e testare tutto il sistema prima del deployment online.

---

## 12. Stack software iniziale

**ESP32**

- Arduino framework oppure ESP-IDF
- LVGL per interfaccia grafica
- **ESP-SR** (WakeNet per wake word + MultiNet per riconoscimento comandi offline)
- **NLU-lite** on-device (lessico intent + matching per distanza di Levenshtein)
- clip audio pre-registrate in Flash per il feedback vocale offline
- Wi-Fi
- HTTP/WebSocket
- I2S audio
- OTA firmware update (incluso aggiornamento del lessico intent)

**Server**

- Linux
- Backend Node.js oppure Python
- REST/WebSocket
- database SQLite/PostgreSQL
- classificatore intent / Tiny LLM locale
- STT locale
- TTS locale

Tutto il più possibile open-source.

---

## 13. Roadmap

### V0 — Test AI (validazione intent)

Obiettivo: verificare che il riconoscimento delle intenzioni in italiano funzioni bene, su **due binari paralleli** da confrontare:

1. **NLU-lite (on-device candidate)**: STT → normalizzazione → matching per distanza tra parole → intent. Da provare prima su PC (simulando l'ESP32) per misurare accuratezza e robustezza agli errori di STT.
2. **NLU server**: STT (Whisper) → classificatore/LLM → intent, come riferimento di qualità "alto".

Nessuna memoria, nessun database, nessun TTS in questa fase.

```text
Voce
 ↓
STT
 ↓
 ├─► NLU-lite (word-distance)  ─► intent   (target on-device)
 └─► NLU server (classifier/LLM) ─► intent   (riferimento)
 ↓
JSON { "intent": "..." }
```

Esempi da testare:

- "Ciao Pippo, hai fame?" → `CHECK_HUNGER`
- "Dai Pippo, mangia!" → `FEED`
- "Facciamo una partita?" → `PLAY`
- "Come stai?" → `CHECK_STATUS`

Può essere realizzato con un servizio online gratuito/economico, senza GPU e senza installare nulla in locale. Una volta validato il riconoscimento delle intenzioni in italiano, si sceglie il modello/server definitivo.

### V1 — Prototipo hardware

ESP32-S3 + display + microfono + speaker. Funzioni: parlare, ascoltare, rispondere, animazioni, touch, Wi-Fi.

### V2 — Tamagotchi

Aggiungere: fame, energia, umore, salute, crescita, minigiochi.

### V3 — Personalità e memoria

Aggiungere: memoria a lungo termine, personalità dinamica, eventi importanti, relazione con l'utente.

### V4 — Server online

Deploy del backend e AI su infrastruttura gratuita.

### V5 — Prodotto

PCB personalizzato, case dedicato, ottimizzazione consumi, UI definitiva, eventuale camera, aggiornamenti OTA, supporto a più Tamagotchi.

---

## 14. MVP definitivo

La prima versione funzionante deve essere:

**Hardware**

- Waveshare ESP32-S3-Touch-LCD-2
- INMP441
- MAX98357A
- speaker
- batteria Li-Po

**Software**

- firmware ESP32
- UI touch
- Game Engine
- **NLU-lite on-device** (ESP-SR + matching per distanza tra parole) con clip audio locali
- server API (fallback online)
- STT (server)
- classificatore intent (+ Tiny LLM opzionale, server)
- TTS (server)
- database memoria

**Esperienza**

L'utente accende il dispositivo, dà un nome alla creatura e inizia a interagire. La creatura:

1. ascolta;
2. comprende (tramite classificazione dell'intent);
3. risponde (tramite il Game Engine);
4. reagisce visivamente;
5. modifica il proprio stato;
6. ricorda gli eventi importanti;
7. sviluppa progressivamente una personalità.

---

## Obiettivo finale

Creare un Tamagotchi AI open-source e a costo operativo zero, in cui un hardware economico diventa il corpo della creatura, il **Game Engine** ne controlla la vita (bisogni, stato, emozioni, tempo) e un modello AI leggero (classificatore di intent, con un Tiny LLM opzionale) ne interpreta la voce.

Il valore del progetto non è la potenza dell'AI, ma la combinazione di:

**corpo + voce + memoria + personalità + stato + tempo.**

## Libreria occhi

[LVGL KAWAII FACE](https://github.com/0015/lvgl_kawaii_face)
[GIF](https://www.pixellab.ai/create?tool=create_image_pro)
