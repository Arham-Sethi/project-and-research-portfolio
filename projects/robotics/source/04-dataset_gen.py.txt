import numpy as np
import matplotlib.pyplot as plt

np.random.seed(42)

# --- Time axis ---
t = np.linspace(0, 20, 800)   # 20 seconds at high resolution

# --- Synthetic "raw" line error (before refinement) ---
error_raw = 50*np.sin(1.2*t) + 18*np.sin(4*t) + 8*np.random.randn(len(t))

# --- Synthetic "refined" line error (after adaptive PID & AGGRO) ---
error_refined = 25*np.sin(1.2*t) + 6*np.sin(4*t) + 4*np.random.randn(len(t))

# --- Derivatives ---
der_raw = np.gradient(error_raw, t)
der_ref = np.gradient(error_refined, t)

# --- Simulated motor output after PID ---
Kp1, Kd1 = 0.22, 4.5      # early tuning constants
Kp2, Kd2 = 0.30, 7.0      # refined adaptive PID

control_raw = Kp1*error_raw + Kd1*der_raw
control_ref = Kp2*error_refined + Kd2*der_ref

# Graph Line error reduction
plt.figure(figsize=(12,5))
plt.plot(t, error_raw, alpha=0.6, label="Before Refinement (Baseline PID)")
plt.plot(t, error_refined, linewidth=2.2, label="After Refinement (Adaptive PID + AGGRO)")

plt.title("Line Tracking Error vs Time")
plt.xlabel("Time (s)")
plt.ylabel("Line Error (QTR Weighted Output)")
plt.legend()
plt.grid(True)
plt.tight_layout()
plt.show()

# Graph: Oscillation Decay Comparison
plt.figure(figsize=(12,5))
plt.plot(t, np.abs(error_raw), label="Baseline Oscillation Envelope", alpha=0.6)
plt.plot(t, np.abs(error_refined), label="Refined Oscillation Envelope", linewidth=2.2)

plt.title("Oscillation Envelope Decay (Error Magnitude)")
plt.xlabel("Time (s)")
plt.ylabel("|Error|")
plt.legend()
plt.grid(True)
plt.tight_layout()
plt.show()

# Graph: Control Output Smoothness
plt.figure(figsize=(12,5))
plt.plot(t, control_raw, alpha=0.55, label="Baseline Control Output")
plt.plot(t, control_ref, linewidth=2.2, label="Refined Control Output")

plt.title("PID Motor Command Output vs Time")
plt.xlabel("Time (s)")
plt.ylabel("Control Output (PID)")
plt.legend()
plt.grid(True)
plt.tight_layout()
plt.show()

# Graph: Soft-Landing ToF Approach
# synthetic ToF distance (mm)
d_raw = np.clip(400 - 20*t + 15*np.sin(0.7*t), 0, 400)
d_ref = np.clip(400 - 20*t + 4*np.sin(0.7*t), 0, 400)

plt.figure(figsize=(12,6))
plt.plot(t, d_raw, alpha=0.6, label="Before Soft-Landing")
plt.plot(t, d_ref, linewidth=2.2, label="After Soft-Landing (Distance Bands)")

plt.title("ToF Distance vs Time During Approach")
plt.xlabel("Time (s)")
plt.ylabel("Distance to Target (mm)")
plt.legend()
plt.grid(True)
plt.tight_layout()
plt.show()
