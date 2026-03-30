import RPi.GPIO as GPIO
import time

# ========== GPIO Setup ==========
GPIO.setmode(GPIO.BCM)
GPIO.setwarnings(False)

# Pin Definitions
GAS_PIN = 17
TRIG = 23
ECHO = 24
LED = 18
SERVO = 25

# Setup GPIO Modes
GPIO.setup(GAS_PIN, GPIO.IN)
GPIO.setup(TRIG, GPIO.OUT)
GPIO.setup(ECHO, GPIO.IN)
GPIO.setup(LED, GPIO.OUT)
GPIO.setup(SERVO, GPIO.OUT)

# Setup Servo PWM
servo_pwm = GPIO.PWM(SERVO, 50)  # 50 Hz PWM
servo_pwm.start(2.5)  # Start at 0 degrees

# ========== Function: Get Distance ==========
def get_distance():
    GPIO.output(TRIG, False)
    time.sleep(0.01)

    GPIO.output(TRIG, True)
    time.sleep(0.00001)
    GPIO.output(TRIG, False)

    start_time = time.time()
    timeout = start_time + 0.02
    while GPIO.input(ECHO) == 0:
        start_time = time.time()
        if time.time() > timeout:
            return None

    stop_time = time.time()
    timeout = stop_time + 0.02
    while GPIO.input(ECHO) == 1:
        stop_time = time.time()
        if time.time() > timeout:
            return None

    elapsed = stop_time - start_time
    distance = (elapsed * 34300) / 2
    return round(distance, 2)

# ========== Function: Blink LED ==========
def blink_led(times=3, delay=0.2):
    for _ in range(times):
        GPIO.output(LED, GPIO.HIGH)
        time.sleep(delay)
        GPIO.output(LED, GPIO.LOW)
        time.sleep(delay)

# ========== Function: Rotate Servo ==========
def rotate_servo_sequence():
    angles = [90, 180,0]
    for angle in angles:
        duty = angle / 18 + 2
        print(f"🔁 Rotating servo to {angle}°")
        servo_pwm.ChangeDutyCycle(duty)
        time.sleep(1)
    servo_pwm.ChangeDutyCycle(0)

# ========== MAIN LOOP ==========
try:
    while True:
        # --- GAS SENSOR (independent action) ---
        if GPIO.input(GAS_PIN) == 0:  # Gas detected
            print("⚠ GAS DETECTED!")
            blink_led()
            rotate_servo_sequence()

        # --- ULTRASONIC DISTANCE (independent action) ---
        distance = get_distance()
        if distance is None:
            print("❌ Distance measurement failed.")
        elif distance > 400:
            print("⚠ Distance out of range (>400 cm)")
        else:
            print(f"📏 Distance: {distance} cm")

        time.sleep(1)

except KeyboardInterrupt:
    print("🛑 Stopping program. Cleaning up GPIO...")
    servo_pwm.stop()
    GPIO.cleanup()
