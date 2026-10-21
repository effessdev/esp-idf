# Hardware interrupts

In this chapter, we are going to write a program which allows us to toggle an LED using a button (we don't need a real button, we'll use a workaround!).

The connections are the same as before (LED & resistor). But we have to connect a jumper wire to GPIO23. Leave the other end of the jumper wire not connected to anything. Touching the other end of this jumper wire to the `GND` pin in the ESP32 will toggle the LED (equivalent to what happens internally when we press a real button).

## Key concepts

### ISR handler

ISR handler stands for **Interrupt Service Routine (ISR) handler**. It's a function that we configure to be run immediately when a hardware event occurs (in this case, the button press).

### `IRAM_ATTR`

While creating ISRs, we prefix `IRAM_ATTR` before the function name:

```c
static void IRAM_ATTR my_isr_handler(void* arg)
{
    // code to toggle the LED
}
```

`IRAM_ATTR` is a preprocessor macro that makes this function load directly into internal instruction RAM (IRAM), instead of running them from the external flash memory. This makes the loading much faster.

!!! question "But aren't functions already stored in RAM?"
    On most standard computers, all programs and functions are fully loaded in to the RAM before they run, **but not in ESP32**.

    ESP32 (and other MCUs) have very limited RAM (less than 1 MB). So, functions remain in the external Flash memory chip and are read on the fly through a small cache.

## How we are going to do this (conceptually)

- We set the button GPIO pin to input mode, which will allow the ESP32 to read the voltage in the button pin.
- We give a constant HIGH voltage to the button pin.
- We put a resistor (also called pull-up resistor) between the voltage source and the pin, allowing us to connect the pin to GND (zero voltage) without causing high current.
- We configure the button pin to trigger a signal (interrupt) when it goes from HIGH to LOW.
- Now, when we touch the other end of the wire to GND, the voltage of the button pin will drop to LOW, triggering the interrupt, and causing the ISR handler to execute.

## The code

```c
#include "driver/gpio.h"
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"

#define BUTTON_PIN GPIO_NUM_23
#define LED_PIN    GPIO_NUM_22

static int led_state = 0;

static void IRAM_ATTR gpio_isr_handler(void* arg) {
    led_state = !led_state;
    gpio_set_level(LED_PIN, led_state);
}

void app_main(void) {
    // Configure LED pin
    gpio_reset_pin(LED_PIN);
    gpio_set_direction(LED_PIN, GPIO_MODE_OUTPUT);

    // Configure Button pin with internal pull-up and falling edge interrupt
    gpio_reset_pin(BUTTON_PIN);
    gpio_set_direction(BUTTON_PIN, GPIO_MODE_INPUT);
    gpio_set_pull_mode(BUTTON_PIN, GPIO_PULLUP_ONLY);
    gpio_set_intr_type(BUTTON_PIN, GPIO_INTR_NEGEDGE);

    // Install GPIO ISR service and add the handler
    gpio_install_isr_service(0);
    gpio_isr_handler_add(BUTTON_PIN, gpio_isr_handler, NULL);

    while (1) {
        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}
```

## Test the code

Now, after building and flashing, touch the button pin to GND. If you did everything correctly, the LED will respond to the input.

Now that the code has been tested, you can commit and push your changes.

## Improving the code

Currently, we are toggling the LED directly inside the ISR handler. This isn't recommended because ISR handlers have very high priority and they will block other running tasks. ISR handlers are kept minimal.

So, we do this:

- ISR handler **doesn't toggle the LED directly**, it adds an item to a queue
- We run a task that constantly waits for items in the queue to process it
- When a new item get added to the queue, the task consumes it, and toggles the LED

Here is the code that uses this approach:

