# calibration.py
# Βαθμονόμηση 3x TCRT5000 με αναλογική έξοδο & ποτενσιόμετρο
#
# Sensor αριστερά : GP26 (A0)
# Sensor κέντρο  : GP27 (A1)
# Sensor δεξιά   : GP28 (A2)

import board
import analogio
import time

# ── Sensors ──────────────────────────────────────────────────────
sensor_left   = analogio.AnalogIn(board.A0)
sensor_center = analogio.AnalogIn(board.A1)
sensor_right  = analogio.AnalogIn(board.A2)

def read_raw():
    return (sensor_left.value, sensor_center.value, sensor_right.value)

# ════════════════════════════════════════════════════════════════
# ΒΗΜΑ 1: ΡΥΘΜΙΣΗ ΠΟΤΕΝΣΙΟΜΕΤΡΟΥ
# Στόχος: να υπάρχει ΜΕΓΑΛΗ διαφορά μεταξύ λευκού και μαύρου
# ════════════════════════════════════════════════════════════════
def pot_tuning_guide():
    """
    Live οδηγός ρύθμισης ποτενσιόμετρου.
    Βάλε ένα sensor πάνω στη γραμμή, ρύθμισε μέχρι το LED να ανάβει,
    μετά σκόπευε σε μέγιστη αναλογική διαφορά.
    """
    print("\n" + "═"*55)
    print("  ΒΗΜΑ 1: ΡΥΘΜΙΣΗ ΠΟΤΕΝΣΙΟΜΕΤΡΟΥ")
    print("═"*55)
    print("  Τοποθέτησε το ρομπότ πάνω στο ΛΕΥΚΟ έδαφος.")
    print("  Στρίβε αργά το ποτενσιόμετρο κάθε sensor μέχρι")
    print("  η τιμή να είναι ΧΑΜΗΛΗ (~2000-8000).")
    print("  Μετά πέρνα στη ΜΑΥΡΗ γραμμή — τιμή πρέπει να")
    print("  ανεβεί σημαντικά (~40000-63000).")
    print("  Ctrl+C για επόμενο βήμα.")
    print("─"*55)
    print(f"{'':4} {'RAW-L':>8} {'RAW-C':>8} {'RAW-R':>8}  "
          f"{'RANGE-L':>8} {'RANGE-C':>8} {'RANGE-R':>8}")
    print("─"*55)

    session_min = [65535, 65535, 65535]
    session_max = [0, 0, 0]

    try:
        while True:
            raw = read_raw()
            for i, v in enumerate(raw):
                if v < session_min[i]: session_min[i] = v
                if v > session_max[i]: session_max[i] = v

            ranges = [session_max[i] - session_min[i] for i in range(3)]

            # Εικονική μπάρα 0-20 χαρακτήρες
            bars = []
            for r in ranges:
                filled = int(r / 65535 * 20)
                bar = f"{'█'*filled}{'░'*(20-filled)}"
                bars.append(bar)

            # Χρωματική ένδειξη range (κείμενο)
            labels = []
            for r in ranges:
                if r > 40000:   labels.append("✓ ΚΑΛΟ ")
                elif r > 20000: labels.append("~ ΜΕΤΡ.")
                else:           labels.append("✗ ΚΑΚΟ ")

            print(f"\r{'':4} "
                  f"{raw[0]:>8} {raw[1]:>8} {raw[2]:>8}  "
                  f"{labels[0]:>8} {labels[1]:>8} {labels[2]:>8}",
                  end="")
            time.sleep(0.05)

    except KeyboardInterrupt:
        print(f"\n\n  Καταγεγραμμένο εύρος:")
        print(f"  L: {session_min[0]:>6} → {session_max[0]:>6}  "
              f"(range={session_max[0]-session_min[0]})")
        print(f"  C: {session_min[1]:>6} → {session_max[1]:>6}  "
              f"(range={session_max[1]-session_min[1]})")
        print(f"  R: {session_min[2]:>6} → {session_max[2]:>6}  "
              f"(range={session_max[2]-session_min[2]})")
        return session_min, session_max

# ════════════════════════════════════════════════════════════════
# ΒΗΜΑ 2: AUTO-CALIBRATION  (min/max capture)
# ════════════════════════════════════════════════════════════════
cal_min = [65535, 65535, 65535]
cal_max = [0,     0,     0    ]

