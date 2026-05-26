# ════════════════════════════════════════════════════════════════
#  LINE FOLLOWER — Maker Pi RP2040
#  PID Controller με μη-γραμμικό κέρδος για κλειστές στροφές
# ════════════════════════════════════════════════════════════════
#
#  ΥΛΙΚΟ:
#  ├─ Αριστερό μοτέρ  : GP8  (M1A) · GP9  (M1B)   ← dual-PWM H-bridge
#  ├─ Δεξί μοτέρ      : GP10 (M2A) · GP11 (M2B)   ← dual-PWM H-bridge
#  ├─ Sensor αριστερά : GP26 (A0)
#  ├─ Sensor κέντρο   : GP27 (A1)
#  ├─ Sensor δεξιά    : GP28 (A2)
#  ├─ Κουμπί START    : GP20
#  └─ Κουμπί STOP     : GP21
#
#  ΧΡΗΣΗ:
#  · Πάτα GP20 → εκκίνηση
#  · Πάτα GP21 → σταμάτημα
# ════════════════════════════════════════════════════════════════

import board
import pwmio
import digitalio
import analogio
import time

# ════════════════════════════════════════════════════════════════
# ΚΟΥΜΠΙΑ
# ════════════════════════════════════════════════════════════════
btn_start = digitalio.DigitalInOut(board.GP20)
btn_stop  = digitalio.DigitalInOut(board.GP21)
btn_start.direction = digitalio.Direction.INPUT
btn_stop.direction  = digitalio.Direction.INPUT
btn_start.pull = digitalio.Pull.UP   # active LOW — πάτημα = False
btn_stop.pull  = digitalio.Pull.UP

# ════════════════════════════════════════════════════════════════
# ΜΟΤΕΡ — dual-PWM H-bridge (όπως ζητάει το Maker Pi RP2040)
# ════════════════════════════════════════════════════════════════
# Κάθε μοτέρ ελέγχεται από ΔΥΟ pins, και τα δύο ως PWM:
#   · εμπρός : PWM στο A,  0 στο B
#   · πίσω   : 0 στο A,    PWM στο B
left_a  = pwmio.PWMOut(board.GP8,  frequency=10000, duty_cycle=0)
left_b  = pwmio.PWMOut(board.GP9,  frequency=10000, duty_cycle=0)
right_a = pwmio.PWMOut(board.GP10, frequency=10000, duty_cycle=0)
right_b = pwmio.PWMOut(board.GP11, frequency=10000, duty_cycle=0)

def _drive(pin_a, pin_b, spd):
    """Οδηγεί ένα μοτέρ: spd -100..+100. Θετικό=εμπρός, αρνητικό=πίσω."""
    duty = int(min(abs(spd), 100) / 100 * 65535)
    if spd >= 0:
        pin_a.duty_cycle = duty
        pin_b.duty_cycle = 0
    else:
        pin_a.duty_cycle = 0
        pin_b.duty_cycle = duty

def set_motors(left_spd, right_spd):
    """Έλεγχος μοτέρ: -100 έως +100. Αρνητικό = πίσω."""
    _drive(left_a,  left_b,  left_spd)
    _drive(right_a, right_b, right_spd)

def stop():
    """Σταμάτημα μοτέρ (coast)."""
    left_a.duty_cycle  = 0
    left_b.duty_cycle  = 0
    right_a.duty_cycle = 0
    right_b.duty_cycle = 0

# ════════════════════════════════════════════════════════════════
# ΑΙΣΘΗΤΗΡΕΣ
# ════════════════════════════════════════════════════════════════
sensor_left   = analogio.AnalogIn(board.A0)   # GP26
sensor_center = analogio.AnalogIn(board.A1)   # GP27
sensor_right  = analogio.AnalogIn(board.A2)   # GP28

# ── Τιμές βαθμονόμησης ──────────────────────────────────────────
CAL_MIN = [3808, 16660, 3600]
CAL_MAX = [61230, 63327, 62127]

