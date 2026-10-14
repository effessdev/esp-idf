# Hardware interrupts

!!! warning "Draft"
    This chapter is not completed yet. Please skip this and check out the next chapter.

Hardware interrupts are hardware signals that instantly pause the normal program flow to handle external events such as a button press, without wasting CPU cycles on continuous polling.

Let's update our `led_blink` project such that the LED can be toggled using a button press.

The connections are the same as before (LED & resistor). But we have to connect a jumper wire to GPIO23. Leave the other end of the jumper wire not connected to anything.

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

## The actual code

```c
#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "driver/gpio.h"

#define LED_PIN       GPIO_NUM_22
#define BUTTON_PIN    GPIO_NUM_23

static int led_state = 0;

// Define the ISR handler
static void IRAM_ATTR gpio_isr_handler(void* arg)
{
    led_state = !led_state;
    gpio_set_level(LED_PIN, led_state);
}

void app_main(void)
{
    gpio_reset_pin(LED_PIN);
    gpio_set_direction(LED_PIN, GPIO_MODE_OUTPUT);

    gpio_reset_pin(BUTTON_PIN);

    // Set button pin to input mode
    gpio_set_direction(BUTTON_PIN, GPIO_MODE_INPUT);
    
    // Enable pull-up resistor in button pin
    // (this will also give a HIGH voltage to the pin)
    gpio_pullup_en(BUTTON_PIN);

    // Set the interrupt type to GPIO_INTR_NEGEDGE, which
    // stands for "when the voltage go from HIGH to LOW"
    gpio_set_intr_type(BUTTON_PIN, GPIO_INTR_NEGEDGE);

    // Install interrupt service (required for properly
    // handing the interrupt over to the ISR handler)
    gpio_install_isr_service(0);

    // Attach the ISR handler to the button pin
    gpio_isr_handler_add(BUTTON_PIN, gpio_isr_handler, NULL);
}
```

## Test the code

Now, after building and flashing, touch the button pin to GND. If you did everything correctly, the LED will respond to the input.

Now that the code has been tested, you can commit and push your changes.

## Improving the code

The current code has 2 problems.

### Problem 1: Heavy work inside ISR handler

Currently, we are toggling the LED directly inside the ISR handler. This is not recommended because ISRs have the highest priority and will block other running tasks. So, ISR handlers should be kept minimal. This is not a problem for our code since there are no other tasks running. But it's a problem if we have.

So, we use a pattern where we:

- Create an event queue (a queue to store events)
- Make the ISR handler add an event item to the queue when an interrupt happens instead of taking action directly
- Create task that continuously consume the items in the queue and take necessary action (like toggling the LED)

### Problem 2: No debounce

You might notice that when you ground the button pin, the LED flickers or quickly reverts to its original state. This happens because of switch bounce (or contact chatter). When a mechanical connection is made, whether using a physical push button or a jumper wire, the metal contacts microscopically bounce against each other rather than making an instantaneous, clean connection. This rapid series of tiny disconnects creates multiple voltage drops that the microcontroller registers as separate button presses within milliseconds.

These extra button presses must be explicitly ignored by checking whether it's too close in time to the previous. This can be done inside the ISR handler.

### Improved code

```c
#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "freertos/queue.h"
#include "driver/gpio.h"
#include "esp_timer.h"

#define LED_PIN         GPIO_NUM_22
#define BUTTON_PIN      GPIO_NUM_23
#define DEBOUNCE_TIME_US 200000 // 200 milliseconds debounce window

static int led_state = 0;
static QueueHandle_t gpio_evt_queue = NULL;

// ISR handler with debounce and queue transmission
static void IRAM_ATTR gpio_isr_handler(void* arg)
{
    static int64_t last_interrupt_time = 0;
    int64_t current_time = esp_timer_get_time();

    // Discard interrupts coming too close together
    if (current_time - last_interrupt_time > DEBOUNCE_TIME_US) {
        last_interrupt_time = current_time;
        uint32_t gpio_num = (uint32_t) arg;

        // Add event to the queue
        xQueueSendFromISR(gpio_evt_queue, &gpio_num, NULL);
    }
}

// Background task to process button events safely
static void button_task(void* arg)
{
    uint32_t io_num;
    for (;;) {
        // Wait indefinitely until an event is received from the queue
        if (xQueueReceive(gpio_evt_queue, &io_num, portMAX_DELAY)) {
            led_state = !led_state;
            gpio_set_level(LED_PIN, led_state);
        }
    }
}

void app_main(void)
{
    // Configure LED pin
    gpio_reset_pin(LED_PIN);
    gpio_set_direction(LED_PIN, GPIO_MODE_OUTPUT);

    // Configure Button pin with internal pull-up
    gpio_reset_pin(BUTTON_PIN);
    gpio_set_direction(BUTTON_PIN, GPIO_MODE_INPUT);
    gpio_pullup_en(BUTTON_PIN);

    // Set interrupt type to falling edge (HIGH to LOW when pressed)
    gpio_set_intr_type(BUTTON_PIN, GPIO_INTR_NEGEDGE);

    // Create a queue to handle GPIO events from the ISR
    gpio_evt_queue = xQueueCreate(10, sizeof(uint32_t));

    // Create the background button handling task
    xTaskCreate(button_task, "button_task", 2048, NULL, 10, NULL);

    // Install GPIO ISR service
    gpio_install_isr_service(0);

    // Attach the ISR handler to the button pin and pass the pin number as argument
    gpio_isr_handler_add(BUTTON_PIN, gpio_isr_handler, (void*) BUTTON_PIN);
}
```

Now, our code is much better!

While you might say this is not a button (it's a wire, obviously), this is what exactly happens inside a physical button. You connect one side of the button to GPIO23, and the other to GND. Pressing it will close the circuit (like touching the wire to GND).

Commit the changes. In the next chapter, we are going to build the next project. Stay tuned!

## Test your knowledge

- What is an ISR handler?
- What is `IRAM_ATTR`?
- Explain how the LED toggles when we touch the button pin to GND (in the new improved code).
