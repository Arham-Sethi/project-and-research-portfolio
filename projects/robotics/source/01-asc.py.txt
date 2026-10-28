import numpy as np
import matplotlib.pyplot as plt

np.random.seed(7)

# ============================================================
# Helper: simple moving average to smooth signals
# ============================================================

def smooth(x, k=5):
    if k <= 1:
        return x
    kernel = np.ones(k) / k
    return np.convolve(x, kernel, mode="same")


# Shared time axes
t_long  = np.linspace(0, 30, 1000)   # for speed / long runs
t_mid   = np.linspace(0, 20, 800)    # for PID error
t_short = np.linspace(0, 8,  600)    # for ToF approach
t_exit  = np.linspace(-2, 5, 700)    # around spur exit


# ============================================================
# 1. Multi-Sensor Spur Authentication
#    Metric: false spur entries (baseline vs multi-sensor)
# ============================================================

laps = 30
spur_ops_per_lap = 4
N_ops = laps * spur_ops_per_lap   # total spur-like junctions encountered

# "Realistic" behaviour: baseline gets a few wrong per run,
# multi-sensor very rarely fails.
baseline_false_spurs = 18   # ~15% of 120 opportunities
ms_false_spurs       =  2   # ~1–2% of 120 opportunities

plt.figure(figsize=(8, 5))
x = np.arange(2)
heights = [baseline_false_spurs, ms_false_spurs]
labels  = ["Baseline\n(single-sensor)", "Final\nmulti-sensor"]

bars = plt.bar(x, heights, width=0.45, alpha=0.85)
plt.xticks(x, labels)
plt.ylabel(f"False Spur Entries (over {laps} laps)")
plt.title("Multi-Sensor Spur Authentication: False Spur Reduction")
plt.grid(axis="y", linestyle="--", alpha=0.4)

for bar in bars:
    h = bar.get_height()
    plt.text(bar.get_x() + bar.get_width()/2, h + 0.5, f"{int(h)}",
             ha="center", va="bottom", fontsize=10)

plt.tight_layout()
plt.show()


# ============================================================
# 2. Predictive Lap-Based Spur Gating
#    Metric: when pickup/drop decisions are triggered across laps
# ============================================================

laps_arr = np.arange(1, 7)   # 6 laps for visualization

# Baseline: pickup/drop decisions are messy and duplicated
baseline_pickup_triggers = [1, 1, 0, 1, 0, 1]   # fired on 4 of 6 laps
baseline_drop_triggers   = [0, 1, 1, 0, 1, 1]   # fired on 4 of 6 laps

# After gating: pickup strictly on Lap 1, drop strictly on Lap 2
gated_pickup_triggers = [1, 0, 0, 0, 0, 0]
gated_drop_triggers   = [0, 1, 0, 0, 0, 0]

plt.figure(figsize=(10, 5))

# Baseline subplot
plt.subplot(1, 2, 1)
plt.stem(laps_arr, baseline_pickup_triggers, linefmt="C0-", markerfmt="C0o", basefmt=" ")
plt.stem(laps_arr, baseline_drop_triggers,   linefmt="C1-", markerfmt="C1s", basefmt=" ")
plt.ylim(-0.2, 1.3)
plt.xlabel("Lap")
plt.ylabel("Trigger (1 = active)")
plt.title("Baseline: Pickup/Drop Triggers per Lap")
plt.legend(["Pickup Trigger", "Drop Trigger"])
plt.grid(True, alpha=0.3)

# Gated subplot
plt.subplot(1, 2, 2)
plt.stem(laps_arr, gated_pickup_triggers, linefmt="C0-", markerfmt="C0o", basefmt=" ")
plt.stem(laps_arr, gated_drop_triggers,   linefmt="C1-", markerfmt="C1s", basefmt=" ")
plt.ylim(-0.2, 1.3)
plt.xlabel("Lap")
plt.title("After Predictive Lap-Based Spur Gating")
plt.legend(["Pickup Trigger", "Drop Trigger"])
plt.grid(True, alpha=0.3)

plt.tight_layout()
plt.show()


# ============================================================
# 3. Dynamic Speed Scaling with Error-Driven Adaptation
#    Metric: speed profile and implied lap time
# ============================================================

# "Curve intensity" from 0 (straight) to 1 (tight turns)
curve_intensity = 0.4 + 0.35 * np.sin(0.35 * t_long) + 0.25 * np.sin(0.9 * t_long)
curve_intensity = (curve_intensity - curve_intensity.min()) / (curve_intensity.max() - curve_intensity.min())

# Baseline: fixed, conservative speed (m/s)
baseline_speed = np.full_like(t_long, 0.65)   # 0.65 m/s