def read_normalized():
    """Επιστρέφει (L, C, R) κανονικοποιημένα 0.0 (λευκό) – 1.0 (γραμμή)."""
    raw = (sensor_left.value, sensor_center.value, sensor_right.value)
    out = []
    for i, v in enumerate(raw):
        span = CAL_MAX[i] - CAL_MIN[i]
        out.append(max(0.0, min(1.0, (v - CAL_MIN[i]) / span)))
    return tuple(out)

def line_position():
    """
    Θέση γραμμής με weighted average.
    -1.0 = τελείως αριστερά · 0.0 = κέντρο · +1.0 = τελείως δεξιά
    None  = γραμμή δεν εντοπίστηκε
    """
    l, c, r = read_normalized()
    total = l + c + r
    if total < 0.15:
        return None
    return (-1.0 * l + 0.0 * c + 1.0 * r) / total

def all_black():
    """True αν και οι 3 αισθητήρες βλέπουν μαύρο (γραμμή τερματισμού)."""
    l, c, r = read_normalized()
    return l > STOP_THRESHOLD and c > STOP_THRESHOLD and r > STOP_THRESHOLD

# ════════════════════════════════════════════════════════════════
# ΠΑΡΑΜΕΤΡΟΙ PID  ← ρύθμισε εδώ αν χρειαστεί
# ════════════════════════════════════════════════════════════════
BASE_SPEED = 80     # Ταχύτητα ευθείας (0–100)
TURN_SPEED = 16     # Ταχύτητα μέσα στη στροφή — χαμήλωσέ το για πιο κλειστές
MAX_SPEED  = 130     # Μέγιστη ταχύτητα μοτέρ
MIN_SPEED  = -9    # Επιτρέπει pivot: εσωτερικός τροχός γυρίζει ανάποδα

Kp = 18.0           # Proportional — άμεση διόρθωση
Ki = 0.22            # Integral     — διόρθωση σταθερού σφάλματος
Kd = 13.0           # Derivative   — απόσβεση ταλαντώσεων

# ── Μη-γραμμικό κέρδος ──────────────────────────────────────────
# Ήπια αντίδραση κοντά στο κέντρο, επιθετική στα άκρα (κλειστές στροφές).
# 0.0 = γραμμικό · μεγαλύτερο = πιο δυνατό «δάγκωμα» στη γωνία.
EDGE_BOOST = 2.8

# ── Ανάκτηση όταν χαθεί η γραμμή ───────────────────────────────
LOST_TIMEOUT = 0.8   # δευτ. αναζήτησης πριν σταματήσει
LOST_SPEED   = 55    # ταχύτητα pivot αναζήτησης

# ── Γραμμή τερματισμού (μαύρο και στους 3 αισθητήρες) ──────────
STOP_THRESHOLD = 0.7   # πάνω από αυτό θεωρείται «μαύρο» (0.0–1.0)
STOP_CONFIRM   = 3     # συνεχόμενες αναγνώσεις για επιβεβαίωση
                       # (αποφεύγει ψευδείς ανιχνεύσεις σε κλειστή στροφή)

# ════════════════════════════════════════════════════════════════
# PID CONTROLLER
# ════════════════════════════════════════════════════════════════
integral       = 0.0
last_error     = 0.0
last_time      = time.monotonic()
last_valid_pos = 0.0
lost_since     = None

def reset_pid():
    """Μηδενίζει την κατάσταση του PID."""
    global integral, last_error, last_time, last_valid_pos, lost_since
    integral       = 0.0
    last_error     = 0.0
    last_time      = time.monotonic()
    last_valid_pos = 0.0
    lost_since     = None

