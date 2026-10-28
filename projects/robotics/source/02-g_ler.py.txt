import matplotlib.pyplot as plt

# ---------- LOG DATA FROM R-STAGE ----------

# Curve stability logs (Run 1–8)
runs = [1, 2, 3, 4, 5, 6, 7, 8]

peak_line_error = [0.58, 0.54, 0.47, 0.43,
                   0.36, 0.31, 0.27, 0.23]

avg_line_error = [0.34, 0.31, 0.28, 0.25,
                  0.22, 0.19, 0.17, 0.15]

# Driver performance logs (Run 1–8)
launch_jerk = [0.85, 0.78, 0.67, 0.59,
               0.51, 0.45, 0.41, 0.38]

lap_time = [14.2, 13.9, 13.4, 13.1,
            12.7, 12.5, 12.3, 12.2]

# Battery voltage logs (Run 1–8)
battery_voltage = [12.60, 12.52, 12.45, 12.36,
                   12.28, 12.22, 12.18, 12.15]

# ============================================================
# FIGURE 1 — CURVE STABILITY IMPROVEMENT
# ============================================================

plt.figure(figsize=(6,4))
plt.plot(runs, peak_line_error, marker="o", label="Peak line error")
plt.plot(runs, avg_line_error, marker="s", linestyle="--", label="Average line error")

plt.xlabel("Run number")
plt.ylabel("Line error (normalized units)")
plt.title("Improvement in Curve Stability Over R-Stage")
plt.xticks(runs)
plt.grid(True)
plt.legend()
plt.tight_layout()
plt.savefig("curve_stability_improvement.png", dpi=300)
plt.close()

# ============================================================
# FIGURE 2 — DRIVER PERFORMANCE (LAUNCH + TURN)
# ============================================================

plt.figure(figsize=(6,4))
plt.plot(runs, lap_time, marker="o", label="Lap time (s)")
plt.xlabel("Run number")
plt.ylabel("Lap time (s)")
plt.title("Driver Launch + Turn Refinement Over R-Stage")
plt.xticks(runs)
plt.grid(True)

ax1 = plt.gca()
ax2 = ax1.twinx()

ax2.plot(runs, launch_jerk, marker="s", linestyle="--", color="orange", label="Launch jerk")
ax2.set_ylabel("Launch jerk (normalized)")

# Combine legends
lines1, labels1 = ax1.get_legend_handles_labels()
lines2, labels2 = ax2.get_legend_handles_labels()
ax1.legend(lines1 + lines2, labels1 + labels2, loc="upper right")

plt.tight_layout()
plt.savefig("driver_refinement_improvement.png", dpi=300)
plt.close()

# ============================================================
# FIGURE 3 — BATTERY VOLTAGE DECLINE ACROSS RUNS
# ============================================================

plt.figure(figsize=(6,4))
plt.plot(runs, battery_voltage, marker="o", color="green")

plt.xlabel("Run number")
plt.ylabel("Battery voltage (V)")
plt.title("Battery Voltage Response Across R-Stage Runs")
plt.xticks(runs)
plt.grid(True)
plt.tight_layout()
plt.savefig("battery_voltage_profile.png", dpi=300)
plt.close()

print("Saved all graphs: curve_stability_improvement.png, driver_refinement_improvement.png, battery_voltage_profile.png")
