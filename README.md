# ClimaGuard

**An ESP32 smart climate controller for poultry houses and greenhouses, with automatic fan control, local alarms and Telegram alerts.**

ClimaGuard keeps a growing space at a safe temperature. It reads temperature and humidity with a DHT22 sensor, switches a ventilation fan through a relay when it gets too warm, and raises an alarm (buzzer, LED and a Telegram message to your phone) if conditions become dangerous or the sensor stops working. A 16x2 LCD shows live readings, and the target temperature can be changed with two buttons or from your phone.

Built for low-cost, real-world farm use: it keeps running by itself when Wi-Fi drops, remembers its settings through power cuts, and fails safe by running the fan if the sensor fails.

## Features

- **Automatic fan control** with hysteresis, so the relay doesn't click on and off around the threshold
- **Two-level protection**: the fan starts at the setpoint, and the alarm triggers at a higher danger limit
- **Local alarm**: buzzer and LED beep together while the alarm is active and stop as soon as it clears
- **Telegram alerts**: instant message when the alarm turns on, a reminder every 5 minutes while it lasts, and a message when it clears
- **Remote commands** from your phone: check status, change the setpoint, send a test alert
- **Adjustable setpoint** with UP and DOWN buttons, saved to flash memory so it survives power loss
- **Sensor failure detection**: if the DHT22 stops responding, the alarm sounds and the fan is forced on
- **Mute button**: press both buttons together to silence the buzzer for the current alarm
- **Non-blocking design**: Telegram runs in its own task, so network problems never freeze the fan, buzzer or display

## How it behaves (default setpoint 30 °C)

| Temperature | Fan | Buzzer, LED and Telegram |
|---|---|---|
| Below 28 °C | Off | Silent |
| 28 to 30 °C | Keeps its last state | Silent |
| 30 to 35 °C | On | Silent |
| 35 °C or higher | On | **Alarm: beeping, LED blinking, Telegram alert** |
| Back to 34 °C or lower | On until 28 °C | Alarm clears, "Back to normal" sent |
| Sensor not responding | Forced on | **Alarm** |

All thresholds follow the setpoint, so raising it to 32 °C moves the fan-on point to 32 °C and the alarm to 37 °C. Humidity is displayed and reported but does not trigger anything yet.

## Hardware

| Part | Quantity | Notes |
|---|---|---|
| ESP32 DevKit | 1 | 2.4 GHz Wi-Fi required |
| DHT22 (AM2302) temperature and humidity sensor | 1 | Add a 10 kΩ pull-up between DATA and 3.3 V if it is the bare 4-pin version |
| 16x2 I2C LCD (address 0x27) | 1 | |
| 1-channel 5 V relay module | 1 | Switches the fan |
| DC fan | 1 | 5 V from VIN for small fans, or a separate supply for 12 V fans |
| Active buzzer | 1 | |
| LED and 220 Ω resistor | 1 each | Alarm indicator |
| Push buttons | 2 | Setpoint up and down |
| 1N4007 diode | 1 | Across the fan if it is a bare motor |
| Breadboard and jumper wires | | |

## Wiring

| Part | ESP32 pin |
|---|---|
| DHT22 DATA | GPIO 4 |
| Relay IN | GPIO 26 |
| Buzzer (+) | GPIO 25 |
| Alarm LED (through 220 Ω) | GPIO 33 |
| LCD SDA / SCL | GPIO 21 / GPIO 22 |
| Button UP | GPIO 18 (other leg to GND) |
| Button DOWN | GPIO 19 (other leg to GND) |

**Power connections**

- DHT22 VCC to 3.3 V, GND to GND
- LCD VCC to 5 V (VIN), GND to GND
- Relay VCC to VIN, GND to GND
- Buzzer (−) and LED cathode to GND

**Fan circuit (switched by the relay)**

1. Fan supply **+** goes to relay **COM**
2. Relay **NO** goes to fan **+**
3. Fan **−** goes back to supply **−** (or to ESP32 GND when powered from VIN)

Use **NO** (normally open) so the fan stays off if the controller loses power. Leave **NC** unconnected.

## Software setup

### 1. Install the libraries

In the Arduino IDE Library Manager, install:

- DHT sensor library (Adafruit) and Adafruit Unified Sensor
- LiquidCrystal I2C
- UniversalTelegramBot (Brian Lough)
- ArduinoJson, **version 6.x** (not 7.x)

Select the board **ESP32 Dev Module**.

### 2. Create your Telegram bot

1. Open Telegram and message **@BotFather**, send `/newbot` and follow the steps. Copy the token it gives you (format: `123456789:AAH...`).
2. Open your new bot and press **Start**. The bot cannot message you until you do.
3. Message **@userinfobot** to get your personal numeric chat ID. This is different from the number inside your token.

