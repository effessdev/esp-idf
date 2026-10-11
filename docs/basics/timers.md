# Timers

This time, we are going to blink an LED using a **timer**.

## Key concepts

### Timers and callback functions

A timer allows us to schedule an action to happen (a function to be called) after a specified amount of time.

The function we give to the timer is called **callback function**. The timer will call it after the specified amount of time. Callback functions aren't some new kind of functions, they are called so because of their purpose.

### Types of timers

There are two common types of timers:

- **One-shot timer:** Invokes its callback once after the specified interval and then stops.
- **Periodic timer:** Invokes its callback repeatedly at the specified interval until it is manually stopped.

For example, assume we give `flash_led()` function as the timer callback. If we start a one-shot timer with a duration of 1 seconds, the LED flashes after 1 second. But if we use a periodic timer, the LED flashes after 1 second, 2 seconds, 3 seconds, and so on.

### The `esp_timer` API

ESP-IDF provides the `esp_timer.h` for creating software timers.

Here are some of it's important functions. You don't have to memorize these. The names are self-explanatory.

| Function | Purpose |
|---|---|
| `esp_timer_create()` | Create a timer |
| `esp_timer_start_once()` | Start a one-shot timer |
| `esp_timer_start_periodic()` | Start a periodic timer |
| `esp_timer_stop()` | Stop a running timer |
| `esp_timer_restart()` | Restart an active timer with a new timeout |
| `esp_timer_delete()` | Delete a timer |

We will use `esp_timer_create()` and `esp_timer_start_once()` in this project.

### Are timers reliable?

Software timers do not guarantee perfectly exact callback execution times. System load and callback scheduling can introduce delays.

## Set up the project

Create a new empty project using the `sample_project` template, as covered in [Your First Project](../getting-started/first-project.md). Name it `led_timer`.

## Setup the circuit

We will use GPIO22 for the LED and GPIO23 for the button, just like in our previous chapters. The circuit is also same as before:

- Connect the positive terminal of the LED to GPIO22.
- Connect the negative terminal to a 220–330 Ω resistor.
- Connect the other end of the resistor to GND.

## What we are going to do

1. Create a `toggle_led` function.
2. Create a timer with `toggle_led` as the callback.
3. Start the timer as a periodic timer with a period of 1 second.

The timer will call `toggle_led()` every second.

## Write the code

```c
#include "driver/gpio.h"
#include "esp_log.h"
#include "esp_timer.h"

#define LED_GPIO GPIO_NUM_22

static const char *TAG = "MAIN";
static bool led_state = false;

// The function to be called every time the timer fires
static void toggle_led(void *arg) {
  led_state = !led_state;
  gpio_set_level(LED_GPIO, led_state);
  ESP_LOGI(TAG, "LED toggled to: %d", led_state);
}

void app_main(void) {
  gpio_reset_pin(LED_GPIO);
  gpio_set_direction(LED_GPIO, GPIO_MODE_OUTPUT);

  // Set up the timer creation arguments
  const esp_timer_create_args_t timer_args = {
      .callback = &toggle_led,   // Called when the timer expires
      .name = "toggle_led_timer" // For debugging purposes only
  };

  // Handle to store the timer instance
  esp_timer_handle_t led_timer;

  // Create the timer instance using timer_args
  esp_timer_create(&timer_args, &led_timer);

  // Start the timer in periodic mode with a 1-second interval (1,000,000
  // microseconds)
  esp_timer_start_periodic(led_timer, 1000000);
}
```

## Improving the code

ESP-IDF provides a function-like macro `ESP_ERROR_CHECK()`. Here is how it works:

we wrap a function inside `ESP_ERROR_CHECK`, like this:

```
ESP_ERROR_CHECK(esp_timer_create())
```

If `esp_timer_create()` fails, it returns a non-zero return code (this behaviour is coded to the `esp_timer_create()` function itself, as well as many other ESP-IDF functions). If that happens, `ESP_ERROR_CHECK` function returns an error, ESP-IDF logs the error and aborts the program.

If we do not use it, and an error occurs, the firmware continues running. We will never know about the error, and it would be really painful to troubleshoot later.

Wrap `esp_timer_create()` and `esp_timer_start_periodic()` inside `ESP_ERROR_CHECK()` to improve the code.

## Final full code

```c
#include "driver/gpio.h"
#include "esp_log.h"
#include "esp_timer.h"

#define LED_GPIO GPIO_NUM_22

static const char *TAG = "MAIN";
static bool led_state = false;

// The function to be called every time the timer fires
static void toggle_led(void *arg) {
  led_state = !led_state;
  gpio_set_level(LED_GPIO, led_state);
  ESP_LOGI(TAG, "LED toggled to: %d", led_state);
}

void app_main(void) {
  gpio_reset_pin(LED_GPIO);
  gpio_set_direction(LED_GPIO, GPIO_MODE_OUTPUT);

  // Set up the timer creation arguments
  const esp_timer_create_args_t timer_args = {
      .callback = &toggle_led,   // Called when the timer expires
      .name = "toggle_led_timer" // For debugging purposes only
  };

  // Handle to store the timer instance
  esp_timer_handle_t led_timer;

  // Create the timer instance using timer_args
  ESP_ERROR_CHECK(esp_timer_create(&timer_args, &led_timer));

  // Start the timer in periodic mode with a 1-second interval (1,000,000
  // microseconds)
  ESP_ERROR_CHECK(esp_timer_start_periodic(led_timer, 1000000));
}
```

Now, you can test the project and ensure it's working.
