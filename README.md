# 📘 STM32 Lab Journey - Nucleo F446RE

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
STM32_Lab/
├── .gitignore           # File configurazione Git (ignora file di build/debug)
├── README.md            # Questo manuale
│
├── 01_Blink_LED/        # Progetto 1: Lampeggio LED base
├── 02_Button_Input/     # Progetto 2: Lettura Tasto e controllo LED
└── ...
```
> **Nota:** Quando si crea un nuovo progetto con STM32CubeMX/IDE, assicurarsi di selezionare la cartella `STM32_Lab` come root, in modo che venga creata la sottocartella specifica (es. `03_Relay/`) al suo interno.

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

### 3. PWM Breathing LED (Effetto Respiro)
Generare un segnale PWM sul pin del LED per variare la luminosità gradualmente.
* **Pin:** `PA5` (Timer 2, Canale 1)
* **Clock CPU:** 100 MHz

#### Calcolo Parametri Timer (1 kHz PWM)
Vogliamo che il Timer conti a 1 MHz (1 tick = 1 µs) e si resetti ogni 1000 tick (1 ms).

1.  **Prescaler (PSC):** Divide il clock della CPU per ottenere la velocità del contatore.
    * `PSC = (Clock_CPU / Frequenza_Target_Contatore) - 1`
    * `PSC = (100.000.000 / 1.000.000) - 1` = **99**
2.  **Period (ARR):** Il numero di tick prima del reset (definisce la frequenza PWM).
    * `ARR = (Frequenza_Contatore / Frequenza_PWM) - 1`
    * `ARR = (1.000.000 / 1.000) - 1` = **999**

```c
/* 1. Avviare il Timer PWM prima del while(1) */
/* USER CODE BEGIN 2 */
HAL_TIM_PWM_Start(&htim2, TIM_CHANNEL_1);
/* USER CODE END 2 */

/* 2. Loop per l'effetto respiro nel while(1) */
/* USER CODE BEGIN 3 */

// Aumenta luminosità (0 -> 999)
for(int duty = 0; duty < 1000; duty += 10)
{
    __HAL_TIM_SET_COMPARE(&htim2, TIM_CHANNEL_1, duty);
    HAL_Delay(10);
}

// Diminuisci luminosità (999 -> 0)
for(int duty = 1000; duty > 0; duty -= 10)
{
    __HAL_TIM_SET_COMPARE(&htim2, TIM_CHANNEL_1, duty);
    HAL_Delay(10);
}

HAL_Delay(500); // Pausa
/* USER CODE END 3 */
```

### 4. Interrupt Pulsante (EXTI)
Gestire la pressione del tasto in modo asincrono (senza bloccare il loop principale) usando una funzione di *callback*.
* **Configurazione CubeMX:**
    * Pin `PC13`: Modalità **GPIO_EXTI13**.
    * NVIC: Abilitare **EXTI line[15:10] interrupts**.

```c
/* ATTENZIONE: Questa funzione va scritta FUORI dal main(),
   nella sezione USER CODE BEGIN 4 */

void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)
{
    // Verifica che l'interrupt arrivi dal Tasto Blu (B1)
    if (GPIO_Pin == B1_Pin)
    {
        // Inverte lo stato del LED
        HAL_GPIO_TogglePin(LD2_GPIO_Port, LD2_Pin);

        // Piccolo ciclo di ritardo per Debounce (solo a scopo didattico per evitare rimbalzi)
        for(int i=0; i<100000; i++);
    }
}
```

### 5. Comunicazione Seriale (UART)
Inviare messaggi di testo al PC tramite la porta USB virtuale (Virtual COM Port) integrata nell'ST-LINK.
* **Periferica:** USART2
* **Pin:** `PA2` (TX) e `PA3` (RX)
* **Configurazione:** 115200 bps, 8 Bits, No Parity, 1 Stop Bit.

```c
/* 1. Aggiungere gli include necessari (USER CODE BEGIN Includes) */
#include <string.h>
#include <stdio.h>