### 3. Configure the sketch

Edit the settings block at the top of the sketch:

```cpp
const char* WIFI_SSID = "YOUR_WIFI_NAME";
const char* WIFI_PASS = "YOUR_WIFI_PASSWORD";
const char* BOT_TOKEN = "YOUR_BOT_TOKEN";
const char* CHAT_ID   = "YOUR_CHAT_ID";
```

> **Never commit your real Wi-Fi password or bot token to GitHub.** Keep the placeholders in the repository. If a token is ever exposed, send `/revoke` to @BotFather to get a new one.

Other useful settings:

| Setting | Default | Meaning |
|---|---|---|
| `DEFAULT_SETPOINT` | 30.0 | Fan turns on at or above this temperature (°C) |
| `HYSTERESIS` | 2.0 | Fan turns off this many degrees below the setpoint |
| `ALARM_MARGIN` | 5.0 | Alarm triggers this many degrees above the setpoint |
| `ALARM_CLEAR_BAND` | 1.0 | Alarm clears this many degrees below its trigger point |
| `RELAY_ACTIVE_LOW` | false | Set to true if your relay switches on when its input is LOW |
| `BUZZER_ACTIVE_LOW` | false | Set to true only for buzzer modules that sound when their input is LOW |
| `FAN_ON_SENSOR_FAIL` | true | Run the fan if the sensor stops responding |

### 4. Upload

Upload the sketch and open the Serial Monitor at **115200 baud**. You should hear one short beep at startup, see readings every 2 seconds, and receive a "Controller online" message in Telegram.

## Telegram commands

| Command | What it does |
|---|---|
| `/status` | Sends current temperature, humidity, setpoint, fan state and alarm state |
| `/set 32` | Changes the setpoint (allowed range 20 to 40 °C) |
| `/test` | Sends a test alert to confirm notifications work |

Only messages from your configured chat ID are accepted.

## Controls

| Action | Result |
|---|---|
| Press UP | Raises the setpoint by 0.5 °C (hold to repeat) |
| Press DOWN | Lowers the setpoint by 0.5 °C (hold to repeat) |
| Press UP and DOWN together | Mutes the buzzer for the current alarm |

## Testing

1. **Relay check**: with the fan disconnected, set the setpoint just above room temperature, then lower it below room temperature. The relay should click on. If it behaves the opposite way, flip `RELAY_ACTIVE_LOW`.
2. **Alarm check**: temporarily set `ALARM_MARGIN` to 1.0 and lower the setpoint until room temperature is 1 °C above it. The buzzer and LED should start and a Telegram alert should arrive. Raise the setpoint and everything should stop. Restore `ALARM_MARGIN` to 5.0 afterwards.
3. **Sensor failure check**: unplug the DHT22 data wire. After about 6 seconds the sensor alarm should trigger and the fan should turn on.

## Troubleshooting

| Problem | Likely cause and fix |
|---|---|
| Buzzer sounds all the time | Flip `BUZZER_ACTIVE_LOW` |
| Fan runs when it should be off | Flip `RELAY_ACTIVE_LOW`, and check the fan is wired to NO, not NC |
| No Telegram messages | Check the full token (with the number and colon), use your personal chat ID, press Start in the bot, and make sure Wi-Fi is 2.4 GHz |
| LCD shows nothing | Adjust the contrast screw on the back, and check the I2C address (0x27 or 0x3F) |
| Temperature shows `--` or sensor alarm | Check DHT22 wiring, the 10 kΩ pull-up and the 3.3 V supply |
| Random reboots | Use a 5 V supply rated 1 A or more, and avoid powering a large fan from the ESP32 |

## Safety notes

- Never power a motor directly from a GPIO pin. Always switch it through the relay.
- Add a flyback diode across any bare DC motor.
- Do not switch mains-voltage loads without a properly rated relay, a fuse and an enclosure.
- This is a hobby and learning project. It should not be the only safeguard for valuable livestock or crops. Keep an independent backup, such as a manual thermometer and regular checks.

## Roadmap

- [ ] Heater control for cold conditions (two-way thermostat)
- [ ] Humidity-based ventilation and misting
- [ ] Day-by-day brooder temperature schedule
- [ ] Ammonia and air-quality sensor
- [ ] Data logging and dashboard
- [ ] Battery backup for power cuts

## Contributing

Issues and pull requests are welcome. If you build your own version or adapt it to a different crop or animal, feel free to share it.

## License

Released under the MIT License. See the `LICENSE` file for details.