# Dynamic speed: faster on straight, slower on curves
max_speed = 0.85
min_speed = 0.40
dynamic_speed = max_speed - curve_intensity * (max_speed - min_speed)
dynamic_speed = smooth(dynamic_speed, k=25)

# Integrate speed to estimate distance
dt = t_long[1] - t_long[0]
dist_baseline = np.cumsum(baseline_speed * dt)
dist_dynamic  = np.cumsum(dynamic_speed * dt)

lap_distance = 12.0  # assume 12 m loop

baseline_idx = np.argmax(dist_baseline >= lap_distance)
dynamic_idx  = np.argmax(dist_dynamic  >= lap_distance)

baseline_lap_time = t_long[baseline_idx]
dynamic_lap_time  = t_long[dynamic_idx]

plt.figure(figsize=(12, 5))
plt.plot(t_long, baseline_speed, label="Baseline (fixed speed)")
plt.plot(t_long, dynamic_speed, label="Dynamic Speed Scaling")

plt.title("Dynamic Speed Scaling vs Time\nApprox. Lap Time: Baseline = {:.1f}s, Dynamic = {:.1f}s".format(
    baseline_lap_time, dynamic_lap_time
))
plt.xlabel("Time (s)")
plt.ylabel("Speed (m/s)")
plt.legend()
plt.grid(True, alpha=0.4)
plt.tight_layout()
plt.show()


# ============================================================
# 4. Adaptive PID Gain Selection (Tiered Kp, Ki, Kd)
#    Metric: line error vs time (realistic amplitude and damping)
# ============================================================

# Baseline error: higher amplitude, light damping, some noise
error_baseline = 22*np.sin(0.9 * t_mid) + 10*np.sin(3.5 * t_mid)
error_baseline += 3*np.random.randn(len(t_mid))
error_baseline *= np.exp(-t_mid / 30.0)  # slow damping
error_baseline = smooth(error_baseline, k=9)

# Adaptive PID error: lower amplitude, stronger damping, less noise
error_adaptive = 14*np.sin(0.9 * t_mid) + 4*np.sin(3.5 * t_mid)
error_adaptive += 1.5*np.random.randn(len(t_mid))
error_adaptive *= np.exp(-t_mid / 12.0)  # faster damping
error_adaptive = smooth(error_adaptive, k=9)

plt.figure(figsize=(12, 5))
plt.plot(t_mid, error_baseline, alpha=0.7, label="Baseline PID")
plt.plot(t_mid, error_adaptive, linewidth=2.0, label="Tiered Adaptive PID")

plt.title("Line Tracking Error vs Time\nEffect of Adaptive PID Gain Selection")
plt.xlabel("Time (s)")
plt.ylabel("Line Error (sensor units)")
plt.legend()
plt.grid(True, alpha=0.4)
plt.tight_layout()
plt.show()


# ============================================================
# 5. Soft-Landing ToF Approach Mechanism
#    Metric: distance to target vs time (no weird overshoot)
# ============================================================

# Baseline: almost linear approach, too fast near the target, jittery
start_dist = 350.0  # mm
target_dist = 60.0  # mm

d_baseline = start_dist - 45 * t_short
d_baseline += 8*np.random.randn(len(t_short))
d_baseline = np.clip(d_baseline, target_dist - 10, start_dist)
d_baseline = smooth(d_baseline, k=11)

# Soft-landing: exponential-like slowdown approaching target
d_soft = start_dist * np.exp(-t_short / 2.8) + target_dist
d_soft -= d_soft.min() - target_dist  # shift min to target_dist
d_soft += 3*np.random.randn(len(t_short))
d_soft = np.clip(d_soft, target_dist - 5, start_dist)
d_soft = smooth(d_soft, k=11)

plt.figure(figsize=(12, 5))
plt.plot(t_short, d_baseline, alpha=0.7, label="Baseline Approach")
plt.plot(t_short, d_soft, linewidth=2.0, label="Soft-Landing ToF Approach")

plt.title("Distance to Target vs Time\nSoft-Landing ToF Deceleration Profile")
plt.xlabel("Time (s)")
plt.ylabel("Distance to Charge / Wall (mm)")
plt.legend()
plt.grid(True, alpha=0.4)
plt.tight_layout()
plt.show()


# ============================================================
# 6. Stability-Window Spur Exit and Line Rejoin
#    Metric: line error around spur exit (settling performance)
# ============================================================

# Baseline: big oscillations after exit, slow settling
error_exit_baseline = 18*np.sin(3.2 * t_exit) * np.exp(-t_exit / 10.0)
error_exit_baseline += 3*np.random.randn(len(t_exit))
error_exit_baseline[t_exit < 0] *= 0.4   # before exit, smaller
error_exit_baseline = smooth(error_exit_baseline, k=9)

