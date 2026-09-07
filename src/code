"""
Smart Cradle - Raspberry Pi Baby Monitoring System (Grove Edition)
--------------------------------------------------------------------
Follows the exact control flow from the project report flowchart:

    Start -> Initialize system -> Setup Wi-Fi (retry until connected)
    -> Read sensor values  <---------------------------------------+
    -> If moisture=1?  No  -> "dry"  -----------------+             |
                       Yes -> "wet"  -----------------+             |
    -> If sound=1?     No  -> (skip swing + motion, jump to temp) --+---+
                       Yes -> "swing the cradle"                    |   |
                              -> If motion detected? No -> "no motion" |
                                                      Yes -> "motion detected"
                              -> (falls through to temp check)      |   |
    -> If temp>30?     No  -> loop back to "Read sensor values" ----+   |
                       Yes -> "display" -> "thingspeak" -> stop         |
                                                                          v
                                                                    (temp check)

Hardware: Raspberry Pi + Grove Base Hat, with:
  - Grove Moisture Sensor (analog)
  - Grove Temperature & Humidity Sensor / DHT11 (digital)
  - Grove PIR Motion Sensor (digital)
  - Grove Sound Sensor / mic (analog) - used for cry detection
  - Grove 16x2 LCD White on Blue (I2C)
  - Servo Motor (PWM)

Install the Grove library first:
    curl -sL https://github.com/Seeed-Studio/grove.py/raw/master/install.sh | sudo bash -s -

Run `grove_gpio` on the Pi to confirm which BCM pin number matches each
physical slot you used, and update the pin numbers below accordingly.

IMPORTANT NOTE ON THE FLOWCHART'S "stop" NODE:
The flowchart's high-temperature branch ends at "stop" with no drawn
arrow back to "Read sensor values". Since this is meant to run as a
continuous monitor rather than a one-shot program, this implementation
treats "stop" as "done handling this event" rather than "end the
program": after logging to ThingSpeak, it loops back to reading sensor
values and keeps monitoring indefinitely.
"""

import time
import socket
import requests
import RPi.GPIO as GPIO

from grove.adc import ADC
from grove.gpio import GPIO as GroveGPIO
from grove.grove_temperature_humidity_sensor import DHT
from grove.display.jhd1802 import JHD1802

# ---------- ThingSpeak Settings ----------
THINGSPEAK_API_KEY = "YOUR_THINGSPEAK_WRITE_API_KEY"
THINGSPEAK_URL = "https://api.thingspeak.com/update"

# ---------- Pin / Slot Definitions ----------
# Analog slots (confirm with your Grove Base Hat's A0/A2/A4/A6 labeling)
SOUND_ADC_CHANNEL = 2        # cry/sound sensor
MOISTURE_ADC_CHANNEL = 0     # moisture sensor

# Digital slots (BCM pin numbers - confirm with `grove_gpio`)
DHT_PIN = 5
PIR_PIN = 16
SERVO_PIN = 12               # PWM-capable slot

# ---------- Thresholds ----------
# These are STARTING POINTS based on Seeed's own documentation/examples,
# NOT measured from your actual hardware. You must run
# sensor_calibration_test.py on your Pi and adjust these before your
# final demo/submission.
#
# Sound: Seeed's own example output for this exact library on a Pi showed
# baseline "quiet room" readings in the ~450-660 range, so the threshold
# needs to sit clearly above that or it will misfire on ambient noise.
# This is a rough starting point pending your own "quiet vs loud" test.
SOUND_THRESHOLD = 900

# Moisture: Seeed's official spec for this sensor lists 0-300 as dry,
# 300-700 as moist, and 700-950 as wet/submerged - but that's on the
# sensor's original 10-bit scale (0-1023). Your Grove Base Hat uses a
# 12-bit ADC (0-4095), roughly 4x that range, so this value is scaled
# up proportionally as a rough estimate. Confirm with your own
# dry-vs-wet test before relying on this.
MOISTURE_WET_THRESHOLD = 2800   # higher reading = wetter, per Seeed's spec

# Temperature: alert threshold in degrees Celsius, per the flowchart's
# "if temp>30" decision.
TEMP_ALERT_THRESHOLD_C = 30

# ---------- Setup ----------
adc = ADC()
dht_sensor = DHT('11', DHT_PIN)
pir = GroveGPIO(PIR_PIN, GroveGPIO.IN)
lcd = JHD1802()