def auto_calibrate(duration=5.0):
    """
    Κούνα αργά το ρομπότ πάνω-κάτω στη γραμμή για 'duration' δευτ.
    Αποθηκεύει min/max για κάθε sensor.
    """
    global cal_min, cal_max
    print("\n" + "═"*55)
    print("  ΒΗΜΑ 2: AUTO-CALIBRATION")
    print("═"*55)
    print(f"  Κούνα το ρομπότ αριστερά-δεξιά στη γραμμή")
    print(f"  για {duration} δευτερόλεπτα...")
    print("─"*55)

    cal_min = [65535, 65535, 65535]
    cal_max = [0,     0,     0    ]
    samples = 0

    start = time.monotonic()
    while time.monotonic() - start < duration:
        elapsed = time.monotonic() - start
        remaining = duration - elapsed
        raw = read_raw()

        for i, v in enumerate(raw):
            if v < cal_min[i]: cal_min[i] = v
            if v > cal_max[i]: cal_max[i] = v

        # Progress bar
        pct = elapsed / duration
        bar_len = 30
        filled = int(pct * bar_len)
        bar = f"[{'█'*filled}{'░'*(bar_len-filled)}] {remaining:.1f}s"

        print(f"\r  {bar}  L:{raw[0]:>6} C:{raw[1]:>6} R:{raw[2]:>6}",
              end="")
        samples += 1
        time.sleep(0.02)

    print(f"\n\n  ✓ {samples} δείγματα ελήφθησαν")
    print(f"\n  {'':10} {'MIN':>8} {'MAX':>8} {'RANGE':>8} {'STATUS':>8}")
    print(f"  {'─'*46}")
    names = ["L (Αριστ.)", "C (Κέντρο)", "R (Δεξιά) "]
    for i, name in enumerate(names):
        r = cal_max[i] - cal_min[i]
        status = "✓ ΚΑΛΟ" if r > 30000 else ("~ ΜΕΤΡ." if r > 15000 else "✗ ΚΑΚΟ")
        print(f"  {name:10} {cal_min[i]:>8} {cal_max[i]:>8} "
              f"{r:>8} {status:>8}")
    return cal_min, cal_max

# ════════════════════════════════════════════════════════════════
# ΒΗΜΑ 3: ΕΠΑΛΗΘΕΥΣΗ  — normalized + position test
# ════════════════════════════════════════════════════════════════
def read_normalized():
    """0.0 = λευκό, 1.0 = μαύρη γραμμή."""
    raw = read_raw()
    out = []
    for i, v in enumerate(raw):
        span = cal_max[i] - cal_min[i]
        if span == 0:
            out.append(0.0)
        else:
            out.append(max(0.0, min(1.0, (v - cal_min[i]) / span)))
    return tuple(out)

def line_position():
    """
    Θέση γραμμής: -1.0 (αριστερά) έως +1.0 (δεξιά), 0.0 = κέντρο.
    Επιστρέφει None αν δεν εντοπιστεί γραμμή.
    """
    l, c, r = read_normalized()
    total = l + c + r
    if total < 0.2:
        return None
    return round((-1.0*l + 0.0*c + 1.0*r) / total, 3)

def verify_calibration(cycles=60):
    """Live επαλήθευση — κούνα το ρομπότ και δες αν η θέση αλλάζει."""
    print("\n" + "═"*55)
    print("  ΒΗΜΑ 3: ΕΠΑΛΗΘΕΥΣΗ")
    print("═"*55)
    print("  Κούνα το ρομπότ — η POS πρέπει να πηγαίνει")
    print("  από -1.0 (αριστερά) έως +1.0 (δεξιά).")
    print("─"*55)
    print(f"  {'NRM-L':>7} {'NRM-C':>7} {'NRM-R':>7}   "
          f"{'VISUAL':<24} {'POS':>6}")
    print("─"*55)

    try:
        while True:
            n = read_normalized()
            pos = line_position()

            # Visual bar: 25 χαρακτήρες, γραμμή δείχνει θέση
            bar = ['─'] * 25
            bar[12] = '┼'  # κέντρο
            if pos is not None:
                idx = int((pos + 1.0) / 2.0 * 24)
                idx = max(0, min(24, idx))
                bar[idx] = '█'
            visual = "".join(bar)

            pos_str = f"{pos:+.3f}" if pos is not None else " None"
            print(f"\r  {n[0]:>7.3f} {n[1]:>7.3f} {n[2]:>7.3f}   "
                  f"{visual}   {pos_str:>6}", end="")
            time.sleep(0.05)

    except KeyboardInterrupt:
        print("\n  Επαλήθευση ολοκληρώθηκε.")

# ════════════════════════════════════════════════════════════════
# ΕΚΤΥΠΩΣΗ ΤΙΜΩΝ ΓΙΑ ΑΝΤΙΓΡΑΦΗ ΣΤΟ ΚΥΡΙΩΣ ΚΩΔΙΚΑ
# ════════════════════════════════════════════════════════════════
def print_cal_values():
    print("\n" + "═"*55)
    print("  ΑΝΤΙΓΡΑΨΕ ΑΥΤΕΣ ΤΙΣ ΤΙΜΕΣ ΣΤΟ line_following.py:")
    print("═"*55)
    print(f"  cal_min = {cal_min}")
    print(f"  cal_max = {cal_max}")
    print("═"*55)

# ════════════════════════════════════════════════════════════════
# MAIN — τρέξε τα βήματα σειριακά
# ════════════════════════════════════════════════════════════════
print("\n" + "★"*55)
print("  ΒΑΘΜΟΝΟΜΗΣΗ TCRT5000 — Maker Pi RP2040")
print("★"*55)

input("\n  Πάτα Enter για να ξεκινήσει το Βήμα 1...")
pot_tuning_guide()

input("\n  Πάτα Enter για Auto-Calibration (Βήμα 2)...")
auto_calibrate(duration=5.0)

input("\n  Πάτα Enter για Επαλήθευση (Βήμα 3)...")
verify_calibration()


print_cal_values()


