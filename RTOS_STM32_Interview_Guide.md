# Embedded Systems & RTOS Interview Guide (STM32 + FreeRTOS)

---

## Table of Contents
1. [What is an Embedded System?](#1-what-is-an-embedded-system)
2. [What is a Microcontroller (STM32)?](#2-what-is-a-microcontroller-stm32)
3. [Bare-Metal (Super Loop) Programming](#3-bare-metal-super-loop-programming)
4. [What is an RTOS?](#4-what-is-an-rtos)
5. [Why Do We Need an RTOS?](#5-why-do-we-need-an-rtos)
6. [Setting up FreeRTOS on STM32 (CubeMX)](#6-setting-up-freertos-on-stm32-cubemx)
7. [Tasks](#7-tasks)
8. [Task States](#8-task-states)
9. [Scheduler](#9-scheduler)
10. [Context Switching](#10-context-switching)
11. [Queue](#11-queue)
12. [Semaphore](#12-semaphore)
13. [Mutex](#13-mutex)
14. [Event Flags (Event Groups)](#14-event-flags-event-groups)
15. [Task Notifications](#15-task-notifications)
16. [Software Timers](#16-software-timers)
17. [Memory Management](#17-memory-management)
18. [Interrupts with RTOS on STM32](#18-interrupts-with-rtos-on-stm32)
19. [Complete STM32 Project Example](#19-complete-stm32-project-example)
20. [Common Bugs & Debugging](#20-common-bugs--debugging)
21. [Rapid-Fire Interview Questions](#21-rapid-fire-interview-questions)
22. [AUTOSAR OS vs. RTOS (Basic & Extended Tasks)](#22-autosar-os-vs-rtos-basic--extended-tasks)

---

## 1. What is an Embedded System?

### Q: What is an embedded system?
An **embedded system** is a computer system built to do **one specific job** (or a small set of jobs) inside a larger device. It combines **hardware** (microcontroller, sensors, actuators) and **software** (firmware). Usually it has limited resources (CPU, RAM, power), and often it has real-time requirements.

**Examples:**
| Domain | Example |
|---|---|
| Consumer | Washing machine, microwave, smartwatch, TV remote |
| Automotive | ABS, airbag controller, engine ECU, infotainment |
| Medical | Pacemaker, insulin pump, ECG monitor |
| Industrial | PLC, motor drives, robotic arms |
| IoT | Smart thermostat, smart meter, wearable fitness band |

### Q: What are the characteristics of an embedded system?
- **Dedicated function:** it does one job, not general-purpose computing.
- **Resource constrained:** small RAM/Flash (KB to MB), low-power CPU.
- **Real-time:** it often has to respond within a deadline.
- **Reliable:** it may run for years without a reboot.
- **Low power:** it is often battery operated.
- **Interacts with hardware:** sensors, motors, communication buses.

### Q: What are the components of an embedded system?
```
+-----------------------------------------------------+
|                 Embedded System                      |
|                                                       |
|  Sensors --> [ Microcontroller ] --> Actuators        |
|  (input)     |  CPU            |     (output)         |
|              |  Flash (code)   |                      |
|              |  RAM (data)     |                      |
|              |  Peripherals    |                      |
|              |  (GPIO, UART,   |                      |
|              |   SPI, I2C, ADC,|                      |
|              |   Timers, DMA)  |                      |
|              +-----------------+                      |
|  Power supply, Clock (crystal), Communication         |
+-----------------------------------------------------+
```

### Q: What is the difference between a microprocessor and a microcontroller?
| Microprocessor (MPU) | Microcontroller (MCU) |
|---|---|
| CPU only; RAM/ROM/peripherals are external | CPU + RAM + Flash + peripherals on one chip |
| High performance, high power | Lower power, low cost |
| Runs a full OS (Linux) | Runs bare-metal or an RTOS |
| e.g., Intel Core, ARM Cortex-A | e.g., STM32 (Cortex-M), AVR, PIC |

### Q: What is firmware?
Software that is permanently stored in the device's non-volatile memory (Flash) and controls the hardware directly.

### Q: What does the `volatile` keyword do, and why does it matter in embedded systems?
`volatile` tells the compiler that a variable can change **outside the normal program flow** (from hardware registers, ISRs, or other tasks), so the compiler must not optimize away reads or writes to it.
```c
volatile uint8_t flag = 0;      // modified in an ISR
void EXTI0_IRQHandler(void) { flag = 1; }
while (!flag);                   // without volatile, this could become an infinite loop
```

---

## 2. What is a Microcontroller (STM32)?

### Q: What is STM32?
**STM32** is a family of 32-bit microcontrollers from **STMicroelectronics**, built on **ARM Cortex-M** cores.

| Series | Core | Use case |
|---|---|---|
| STM32F0 / G0 | Cortex-M0/M0+ | Low cost |
| STM32F1 | Cortex-M3 | General purpose ("Blue Pill") |
| STM32F4 | Cortex-M4F (FPU) | DSP, high performance |
| STM32F7 / H7 | Cortex-M7 | Very high performance |
| STM32L series | Cortex-M0+/M4 | Ultra low power |
| STM32WB / WL | Cortex-M4 + radio | BLE / LoRa wireless |

**Common board:** STM32 **Nucleo-F446RE** (onboard LED = PA5, user button = PC13, UART2 over ST-Link USB).

### Q: What tools are used for STM32 development?
- **STM32CubeIDE:** IDE (Eclipse + GCC + debugger).
- **STM32CubeMX:** graphical tool for pins, clocks and middleware (FreeRTOS); it generates code.
- **HAL / LL libraries:** hardware abstraction drivers.
- **ST-Link:** debugger and programmer.
- **CMSIS-RTOS v2:** a standard RTOS API layer that wraps FreeRTOS.

### Q: What are the key Cortex-M features that matter for an RTOS?
- **SysTick timer:** generates the RTOS tick.
- **PendSV exception:** used for context switching.
- **SVC instruction:** starts the first task.
- **Two stack pointers:** MSP (main, used by ISRs/kernel) and PSP (process, used by tasks).
- **NVIC:** nested vectored interrupt controller with priorities.

---

## 3. Bare-Metal (Super Loop) Programming

### Q: What is a super loop?
The simplest firmware architecture: one infinite loop in `main()` that calls every function in turn.

```c
int main(void)
{
    HAL_Init();
    SystemClock_Config();
    MX_GPIO_Init();

    while (1)
    {
        ReadSensors();       // 5 ms
        UpdateDisplay();     // 50 ms  <-- blocks everything else
        CheckButtons();      // must react within 10 ms! (missed)
        SendDataUART();      // 20 ms
        BlinkLED();
    }
}
```

### Q: What are the problems with a super loop?
1. **Poor responsiveness:** a slow function delays all the others.
2. **`HAL_Delay()` blocks everything:** the CPU wastes time busy-waiting.
3. **No priorities:** urgent jobs wait for unimportant ones.
4. **Hard to scale:** adding a feature changes the timing of all the others.
5. **Complex state machines:** to avoid blocking, you end up writing complicated non-blocking code.
6. **Tight coupling:** modules are hard to develop and test separately.

---

## 4. What is an RTOS?

### Q: What is an RTOS?
An **RTOS (Real-Time Operating System)** is a small operating system kernel for embedded systems. It manages multiple **tasks** (threads) and guarantees **deterministic** (predictable) timing, so the most important work always finishes **within its deadline**.

> **"Real-time" does NOT mean "fast". It means "predictable and on time".**

```
            +-----------+   +-----------+   +-----------+
            |  Task 1   |   |  Task 2   |   |  Task 3   |
            +-----+-----+   +-----+-----+   +-----+-----+
                  \               |               /
             +----------------------------------------+
             |                 RTOS KERNEL             |
             | Scheduler | Context Switch | Queue      |
             | Semaphore | Mutex | Event Flags | Timers|
             +--------------------+-------------------+
                                  |
                       +----------v----------+
                       |   MCU (STM32)       |
                       +---------------------+
```

### Q: What are the main components of an RTOS?
| Component | Purpose |
|---|---|
| **Task** | Independent unit of work with its own stack and priority |
| **Scheduler** | Decides which task runs next |
| **Context switching** | Saves and restores CPU registers when switching tasks |
| **Queue** | Passes data between tasks or from ISR to task |
| **Semaphore** | Signals and synchronizes |
| **Mutex** | Gives exclusive access to a shared resource |
| **Event flags** | Waits on multiple conditions |
| **Software timers** | Calls a function after a delay or periodically |
| **Memory management** | Allocates task stacks and kernel objects |

### Q: What are hard, firm and soft real-time systems?
| Type | If a deadline is missed | Example |
|---|---|---|
| **Hard** | System failure, possibly dangerous | Airbag, pacemaker, ABS |
| **Firm** | The result is useless but the system survives | Video frame decoding |
| **Soft** | Quality drops | Audio streaming, UI |

### Q: What is the difference between an RTOS and a GPOS?
| Feature | RTOS | GPOS (Linux/Windows) |
|---|---|---|
| Goal | Determinism | Throughput / fairness |
| Latency | Microseconds, bounded | Milliseconds, unbounded |
| Size | 5–20 KB | MBs–GBs |
| Scheduling | Priority preemptive | Time sharing / fair scheduling |
| MMU required | No | Yes |
| Boot time | Milliseconds | Seconds |

### Q: What are some popular RTOSes?
FreeRTOS (most common; Amazon), Zephyr (Linux Foundation), Azure RTOS ThreadX (now Eclipse ThreadX), Keil RTX, Micrium uC/OS, VxWorks, QNX, embOS.

### Q: What is determinism?
The same input always produces the same output **within the same bounded time**. Worst-case execution time (WCET) is known.

### Q: What is latency? What is jitter?
- **Interrupt latency:** time from the interrupt signal to the first ISR instruction.
- **Scheduling latency:** time from an event to the start of the task that handles it.
- **Jitter:** variation in timing, e.g., a 10 ms periodic task actually runs at 9.8–10.3 ms.

---

## 5. Why Do We Need an RTOS?

### Q: Why use an RTOS instead of a super loop?
1. **Multitasking:** several jobs appear to run in parallel.
2. **Priority-based response:** urgent work (motor control) preempts less important work (logging).
3. **Deterministic timing:** deadlines are guaranteed.
4. **Efficient CPU use:** blocked tasks use 0% CPU, unlike busy-waiting with `HAL_Delay`.
5. **Modularity:** each feature is its own task, which is easier to develop, test and maintain.
6. **Team development:** different engineers own different tasks.
7. **Built-in IPC:** queues, semaphores and mutexes are tested and thread-safe.
8. **Low power:** the idle task can put the MCU to sleep (tickless idle).
9. **Scalability:** you can add features without rewriting the timing.
10. **Ecosystem:** TCP/IP, USB, file systems and BLE stacks often require an RTOS.

### Q: When should you NOT use an RTOS?
- Very simple applications (e.g., blinking an LED or reading one sensor).
- Extremely small MCUs (< 4 KB RAM).
- When the extra code size and complexity are not justified.
- When interrupts plus a simple loop already meet every deadline.

### Q: Show the same job written as a super loop and as an RTOS.

**Super loop (blocking):**
```c
while (1)
{
    HAL_GPIO_TogglePin(GPIOA, GPIO_PIN_5);
    HAL_Delay(500);                 // CPU is stuck here
    if (HAL_GPIO_ReadPin(GPIOC, GPIO_PIN_13) == GPIO_PIN_RESET)
        printf("Button\r\n");       // response up to 500 ms late!
}
```

**RTOS (each task independent):**
```c
void LedTask(void *arg)
{
    for (;;) { HAL_GPIO_TogglePin(GPIOA, GPIO_PIN_5); vTaskDelay(pdMS_TO_TICKS(500)); }
}

void ButtonTask(void *arg)
{
    for (;;)
    {
        if (HAL_GPIO_ReadPin(GPIOC, GPIO_PIN_13) == GPIO_PIN_RESET)
            printf("Button\r\n");    // responds within 10 ms
        vTaskDelay(pdMS_TO_TICKS(10));
    }
}
```

---

## 6. Setting up FreeRTOS on STM32 (CubeMX)

### Steps in STM32CubeIDE
1. **File > New > STM32 Project**, then select the board (e.g., NUCLEO-F446RE).
2. **System Core > SYS > Timebase Source: TIM6** (not SysTick).
   - *Why?* FreeRTOS uses **SysTick** for its tick, so HAL needs a different timer for `HAL_GetTick()`.
3. **Middleware > FREERTOS > Interface: CMSIS_V2**.
4. In the **Tasks and Queues** tab, add tasks, queues, semaphores and mutexes.
5. In **Config parameters**, set `TOTAL_HEAP_SIZE`, `USE_PREEMPTION`, `TICK_RATE_HZ`, and so on.
6. Generate the code. Write your logic inside the generated task functions in `main.c` or `freertos.c`.

### Important `FreeRTOSConfig.h` settings
```c
#define configUSE_PREEMPTION                    1
#define configCPU_CLOCK_HZ                      (SystemCoreClock)
#define configTICK_RATE_HZ                      ((TickType_t)1000)   // 1 ms tick
#define configMAX_PRIORITIES                    (56)
#define configMINIMAL_STACK_SIZE                ((uint16_t)128)      // words, not bytes!
#define configTOTAL_HEAP_SIZE                   ((size_t)15360)
#define configUSE_MUTEXES                       1
#define configUSE_RECURSIVE_MUTEXES             1
#define configUSE_COUNTING_SEMAPHORES           1
#define configUSE_TIMERS                        1
#define configCHECK_FOR_STACK_OVERFLOW          2
#define configUSE_MALLOC_FAILED_HOOK            1
#define configLIBRARY_MAX_SYSCALL_INTERRUPT_PRIORITY 5
```

### Q: Native FreeRTOS API vs. CMSIS-RTOS v2 API?
| Native FreeRTOS | CMSIS-RTOS v2 |
|---|---|
| `xTaskCreate()` | `osThreadNew()` |
| `vTaskDelay()` | `osDelay()` |
| `xQueueSend()` | `osMessageQueuePut()` |
| `xSemaphoreTake()` | `osSemaphoreAcquire()` / `osMutexAcquire()` |
| `xEventGroupSetBits()` | `osEventFlagsSet()` |

CMSIS is portable across RTOSes. Native FreeRTOS is more widely documented and is what interviewers usually ask about. **This guide uses the native API.**

### Redirect `printf` to UART2 (used in the examples)
```c
#include <stdio.h>
extern UART_HandleTypeDef huart2;

int __io_putchar(int ch)
{
    HAL_UART_Transmit(&huart2, (uint8_t *)&ch, 1, HAL_MAX_DELAY);
    return ch;
}
```

---

## 7. Tasks

### Q: What is a task?
An independent function that runs forever and has:
- its **own stack** (local variables, saved context),
- a **priority**,
- a **TCB (Task Control Block)**: a kernel structure that stores the stack pointer, priority, state, name and other details.

### Q: Show how to create tasks on STM32.
```c
#include "FreeRTOS.h"
#include "task.h"

TaskHandle_t hLedTask, hUartTask;

void LedTask(void *argument)
{
    for (;;)
    {
        HAL_GPIO_TogglePin(GPIOA, GPIO_PIN_5);      // Nucleo LD2
        vTaskDelay(pdMS_TO_TICKS(500));
    }
}

void UartTask(void *argument)
{
    uint32_t count = 0;
    for (;;)
    {
        printf("Hello from task, count=%lu\r\n", count++);
        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}

int main(void)
{
    HAL_Init();
    SystemClock_Config();
    MX_GPIO_Init();
    MX_USART2_UART_Init();

    xTaskCreate(LedTask,  "LED",  128, NULL, 1, &hLedTask);
    xTaskCreate(UartTask, "UART", 256, NULL, 2, &hUartTask);

    vTaskStartScheduler();      // never returns if successful
    while (1);                  // reached only if there is not enough heap
}
```

### Q: What are the parameters of `xTaskCreate`?
```c
BaseType_t xTaskCreate(
    TaskFunction_t pvTaskCode,     // task function
    const char *pcName,            // name (for debugging)
    uint16_t usStackDepth,         // stack size in WORDS (128 words = 512 bytes on a 32-bit MCU)
    void *pvParameters,            // argument passed to the task
    UBaseType_t uxPriority,        // priority (0 = lowest = idle)
    TaskHandle_t *pxCreatedTask);  // handle (optional)
```

### Q: How do you pass parameters to a task (one function, several tasks)?
```c
typedef struct { GPIO_TypeDef *port; uint16_t pin; uint32_t period; } LedCfg_t;

static LedCfg_t led1 = { GPIOA, GPIO_PIN_5, 200 };
static LedCfg_t led2 = { GPIOB, GPIO_PIN_0, 700 };   // must be static/global, not on main's stack!

void BlinkTask(void *arg)
{
    LedCfg_t *cfg = (LedCfg_t *)arg;
    for (;;)
    {
        HAL_GPIO_TogglePin(cfg->port, cfg->pin);
        vTaskDelay(pdMS_TO_TICKS(cfg->period));
    }
}

xTaskCreate(BlinkTask, "Blink1", 128, &led1, 1, NULL);
xTaskCreate(BlinkTask, "Blink2", 128, &led2, 1, NULL);
```

### Q: Why should a task never return?
In FreeRTOS, returning from a task function is **undefined behaviour** (on Cortex-M it calls `prvTaskExitError`, which traps the system). A task must loop forever or call `vTaskDelete(NULL)`.

### Q: How do you choose the stack size? How do you detect a stack overflow?
- Start with an estimate (local variables + function call depth + printf, which is heavy).
- Measure: `uxTaskGetStackHighWaterMark(NULL)` returns the minimum free stack seen so far, in words.
- Enable `configCHECK_FOR_STACK_OVERFLOW 2` and implement the hook:
```c
void vApplicationStackOverflowHook(TaskHandle_t xTask, char *pcTaskName)
{
    printf("STACK OVERFLOW in %s\r\n", pcTaskName);
    __disable_irq();
    while (1);
}
```

### Q: Static vs. dynamic task creation?
- `xTaskCreate()` allocates the TCB and stack from the FreeRTOS heap (dynamic).
- `xTaskCreateStatic()` uses buffers you provide (static). No heap is needed, which is preferred in safety-critical systems (MISRA, ISO 26262).

---

## 8. Task States

```
                     xTaskCreate()
                          |
                          v
         +---------->  READY  <-------------------+
         |               |                         |
         | preempted /   | scheduler picks         | event occurs /
         | taskYIELD()   | highest priority        | timeout expires
         |               v                         |
         +---------- RUNNING --------------------> BLOCKED
                         |   vTaskDelay, xQueueReceive,
                         |   xSemaphoreTake (waiting)
                         |
          vTaskSuspend() |            vTaskResume()
                         v          (from any state)
                     SUSPENDED  --------> READY
```

### Q: Explain each task state.
| State | Meaning |
|---|---|
| **Running** | Currently executing on the CPU (only one on a single core) |
| **Ready** | Able to run; waiting for a higher or equal priority task to give up the CPU |
| **Blocked** | Waiting for an event (queue, semaphore, delay) **with an optional timeout**; uses no CPU |
| **Suspended** | Stopped explicitly; no timeout; only `vTaskResume()` brings it back |
| **Deleted** | Removed; the idle task frees its memory |

### Q: Blocked vs. Suspended?
- **Blocked:** wakes automatically when its event happens or its timeout expires.
- **Suspended:** never wakes automatically; another piece of code must call `vTaskResume()`.

### Example: suspend and resume a task with the button
```c
void ButtonTask(void *arg)
{
    uint8_t suspended = 0;
    for (;;)
    {
        if (HAL_GPIO_ReadPin(GPIOC, GPIO_PIN_13) == GPIO_PIN_RESET)
        {
            if (suspended) vTaskResume(hLedTask);
            else           vTaskSuspend(hLedTask);
            suspended = !suspended;
            vTaskDelay(pdMS_TO_TICKS(300));   // debounce
        }
        vTaskDelay(pdMS_TO_TICKS(20));
    }
}
```

### Q: How do you read a task's state at run time?
```c
eTaskState s = eTaskGetState(hLedTask); // eRunning, eReady, eBlocked, eSuspended, eDeleted
```

---

## 9. Scheduler

### Q: What is the scheduler?
The part of the kernel that decides **which ready task runs next**. FreeRTOS uses **fixed-priority preemptive scheduling with round-robin time slicing** for tasks of equal priority.

### Q: Explain the scheduling algorithms.
| Algorithm | Description |
|---|---|
| **Preemptive priority** | The highest-priority ready task always runs; it interrupts lower-priority ones immediately |
| **Cooperative** | A task runs until it yields or blocks voluntarily |
| **Round-robin / time slicing** | Equal-priority tasks share the CPU, one tick each |
| **Rate Monotonic (RMS)** | Static priorities: shorter period = higher priority (design method) |
| **EDF (Earliest Deadline First)** | Dynamic: the task with the closest deadline runs first |

### Q: How does preemption work? (timeline)
```
Priority
  3 |          [ High ]                    [ High ]
  2 |     [Med]        [Med]
  1 | [Low]                  [ Low ...  ]
    +-----------------------------------------------> time
         ^ High becomes ready -> preempts immediately
```

### Q: What is the tick?
A periodic interrupt (**SysTick** on STM32) at `configTICK_RATE_HZ` (usually 1 kHz = 1 ms). On each tick, the kernel:
- increments the tick count,
- unblocks tasks whose delay has expired,
- performs time slicing among equal-priority tasks.

### Q: What is the idle task?
It is created automatically by `vTaskStartScheduler()` at priority 0 and runs when no other task is ready. It:
- frees the memory of deleted tasks,
- runs `vApplicationIdleHook()`, which can be used for low-power `__WFI()` sleep.

### Q: What is starvation? How do you prevent it?
A low-priority task never runs because higher-priority tasks never block.
**Fix:** high-priority tasks must block (delay, wait on a queue or semaphore); never busy-wait.

### Q: `vTaskDelay` vs. `vTaskDelayUntil`?
```c
// Relative delay: the period = execution time + 10 ms, so it DRIFTS
void TaskA(void *a) { for (;;) { DoWork(); vTaskDelay(pdMS_TO_TICKS(10)); } }

// Absolute delay: exactly every 10 ms, NO DRIFT (use for control loops)
void TaskB(void *a)
{
    TickType_t last = xTaskGetTickCount();
    for (;;) { DoWork(); vTaskDelayUntil(&last, pdMS_TO_TICKS(10)); }
}
```

### Q: Can you control the scheduler manually?
```c
vTaskSuspendAll();   // pauses the scheduler (interrupts stay enabled)
/* ... non-preemptible code ... */
xTaskResumeAll();
taskYIELD();         // voluntarily give up the CPU to an equal-priority task
```

### Q: How do you measure CPU usage per task?
Enable `configGENERATE_RUN_TIME_STATS` (using a timer such as TIM2 as the counter) and call `vTaskGetRunTimeStats(buf)`.

---

## 10. Context Switching

### Q: What is context switching?
Saving the **CPU state (registers)** of the currently running task and restoring the state of the next task, so that each task continues exactly where it stopped.

### Q: What is the "context" on a Cortex-M4?
- Core registers **R0–R12, SP, LR, PC, xPSR**
- FPU registers **S0–S31, FPSCR** (if the FPU is used; lazy stacking)

### Q: How does context switching work on STM32 (Cortex-M)?
```
Task A running (uses PSP)
      |
  SysTick ISR / taskYIELD() / xQueueSend unblocks a higher-priority task
      |
  Set PendSV pending  (SCB->ICSR |= PENDSVSET)
      |
  PendSV_Handler (lowest priority, runs after all other ISRs)
      |-- Hardware auto-pushes: R0-R3, R12, LR, PC, xPSR  -> Task A stack
      |-- Software pushes:       R4-R11 (+S16-S31 if FPU)  -> Task A stack
      |-- Save PSP into TCB_A->pxTopOfStack
      |-- vTaskSwitchContext() -> pxCurrentTCB = TCB_B
      |-- Load PSP from TCB_B->pxTopOfStack
      |-- Software pops R4-R11
      |-- Exception return -> hardware pops R0-R3, R12, LR, PC, xPSR
      v
Task B running
```

### Q: Why is PendSV set to the lowest priority?
So that the context switch **never preempts another ISR**. It runs only after every pending interrupt has completed, which keeps ISR latency low and the kernel consistent.

### Q: Why use PendSV instead of doing the switch inside SysTick?
SysTick could fire while another ISR is running. Switching tasks at that moment would corrupt the ISR. PendSV defers the switch until it is safe.

### Q: What are SVC, MSP and PSP?
- **SVC:** used once by `vPortStartFirstTask()` to launch the first task.
- **MSP (Main Stack Pointer):** used by ISRs and the kernel.
- **PSP (Process Stack Pointer):** used by tasks; each task has its own stack area.

### Q: What is the cost of a context switch?
Roughly 100–200 CPU cycles on a Cortex-M4 (about 1 µs at 168 MHz). Too many switches waste CPU time, so avoid an unnecessarily high tick rate or tiny delays.

---

## 11. Queue

### Q: What is a queue?
A **thread-safe FIFO buffer** for passing **data** between tasks or from an ISR to a task.
- Data is **copied** into the queue (copy by value).
- The sender blocks if the queue is full; the receiver blocks if it is empty (both with a timeout).
- Several tasks can send to or receive from the same queue.

```
 Producer Task --> [ item | item | item |      ] --> Consumer Task
                      Queue (length = N, item size = S)
```

### Example 1: ADC sensor task to display task
```c
#include "queue.h"

QueueHandle_t xTempQueue;

typedef struct {
    uint32_t timestamp;
    float    temperature;
} SensorData_t;

void SensorTask(void *arg)
{
    SensorData_t data;
    for (;;)
    {
        HAL_ADC_Start(&hadc1);
        HAL_ADC_PollForConversion(&hadc1, 10);
        uint32_t raw = HAL_ADC_GetValue(&hadc1);

        data.timestamp   = xTaskGetTickCount();
        data.temperature = (raw * 3.3f / 4095.0f) * 100.0f;  // LM35: 10 mV/C

        if (xQueueSend(xTempQueue, &data, pdMS_TO_TICKS(10)) != pdPASS)
            printf("Queue full!\r\n");

        vTaskDelay(pdMS_TO_TICKS(200));
    }
}

void DisplayTask(void *arg)
{
    SensorData_t rx;
    for (;;)
    {
        if (xQueueReceive(xTempQueue, &rx, portMAX_DELAY) == pdPASS)
            printf("[%lu ms] Temp = %.2f C\r\n", rx.timestamp, rx.temperature);
    }
}

// in main():
xTempQueue = xQueueCreate(10, sizeof(SensorData_t));
```

### Example 2: UART receive interrupt to command task
```c
QueueHandle_t xUartRxQueue;
uint8_t rxByte;

void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart)
{
    if (huart->Instance == USART2)
    {
        BaseType_t xWoken = pdFALSE;
        xQueueSendFromISR(xUartRxQueue, &rxByte, &xWoken);
        HAL_UART_Receive_IT(&huart2, &rxByte, 1);       // re-arm the receive
        portYIELD_FROM_ISR(xWoken);
    }
}

void CommandTask(void *arg)
{
    char cmd[32]; uint8_t idx = 0, ch;
    HAL_UART_Receive_IT(&huart2, &rxByte, 1);
    for (;;)
    {
        xQueueReceive(xUartRxQueue, &ch, portMAX_DELAY);
        if (ch == '\r' || ch == '\n')
        {
            cmd[idx] = '\0'; idx = 0;
            if      (strcmp(cmd, "on")  == 0) HAL_GPIO_WritePin(GPIOA, GPIO_PIN_5, GPIO_PIN_SET);
            else if (strcmp(cmd, "off") == 0) HAL_GPIO_WritePin(GPIOA, GPIO_PIN_5, GPIO_PIN_RESET);
            else printf("Unknown: %s\r\n", cmd);
        }
        else if (idx < sizeof(cmd) - 1) cmd[idx++] = ch;
    }
}
// main: xUartRxQueue = xQueueCreate(64, sizeof(uint8_t));
```

### Q: Queue API summary
| API | Use |
|---|---|
| `xQueueCreate(len, size)` | Create a queue |
| `xQueueSend / xQueueSendToBack` | Add to the back |
| `xQueueSendToFront` | Add to the front (urgent item) |
| `xQueueReceive` | Remove and read |
| `xQueuePeek` | Read without removing |
| `xQueueOverwrite` | For a queue of length 1 (latest value / "mailbox") |
| `uxQueueMessagesWaiting` | Number of items in the queue |
| `...FromISR` | ISR-safe versions |

### Q: Why use a queue instead of a global variable?
Global variables need manual protection, can cause race conditions, and force the reader to poll. Queues are thread-safe, ordered, buffered, and let tasks **block efficiently** (0% CPU while waiting).

### Q: How do you send large data efficiently?
Send a **pointer** to a buffer instead of copying the whole buffer, and make sure the buffer stays valid until the receiver has used it (e.g., use a memory pool).

### Q: What is a queue set?
`xQueueCreateSet()` lets a task block on **several queues and semaphores at once**.

---

## 12. Semaphore

### Q: What is a semaphore?
A kernel object used for **signalling and synchronization**. It holds a count; `Give` increments it and `Take` decrements it (and blocks if the count is 0).

### Q: What types are there?
| Type | Count | Use case |
|---|---|---|
| **Binary** | 0 or 1 | Event signalling (ISR to task) |
| **Counting** | 0..N | Counting events or managing N identical resources |

### Example 1: binary semaphore for deferred interrupt processing (button EXTI)
```c
#include "semphr.h"

SemaphoreHandle_t xButtonSem;

// PC13 configured as EXTI, falling edge, NVIC priority >= 5
void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)
{
    if (GPIO_Pin == GPIO_PIN_13)
    {
        BaseType_t xWoken = pdFALSE;
        xSemaphoreGiveFromISR(xButtonSem, &xWoken);
        portYIELD_FROM_ISR(xWoken);
    }
}

void ButtonHandlerTask(void *arg)
{
    for (;;)
    {
        if (xSemaphoreTake(xButtonSem, portMAX_DELAY) == pdTRUE)
        {
            printf("Button pressed! Doing heavy processing...\r\n");
            HAL_GPIO_TogglePin(GPIOA, GPIO_PIN_5);
        }
    }
}
// main: xButtonSem = xSemaphoreCreateBinary();
```
**Why?** Keep ISRs **short**. The ISR only signals; the heavy work runs in a task, where it can be preempted and can call blocking APIs.

### Example 2: counting semaphore for a pool of 3 buffers
```c
SemaphoreHandle_t xBufPool;

void WorkerTask(void *arg)
{
    for (;;)
    {
        if (xSemaphoreTake(xBufPool, pdMS_TO_TICKS(100)) == pdTRUE)
        {
            printf("%s got a buffer\r\n", pcTaskGetName(NULL));
            vTaskDelay(pdMS_TO_TICKS(500));      // use the buffer
            xSemaphoreGive(xBufPool);            // return it
        }
        else printf("%s: no buffer free\r\n", pcTaskGetName(NULL));
        vTaskDelay(pdMS_TO_TICKS(50));
    }
}
// main: xBufPool = xSemaphoreCreateCounting(3, 3);  // max = 3, initial = 3
```

### Example 3: counting semaphore to count events (no event lost)
```c
xPulseSem = xSemaphoreCreateCounting(100, 0);
// ISR: xSemaphoreGiveFromISR(xPulseSem, &w);  (each pulse adds 1)
// Task: while (xSemaphoreTake(xPulseSem, 0) == pdTRUE) pulses++;
```

---

## 13. Mutex

### Q: What is a mutex?
**MUTual EXclusion:** a lock that allows **only one task at a time** to access a shared resource (UART, I2C, SPI, LCD, shared struct).
- It has **ownership**: only the task that took it may give it back.
- It supports **priority inheritance**, which reduces priority inversion.
- It **cannot be used from an ISR**.

### Example: protecting UART2 shared by several tasks
**Without a mutex**, the output is garbled:
```
TaHello sk2: frorm Taskm 1
```
**With a mutex:**
```c
SemaphoreHandle_t xUartMutex;

void SafePrintf(const char *fmt, ...)
{
    char buf[128];
    va_list args;
    va_start(args, fmt);
    vsnprintf(buf, sizeof(buf), fmt, args);
    va_end(args);

    if (xSemaphoreTake(xUartMutex, pdMS_TO_TICKS(100)) == pdTRUE)
    {
        HAL_UART_Transmit(&huart2, (uint8_t *)buf, strlen(buf), HAL_MAX_DELAY);
        xSemaphoreGive(xUartMutex);
    }
}

void Task1(void *a) { for (;;) { SafePrintf("Hello from Task 1\r\n"); vTaskDelay(10); } }
void Task2(void *a) { for (;;) { SafePrintf("Hello from Task 2\r\n"); vTaskDelay(15); } }

// main: xUartMutex = xSemaphoreCreateMutex();
```

### Example: protecting a shared I2C bus (sensor + EEPROM)
```c
SemaphoreHandle_t xI2CMutex;

HAL_StatusTypeDef I2C_Read_Safe(uint16_t addr, uint8_t reg, uint8_t *data, uint16_t len)
{
    HAL_StatusTypeDef st = HAL_ERROR;
    if (xSemaphoreTake(xI2CMutex, pdMS_TO_TICKS(50)) == pdTRUE)
    {
        st = HAL_I2C_Mem_Read(&hi2c1, addr << 1, reg, I2C_MEMADD_SIZE_8BIT, data, len, 100);
        xSemaphoreGive(xI2CMutex);
    }
    return st;
}
```

### Q: What is the difference between a mutex and a binary semaphore? (MOST ASKED)
| Feature | Mutex | Binary Semaphore |
|---|---|---|
| Purpose | **Locking** a resource | **Signalling** an event |
| Ownership | Yes; only the owner can release it | No; any task or ISR can give it |
| Priority inheritance | **Yes** | No |
| Use in an ISR | **No** | Yes (`GiveFromISR`) |
| Initial state | Available (1) | Empty (0) |
| Analogy | Key to a bathroom | Doorbell / flag |

### Q: What is priority inversion? (Mars Pathfinder, 1997)
```
Priority
 High (H) |      [H runs]--wants mutex--X (blocked).............[H gets mutex]
 Med  (M) |                         [====== M runs for a long time ======]
 Low  (L) | [L takes mutex]...[L]                                    [L gives]
          +--------------------------------------------------------------> time
          H is effectively blocked by M, a LOWER priority task -> INVERSION!
```
**Solutions:**
1. **Priority inheritance** (FreeRTOS mutexes): L temporarily runs at H's priority, so M cannot preempt it.
2. **Priority ceiling protocol:** the mutex holder is raised to the highest priority of any task that uses the mutex.
3. Keep critical sections short; avoid sharing resources across very different priorities.

### Q: What is a deadlock?
```
Task A: Take(M1) ... Take(M2)   <- waits for M2, which B holds
Task B: Take(M2) ... Take(M1)   <- waits for M1, which A holds
=> both are blocked forever
```
**Prevention:**
- Always take mutexes in the **same global order**.
- Use **timeouts** instead of `portMAX_DELAY` and handle failure.
- Avoid holding one lock while waiting for another.
- Use a single gatekeeper task that owns the resource.

### Q: What is a recursive mutex?
The same task can take it several times; it must give it back the same number of times.
```c
xRecMutex = xSemaphoreCreateRecursiveMutex();
xSemaphoreTakeRecursive(xRecMutex, portMAX_DELAY);
xSemaphoreTakeRecursive(xRecMutex, portMAX_DELAY);   // OK, no self-deadlock
xSemaphoreGiveRecursive(xRecMutex);
xSemaphoreGiveRecursive(xRecMutex);
```

### Q: What is a gatekeeper task?
A single task that **owns** a resource; other tasks send requests to it through a queue. No mutex is needed, and there is no priority inversion on that resource.

---

## 14. Event Flags (Event Groups)

### Q: What are event flags?
A group of bits (24 usable bits in a 32-bit `EventBits_t`) in which **each bit represents an event**. Tasks can wait for:
- **ANY** of the bits (OR logic), or
- **ALL** of the bits (AND logic).

Several tasks can wait on the same group, and all of them are woken together (broadcast).

### Example: start the cloud upload only when WiFi, sensor and RTC are all ready
```c
#include "event_groups.h"

#define EVT_WIFI_READY    (1 << 0)
#define EVT_SENSOR_READY  (1 << 1)
#define EVT_RTC_SYNCED    (1 << 2)
#define EVT_ALL_READY     (EVT_WIFI_READY | EVT_SENSOR_READY | EVT_RTC_SYNCED)

EventGroupHandle_t xSysEvents;

void WifiInitTask(void *a)
{
    vTaskDelay(pdMS_TO_TICKS(2000));   // simulate connecting
    printf("WiFi connected\r\n");
    xEventGroupSetBits(xSysEvents, EVT_WIFI_READY);
    vTaskDelete(NULL);
}

void SensorInitTask(void *a)
{
    vTaskDelay(pdMS_TO_TICKS(500));
    printf("Sensor ready\r\n");
    xEventGroupSetBits(xSysEvents, EVT_SENSOR_READY);
    vTaskDelete(NULL);
}

void RtcSyncTask(void *a)
{
    vTaskDelay(pdMS_TO_TICKS(1200));
    printf("RTC synced\r\n");
    xEventGroupSetBits(xSysEvents, EVT_RTC_SYNCED);
    vTaskDelete(NULL);
}

void CloudTask(void *a)
{
    EventBits_t bits = xEventGroupWaitBits(
        xSysEvents,
        EVT_ALL_READY,
        pdFALSE,        // xClearOnExit: keep the bits set
        pdTRUE,         // xWaitForAllBits: AND logic
        portMAX_DELAY);

    printf("All ready (0x%02lX), starting upload\r\n", bits);
    for (;;) { printf("Uploading...\r\n"); vTaskDelay(pdMS_TO_TICKS(5000)); }
}

// main: xSysEvents = xEventGroupCreate();
```

### Example: wait for ANY event (error or stop button)
```c
EventBits_t b = xEventGroupWaitBits(xSysEvents, EVT_ERROR | EVT_STOP,
                                    pdTRUE, pdFALSE, portMAX_DELAY);  // OR logic
if (b & EVT_ERROR) HandleError();
if (b & EVT_STOP)  StopMotor();
```

### Q: How do you set event bits from an ISR?
`xEventGroupSetBitsFromISR()` defers the work to the timer daemon task, so it requires `configUSE_TIMERS = 1`.

### Q: What is `xEventGroupSync()`?
A **barrier**: several tasks each set their own bit and wait until all bits are set, then they all continue together (task rendezvous).

### Q: Event flags vs. semaphore vs. queue?
| | Semaphore | Queue | Event Group |
|---|---|---|---|
| Carries data | No | **Yes** | No (bits only) |
| Multiple conditions | No | No | **Yes (AND/OR)** |
| Wakes multiple tasks | No (one) | No (one) | **Yes (broadcast)** |
| Usable from an ISR | Yes | Yes | Yes (deferred) |

---

## 15. Task Notifications

### Q: What are task notifications?
Every task has a built-in 32-bit **notification value**. You can signal a task **directly**, without creating a separate object. This is **up to 45% faster and uses less RAM** than semaphores or queues.
**Limitation:** there is only one receiver (a specific task), and the sender cannot block waiting for the receiver.

### Example: replace a binary semaphore (ISR to task)
```c
TaskHandle_t hProcTask;

void HAL_ADC_ConvCpltCallback(ADC_HandleTypeDef *hadc)
{
    BaseType_t w = pdFALSE;
    vTaskNotifyGiveFromISR(hProcTask, &w);
    portYIELD_FROM_ISR(w);
}

void ProcTask(void *a)
{
    for (;;)
    {
        ulTaskNotifyTake(pdTRUE, portMAX_DELAY);   // acts like a binary semaphore
        uint32_t v = HAL_ADC_GetValue(&hadc1);
        printf("ADC = %lu\r\n", v);
        HAL_ADC_Start_IT(&hadc1);
    }
}
```

---

## 16. Software Timers

### Q: What is a software timer?
A callback that runs after a delay (**one-shot**) or periodically (**auto-reload**). The callback runs in the **timer daemon (service) task**, not in an ISR.
**Rules:** never block inside a timer callback; keep it short.

```c
#include "timers.h"

TimerHandle_t xHeartbeatTimer, xBacklightTimer;

void HeartbeatCb(TimerHandle_t t) { HAL_GPIO_TogglePin(GPIOA, GPIO_PIN_5); }
void BacklightOffCb(TimerHandle_t t) { LCD_BacklightOff(); }

// main:
xHeartbeatTimer = xTimerCreate("HB", pdMS_TO_TICKS(1000), pdTRUE,  NULL, HeartbeatCb);
xBacklightTimer = xTimerCreate("BL", pdMS_TO_TICKS(5000), pdFALSE, NULL, BacklightOffCb);
xTimerStart(xHeartbeatTimer, 0);

// On a key press: turn the backlight on and restart the 5 s timeout
LCD_BacklightOn();
xTimerReset(xBacklightTimer, 0);
```

---

## 17. Memory Management

### Q: What heap schemes does FreeRTOS provide?
| Scheme | Free supported | Combines adjacent free blocks | Notes |
|---|---|---|---|
| heap_1 | No | - | Allocate only; simplest; safety-critical use |
| heap_2 | Yes | No | Fragments; legacy |
| heap_3 | Yes | - | Wraps the C library `malloc`/`free` (not deterministic) |
| **heap_4** | Yes | **Yes** | **Most common; the STM32CubeMX default** |
| heap_5 | Yes | Yes | Heap can span several RAM regions (e.g., SRAM1 + CCM) |

### Q: Why avoid `malloc` in embedded systems?
- **Not deterministic** (its execution time varies).
- **Fragmentation** over long run times.
- **Not thread-safe** in some C libraries.
**Best practice:** allocate everything at startup, or use static allocation.

### Q: How do you detect that the heap ran out?
```c
void vApplicationMallocFailedHook(void) { printf("HEAP EXHAUSTED\r\n"); while (1); }
size_t freeNow = xPortGetFreeHeapSize();
size_t minEver = xPortGetMinimumEverFreeHeapSize();
```

---

## 18. Interrupts with RTOS on STM32

### Q: What are the rules for ISRs in an RTOS?
1. **Use only `...FromISR()` APIs** inside ISRs.
2. **Never block** in an ISR (no `vTaskDelay`, and no `xQueueReceive` with a timeout).
3. Keep ISRs **short**; defer the work to a task (semaphore or notification).
4. Call **`portYIELD_FROM_ISR(xHigherPriorityTaskWoken)`** at the end.
5. The **NVIC priority** must be numerically **>= `configLIBRARY_MAX_SYSCALL_INTERRUPT_PRIORITY`** (e.g., 5) to call FreeRTOS APIs.

### Q: Explain the STM32 interrupt priority numbering.
On Cortex-M, a **lower number means a higher urgency**.
```
Priority 0..4  -> Above the kernel: FreeRTOS APIs NOT allowed; never masked by the kernel
                  (use for ultra-fast ISRs, e.g., motor PWM fault)
Priority 5..15 -> Can call ...FromISR() APIs; masked during critical sections
```
```c
HAL_NVIC_SetPriority(EXTI15_10_IRQn, 6, 0);   // OK for FreeRTOS calls
HAL_NVIC_SetPriority(USART2_IRQn,   5, 0);    // OK
HAL_NVIC_SetPriority(TIM1_BRK_IRQn,  2, 0);   // fast, NO RTOS calls inside
```
A wrong priority causes `configASSERT` failures or hard faults. **This is a very common STM32 + FreeRTOS bug.**

### Q: What is `xHigherPriorityTaskWoken`?
It is set to `pdTRUE` by a `FromISR` API if the call unblocked a task with a higher priority than the interrupted task. `portYIELD_FROM_ISR` then triggers PendSV, so the woken task runs **immediately after the ISR exits**, not at the next tick.

### Q: What is a critical section?
```c
taskENTER_CRITICAL();              // masks interrupts at or below MAX_SYSCALL priority
shared_counter++;
taskEXIT_CRITICAL();

UBaseType_t s = taskENTER_CRITICAL_FROM_ISR();   // ISR version
/* ... */
taskEXIT_CRITICAL_FROM_ISR(s);
```
Keep it **very short** (a few instructions). It raises interrupt latency for every interrupt it masks.

---

## 19. Complete STM32 Project Example

### Smart Temperature Controller (NUCLEO-F446RE)
**Hardware:** LM35 on PA0 (ADC1_IN0), fan via PWM on PA6 (TIM3_CH1), LED on PA5, button on PC13, UART2 for logging.

```
  [ADC ISR] --notify--> [SensorTask  P4] --queue--> [ControlTask P3] --> Fan PWM (TIM3)
                                                          |
                                                          | queue (log msgs)
                                                          v
  [Button EXTI] --sem--> [ButtonTask P3]           [LoggerTask  P1] --mutex--> UART2
                              |
                              +-- event group: MODE_AUTO / MODE_MANUAL
  [Heartbeat software timer] --> LED toggle every 1 s
```

```c
/* ===================== main.c (core parts) ===================== */
#include "main.h"
#include "FreeRTOS.h"
#include "task.h"
#include "queue.h"
#include "semphr.h"
#include "event_groups.h"
#include "timers.h"
#include <stdio.h>
#include <string.h>
#include <stdarg.h>

extern ADC_HandleTypeDef  hadc1;
extern TIM_HandleTypeDef  htim3;
extern UART_HandleTypeDef huart2;

/* ---------- RTOS objects ---------- */
TaskHandle_t       hSensorTask;
QueueHandle_t      xTempQueue;
QueueHandle_t      xLogQueue;
SemaphoreHandle_t  xButtonSem;
SemaphoreHandle_t  xUartMutex;
EventGroupHandle_t xModeEvents;
TimerHandle_t      xHeartbeat;

#define MODE_AUTO    (1 << 0)
#define MODE_MANUAL  (1 << 1)

typedef struct { char text[64]; } LogMsg_t;

/* ---------- Helpers ---------- */
static void Log(const char *fmt, ...)
{
    LogMsg_t m;
    va_list a; va_start(a, fmt);
    vsnprintf(m.text, sizeof(m.text), fmt, a);
    va_end(a);
    xQueueSend(xLogQueue, &m, 0);              // never block the caller
}

static void Fan_SetDuty(uint8_t pct)
{
    uint32_t arr = __HAL_TIM_GET_AUTORELOAD(&htim3);
    __HAL_TIM_SET_COMPARE(&htim3, TIM_CHANNEL_1, (arr + 1) * pct / 100);
}

/* ---------- ISRs / callbacks ---------- */
void HAL_ADC_ConvCpltCallback(ADC_HandleTypeDef *hadc)
{
    BaseType_t w = pdFALSE;
    vTaskNotifyGiveFromISR(hSensorTask, &w);
    portYIELD_FROM_ISR(w);
}

void HAL_GPIO_EXTI_Callback(uint16_t pin)
{
    if (pin == GPIO_PIN_13)
    {
        BaseType_t w = pdFALSE;
        xSemaphoreGiveFromISR(xButtonSem, &w);
        portYIELD_FROM_ISR(w);
    }
}

static void HeartbeatCb(TimerHandle_t t) { HAL_GPIO_TogglePin(GPIOA, GPIO_PIN_5); }

/* ---------- Tasks ---------- */
void SensorTask(void *arg)                      // Priority 4
{
    TickType_t last = xTaskGetTickCount();
    for (;;)
    {
        HAL_ADC_Start_IT(&hadc1);
        if (ulTaskNotifyTake(pdTRUE, pdMS_TO_TICKS(50)))
        {
            uint32_t raw = HAL_ADC_GetValue(&hadc1);
            float tempC = raw * 3.3f / 4095.0f * 100.0f;
            xQueueOverwrite(xTempQueue, &tempC); // always keep the latest value
        }
        else Log("ADC timeout\r\n");
        vTaskDelayUntil(&last, pdMS_TO_TICKS(100));   // exactly 10 Hz
    }
}

void ControlTask(void *arg)                     // Priority 3
{
    float t;
    for (;;)
    {
        if (xQueuePeek(xTempQueue, &t, portMAX_DELAY) == pdPASS)
        {
            EventBits_t mode = xEventGroupGetBits(xModeEvents);
            uint8_t duty;
            if (mode & MODE_AUTO)
            {
                if      (t < 25.0f) duty = 0;
                else if (t < 30.0f) duty = 40;
                else if (t < 35.0f) duty = 70;
                else                duty = 100;
            }
            else duty = 100;                     // manual = full speed
            Fan_SetDuty(duty);
            Log("T=%.1fC Fan=%u%% Mode=%s\r\n", t, duty,
                (mode & MODE_AUTO) ? "AUTO" : "MANUAL");
        }
        vTaskDelay(pdMS_TO_TICKS(500));
    }
}

void ButtonTask(void *arg)                      // Priority 3
{
    for (;;)
    {
        xSemaphoreTake(xButtonSem, portMAX_DELAY);
        vTaskDelay(pdMS_TO_TICKS(50));           // debounce
        if (xEventGroupGetBits(xModeEvents) & MODE_AUTO)
        {
            xEventGroupClearBits(xModeEvents, MODE_AUTO);
            xEventGroupSetBits(xModeEvents, MODE_MANUAL);
        }
        else
        {
            xEventGroupClearBits(xModeEvents, MODE_MANUAL);
            xEventGroupSetBits(xModeEvents, MODE_AUTO);
        }
        Log("Mode toggled\r\n");
        xSemaphoreTake(xButtonSem, 0);           // drop presses caused by bounce
    }
}

void LoggerTask(void *arg)                      // Priority 1 (lowest)
{
    LogMsg_t m;
    for (;;)
    {
        xQueueReceive(xLogQueue, &m, portMAX_DELAY);
        if (xSemaphoreTake(xUartMutex, pdMS_TO_TICKS(100)) == pdTRUE)
        {
            HAL_UART_Transmit(&huart2, (uint8_t *)m.text, strlen(m.text), 100);
            xSemaphoreGive(xUartMutex);
        }
    }
}

/* ---------- Hooks ---------- */
void vApplicationStackOverflowHook(TaskHandle_t t, char *name)
{ __disable_irq(); while (1); }
void vApplicationMallocFailedHook(void)
{ __disable_irq(); while (1); }
void vApplicationIdleHook(void)
{ __WFI(); }                                     // sleep until the next interrupt

/* ---------- main ---------- */
int main(void)
{
    HAL_Init();
    SystemClock_Config();
    MX_GPIO_Init();
    MX_ADC1_Init();
    MX_TIM3_Init();
    MX_USART2_UART_Init();
    HAL_TIM_PWM_Start(&htim3, TIM_CHANNEL_1);

    xTempQueue  = xQueueCreate(1, sizeof(float));      // mailbox
    xLogQueue   = xQueueCreate(16, sizeof(LogMsg_t));
    xButtonSem  = xSemaphoreCreateBinary();
    xUartMutex  = xSemaphoreCreateMutex();
    xModeEvents = xEventGroupCreate();
    xEventGroupSetBits(xModeEvents, MODE_AUTO);
    xHeartbeat  = xTimerCreate("HB", pdMS_TO_TICKS(1000), pdTRUE, NULL, HeartbeatCb);

    configASSERT(xTempQueue && xLogQueue && xButtonSem && xUartMutex && xModeEvents && xHeartbeat);

    xTaskCreate(SensorTask,  "Sensor",  256, NULL, 4, &hSensorTask);
    xTaskCreate(ControlTask, "Control", 384, NULL, 3, NULL);   // printf-style formatting uses stack
    xTaskCreate(ButtonTask,  "Button",  256, NULL, 3, NULL);
    xTaskCreate(LoggerTask,  "Logger",  256, NULL, 1, NULL);
    xTimerStart(xHeartbeat, 0);

    vTaskStartScheduler();
    while (1);
}
```

### What this project demonstrates
| Concept | Where it is used |
|---|---|
| Tasks and priorities | 4 tasks, P1–P4 |
| Scheduler / preemption | Sensor (P4) preempts Logger (P1) |
| `vTaskDelayUntil` | Exact 10 Hz sampling |
| Task notification | ADC ISR to Sensor task |
| Queue (mailbox) | Latest temperature (`xQueueOverwrite` / `xQueuePeek`) |
| Queue (FIFO) | Log messages to Logger |
| Binary semaphore | Button ISR to Button task |
| Mutex | UART2 protection |
| Event group | AUTO/MANUAL mode |
| Software timer | Heartbeat LED |
| Idle hook | Low-power `__WFI()` |
| Hooks | Stack overflow and malloc failure detection |

> **Note:** printing `%f` with newlib-nano requires the linker flag `-u _printf_float` (STM32CubeIDE: Project Properties > C/C++ Build > Settings > MCU Settings > "Use float with printf").

---

## 20. Common Bugs & Debugging

| Symptom | Likely cause | Fix |
|---|---|---|
| HardFault right after the scheduler starts | Stack too small, or the heap is too small for the tasks | Increase the stack/heap; check the return values of `xTaskCreate` |
| `configASSERT` failure in `vPortValidateInterruptPriority` | ISR priority is numerically < 5 and calls an RTOS API | Set the NVIC priority to 5 or higher |
| `HAL_Delay` hangs | HAL timebase on SysTick conflicts with FreeRTOS | Use TIM6 as the HAL timebase |
| A low-priority task never runs | Starvation; a high-priority task busy-loops | Add blocking calls |
| Garbled UART output | Shared resource without protection | Mutex or gatekeeper task |
| System freezes | Deadlock or `portMAX_DELAY` everywhere | Lock ordering and timeouts |
| Random crashes | Stack overflow | `configCHECK_FOR_STACK_OVERFLOW 2` plus high-water mark checks |
| `printf` with floats prints nothing | newlib-nano without float support | `-u _printf_float` |
| Task parameter is corrupted | A pointer to a local variable in `main` was passed | Use static or global storage |

**Debug tools:** STM32CubeIDE FreeRTOS task-aware view, SEGGER SystemView, Percepio Tracealyzer, `vTaskList()`, `vTaskGetRunTimeStats()`.

---

## 21. Rapid-Fire Interview Questions

### Embedded basics

1. **What is an embedded system?** A dedicated computer inside a device that does a specific function with limited resources.
2. **MCU vs. MPU?** An MCU has CPU, memory and peripherals on one chip; an MPU is a CPU only.
3. **What does `volatile` do?** It prevents the compiler from optimizing away accesses to variables changed by hardware, ISRs or other tasks.
4. **What is an ISR?** An Interrupt Service Routine: the function that runs automatically when a hardware event occurs.
5. **What is a watchdog timer?** A hardware timer that resets the MCU if the software stops refreshing it (because it hung).
6. **What is DMA?** Direct Memory Access: it moves data between peripherals and memory without the CPU.
7. **Polling vs. interrupts?** Polling checks repeatedly and wastes CPU time; an interrupt notifies the CPU only when an event occurs.
8. **Big vs. little endian?** The byte order of multi-byte values; ARM Cortex-M is little-endian by default.
9. **What do `static` inside a function and `static` at file scope mean?** Inside a function, the value persists between calls; at file scope, the symbol is visible only in that file.
10. **Name the communication protocols.** UART (asynchronous, 2 wires), SPI (synchronous, 4 wires, fast, full-duplex), I2C (2 wires, addressed, multi-device), CAN (automotive, differential).

### RTOS

11. **What is an RTOS?** An OS that guarantees deterministic timing for multitasking embedded systems.
12. **Why use an RTOS?** Multitasking, priorities, determinism, modularity, efficient CPU use and built-in IPC.
13. **Does real-time mean fast?** No. It means predictable and meeting deadlines.
14. **Which scheduling does FreeRTOS use?** Fixed-priority preemptive with round-robin among equal priorities.
15. **What is a context switch?** Saving the current task's registers and restoring the next task's registers (done in PendSV on Cortex-M).
16. **Why is PendSV set to the lowest priority?** So a context switch never interrupts an ISR.
17. **What are the task states?** Running, Ready, Blocked, Suspended (plus Deleted).
18. **Mutex vs. semaphore?** A mutex locks a resource (ownership, priority inheritance); a semaphore signals an event (no ownership, ISR-safe).
19. **What is priority inversion and how is it fixed?** A high-priority task is blocked by a lower-priority one through a shared mutex; fix it with priority inheritance or a priority ceiling.
20. **What is a deadlock and how is it prevented?** Circular waiting on locks; prevent it with consistent lock ordering and timeouts.
21. **Queue vs. global variable?** A queue is thread-safe, buffered and blocking; a global variable needs manual protection.
22. **Event group vs. semaphore?** An event group waits on several bits with AND/OR logic and can wake several tasks.
23. **Can you call `vTaskDelay` in an ISR?** No. Use only `FromISR` APIs, and never block.
24. **What is `portYIELD_FROM_ISR`?** It forces a context switch on ISR exit if a higher-priority task was woken.
25. **What is the idle task?** Priority 0; runs when nothing else can, cleans up deleted tasks, and can be used for low-power sleep.
26. **What is the tick?** The periodic SysTick interrupt that drives delays and time slicing.
27. **What is tickless idle?** The tick stops during long idle periods to save power (`configUSE_TICKLESS_IDLE`).
28. **Why use TIM6 as the HAL timebase with FreeRTOS?** SysTick is reserved for the FreeRTOS tick.
29. **Which heap scheme is most commonly used?** heap_4, which combines adjacent free blocks.
30. **Task notification vs. semaphore?** A notification is faster and lighter but has a single receiver task.
31. **What is reentrancy?** A function that is safe to call from several tasks or ISRs at the same time (no unprotected shared state).
32. **What is a critical section?** A short code region with interrupts or the scheduler disabled to protect shared data.
33. **What is jitter?** Variation in task timing; reduce it with `vTaskDelayUntil` and a good priority design.
34. **What is Rate Monotonic Scheduling?** Shorter period = higher priority; schedulable if utilization <= n(2^(1/n) - 1), about 69% for many tasks.
35. **Hard vs. soft real-time?** Hard: a missed deadline is a failure. Soft: a missed deadline only reduces quality.

---

---

## 22. AUTOSAR OS vs. RTOS (Basic & Extended Tasks)

### Q: Is AUTOSAR OS an RTOS?
**Yes.** AUTOSAR OS **is an RTOS**. It is a real-time OS **specification (standard)** written for automotive ECUs. It is not a separate category of operating system.

```
                     RTOS (category: any real-time OS)
                                 |
        +------------------------+-------------------------+
        |                        |                         |
  General-purpose RTOS     Standard-based RTOS        Safety / domain RTOS
  (own API, no standard)   (follow a specification)
   - FreeRTOS               - OSEK/VDX OS  (ISO 17356-3)   - QNX, VxWorks
   - Zephyr                 - AUTOSAR OS   (extends OSEK)  - SAFERTOS
   - ThreadX                - POSIX RTOS profiles (PSE51/52)
                            - ITRON (Japan)
                            - ARINC 653 (avionics)
```

### Q: What is the difference between a "general RTOS" (FreeRTOS) and AUTOSAR OS?
| Feature | FreeRTOS (general RTOS) | AUTOSAR OS |
|---|---|---|
| What it is | A product (source code) | A **specification**; vendors implement it |
| Standard | No formal standard API (CMSIS-RTOS wrapper is optional) | Based on **OSEK/VDX (ISO 17356-3)** + AUTOSAR SWS OS |
| Configuration | **Dynamic**: tasks/queues created at run time (`xTaskCreate`) | **Static**: everything defined at build time in an **OIL/ARXML** config file, then generated |
| Task creation at run time | Yes | **No** |
| Task types | One type (any task can block) | **Basic** and **Extended** |
| Inter-task communication | Queues, semaphores, mutexes, event groups | **Events**, **Resources** (priority ceiling), IOC (inter-OS-application communication) |
| Mutual exclusion | Mutex with **priority inheritance** | **Resources** with **priority ceiling protocol** (no deadlock, no inversion) |
| Timing | Software timers | **Counters, Alarms, Schedule Tables** |
| Safety features | Basic (stack overflow check) | **Memory protection, timing protection, OS-Applications** (Scalability Classes SC1–SC4) |
| Safety certification | SAFERTOS variant | Implementations certified to **ISO 26262 up to ASIL-D** |
| Typical use | IoT, consumer, industrial | Automotive ECUs (engine, brakes, body control) |
| Cost | Free (MIT license) | Commercial (Vector, ETAS, Elektrobit, ...) |

### Q: Which standard does an RTOS follow?
**An RTOS does not have to follow any standard.** It depends on the RTOS:
| Standard | Domain | Examples |
|---|---|---|
| **OSEK/VDX OS (ISO 17356-3)** | Automotive (older) | ERIKA Enterprise, Trampoline |
| **AUTOSAR OS** (extends OSEK) | Automotive (current) | Vector MICROSAR OS, ETAS RTA-OS, EB tresos AutoCore OS |
| **POSIX (IEEE 1003.13 PSE51–54)** | General / industrial | QNX, VxWorks, NuttX, RTEMS |
| **ARINC 653** | Avionics | VxWorks 653, PikeOS, LynxOS-178 |
| **ITRON / μITRON** | Japanese consumer electronics | TOPPERS |
| **CMSIS-RTOS v2** | ARM Cortex-M API (a wrapper API) | Keil RTX5, FreeRTOS via wrapper |
| *None (own API)* | General embedded | **FreeRTOS**, Zephyr, ThreadX |

### Q: Is AUTOSAR OS defined by MICROSAR?
**No.** That reverses the relationship:
- **AUTOSAR** (the consortium: BMW, Bosch, Continental, Toyota, VW, ...) **writes the specification** (the SWS OS document).
- **MICROSAR** is **Vector Informatik's product**, i.e. its **implementation** of the AUTOSAR standard. **MICROSAR OS** is Vector's AUTOSAR OS.

```
 AUTOSAR specification (the "rules")
            |
            | implemented by different vendors
   +--------+-------------+--------------------+---------------------+
   |                      |                    |                     |
 Vector               ETAS (Bosch)         Elektrobit             Siemens (Mentor)
 MICROSAR OS          RTA-OS               EB tresos AutoCore OS  Capital VSTAR OS
```
All of these follow the **same AUTOSAR API** (`ActivateTask`, `WaitEvent`, `SetEvent`, `GetResource`, ...), so application code can be ported between vendors.

### Q: What are the AUTOSAR OS Scalability Classes?
| Class | Adds |
|---|---|
| **SC1** | OSEK OS + Schedule Tables (no protection) |
| **SC2** | SC1 + **Timing protection** (execution-time budgets) |
| **SC3** | SC1 + **Memory protection** (MPU) + OS-Applications |
| **SC4** | SC2 + SC3 (full protection; used for ASIL-D) |

### Q: What are the conformance classes (OSEK/AUTOSAR)?
| Class | Task types | Multiple activations | Tasks per priority |
|---|---|---|---|
| **BCC1** | Basic only | No | One |
| **BCC2** | Basic only | Yes | Several |
| **ECC1** | Basic + Extended | No (basic) | One |
| **ECC2** | Basic + Extended | Yes (basic) | Several |

### Q: What is the difference between basic and extended tasks?

**Basic task: 3 states (no Waiting)**
```
              activate
 SUSPENDED ────────────► READY
     ▲                  │   ▲
     │ terminate        │   │ preempt
     │                  ▼   │
     └──────────────── RUNNING
```

**Extended task: 4 states (with Waiting)**
```
              activate
 SUSPENDED ────────────► READY ◄───────────┐
     ▲                  │   ▲               │ SetEvent()
     │ terminate        │   │ preempt       │
     │                  ▼   │               │
     └──────────────── RUNNING ──────────► WAITING
                              WaitEvent()
```

| Feature | Basic Task | Extended Task |
|---|---|---|
| States | Suspended, Ready, Running (**3**) | + **Waiting** (**4**) |
| Can wait for events | No | Yes (`WaitEvent`) |
| Stack | **Can share one stack** | Needs its **own stack** |
| RAM usage | Low | Higher |
| Execution | Runs to completion | Can pause in the middle |
| Multiple activations | Yes (BCC2/ECC2) | No |
| Typical use | Cyclic 1/10/100 ms jobs | CAN handling, state machines, communication |

### Q: Why does a basic task have no Waiting state?
By design it **runs to completion and never blocks halfway through**. This means:
1. Its worst-case execution time is **easy to analyze** (deterministic).
2. Several basic tasks can **share a single stack**, which saves RAM on small ECUs.
3. There is **no event-handling overhead**.

### Example code (AUTOSAR OS API)
```c
/* Basic task: cyclic, activated every 10 ms by an Alarm */
TASK(Task_10ms)
{
    ReadSensors();
    RunControlAlgorithm();
    UpdateActuators();
    TerminateTask();                 /* must end, cannot wait */
}

/* Extended task: waits for events */
TASK(Task_CanHandler)
{
    EventMaskType ev;
    for (;;)
    {
        WaitEvent(EV_CAN_RX | EV_TIMEOUT);   /* RUNNING -> WAITING */
        GetEvent(Task_CanHandler, &ev);
        ClearEvent(ev);
        if (ev & EV_CAN_RX)  ProcessCanMessage();
        if (ev & EV_TIMEOUT) HandleTimeout();
    }
}

/* Category 2 ISR (allowed to call OS services) */
ISR(CAN_RX_ISR)
{
    SetEvent(Task_CanHandler, EV_CAN_RX);   /* WAITING -> READY */
}

/* Resource = mutual exclusion with priority ceiling */
TASK(Task_Logger)
{
    GetResource(RES_SPI);
    Spi_WriteFlash();
    ReleaseResource(RES_SPI);
    TerminateTask();
}
```

### Example static configuration (OIL file, OSEK style)
```
TASK Task_10ms {
    PRIORITY = 5;
    SCHEDULE = FULL;          /* preemptive */
    ACTIVATION = 1;
    AUTOSTART = FALSE;
    RESOURCE = RES_SPI;
};

TASK Task_CanHandler {
    PRIORITY = 8;
    SCHEDULE = FULL;
    ACTIVATION = 1;
    EVENT = EV_CAN_RX;        /* having EVENTs makes it an EXTENDED task */
    EVENT = EV_TIMEOUT;
};

ALARM Alarm_10ms {
    COUNTER = SystemTimer;
    ACTION = ACTIVATETASK { TASK = Task_10ms; };
    AUTOSTART = TRUE { ALARMTIME = 10; CYCLETIME = 10; };
};
```
In AUTOSAR the same data is written in **ARXML** and edited with tools such as **Vector DaVinci Configurator** or **EB tresos Studio**.

### Q: What are Category 1 and Category 2 ISRs?
| Category 1 ISR | Category 2 ISR |
|---|---|
| Does **not** call OS services | **Can** call OS services (`SetEvent`, `ActivateTask`) |
| Fastest, lowest overhead | Managed by the OS (slightly more overhead) |
| Like FreeRTOS ISRs above `MAX_SYSCALL` priority | Like FreeRTOS ISRs that use `...FromISR` |

### FreeRTOS to AUTOSAR OS mapping
| FreeRTOS | AUTOSAR OS |
|---|---|
| `xTaskCreate()` (run time) | Static config (OIL/ARXML), `ActivateTask()` |
| Blocked state | **Waiting** state (extended tasks only) |
| `xEventGroupWaitBits` / `SetBits` | `WaitEvent` / `SetEvent` |
| Mutex (priority inheritance) | **Resource** (`GetResource`, priority ceiling) |
| Software timer | **Alarm** / **Schedule Table** |
| `vTaskDelete(NULL)` | `TerminateTask()` |
| ISR using `FromISR` API | Category 2 ISR |

### Interview answer (short)
> "AUTOSAR OS **is** an RTOS. It is a standardized RTOS specification for automotive, based on OSEK/VDX (ISO 17356-3). Unlike FreeRTOS, it is statically configured, uses basic and extended tasks, Resources with priority ceiling, Alarms and Schedule Tables, and adds memory and timing protection (SC1–SC4) for ISO 26262. The AUTOSAR consortium writes the spec; vendors implement it. **Vector MICROSAR OS**, ETAS RTA-OS and EB tresos AutoCore OS are all implementations. A general RTOS like FreeRTOS does not follow a standard and has its own API."

---

*Practice tip: build each example on a Nucleo board one by one (Tasks, then Queue, then Semaphore, then Mutex, then Event Group, then the full project), and watch the UART output in a serial terminal at 115200 baud.*
