import numpy as np
import matplotlib.pyplot as plt

# Synthetic predictive data
np.random.seed(42)
n = 20

manual = np.clip(np.random.normal(3.2, 0.4, n), 0.5, None)
assist = np.clip(np.random.normal(1.8, 0.25, n), 0.5, None)

manual_avg = manual.mean()
assist_avg = assist.mean()

# Clean minimalist plot --------------------------------------------

plt.figure(figsize=(6,4))                # smaller, more balanced
bars = plt.bar(
    ["Manual", "Button Assist"],
    [manual_avg, assist_avg],
    width=0.45
)

# Annotate bars
for bar, value in zip(bars, [manual_avg, assist_avg]):
    plt.text(
        bar.get_x() + bar.get_width()/2,
        value + 0.05,
        f"{value:.2f} s",
        ha="center",
        va="bottom",
        fontsize=11
    )

# Title and labels (minimal)
plt.ylabel("Average Stacking Time (s)", fontsize=11)
plt.title("Manual vs Button-Assist Stacking Time\n(Synthetic Predictive Data)", fontsize=12)

# Reduce visual noise
plt.grid(axis="y", linestyle="--", alpha=0.3)
plt.ylim(0, max(manual_avg, assist_avg) + 0.7)

plt.tight_layout()
plt.show()