/* 2. Inserire nel ciclo while(1) */
char msg[] = "Hello from STM32!\r\n"; // \r\n per andare a capo

// Invia la stringa via UART
// &huart2      -> Handle della periferica
// (uint8_t*)msg -> Casting del messaggio a array di byte
// strlen(msg)  -> Calcola lunghezza stringa (richiede <string.h>)
// 100          -> Timeout in ms
HAL_UART_Transmit(&huart2, (uint8_t*)msg, strlen(msg), 100);

HAL_Delay(1000); // Invia ogni secondo
```

### 6. Internal Temp Sensor (ADC) & Non-Blocking I/O
* **Obiettivo:** Leggere la temperatura della CPU e inviarla via UART, gestendo il flusso con il Tasto Blu in modo reattivo (senza `HAL_Delay` bloccanti).
* **Periferica:** ADC1 (Channel: Temperature Sensor).
* **Configurazione:** Sampling Time **480 Cycles** (necessario per stabilità).
* **Logica Avanzata:** Utilizzo di `HAL_GetTick()` per gestire le tempistiche. Questo permette al processore di controllare il bottone migliaia di volte al secondo anche mentre aspetta di inviare i dati.

**Codice Sorgente (`main.c`):**

```c
/* 1. Definizioni Indirizzi Calibrazione (Specifici per STM32F446) */
#define TS_CAL1_ADDR ((uint16_t*)((uint32_t)0x1FFF7A2C)) // Calibrazione a 30°C
#define TS_CAL2_ADDR ((uint16_t*)((uint32_t)0x1FFF7A2E)) // Calibrazione a 110°C

/* 2. Variabili Globali (USER CODE BEGIN PV) */
uint8_t system_active = 1;    // Flag stato: 1=ON, 0=PAUSA
uint32_t last_time_sent = 0;  // Timer software

/* 3. Loop Principale (Non-Blocking) */
while (1)
{
    // A. Controllo Tasto Reattivo (Polling continuo)
    if (HAL_GPIO_ReadPin(B1_GPIO_Port, B1_Pin) == GPIO_PIN_RESET)
    {
        system_active = !system_active; // Toggle ON/OFF
        HAL_Delay(300); // Debounce
    }

    // B. Logica a tempo (Esegue ogni 1000ms senza bloccare il tasto)
    if (system_active && (HAL_GetTick() - last_time_sent > 1000))
    {
        last_time_sent = HAL_GetTick(); // Reset timer

        // Avvio ADC e Lettura
        HAL_ADC_Start(&hadc1);
        if (HAL_ADC_PollForConversion(&hadc1, 100) == HAL_OK)
        {
            uint16_t rawValue = HAL_ADC_GetValue(&hadc1);

            // Calcolo Temperatura (Interpolazione Lineare)
            int32_t cal1 = *TS_CAL1_ADDR;
            int32_t cal2 = *TS_CAL2_ADDR;
            float temp = ((float)(rawValue - cal1) / (float)(cal2 - cal1)) * 80.0f + 30.0f;

            // Stampa "Float" usando interi (Workaround per sprintf nano-lib)
            int p_int = (int)temp;
            int p_dec = (int)((temp - p_int) * 100);

            char msg[50];
            sprintf(msg, "Stato: ON | Temp CPU: %d.%02d C\r\n", p_int, p_dec);
            HAL_UART_Transmit(&huart2, (uint8_t*)msg, strlen(msg), 100);
            
            HAL_GPIO_TogglePin(LD2_GPIO_Port, LD2_Pin); // Blink Feedback
        }
        HAL_ADC_Stop(&hadc1);
    }
    
    // Feedback visivo in pausa
    if (!system_active) HAL_GPIO_WritePin(LD2_GPIO_Port, LD2_Pin, GPIO_PIN_RESET);
}
```

### 7. Real Time Clock (RTC) & Alarm (LSE Crystal)
* **Obiettivo:** Trasformare il microcontrollore in un orologio preciso usando il quarzo esterno e scatenare un evento (Sveglia) tramite interrupt.
* **Periferica:** RTC (Real Time Clock).
* **Clock Source:** **LSE** (Low Speed External) a 32.768 kHz (Cristallo fisico sulla scheda).
* **Precisione:** Elevata (errore di pochi secondi al mese).

**Configurazione Hardware (CubeMX):**
1.  **RCC:** Low Speed Clock (LSE) -> **Crystal/Ceramic Resonator**.
2.  **Clock Config:** RTC Source Mux -> **LSE**.
3.  **RTC Parameter:** * `Asynchronous Prediv`: **127**
    * `Synchronous Prediv`: **255**
    * *(127+1) * (255+1) = 32768 Hz -> 1 Hz esatto.*
4.  **NVIC:** Abilitare **RTC alarm interrupt through EXTI line 17**.

**Codice Sorgente (`main.c`):**

**A. Setup Iniziale (USER CODE BEGIN 2)**
Impostiamo l'orario a 12:00:00 e la sveglia a 12:00:10.

```c
// 1. Imposta Orario Iniziale
RTC_TimeTypeDef sTime = {0};
sTime.Hours = 0x12; // 12 (Formato BCD)
sTime.Minutes = 0x00;
sTime.Seconds = 0x00;
HAL_RTC_SetTime(&hrtc, &sTime, RTC_FORMAT_BCD);