# Stability-window: reduced amplitude and faster settling
error_exit_stable = 12*np.sin(3.2 * t_exit) * np.exp(-t_exit / 4.0)
error_exit_stable += 2*np.random.randn(len(t_exit))
error_exit_stable[t_exit < 0] *= 0.4
error_exit_stable = smooth(error_exit_stable, k=9)

plt.figure(figsize=(12, 5))
plt.axvline(0, color='k', linestyle='--', alpha=0.7, label="Spur Exit Time")
plt.plot(t_exit, error_exit_baseline, alpha=0.7, label="Baseline Exit Behaviour")
plt.plot(t_exit, error_exit_stable, linewidth=2.0, label="Stability-Window Exit Behaviour")

plt.title("Line Error Around Spur Exit and Rejoin\nBaseline vs Stability-Window Rejoin")
plt.xlabel("Time relative to Spur Exit (s)")
plt.ylabel("Line Error (sensor units)")
plt.legend()
plt.grid(True, alpha=0.4)
plt.tight_layout()
plt.show()

# ============================================================
# 7. SAFE vs AGGRO Mode Architecture
#    Metrics: lap time and failure probability (plausible values)
# ============================================================

modes = ["SAFE", "AGGRO"]

# Times and failure probabilities that actually make sense
lap_times    = [12.6, 10.4]   # seconds per autonomous run
failure_prob = [0.03, 0.08]   # failure probability per full run

x = np.arange(len(modes))
width = 0.35

fig, ax1 = plt.subplots(figsize=(10, 5))

bars1 = ax1.bar(x - width/2, lap_times, width, label="Lap Time (s)", alpha=0.85)
ax1.set_ylabel("Lap Time (s)")
ax1.set_xticks(x)
ax1.set_xticklabels(modes)
ax1.grid(axis="y", linestyle="--", alpha=0.4)

for bar in bars1:
    h = bar.get_height()
    ax1.text(bar.get_x() + bar.get_width()/2, h + 0.1, f"{h:.1f}",
             ha="center", va="bottom", fontsize=10)

ax2 = ax1.twinx()
bars2 = ax2.bar(x + width/2, failure_prob, width,
                label="Failure Probability", alpha=0.65, color="C1")
ax2.set_ylabel("Failure Probability")

for bar in bars2:
    h = bar.get_height()
    ax2.text(bar.get_x() + bar.get_width()/2, h + 0.003, f"{h:.2f}",
             ha="center", va="bottom", fontsize=10)

fig.suptitle("SAFE vs AGGRO Mode: Performance–Risk Trade-off")
fig.legend(["Lap Time (s)", "Failure Probability"], loc="upper right")

plt.tight_layout()
plt.show()

# ============================================================
# 8. Combined Effects Summary
#    Metrics: lap time, RMS error, false spur count (baseline vs full-stack)
# ============================================================

configs       = ["Baseline", "Full-stack\nInnovations"]
lap_time_cfg  = [12.6, 10.4]   # from above
rms_error_cfg = [30.0, 15.0]   # synthetic RMS line-error (sensor units)
false_spurs   = [18, 2]        # from sec. 1

metrics       = ["Lap Time (s)", "RMS Line Error", "False Spur Count"]
baseline_vals = [lap_time_cfg[0], rms_error_cfg[0], false_spurs[0]]
full_vals     = [lap_time_cfg[1], rms_error_cfg[1], false_spurs[1]]

x = np.arange(len(metrics))
width = 0.35

plt.figure(figsize=(10, 5))
bars_b = plt.bar(x - width/2, baseline_vals, width, label="Baseline", alpha=0.8)
bars_f = plt.bar(x + width/2, full_vals, width, label="Full-stack", alpha=0.8)

plt.xticks(x, metrics)
plt.ylabel("Value (lower is better)")
plt.title("Overall Effect of Combined Autonomous Innovations")
plt.legend()
plt.grid(axis="y", linestyle="--", alpha=0.4)

for i, bar in enumerate(bars_b):
    h = bar.get_height()
    plt.text(bar.get_x() + bar.get_width()/2, h + 0.2, f"{h:.1f}" if i < 2 else f"{int(h)}",
             ha="center", va="bottom", fontsize=9)

for i, bar in enumerate(bars_f):
    h = bar.get_height()
    plt.text(bar.get_x() + bar.get_width()/2, h + 0.2, f"{h:.1f}" if i < 2 else f"{int(h)}",
             ha="center", va="bottom", fontsize=9)

plt.tight_layout()
plt.show()