def pid_step():
    """
    Ένα βήμα PID.
    Επιστρέφει (left_speed, right_speed).
    """
    global integral, last_error, last_time, last_valid_pos, lost_since

    now = time.monotonic()
    dt  = max(now - last_time, 0.001)
    last_time = now

    pos = line_position()

    # ── Γραμμή εντοπίστηκε ──────────────────────────────────────
    if pos is not None:
        lost_since     = None
        last_valid_pos = pos
        error          = pos

        integral  += error * dt
        integral   = max(-1.5, min(1.5, integral))   # anti-windup

        derivative = (error - last_error) / dt
        last_error = error

        # Μη-γραμμικό error: ήπιο στο κέντρο, δυνατό στα άκρα
        error_boosted = error * (1.0 + EDGE_BOOST * abs(error))

        correction = Kp * error_boosted + Ki * integral + Kd * derivative
        correction = max(-100.0, min(100.0, correction))

        # Δυναμική ταχύτητα: κόβει αυτόματα στις στροφές
        base = BASE_SPEED - (BASE_SPEED - TURN_SPEED) * min(abs(error), 1.0)

        L = int(max(MIN_SPEED, min(MAX_SPEED, base - correction)))
        R = int(max(MIN_SPEED, min(MAX_SPEED, base + correction)))
        return L, R

    # ── Γραμμή χάθηκε ───────────────────────────────────────────
    if lost_since is None:
        lost_since = now
        integral   = 0.0   # μηδένισε για να μην τινάξει όταν ξαναβρεθεί
        last_error = 0.0

    if now - lost_since > LOST_TIMEOUT:
        return 0, 0   # σταμάτα — δεν βρέθηκε γραμμή

    # Στρίψε προς την τελευταία γνωστή κατεύθυνση
    if last_valid_pos <= 0:
        return -LOST_SPEED, LOST_SPEED   # γραμμή ήταν αριστερά
    else:
        return LOST_SPEED, -LOST_SPEED   # γραμμή ήταν δεξιά

# ════════════════════════════════════════════════════════════════
# ΒΟΗΘΗΤΙΚΕΣ ΣΥΝΑΡΤΗΣΕΙΣ ΚΟΥΜΠΙΩΝ
# ════════════════════════════════════════════════════════════════
def wait_for_start():
    """Περιμένει πάτημα του GP20 (START)."""
    print("  Πάτα GP20 για εκκίνηση...")
    while btn_start.value:      # True = δεν πατήθηκε
        time.sleep(0.05)
    time.sleep(0.05)            # debounce

def stop_pressed():
    """Επιστρέφει True αν πατήθηκε το GP21 (STOP)."""
    return not btn_stop.value   # False = πατήθηκε (active LOW)

# ════════════════════════════════════════════════════════════════
# ΚΥΡΙΩΣ ΠΡΟΓΡΑΜΜΑ
# ════════════════════════════════════════════════════════════════
print("╔══════════════════════════════════════╗")
print("║   LINE FOLLOWER — Maker Pi RP2040    ║")
print("║   GP20 = START  |  GP21 = STOP       ║")
print("╚══════════════════════════════════════╝")

while True:
    # ── Αναμονή εκκίνησης ───────────────────────────────────────
    wait_for_start()
    reset_pid()
    print("  ▶ Εκκίνηση!")

    # ── Κύριος βρόχος ───────────────────────────────────────────
    try:
        black_count = 0          # μετρητής επιβεβαίωσης γραμμής τερματισμού
        while True:
            # Έλεγχος κουμπιού STOP
            if stop_pressed():
                stop()
                print("  ■ Σταμάτησε — πάτα GP20 για επανεκκίνηση.")
                time.sleep(0.3)   # debounce
                break

            # Γραμμή τερματισμού — μαύρο και στους 3 αισθητήρες
            if all_black():
                black_count += 1
                if black_count >= STOP_CONFIRM:
                    stop()
                    print("  ■ Γραμμή τερματισμού! Πάτα GP20 για επανεκκίνηση.")
                    time.sleep(0.3)
                    break
            else:
                black_count = 0   # μηδένισε αν δεν είναι πια όλα μαύρα

            # PID βήμα
            L, R = pid_step()
            set_motors(L, R)
            time.sleep(0.01)      # ~100Hz loop

    except KeyboardInterrupt:
        stop()
        print("  ■ Διακοπή από χρήστη.")
        break
