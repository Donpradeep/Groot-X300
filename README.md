#GrootX300

import time
from machine import Pin, ADC, I2C, PWM
from i2c_lcd import I2cLcd 
from time import sleep
import math
import random
import network
from umqtt.simple import MQTTClient

# --- 1. System State ---
mode = "LIGHT"
manual_mode = False 
last_mode = None
last_surprise_time = 0
surprise_cooldown = 5000  
surprise_duration = 2000  

# --- 2. MQTT Config ---
MQTT_BROKER = "10.235.145.114" 
CLIENT_ID = "EmoBot_Final_v2"
TOPIC_CONTROL = b"emobot/control"

# --- 3. Hardware Initialization ---
i2c = I2C(0, sda=Pin(21), scl=Pin(22), freq=400000)
lcd = None
try:
    lcd = I2cLcd(i2c, 0x27, 2, 16)
    except Exception as e:
        print("LCD Error:", e)

        # Sensors
        ldr = ADC(Pin(34))
        ir_sensor = Pin(14, Pin.IN)
        touch_sensor = Pin(12, Pin.IN)
        pir_sensor = Pin(23, Pin.IN) 

        # Motor & LEDs
        r_led = PWM(Pin(27)); g_led = PWM(Pin(18)); b_led = PWM(Pin(19))
        in1 = Pin(32, Pin.OUT); in2 = Pin(33, Pin.OUT); ena = PWM(Pin(4))
        in3 = Pin(25, Pin.OUT); in4 = Pin(13, Pin.OUT); enb = PWM(Pin(5))

        for p in [r_led, g_led, b_led, ena, enb]:
            p.freq(1000); p.duty(0)

            buzzer = PWM(Pin(2))
            buzzer.duty(0)

            ldr.atten(ADC.ATTN_11DB)

            # --- 4. Custom LCD Bitmaps ---
            eye_top = bytearray([0x00, 0x0E, 0x1F, 0x1F, 0x1F, 0x1F, 0x1F, 0x1F])
            eye_bot = bytearray([0x1F, 0x1F, 0x1F, 0x1F, 0x1F, 0x1F, 0x0E, 0x00])
            eye_close = bytearray([0x00, 0x00, 0x00, 0x00, 0x1F, 0x00, 0x00, 0x00])
            big_z = bytearray([0x1F, 0x02, 0x04, 0x08, 0x10, 0x1F, 0x00, 0x00])
            surp_top = bytearray([0x0E, 0x11, 0x11, 0x11, 0x11, 0x11, 0x11, 0x11])
            surp_bot = bytearray([0x11, 0x11, 0x11, 0x11, 0x11, 0x11, 0x11, 0x0E])

            if lcd:
                lcd.custom_char(0, eye_top); lcd.custom_char(1, eye_bot)
                    lcd.custom_char(2, eye_close); lcd.custom_char(3, big_z)
                        lcd.custom_char(5, surp_top); lcd.custom_char(6, surp_bot)

                        # --- 5. Helper Functions ---
                        def set_leds(r, g, b):
                            r_led.duty(r); g_led.duty(g); b_led.duty(b)

                            def drive(dir, speed=1023):
                                if dir == "stop":
                                        ena.duty(0); enb.duty(0)
                                                in1.value(0); in2.value(0); in3.value(0); in4.value(0)
                                                        return
                                                            ena.duty(speed); enb.duty(speed)
                                                                if dir == "forward": in1.value(0); in2.value(1); in3.value(1); in4.value(0)
                                                                    elif dir == "backward": in1.value(1); in2.value(0); in3.value(0); in4.value(1)
                                                                        elif dir == "left": in1.value(1); in2.value(0); in3.value(1); in4.value(0)
                                                                            elif dir == "right": in1.value(0); in2.value(1); in3.value(0); in4.value(1)

                                                                            def on_message(topic, msg):
                                                                                global manual_mode
                                                                                    cmd = msg.decode().strip().lower()
                                                                                        print("MQTT CMD DETECTED:", cmd)
                                                                                            manual_mode = (cmd != "stop")
                                                                                                drive(cmd)

                                                                                                # --- 6. MQTT Connection ---
                                                                                                wlan = network.WLAN(network.STA_IF)
                                                                                                mqtt_client = None
                                                                                                if wlan.isconnected():
                                                                                                    try:
                                                                                                            mqtt_client = MQTTClient(CLIENT_ID, MQTT_BROKER, keepalive=60)
                                                                                                                    mqtt_client.set_callback(on_message)
                                                                                                                            mqtt_client.connect()
                                                                                                                                    mqtt_client.subscribe(TOPIC_CONTROL)
                                                                                                                                        except:
                                                                                                                                                print("MQTT Connection Failed")

                                                                                                                                                # --- 7. Fast Startup Sequence ---
                                                                                                                                                if lcd:
                                                                                                                                                    tests = [("IR", ir_sensor.value), ("TOUCH", touch_sensor.value), ("PIR", pir_sensor.value), ("LDR", ldr.read)]
                                                                                                                                                        for name, func in tests:
                                                                                                                                                                lcd.clear(); lcd.putstr(name); lcd.move_to(0, 1); lcd.putstr("VAL: " + str(func()))
                                                                                                                                                                        sleep(0.2)
                                                                                                                                                                            lcd.clear(); set_leds(0, 1023, 0); sleep(0.5)

                                                                                                                                                                            print("--- EMO BOT ACTIVE (GAS REMOVED) ---")

                                                                                                                                                                            # --- 8. Main Loop ---
                                                                                                                                                                            while True:
                                                                                                                                                                                now = time.ticks_ms()
                                                                                                                                                                                    if mqtt_client:
                                                                                                                                                                                            try: mqtt_client.check_msg()
                                                                                                                                                                                                    except: pass

                                                                                                                                                                                                        # Read Inputs
                                                                                                                                                                                                            ir, touch, pir, light = ir_sensor.value(), touch_sensor.value(), pir_sensor.value(), ldr.read()

                                                                                                                                                                                                                # Mode Selection
                                                                                                                                                                                                                    if manual_mode: mode = "LIGHT" 
                                                                                                                                                                                                                        elif ir == 0: mode = "OBJECT"
                                                                                                                                                                                                                            elif touch == 1: mode = "SHY"
                                                                                                                                                                                                                                elif light > 3000: mode = "DARK"
                                                                                                                                                                                                                                    elif pir == 1 and time.ticks_diff(now, last_surprise_time) > surprise_cooldown:
                                                                                                                                                                                                                                            mode = "SURPRISED"; last_surprise_time = now
                                                                                                                                                                                                                                                elif last_mode == "SURPRISED" and time.ticks_diff(now, last_surprise_time) < surprise_duration:
                                                                                                                                                                                                                                                        mode = "SURPRISED"
                                                                                                                                                                                                                                                            else: mode = "LIGHT"

                                                                                                                                                                                                                                                                # State Transitions
                                                                                                                                                                                                                                                                    if mode != last_mode:
                                                                                                                                                                                                                                                                            if lcd: lcd.clear()
                                                                                                                                                                                                                                                                                    buzzer.duty(0)
                                                                                                                                                                                                                                                                                            
                                                                                                                                                                                                                                                                                                    if mode == "OBJECT":
                                                                                                                                                                                                                                                                                                                set_leds(1023, 0, 0); drive("stop")
                                                                                                                                                                                                                                                                                                                            if lcd: lcd.move_to(0, 1); lcd.putstr("OBJECT DETECTED")
                                                                                                                                                                                                                                                                                                                                    elif mode == "SURPRISED":
                                                                                                                                                                                                                                                                                                                                                set_leds(1023, 1023, 1023)
                                                                                                                                                                                                                                                                                                                                                            if lcd:
                                                                                                                                                                                                                                                                                                                                                                            lcd.move_to(6, 0); lcd.putchar(chr(5)); lcd.move_to(9, 0); lcd.putchar(chr(5))
                                                                                                                                                                                                                                                                                                                                                                                            lcd.move_to(6, 1); lcd.putchar(chr(6)); lcd.move_to(9, 1); lcd.putchar(chr(6))
                                                                                                                                                                                                                                                                                                                                                                                                    elif mode == "SHY":
                                                                                                                                                                                                                                                                                                                                                                                                                if lcd:
                                                                                                                                                                                                                                                                                                                                                                                                                                lcd.move_to(1, 0); lcd.putchar(chr(0)); lcd.move_to(14, 0); lcd.putchar(chr(0))
                                                                                                                                                                                                                                                                                                                                                                                                                                                lcd.move_to(6, 1); lcd.putstr("////")
                                                                                                                                                                                                                                                                                                                                                                                                                                                        elif mode == "DARK":
                                                                                                                                                                                                                                                                                                                                                                                                                                                                    set_leds(0, 0, 0)
                                                                                                                                                                                                                                                                                                                                                                                                                                                                                if lcd:
                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                for p in [4, 7, 10]: lcd.move_to(p, 0); lcd.putchar(chr(3))
                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        elif mode == "LIGHT":
                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    set_leds(0, 1023, 0)
                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                if lcd:
                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                lcd.move_to(6, 0); lcd.putchar(chr(0)); lcd.move_to(9, 0); lcd.putchar(chr(0))
                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                lcd.move_to(6, 1); lcd.putchar(chr(1)); lcd.move_to(9, 1); lcd.putchar(chr(1))
                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        last_mode = mode

                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            # Continuous Effects
                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                if mode == "SHY":
                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        pulse = int((math.sin(time.ticks_ms() / 1000 * 2 * math.pi) + 1) * 511)
                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                set_leds(0, 0, pulse)
                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        if time.ticks_ms() % 1000 < 50:
                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    buzzer.freq(2000); buzzer.duty(100)
                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            else: buzzer.duty(0)
                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                elif mode == "OBJECT":
                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        buzzer.freq(1200); buzzer.duty(512); sleep(0.05); buzzer.duty(0)
                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            elif mode == "LIGHT" and random.random() > 0.98 and lcd:
                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    lcd.move_to(6, 0); lcd.putchar(chr(2)); lcd.move_to(9, 0); lcd.putchar(chr(2))
                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            sleep(0.05)
                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    lcd.move_to(6, 0); lcd.putchar(chr(0)); lcd.move_to(9, 0); lcd.putchar(chr(0))
                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        else:
                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                if mode not in ["SHY", "OBJECT"]: buzzer.duty(0)

                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    sleep(0.02)
                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    
