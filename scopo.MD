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

L'ESP32 non esegue il modello linguistico principale: si occupa solo di acquisire audio, mostrare l'interfaccia e riprodurre l'audio di risposta. Tutta l'intelligenza (STT, classificazione intent, Game Engine, TTS) vive sul server.

### Architettura corretta (Game Engine al centro)

```text
                         TAMAGOTCHI
                             │
                             ▼
                     ┌─────────────┐
                     │ GAME ENGINE │
                     │             │
                     │ Fame        │
                     │ Sete        │
                     │ Energia     │
                     │ Umore       │
                     │ Sonno       │
                     │ Età         │
                     │ Bisogni     │
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

- fame, energia, umore, salute;
- età, esperienza, crescita, evoluzione;
- emozioni e animazioni;
- regole del gioco.

### Flusso end-to-end

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

## 5. Memoria e personalità

Ogni Tamagotchi possiede un profilo persistente. Esempio:

```json
{
  "name": "Pippo",
  "age_days": 12,
  "mood": 91,
  "hunger": 42,
  "energy": 76,

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

Flusso principale:

```text
UTENTE PARLA
     ↓
ESP32 registra audio
     ↓
Wi-Fi
     ↓
Speech-to-Text sul server
     ↓
Classificazione intent
     ↓
Game Engine
     ↓
Text-to-Speech (opzionale)
     ↓
ESP32
     ↓
SPEAKER
```

Il sistema deve privilegiare risposte brevi per ridurre latenza e consumo.

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
| Fame | diminuisce nel tempo |
| Energia | diminuisce con le attività |
| Umore | dipende dalle interazioni |
| Salute | influenzata da fame, sonno e cura |
| Affetto | cresce con le interazioni |
| Esperienza | cresce nel tempo |
| Età | aumenta automaticamente |
| Personalità | evolve in base alle esperienze |
| Memoria | conserva eventi importanti |

### Il Tamagotchi vive autonomamente

Caratteristica fondamentale: il bambino non deve interagire perché il gioco funzioni. Il Game Engine continua a funzionare grazie a un clock interno:

```text
TIME
 │
 ├── ogni minuto
 │     ├── fame aumenta
 │     ├── energia cambia
 │     └── umore cambia
 │
 ├── ogni ora
 │     └── bisogni aggiornati
 │
 └── eventi
       ├── fame critica
       ├── sonno
       ├── tristezza
       └── richiesta attenzione
```

Esempio di sequenza autonoma:

1. **08:00** — Pippo è felice.
2. **12:00** — Fame al 60%: Pippo mostra un'animazione di fame.
3. **14:00** — Fame all'85%: Pippo diventa triste e visualizza un segnale di richiesta attenzione.
4. Il bambino chiede: *"Vuoi mangiare?"* → AI restituisce `CHECK_HUNGER` → Game Engine valuta `hunger > 80` → risposta `YES` → Pippo mostra fame sul display.
5. Il bambino dice: *"Mangia!"* → AI restituisce `FEED` → Game Engine aggiorna `hunger = 20`, `mood += 10`, `affection += 2` → il display mostra Pippo felice mentre mangia.

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
- Wi-Fi
- HTTP/WebSocket
- I2S audio
- OTA firmware update

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

Obiettivo: verificare che il riconoscimento vocale delle intenzioni in italiano funzioni bene, prima di comprare l'hardware. Nessuna memoria, nessun database, nessun TTS.

```text
Voce
 ↓
STT
 ↓
Classificazione intent
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
- server API
- STT
- classificatore intent (+ Tiny LLM opzionale)
- TTS
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