GPIO.setmode(GPIO.BCM)
GPIO.setup(SERVO_PIN, GPIO.OUT)
servo = GPIO.PWM(SERVO_PIN, 50)  # 50Hz standard for servos
servo.start(0)


def is_wifi_connected():
    """Quick check for internet connectivity (used for the 'Setup Wifi' loop)."""
    try:
        socket.create_connection(("8.8.8.8", 53), timeout=3)
        return True
    except OSError:
        return False


def setup_wifi():
    """Loop until Wi-Fi/internet connectivity is established, per the flowchart."""
    print("Setting up Wi-Fi...")
    while not is_wifi_connected():
        print("Wi-Fi not established, retrying...")
        time.sleep(3)
    print("Wi-Fi connected.")


def swing_cradle():
    """Turn the servo motor on to rock the cradle."""
    for angle in (60, 120, 60):
        duty = 2 + (angle / 18)
        servo.ChangeDutyCycle(duty)
        time.sleep(0.5)
    servo.ChangeDutyCycle(0)  # stop sending signal to avoid jitter


def send_to_thingspeak(temperature, humidity, moisture_value):
    payload = {
        "api_key": THINGSPEAK_API_KEY,
        "field1": temperature,
        "field2": humidity,
        "field3": moisture_value,
    }
    try:
        response = requests.get(THINGSPEAK_URL, params=payload, timeout=5)
        print(f"ThingSpeak update sent. Response: {response.status_code}")
    except requests.exceptions.RequestException as e:
        print(f"Error sending to ThingSpeak: {e}")


def main():
    print("Initializing system...")
    lcd.setCursor(0, 0)
    lcd.write("Smart Cradle")
    lcd.setCursor(0, 1)
    lcd.write("Initializing...")
    time.sleep(2)

    setup_wifi()

    try:
        while True:
            # ---- Read sensor values ----
            sound_value = adc.read(SOUND_ADC_CHANNEL)
            humidity, temperature = dht_sensor.read()
            motion_detected = pir.read()
            moisture_value = adc.read(MOISTURE_ADC_CHANNEL)

            lcd.clear()

            # ---- Moisture check (display only - always continues) ----
            is_wet = moisture_value > MOISTURE_WET_THRESHOLD
            if not is_wet:
                lcd.setCursor(0, 0)
                lcd.write("Dry")
                print("Moisture: dry")
            else:
                lcd.setCursor(0, 0)
                lcd.write("Wet")
                print("Moisture: wet")

            # ---- Sound / cry check ----
            # No -> skip cradle swing AND motion check, go straight to temp check.
            # Yes -> swing the cradle, then also run the motion check below.
            cry_detected = sound_value > SOUND_THRESHOLD
            if cry_detected:
                print("Cry detected - swinging cradle")
                swing_cradle()

                # ---- Motion check (only reached when a cry was detected) ----
                if not motion_detected:
                    lcd.setCursor(0, 1)
                    lcd.write("No motion")
                    print("No motion detected")
                else:
                    lcd.setCursor(0, 1)
                    lcd.write("Movement!")
                    print("Motion detected")
            else:
                print("No cry detected - skipping cradle swing and motion check")

            # ---- Temperature check ----
            temp_valid = temperature is not None
            if not temp_valid:
                print("Temperature read error")
                time.sleep(2)
                continue  # back to Read sensor values

            temp_alert = temperature > TEMP_ALERT_THRESHOLD_C
            if not temp_alert:
                # Per flowchart: temp <= 30 -> loop straight back to
                # Read sensor values, skipping display and ThingSpeak.
                time.sleep(2)
                continue

            # ---- temp > 30: display, log to ThingSpeak, then stop ----
            lcd.setCursor(0, 0)
            lcd.write(f"ALERT {temperature:.0f}C")
            print(f"ALERT: High temperature detected - {temperature}C "
                  f"(threshold: {TEMP_ALERT_THRESHOLD_C}C)")

            send_to_thingspeak(temperature, humidity, moisture_value)

            print("High-temperature event logged - resuming monitoring.")
            time.sleep(2)
            continue

    except KeyboardInterrupt:
        print("Shutting down Smart Cradle system...")
    finally:
        servo.stop()
        lcd.clear()
        GPIO.cleanup()


if __name__ == "__main__":
    main()