// 2. Imposta Allarme (+10 secondi)
RTC_AlarmTypeDef sAlarm = {0};
sAlarm.AlarmTime.Hours = 0x12;
sAlarm.AlarmTime.Minutes = 0x00;
sAlarm.AlarmTime.Seconds = 0x10; // Suona alle 12:00:10
sAlarm.Alarm = RTC_ALARM_A;
sAlarm.AlarmMask = RTC_ALARMMASK_DATEWEEKDAY; // Ignora la data, guarda solo l'ora
HAL_RTC_SetAlarm_IT(&hrtc, &sAlarm, RTC_FORMAT_BCD);

char msg[] = "RTC Avviato (LSE). Allarme tra 10 secondi...\r\n";
HAL_UART_Transmit(&huart2, (uint8_t*)msg, strlen(msg), 100);
```

**B. Gestione Interrupt (USER CODE BEGIN 4)**
La funzione chiamata quando scatta l'allarme.

```c
void HAL_RTC_AlarmAEventCallback(RTC_HandleTypeDef *hrtc)
{
    char msg[] = "\r\n[ALARM] DRIIIN! SVEGLIA !!!\r\n";
    HAL_UART_Transmit(&huart2, (uint8_t*)msg, strlen(msg), 100);
    
    // Accendi il LED fisso
    HAL_GPIO_WritePin(LD2_GPIO_Port, LD2_Pin, GPIO_PIN_SET);
}
```

**C. Loop di lettura (while 1)**
Visualizza l'orario corrente.
*Nota: Bisogna sempre leggere PRIMA l'orario e POI la data per sbloccare i registri shadow.*

```c
RTC_TimeTypeDef gTime;
RTC_DateTypeDef gDate;

HAL_RTC_GetTime(&hrtc, &gTime, RTC_FORMAT_BIN);
HAL_RTC_GetDate(&hrtc, &gDate, RTC_FORMAT_BIN);

char msg[30];
sprintf(msg, "Time: %02d:%02d:%02d\r", gTime.Hours, gTime.Minutes, gTime.Seconds);
HAL_UART_Transmit(&huart2, (uint8_t*)msg, strlen(msg), 100);

