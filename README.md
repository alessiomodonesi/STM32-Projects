# 📘 STM32 Projects Journey - Nucleo F446RE

Questa repository documenta il percorso di apprendimento e i progetti sviluppati sulla scheda di sviluppo **STM32 Nucleo-F446RE**.
Contiene il codice sorgente, le configurazioni hardware e le note operative per replicare gli esperimenti.

---

## 🛠 Hardware Setup

* **Scheda:** STMicroelectronics NUCLEO-F446RE (MCU: STM32F446RET6, 180MHz, Cortex-M4)
* **Connessione:** Cavo USB Type-A/C to **Mini-B**
* **OS:** macOS (Apple Silicon M1/M2/M3)
* **Pinout Reference:** [Link al Datasheet Ufficiale](datasheet.pdf) o vedi la serigrafia sulla scheda.
  
### 🚨 Configurazione Iniziale su Mac (Fondamentale)
Prima di iniziare, è necessario aggiornare il firmware del debugger ST-LINK integrato per garantire la compatibilità con i Mac recenti.

1.  Scaricare e installare **STM32CubeProgrammer**.
2.  Collegare la scheda via USB.
3.  Aprire CubeProgrammer > Cliccare **Firmware Upgrade** > **Open in update mode** > **Upgrade**.
4.  Chiudere CubeProgrammer (non serve tenerlo aperto durante lo sviluppo).

---

## 💻 Software & IDE

* **IDE Principale:** STM32CubeIDE (Eclipse-based)
* **Configuratore:** STM32CubeMX (Integrato nell'IDE o Standalone)

### 🐛 Troubleshooting: Menu "STM32 Project" Mancante
Se su macOS non compare la voce *File -> New -> STM32 Project*:
* **Soluzione:** Andare su **File -> New -> Other...** e cercare `STM32` nella barra di ricerca. Selezionare **STM32 Project** da lì.
* **Alternativa:** Usare **STM32CubeMX** esterno per generare il codice e importarlo successivamente nell'IDE.

---

## 📂 Struttura della Repository

Questa è una **Monorepo**. Ogni esercizio o progetto ha la sua cartella dedicata per mantenere l'indipendenza delle librerie HAL.

```text
STM32_Projects/
├── .gitignore           # File configurazione Git (ignora file di build/debug)
├── README.md            # Questo manuale
│
├── 01_Blink_LED/        # Progetto 1: Lampeggio LED base
├── 02_Button_Input/     # Progetto 2: Lettura Tasto e controllo LED
└── ...
```
> **Nota:** Quando si crea un nuovo progetto con STM32CubeMX/IDE, assicurarsi di selezionare la cartella `STM32_Projects` come root, in modo che venga creata la sottocartella specifica (es. `03_Relay/`) al suo interno.

## 📝 Snippet di Codice Utili

Tutto il codice utente va scritto nel file `Core/Src/main.c`, rigorosamente tra i commenti `/* USER CODE BEGIN ... */` e `/* USER CODE END ... */`.

### 1. Lampeggio LED (Blink)
Far lampeggiare il LED Verde (LD2) integrato sulla scheda.
* **Pin:** `PA5` (Definito come `LD2_Pin`)

```c
/* Inserire nel ciclo while(1) */
HAL_GPIO_TogglePin(LD2_GPIO_Port, LD2_Pin); // Inverte stato (ON/OFF)
HAL_Delay(500);                             // Attende 500ms
```
### 2. Lettura Pulsante (Digital Input)
Accendere il LED solo quando si preme il Tasto Blu (B1).
* **Pin:** `PC13` (Definito come `B1_Pin`)
* **Logica:** Il tasto è *Active Low* (0 = Premuto, 1 = Rilasciato).

```c
/* Inserire nel ciclo while(1) */
// Legge lo stato del pin. GPIO_PIN_RESET (0) significa PREMUTO.
if (HAL_GPIO_ReadPin(B1_GPIO_Port, B1_Pin) == GPIO_PIN_RESET)
{
    // Accendi il LED
    HAL_GPIO_WritePin(LD2_GPIO_Port, LD2_Pin, GPIO_PIN_SET);
}
else
{
    // Spegni il LED
    HAL_GPIO_WritePin(LD2_GPIO_Port, LD2_Pin, GPIO_PIN_RESET);
}
```

## ⚙️ Gestione Git (.gitignore)

Per evitare di caricare file spazzatura (compilati, debug, impostazioni locali), creare un file chiamato `.gitignore` nella cartella principale (root) e incollarci dentro questo contenuto:

```text
# Cartelle di output compilazione
Debug/
Release/
build/
Bin/

# File temporanei IDE e Metadata
.settings/
.mxproject
*.launch
*.su
*.d
*.o
*.elf
*.map

# Mac OS system files
.DS_Store
```

---

## 🧪 Esperimenti "Stand-alone" (Senza Breadboard)

In attesa della breadboard, questi esperimenti sfruttano l'hardware già integrato sulla scheda Nucleo (LED, Button, Debugger, Sensori interni) per esplorare le periferiche avanzate dell'STM32.

### 1. Manipolazione del Clock (RCC)
* **Obiettivo:** Capire l'albero dei clock.
* **Attività:** Modificare la frequenza della CPU tramite CubeMX (es. da 180MHz a 16MHz) e osservare visivamente come cambia la velocità di esecuzione (es. la velocità di lampeggio) senza modificare il codice.

### 2. PWM & "Breathing" LED
* **Obiettivo:** Generazione segnali e Duty Cycle.
* **Attività:** Configurare il Timer (TIM2 o TIM5) connesso al LED Verde (PA5) in modalità PWM Generation. Variare il *Duty Cycle* nel tempo per creare un effetto "respiro" (fading) invece del lampeggio netto.

### 3. Interrupt Esterni (EXTI)
* **Obiettivo:** Abbandonare il Polling per un codice reattivo.
* **Attività:** Configurare il Tasto Blu (PC13) in modalità Interrupt. Gestire l'accensione del LED tramite la funzione di callback `HAL_GPIO_EXTI_Callback`, liberando il loop principale.

### 4. Comunicazione Seriale (UART)
* **Obiettivo:** Inviare dati al PC.
* **Attività:** Attivare la USART2 (connessa via USB). Usare `printf` o `HAL_UART_Transmit` per inviare messaggi di debug ("Hello World", stato del bottone) visualizzabili su un terminale (es. CoolTerm, PuTTY, Serial Monitor).

### 5. Sensori Interni (ADC)
* **Obiettivo:** Leggere segnali analogici senza sensori esterni.
* **Attività:** Configurare l'ADC per leggere il **Sensore di Temperatura** interno e il canale **VREFINT** (Voltage Reference). Inviare i dati letti via UART per monitorare la temperatura della CPU in tempo reale.

---

## 🚀 Roadmap Aggiornata

- [x] Setup Ambiente e Firmware Update
- [x] Hello World (Blink LED)
- [ ] **Exp 1: Clock & Frequenze**
- [ ] **Exp 2: PWM Breathing LED**
- [ ] **Exp 3: Interrupts (EXTI)**
- [ ] **Exp 4: UART Communication**
- [ ] **Exp 5: Internal Temp Sensor (ADC)**
- [ ] Digital Input (Button Reading)
- [ ] Lettura Analogica Esterna (Potenziometro)
- [ ] Integrazione Relè e Transistor