```c
#include "driver/gpio.h"
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "freertos/queue.h"

#define BUTTON_PIN GPIO_NUM_23
#define LED_PIN    GPIO_NUM_22

static QueueHandle_t gpio_evt_queue = NULL;

static void IRAM_ATTR gpio_isr_handler(void* arg) {
    uint32_t gpio_num = (uint32_t) arg;
    xQueueSendFromISR(gpio_evt_queue, &gpio_num, NULL);
}

static void gpio_task(void* arg) {
    uint32_t io_num;
    int led_state = 0;
    
    while (1) {
        if (xQueueReceive(gpio_evt_queue, &io_num, portMAX_DELAY)) {
            led_state = !led_state;
            gpio_set_level(LED_PIN, led_state);
        }
    }
}

void app_main(void) {
    // Configure LED pin
    gpio_reset_pin(LED_PIN);
    gpio_set_direction(LED_PIN, GPIO_MODE_OUTPUT);

    // Configure Button pin with internal pull-up and falling edge interrupt
    gpio_reset_pin(BUTTON_PIN);
    gpio_set_direction(BUTTON_PIN, GPIO_MODE_INPUT);
    gpio_set_pull_mode(BUTTON_PIN, GPIO_PULLUP_ONLY);
    gpio_set_intr_type(BUTTON_PIN, GPIO_INTR_NEGEDGE);

    // Create a queue to handle GPIO events from ISR
    gpio_evt_queue = xQueueCreate(10, sizeof(uint32_t));

    // Start the task to process queue items
    xTaskCreate(gpio_task, "gpio_task", 2048, NULL, 10, NULL);

    // Install GPIO ISR service and add the handler
    gpio_install_isr_service(0);
    gpio_isr_handler_add(BUTTON_PIN, gpio_isr_handler, (void*) BUTTON_PIN);
}
```

## Adding debouncing

You might have noticed that the LED flickers when we press the button. This is because the press is almost never *clean*. We will get multiple hardware interrupts for a single button press, extremely close to each other.

We can ignore those *dirty* interrupts by checking whether they are too close to the last interrupt.

```c
#include "driver/gpio.h"
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "freertos/queue.h"

#define BUTTON_PIN GPIO_NUM_23
#define LED_PIN    GPIO_NUM_22
#define DEBOUNCE_TIME_MS 50

static QueueHandle_t gpio_evt_queue = NULL;

static void IRAM_ATTR gpio_isr_handler(void* arg) {
    uint32_t gpio_num = (uint32_t) arg;
    xQueueSendFromISR(gpio_evt_queue, &gpio_num, NULL);
}

static void gpio_task(void* arg) {
    uint32_t io_num;
    int led_state = 0;
    TickType_t last_tick = 0;
    const TickType_t debounce_ticks = pdMS_TO_TICKS(DEBOUNCE_TIME_MS);
    
    while (1) {
        if (xQueueReceive(gpio_evt_queue, &io_num, portMAX_DELAY)) {
            TickType_t current_tick = xTaskGetTickCount();
            
            // Simple software debounce by checking time elapsed since last valid press
            if ((current_tick - last_tick) > debounce_ticks) {
                last_tick = current_tick;
                led_state = !led_state;
                gpio_set_level(LED_PIN, led_state);
            }
        }
    }
}

void app_main(void) {
    // Configure LED pin
    gpio_reset_pin(LED_PIN);
    gpio_set_direction(LED_PIN, GPIO_MODE_OUTPUT);

    // Configure Button pin with internal pull-up and falling edge interrupt
    gpio_reset_pin(BUTTON_PIN);
    gpio_set_direction(BUTTON_PIN, GPIO_MODE_INPUT);
    gpio_set_pull_mode(BUTTON_PIN, GPIO_PULLUP_ONLY);
    gpio_set_intr_type(BUTTON_PIN, GPIO_INTR_NEGEDGE);

    // Create a queue to handle GPIO events from ISR
    gpio_evt_queue = xQueueCreate(10, sizeof(uint32_t));

    // Start the task to process queue items
    xTaskCreate(gpio_task, "gpio_task", 2048, NULL, 10, NULL);

    // Install GPIO ISR service and add the handler
    gpio_install_isr_service(0);
    gpio_isr_handler_add(BUTTON_PIN, gpio_isr_handler, (void*) BUTTON_PIN);
}
```

## Test your knowledge

- What is an ISR handler?
- What is `IRAM_ATTR`?