HAL_Delay(1000);
```

### 8. External Blink (GPIO Output)
* **Obiettivo:** Pilotare un carico esterno (LED) su breadboard.
* **Hardware:** LED, Resistenza 220Ω, Breadboard.
* **Pin:** `PA0` (Configurato come GPIO_Output).
* **Teoria:** La resistenza è necessaria per limitare la corrente e proteggere il pin (Legge di Ohm: $I = V/R$).

**Collegamento:**
* **Anodo LED (+):** Collegato al Pin `PA0`.
* **Catodo LED (-):** Collegato alla Resistenza -> GND.

```c
/* Nel while(1) */
HAL_GPIO_TogglePin(GPIOA, GPIO_PIN_0); // Toggle Pin PA0
HAL_Delay(500);
```

### 9. Il Potenziometro (ADC Input)
* **Obiettivo:** Leggere una tensione analogica variabile (da 0V a 3.3V) tramite un trimmer/potenziometro e visualizzare il voltaggio sul PC.
* **Hardware:** Trimmer 10kΩ, Breadboard.
* **Pin:** `PA0` (Configurato come **ADC1_IN0**).
* **Teoria:** Il potenziometro agisce come un partitore di tensione variabile. L'ADC a 12-bit converte la tensione in un numero intero da 0 a 4095.

**Collegamento:**
* **Pin 1 Trimmer:** 3.3V (Linea Rossa)
* **Pin 2 Trimmer (Centrale/Wiper):** Pin `PA0` (Segnale)
* **Pin 3 Trimmer:** GND (Linea Blu)

```c
/* Nel while(1) */
HAL_ADC_Start(&hadc1);
if (HAL_ADC_PollForConversion(&hadc1, 100) == HAL_OK)
{
    // 1. Lettura valore grezzo (0 - 4095)
    uint32_t rawValue = HAL_ADC_GetValue(&hadc1);

    // 2. Conversione in Volt (Vref = 3.3V)
    float voltage = (float)rawValue / 4095.0f * 3.3f;

    // 3. Stampa (Trucco per %f con nano-lib)
    int v_int = (int)voltage;
    int v_dec = (int)((voltage - v_int) * 100);

    char msg[50];
    // \r finale permette di sovrascrivere la riga nel terminale
    sprintf(msg, "Raw: %4lu | Volt: %d.%02d V\r", rawValue, v_int, v_dec);
    HAL_UART_Transmit(&huart2, (uint8_t*)msg, strlen(msg), 100);
}
HAL_ADC_Stop(&hadc1);
HAL_Delay(100);
```

### 10. Pulsante Esterno & Debouncing Hardware
* **Obiettivo:** Leggere un input digitale da un pulsante esterno eliminando i falsi contatti (rimbalzi meccanici) tramite un filtro hardware.
* **Hardware:** Pulsante, Resistenza 10kΩ (Pull-Down), Condensatore 100nF.
* **Pin:** `PA10` (Label: `MY_BTN`, Header Arduino: **D2**).
* **Teoria:**
    * **Pull-Down:** La resistenza tiene il pin a 0V (GND) quando il tasto non è premuto.
    * **Filtro RC:** Il condensatore in parallelo alla resistenza assorbe le vibrazioni rapide delle lamelle del pulsante ("Bouncing"), fornendo al microcontrollore un segnale pulito (0 o 1 netto).

**Collegamento (Logica Active High):**
* **Lato 1 Pulsante:** 3.3V.
* **Lato 2 Pulsante (Segnale):** Pin `PA10`.
* **Nodo Segnale:** Collegare qui anche la Resistenza verso GND e il Condensatore verso GND.

```c
/* Nel while(1) */
// Con Pull-Down esterno:
// GPIO_PIN_SET (1)   = Tasto Premuto (3.3V)
// GPIO_PIN_RESET (0) = Tasto Rilasciato (0V via Resistenza)

if (HAL_GPIO_ReadPin(MY_BTN_GPIO_Port, MY_BTN_Pin) == GPIO_PIN_SET)
{
    // Accendi LED
    HAL_GPIO_WritePin(LD2_GPIO_Port, LD2_Pin, GPIO_PIN_SET);
}
else
{
    // Spegni LED
    HAL_GPIO_WritePin(LD2_GPIO_Port, LD2_Pin, GPIO_PIN_RESET);
}
HAL_Delay(10);
```

#### 🔬 Deep Dive: Visualizzare il Rimbalzo (Bouncing)
Per dimostrare la necessità del condensatore, si può utilizzare questo codice di test "instabile".
Rimuovendo `HAL_Delay` e contando ogni fronte di salita (0 -> 1), questo programma mostra come una singola pressione meccanica generi decine di segnali spuri se il filtro hardware (condensatore) viene rimosso.

**Istruzioni per il test:**
1. Rimuovere il condensatore dal circuito.
2. Caricare questo codice.
3. Premere il tasto una volta.
4. Osservare sul Serial Monitor come il contatore aumenti di 10-20 unità per un singolo click.
5. Reinserire il condensatore: il contatore aumenterà di 1 unità esatta.

```c
/* USER CODE BEGIN PV */
uint8_t stato_precedente = 0;
uint32_t counter = 0;
/* USER CODE END PV */

