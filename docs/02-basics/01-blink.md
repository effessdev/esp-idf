# Blinking an LED

Let's have a look at our ESP32 microcontroller:

<img width="785" alt="image" src="https://github.com/user-attachments/assets/a5708d32-d09b-4e50-b8bb-e7a262bb795e" />

As you can see, there are a lot of pins underneath it. These are called **GPIO pins** (General Purpose Input Output pins). These pins are the primary way in which the ESP32 control external hardware. Each pin can be turned HIGH voltage or LOW voltage.

What we are going to now is to make the GPIO22 pin HIGH and LOW continuously (like it's blinking), and connect an LED to that pin so it will turn ON and OFF. Don't ask me why I choose GPIO22.

## Create a new empty project

Create a new empty project using the `sample_project` template. We covered this here: [Your first project](/01-getting-started/02-first-project). But try to do it yourself without looking.

## Write the code

I am not going to give you the code. Here is what you have to do:

- Go to any free chatbot (ChatGPT, Gemini, Claude, etc.)
- Paste your `main.c` file.
- Ask it to write the code to blink GPIO22.
- Paste it back.

If you have an AI agent (like Claude Code or GitHub Copilot), you can ask it to directly edit `main.c`.

## Build & flash

I hope you remember how to do this. If not, check [Your First Project](/01-getting-started/02-first-project/).

## Connect your LED

Now, your GPIO22 pin must be turning ON and OFF repeatedly. But we can't see it yet. Time to set up the circuit:

- Connect the positive terminal of your LED to GPIO22.
- Connect the negative terminal of your LED to a 220 Ω to 330 Ω resistor (this is important; without a resistor, you might damage your microcontroller).
- Connect the other end of the resistor to the GND pin (ground pin) in the ESP32.

Now, you should see your LED blinking. Congrats!