/* USER CODE BEGIN WHILE */
while (1)
{
    // 1. Lettura diretta (senza delay per massima sensibilità)
    uint8_t stato_attuale = HAL_GPIO_ReadPin(MY_BTN_GPIO_Port, MY_BTN_Pin);

    // 2. Rilevamento Fronte di Salita (Rising Edge: 0 -> 1)
    if (stato_attuale == GPIO_PIN_SET && stato_precedente == GPIO_PIN_RESET)
    {
        counter++; // Conta il segnale (anche se è un rimbalzo)

        // Stampa il conteggio
        char msg[50];
        sprintf(msg, "Segnali rilevati: %lu\r\n", counter);
        HAL_UART_Transmit(&huart2, (uint8_t*)msg, strlen(msg), 10);
        
        // Toggle LED per feedback visivo
        HAL_GPIO_TogglePin(LD2_GPIO_Port, LD2_Pin);
    }

    // Aggiorna lo stato per il prossimo ciclo
    stato_precedente = stato_attuale;
    
    // NESSUN DELAY: Il loop gira alla massima frequenza della CPU
}
```

### 11. Transistor come Interruttore (BJT NPN)
* **Obiettivo:** Pilotare un carico ad alta corrente (3 LED in parallelo alimentati a 5V) usando un pin dell'STM32 (3.3V) tramite un transistor.
* **Hardware:** Transistor NPN (S8050), 3x LED, Resistenze (220Ω per LED, 1kΩ per Base).
* **Pin:** `PA0` (Segnale di controllo).
* **Teoria:** Il transistor agisce come un interruttore controllato in corrente (Low-Side Switch). Una piccola corrente alla Base (dal microcontrollore) permette il passaggio di una grande corrente dal Collettore all'Emettitore.

**Collegamento (S8050 E-B-C):**
* **Emettitore (Sinistra):** Diretto a GND.
* **Base (Centro):** Resistenza 1kΩ -> Pin `PA0`.
* **Collettore (Destra):** Collegato ai Catodi (-) dei LED. Gli Anodi (+) dei LED vanno a 5V.

```c
/* Nel while(1) - Codice Blink standard */
HAL_GPIO_TogglePin(GPIOA, GPIO_PIN_0);
HAL_Delay(500);
```

### 12. Controllo Relè & Diodo di Ricircolo
* **Obiettivo:** Pilotare un carico induttivo meccanico (Relè) isolando il microcontrollore dal circuito di potenza.
* **Hardware:** Relè 5V (Omron G5V-1), Transistor NPN (S8050), Diodo (1N4007), Resistenza 1kΩ.
* **Pin:** `PA0` (Segnale).
* **Teoria:**
    * **Inductive Kickback:** Quando la bobina del relè viene spenta, genera un picco di tensione inversa (centinaia di Volt) che distruggerebbe il transistor.
    * **Flyback Diode:** Il diodo 1N4007, messo in antiparallelo alla bobina, "mangia" questa scarica proteggendo il circuito.

**Collegamento (Circuito di Pilotaggio):**
* **Transistor Emettitore:** Diretto a GND.
* **Transistor Base:** Resistenza 1kΩ -> Pin `PA0`.
* **Transistor Collettore:** Collegato al **Pin 1** della Bobina del Relè + **Anodo** (lato nero) del Diodo.

**Collegamento (Circuito di Potenza/Bobina):**
* **Pin 6 Bobina Relè:** Collegato ai **5V**.
* **Catodo Diodo (Striscia Grigia):** Collegato anch'esso ai **5V** (In parallelo alla bobina).

⚠️ **ATTENZIONE:** Il diodo va montato con la striscia verso il positivo (5V). Se montato al contrario, crea un cortocircuito.

```c
/* Nel while(1) - Si sente il CLICK-CLACK */
HAL_GPIO_TogglePin(GPIOA, GPIO_PIN_0);
HAL_Delay(1000);
```

#### Relè con Toggle e Feedback Esterno
* **Obiettivo:** Sistema di automazione completo su breadboard. Un pulsante attiva/disattiva un Relè e un LED di segnalazione esterno.
* **Hardware:**
    * **Output Potenza:** Relè 5V pilotato da Transistor S8050 su Pin `PA0`.
    * **Output Segnalazione:** LED Esterno su Pin `PA1`.
    * **Input:** Pulsante (con filtro RC) su Pin `PA10`.
* **Configurazione:**
    * `PA0` -> GPIO_Output (Relè).
    * `PA1` -> GPIO_Output (LED Esterno).
    * `PA10` -> GPIO_Input (Pulsante).

**Logica di Controllo:**
Pressione Tasto -> Inversione Stato -> Attivazione simultanea di Relè (Click) e LED (Luce).

```c
/* Logica nel while(1) */
if (stato_pulsante_att == GPIO_PIN_SET && stato_pulsante_prec == GPIO_PIN_RESET)
{
    stato_sistema = !stato_sistema; // Toggle
    
    if(stato_sistema) {
        HAL_GPIO_WritePin(GPIOA, GPIO_PIN_0, GPIO_PIN_SET); // Relè ON
        HAL_GPIO_WritePin(GPIOA, GPIO_PIN_1, GPIO_PIN_SET); // Ext LED ON
    } else {
        HAL_GPIO_WritePin(GPIOA, GPIO_PIN_0, GPIO_PIN_RESET); // Relè OFF
        HAL_GPIO_WritePin(GPIOA, GPIO_PIN_1, GPIO_PIN_RESET); // Ext LED OFF
    }
    HAL_Delay(50);
}
```

### 13. Shift Register (74HC595) - Effetto Supercar
* **Obiettivo:** Espandere le uscite digitali (da 3 a 8) per creare effetti luminosi complessi utilizzando un registro a scorrimento (Shift Register).
* **Hardware:** Chip 74HC595, 8x LED, 8x Resistenze 220Ω.
* **Pin Nucleo:**
    * `PA9` (D8) -> **DS** (Data - Pin 14 Chip)
    * `PC7` (D9) -> **ST_CP** (Latch - Pin 12 Chip)
    * `PB6` (D10) -> **SH_CP** (Clock - Pin 11 Chip)
* **Pin Chip Critici:**
    * **VCC (16) & MR (10):** Collegati a 3.3V.
    * **GND (8) & OE (13):** Collegati a GND.
    * **Q0 (Pin 15):** Prima uscita LED (Attenzione: è separata dalle altre Q1-Q7).

**Teoria:** Il chip converte una comunicazione seriale (bit inviati uno alla volta) in un'uscita parallela. Carichiamo un byte intero (8 bit) e poi diamo il comando "Latch" per accendere tutti i LED insieme.

```c
/* Funzione ShiftOut (Bit-Banging manuale) */
void ShiftOut(uint8_t data) {
    HAL_GPIO_WritePin(GPIOC, GPIO_PIN_7, GPIO_PIN_RESET); // Latch GIÙ (Inizio trasmissione)
    
    for (int i = 7; i >= 0; i--) {
        HAL_GPIO_WritePin(GPIOB, GPIO_PIN_6, GPIO_PIN_RESET); // Clock GIÙ
        
        // Scriviamo il bit (MSB first)
        if ((data >> i) & 1) 
            HAL_GPIO_WritePin(GPIOA, GPIO_PIN_9, GPIO_PIN_SET);
        else                 
            HAL_GPIO_WritePin(GPIOA, GPIO_PIN_9, GPIO_PIN_RESET);
            
        HAL_GPIO_WritePin(GPIOB, GPIO_PIN_6, GPIO_PIN_SET);   // Clock SU (Salva il bit)
    }
    HAL_GPIO_WritePin(GPIOC, GPIO_PIN_7, GPIO_PIN_SET);   // Latch SU (Mostra output)
}

/* Esempio utilizzo: Effetto Supercar */
for (int i = 0; i < 8; i++) {
    ShiftOut(1 << i); // Sposta il bit acceso
    HAL_Delay(100);
}
```

### 14. Orecchio Elettronico (Microfono & Op-Amp LM358)
* **Obiettivo:** Amplificare il segnale analogico di un microfono (pochi mV) per visualizzare la forma d'onda della voce sul PC tramite Serial Plotter.
* **Hardware:**
    * Microfono a Elettrete.
    * Op-Amp **LM358** (Configurazione Non-Invertente).
    * Resistenza Feedback: **1 MΩ** (Modificata per guadagno elevato).
    * Resistenza Input (GND): **1 kΩ**.
    * Partitore Bias: 2x **10 kΩ**.
    * Condensatori: 2x **100 nF** (Accoppiamento AC).
* **Pin Nucleo:** `PA0` configurato come **ADC1_IN0** (Continuous Mode).

**Teoria & Calcoli:**
* **Bias DC (Centraggio):** Il segnale audio oscilla tra positivo e negativo. Poiché l'STM32 legge solo 0-3.3V, usiamo un partitore di tensione per centrare il silenzio a **1.65V** (metà scala, circa valore ADC 2048).
* **Guadagno (Gain):** Abbiamo configurato l'amplificatore per moltiplicare il segnale x1000.
  $$Gain = 1 + \frac{R_{feedback}}{R_{input}} = 1 + \frac{1.000.000}{1.000} \approx 1001 \text{ volte}$$
* **Filtro Passa-Alto:** Il condensatore da 100nF in ingresso forma un filtro con la resistenza da 1kΩ ($f_c \approx 16Hz$) per rimuovere la componente DC ma lasciar passare tutta la voce.

**Risultati Sperimentali:**
* **Silenzio:** Valore ADC stabile attorno a **2000-2050** (1.65V).
* **Voce/Fischio:** Oscillazione ampia tra **1200 e 3000** (grazie al Gain 1000x).

```c
/* CONFIGURAZIONE: ADC1 Continuous Mode, USART2 115200 baud */

/* Nel main.c - USER CODE BEGIN 2 */
HAL_ADC_Start(&hadc1); // Avvia l'ADC

/* Nel main.c - USER CODE BEGIN WHILE */
while (1)
{
    // Attendi la conversione
    if (HAL_ADC_PollForConversion(&hadc1, 10) == HAL_OK)
    {
        // 1. Leggi il valore (0-4095)
        uint32_t val = HAL_ADC_GetValue(&hadc1);
        
        // 2. Formatta per Serial Plotter (Numero + a capo)
        char msg[20];
        sprintf(msg, "%lu\r\n", val); 
        HAL_UART_Transmit(&huart2, (uint8_t*)msg, strlen(msg), 10);
    }
    
    // Piccolo ritardo per rendere il grafico leggibile
    HAL_Delay(1); 
}
```

### 15. Costante di Tempo RC (Capacimetro Scientifico)
* **Obiettivo:** Visualizzare la curva di carica di un condensatore e misurarne la capacità sfruttando la legge fisica della costante di tempo ($\tau = R \times C$).
* **Hardware:**
    * Condensatore Elettrolitico (es. 100µF o 47µF). *Attenzione alla polarità!*
    * Resistenza Nota (es. 10kΩ).
* **Pin Nucleo:**
    * `PA0` (ADC1_IN0) -> **Sonda** (Collegato al nodo positivo del condensatore).
    * `PA1` (GPIO_Output) -> **Alimentazione** (Collegato alla resistenza).
* **Teoria:** Un condensatore si carica attraverso una resistenza seguendo una curva esponenziale. Il tempo necessario per raggiungere il **63.2%** della carica massima è pari a $\tau = R \cdot C$.
    * *Esempio:* Con $R=10k\Omega$ e $C=100\mu F$, $\tau = 1.0$ secondi.

**Collegamento:**
* **GND** -> Condensatore (-)
* **PA1** -> Resistenza -> Condensatore (+) -> **PA0**

```c
/* Nel main.c - Ciclo di Carica e Scarica per Plotter */
while (1)
{
    // --- FASE 1: SCARICA (Discharge) ---
    HAL_GPIO_WritePin(GPIOA, GPIO_PIN_1, GPIO_PIN_RESET); // Metti a terra (0V)

    // Aspettiamo che si scarichi (visualizzando lo zero)
    for(int i=0; i<200; i++) {
        uint32_t val = HAL_ADC_GetValue(&hadc1);
        char msg[50];
        // Inviamo anche 0 e 4095 per bloccare la scala del Plotter
        sprintf(msg, "0 4095 %lu\r\n", val);
        HAL_UART_Transmit(&huart2, (uint8_t*)msg, strlen(msg), 10);
        HAL_Delay(50); // Campionamento lento per vedere tutto il processo
    }

    // --- FASE 2: CARICA (Charge) ---
    HAL_GPIO_WritePin(GPIOA, GPIO_PIN_1, GPIO_PIN_SET); // Dai corrente (3.3V)

    // Visualizziamo la curva di salita ("Pinna di Squalo")
    for(int i=0; i<200; i++) {
        if (HAL_ADC_PollForConversion(&hadc1, 10) == HAL_OK) {
             uint32_t val = HAL_ADC_GetValue(&hadc1);
             char msg[50];
             sprintf(msg, "0 4095 %lu\r\n", val);
             HAL_UART_Transmit(&huart2, (uint8_t*)msg, strlen(msg), 10);
        }
        HAL_Delay(50);
    }
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

### 6. RTC (Real Time Clock) & Allarme
* **Obiettivo:** Gestione del tempo reale e Low Power.
* **Attività:** Configurare l'RTC interno usando l'oscillatore a bassa frequenza (LSI o LSE). Impostare l'orario corrente e configurare un **Allarme** che scatta dopo 10 secondi per attivare un evento (es. accendere il LED), simulando una sveglia.

---

## 🚀 Prossimi Passaggi (Roadmap)

### ⚪️ Fase 0: On-Board
- [x] **0. Setup Ambiente e Firmware Update**
- [x] **01. Hello World (Blink LED)**
- [x] **02. Clock & Frequenze**
- [x] **03. PWM Breathing LED**
- [x] **04. Interrupts (EXTI)**
- [x] **05. UART Communication**
- [x] **06. Internal Temp Sensor (ADC)**
- [x] **07. RTC & Alarm**

### 🟢 Fase 1: Breadboard Fundamentals
Questi esperimenti servono a prendere confidenza con i collegamenti fisici, la breadboard e l'uso del Multimetro.
- [x] **08. External Blink** (GPIO Output & Legge di Ohm)
- [x] **09. Il Potenziometro** (ADC Input & Partitore di Tensione)
- [x] **10. Pulsante Esterno** (Input & Hardware Debounce con filtro RC)

### 🟡 Fase 2: Potenza & Switching
Gestione di carichi che richiedono più corrente di quella che il microcontrollore può erogare.
- [x] **11. Transistor come Interruttore** (BJT NPN per pilotare carichi)
- [x] **12. Relè & Diodo di Ricircolo** (Isolamento galvanico e protezione da sovratensioni)

### 🟠 Fase 3: Logica Digitale
- [x] **13. Shift Register** (74HC595 - Espansione uscite / Effetto Supercar)

### 🔴 Fase 4: Analogica Avanzata
- [x] **14. Microfono & Op-Amp** (Amplificazione operazionale di segnali deboli)
- [x] **15. Costante di Tempo RC** (Misura della capacità di un condensatore usando la fisica)
