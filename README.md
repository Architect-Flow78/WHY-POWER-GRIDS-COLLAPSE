# WHY-POWER-GRIDS-COLLAPSE
# power_grid_blackout_phase.py
# WHY POWER GRIDS COLLAPSE — PHASE MATHEMATICS
# Blackout as Cascade Desynchronization Below Critical Coupling K_c
#
# GOOGLE COLAB:
#   1. colab.research.google.com -> New notebook
#   2. Paste entire file -> Shift+Enter
#   3. Download PNG + GIF from Files panel
#
# REAL PROBLEM:
#   European grid: 50 Hz synchronization across 400+ million people
#   Traditional grid: coal/gas generators — tight mechanical coupling
#   Renewable grid: solar/wind — variable frequency, loose coupling
#
#   2003 Italy blackout:    56 million people, cascade in 2.5 minutes
#   2006 European blackout: 15 million people, frequency deviation 0.8 Hz
#   2019 UK blackout:       1 million people, renewable instability
#
#   Engineers call it: "loss of synchronism" / "frequency instability"
#   Phase mathematics: K drops below K_c. r collapses. Cascade.
#
# PHASE MATHEMATICS DIAGNOSIS:
#   Generator = phase oscillator   omega_i = natural frequency
#   Grid sync  = order parameter   r = |mean(exp(2πi·theta))|
#   r → 1: all generators in phase = stable grid = lights on
#   r → 0: phases scattered       = cascade blackout
#
#   Critical coupling:
#     K_c = 2 · sigma_omega / pi
#   sigma_omega = generator frequency spread
#
#   Traditional grid: K >> K_c  (tight coupling, r → 0.9)
#   Renewable grid:   K ~ K_c   (loose coupling, r → 0.16)
#   Blackout zone:    K < K_c   (cascade inevitable)

try:
    import google.colab
    IN_COLAB = True
    SAVE_PATH = '/content/'
    print("Google Colab detected")
except ImportError:
    IN_COLAB = False
    SAVE_PATH = ''

import numpy as np
import matplotlib
matplotlib.use('Agg')
import matplotlib.pyplot as plt
import matplotlib.animation as animation
from scipy.integrate import odeint

# Real grid parameters
FREQ_NOMINAL   = 50.0    # Hz, European grid standard
SIGMA_TRAD     = 0.05    # Hz spread, traditional generators (tight)
SIGMA_RENEW    = 0.30    # Hz spread, renewable-heavy grid (loose)
BLACKOUT_2003  = 2.5     # minutes, Italy 2003 cascade duration
PEOPLE_ITALY   = 56e6    # affected

N = 30
np.random.seed(7)
omega_grid = np.random.normal(0, SIGMA_RENEW, N)
K_c        = 2 * np.std(omega_grid) / np.pi
K_stable   = K_c * 4.0   # traditional grid coupling
K_blackout = K_c * 0.60  # renewable grid — below threshold

print("=" * 60)
print("POWER GRID STABILITY — PHASE MATHEMATICS DIAGNOSIS")
print("=" * 60)
print(f"  N = {N} generators")
print(f"  sigma_omega = {SIGMA_RENEW:.2f} Hz (renewable spread)")
print(f"  K_c = {K_c:.4f}")
print(f"  K_stable   / K_c = {K_stable/K_c:.1f}  (traditional grid)")
print(f"  K_blackout / K_c = {K_blackout/K_c:.1f}  (renewable grid)")

def kuramoto(theta, t, omega, K, N):
    return omega + K/N * np.array([
        np.sum(np.sin(2*np.pi*(theta - theta[i]))) for i in range(N)])

theta0 = np.random.uniform(0, 1, N)
t_sim  = np.linspace(0, 40, 500)

sol_s  = odeint(kuramoto, theta0.copy(), t_sim, args=(omega_grid, K_stable,   N))
sol_b  = odeint(kuramoto, theta0.copy(), t_sim, args=(omega_grid, K_blackout, N))

r_s = np.array([np.abs(np.mean(np.exp(2j*np.pi*sol_s[i]))) for i in range(500)])
r_b = np.array([np.abs(np.mean(np.exp(2j*np.pi*sol_b[i]))) for i in range(500)])

# Cascade detection: r drops below 0.37 = blackout starts
idx_b = next((i for i,v in enumerate(r_b) if v < 0.37), 499)
t_cascade = t_sim[idx_b]

print(f"\n  r_stable   = {r_s[-1]:.3f}  — grid synchronized, lights on")
print(f"  r_blackout = {r_b[-1]:.3f}  — cascade, blackout")
print(f"  Cascade starts at t = {t_cascade:.1f}")

# D_ij
pairs = [(i,j) for i in range(N) for j in range(i+1,N)][:60]
D_s = np.array([np.mean([1-np.cos(2*np.pi*(sol_s[t][i]-sol_s[t][j])) for i,j in pairs]) for t in range(500)])
D_b = np.array([np.mean([1-np.cos(2*np.pi*(sol_b[t][i]-sol_b[t][j])) for i,j in pairs]) for t in range(500)])

# K sweep
K_ratio = np.linspace(0.1, 6.0, 300)
r_sweep = np.zeros(300)
t_test  = np.linspace(0, 30, 200)
for ki, kr in enumerate(K_ratio):
    sol_t = odeint(kuramoto, theta0.copy(), t_test,
                   args=(omega_grid, kr*K_c, N), rtol=1e-3, atol=1e-5)
    r_sweep[ki] = np.abs(np.mean(np.exp(2j*np.pi*sol_t[-1])))

# ─── STATIC FIGURE ─────────────────────────────────────────────
BG = '#03030f'
fig = plt.figure(figsize=(20, 13))
fig.patch.set_facecolor(BG)
fig.suptitle(
    'Why Power Grids Collapse  —  Blackout as Phase Transition Below K_c\n'
    'r = |mean(exp(2πiθ))|     K_c = 2σ_ω/π     D_ij = 1−cos(2π·Δθ)     '
    'European 50Hz Grid  |  Toroidal Phase Mathematics',
    fontsize=10, color='white', fontweight='bold')

axes = []
for i in range(6):
    proj = 'polar' if i == 1 else None
    ax = fig.add_subplot(2, 3, i+1, projection=proj)
    ax.set_facecolor(BG)
    axes.append(ax)

# 1 — r(t)
axes[0].plot(t_sim, r_s, '#33ff99', lw=2.5, label=f'K/K_c={K_stable/K_c:.1f} traditional grid')
axes[0].plot(t_sim, r_b, '#ff3344', lw=2.5, label=f'K/K_c={K_blackout/K_c:.1f} renewable grid')
axes[0].axhline(0.37, color='white', lw=1.2, ls='--', alpha=0.5, label='r=1/e blackout threshold')
axes[0].axvline(t_cascade, color='#ff3344', lw=1.5, ls=':', alpha=0.8, label=f'cascade at t={t_cascade:.1f}')
axes[0].fill_between(t_sim, r_b, r_s, alpha=0.09, color='lime')
axes[0].set_xlabel('Time', color='white')
axes[0].set_ylabel('r = grid synchronization', color='white')
axes[0].set_title('Grid synchronization r(t)\nr→1: stable   r→0: BLACKOUT', color='white', fontsize=8)
axes[0].legend(fontsize=6, facecolor=BG, labelcolor='white')
axes[0].tick_params(colors='white')
for sp in axes[0].spines.values(): sp.set_edgecolor('#222')

# 2 — polar portrait
ts = sol_s[-1] % 1.0
tb = sol_b[-1] % 1.0
mcp = np.angle(np.mean(np.exp(2j*np.pi*ts))) / (2*np.pi) % 1.0
dist_s = np.minimum(np.abs(ts - mcp), 1 - np.abs(ts - mcp))
cs = plt.cm.YlGn(0.4 + 0.6*(1 - np.clip(dist_s*4, 0, 1)))
cb = plt.cm.hsv(tb)
for j in range(N):
    axes[1].scatter([2*np.pi*tb[j]], [0.55], color=cb[j], s=28, alpha=0.75)
    axes[1].scatter([2*np.pi*ts[j]], [0.85], color=cs[j], s=28, alpha=0.85)
axes[1].annotate('', xy=(np.angle(np.mean(np.exp(2j*np.pi*tb))), r_b[-1]*0.5), xytext=(0,0),
    arrowprops=dict(arrowstyle='->', color='#ff3344', lw=3))
axes[1].annotate('', xy=(np.angle(np.mean(np.exp(2j*np.pi*ts))), r_s[-1]*0.8), xytext=(0,0),
    arrowprops=dict(arrowstyle='->', color='#33ff99', lw=3))
axes[1].set_title(f'Generator phase distribution\nRed r={r_b[-1]:.2f} scattered = BLACKOUT\n'
                  f'Green r={r_s[-1]:.2f} locked = STABLE', color='white', fontsize=8)
axes[1].tick_params(colors='#333')

# 3 — r vs K/K_c
axes[2].plot(K_ratio, r_sweep, '#33aaff', lw=2.5)
axes[2].axvline(1.0, color='white', lw=1.5, ls='--', alpha=0.5, label='K = K_c')
axes[2].axvline(K_blackout/K_c, color='#ff3344', lw=2, ls=':', label=f'Renewable K/K_c={K_blackout/K_c:.1f}')
axes[2].axvline(K_stable/K_c,   color='#33ff99', lw=2, ls=':', label=f'Traditional K/K_c={K_stable/K_c:.1f}')
axes[2].fill_between(K_ratio, 0, r_sweep, where=(K_ratio<1.0), alpha=0.15, color='red',   label='blackout zone')
axes[2].fill_between(K_ratio, 0, r_sweep, where=(K_ratio>1.0), alpha=0.10, color='green', label='stable zone')
axes[2].set_xlabel('K / K_c  (grid coupling / threshold)', color='white')
axes[2].set_ylabel('r = synchronization', color='white')
axes[2].set_title('r vs coupling strength\nLeft of K_c: blackout inevitable\nRight: stable grid', color='white', fontsize=8)
axes[2].legend(fontsize=6, facecolor=BG, labelcolor='white')
axes[2].tick_params(colors='white')
for sp in axes[2].spines.values(): sp.set_edgecolor('#222')

# 4 — D_ij
axes[3].plot(t_sim, D_b, '#ff3344', lw=2.5, label='D_ij blackout → 1')
axes[3].plot(t_sim, D_s, '#33ff99', lw=2.5, label='D_ij stable → 0')
axes[3].fill_between(t_sim, D_s, D_b, alpha=0.10, color='red', label='power lost here')
axes[3].set_xlabel('Time', color='white')
axes[3].set_ylabel('D_ij = 1−cos(2π·Δθ)', color='white')
axes[3].set_title('Phase distance D_ij\nD_ij→0: generators in sync\nD_ij→1: cascade failure', color='white', fontsize=8)
axes[3].legend(fontsize=7, facecolor=BG, labelcolor='white')
axes[3].tick_params(colors='white')
for sp in axes[3].spines.values(): sp.set_edgecolor('#222')

# 5 — real events
axes[4].set_facecolor(BG)
events  = ['Italy 2003\n56M people', 'Europe 2006\n15M people', 'UK 2019\n1M people', 'Phase-stable\ngrid (model)']
impacts = [56, 15, 1, 0]
colors5 = ['#ff2222', '#ff4444', '#ff6666', '#33ff99']
bars = axes[4].bar(events, impacts, color=colors5, alpha=0.85, width=0.5)
for b, v in zip(bars, impacts):
    label = f'{v}M affected' if v > 0 else 'ZERO\nblackout'
    axes[4].text(b.get_x()+b.get_width()/2, b.get_height()+0.3,
        label, ha='center', color='white', fontsize=8, fontweight='bold')
axes[4].set_ylabel('People affected (millions)', color='white')
axes[4].set_title('Real blackout events\nvs phase-stable grid', color='white', fontsize=8)
axes[4].tick_params(colors='white')
for sp in axes[4].spines.values(): sp.set_edgecolor('#222')

# 6 — summary
axes[5].axis('off')
txt = (
    "BLACKOUT = PHASE TRANSITION BELOW K_c\n"
    "─────────────────────────────────────\n"
    "Real documented events:\n"
    "  Italy 2003:   56M people, 2.5 min cascade\n"
    "  Europe 2006:  15M people, 0.8 Hz deviation\n"
    "  UK 2019:       1M people, renewable instability\n\n"
    "Phase mathematics:\n"
    "  theta_i  generator phase oscillator\n"
    "  r = |mean(exp(2πiθ))|  sync parameter\n"
    "  D_ij = 1-cos(2πΔθ)     phase distance\n"
    f"  sigma_omega = {SIGMA_RENEW:.2f} Hz  K_c={K_c:.4f}\n\n"
    f"  Traditional: K/K_c={K_stable/K_c:.1f}   r={r_s[-1]:.2f}  STABLE\n"
    f"  Renewable:   K/K_c={K_blackout/K_c:.1f}   r={r_b[-1]:.2f}  BLACKOUT\n\n"
    "Blackout is not a technical failure.\n"
    "It is a phase transition.\n"
    "Threshold: K_c = 2·sigma_omega/pi\n\n"
    "Exact K_c for your grid topology:\n"
    "  requires network parameters."
)
axes[5].text(0.03, 0.97, txt, transform=axes[5].transAxes,
    color='white', fontsize=7.8, va='top', family='monospace',
    bbox=dict(boxstyle='round', facecolor=BG, edgecolor='#33ff99', alpha=0.9))

plt.tight_layout()
png_path = SAVE_PATH + 'power_grid_blackout_phase.png'
plt.savefig(png_path, dpi=150, bbox_inches='tight', facecolor=BG)
plt.close()
print(f"\nSaved: {png_path}")

# ─── ANIMATION ─────────────────────────────────────────────────
print("Generating animation...")
NF   = 90
t_a  = np.linspace(0, 40, NF)
sa_s = odeint(kuramoto, theta0.copy(), t_a, args=(omega_grid, K_stable,   N))
sa_b = odeint(kuramoto, theta0.copy(), t_a, args=(omega_grid, K_blackout, N))
ra_s = np.array([np.abs(np.mean(np.exp(2j*np.pi*sa_s[i]))) for i in range(NF)])
ra_b = np.array([np.abs(np.mean(np.exp(2j*np.pi*sa_b[i]))) for i in range(NF)])

fig_a = plt.figure(figsize=(14, 7))
fig_a.patch.set_facecolor(BG)

def draw_frame(frame):
    fig_a.clf()
    fig_a.patch.set_facecolor(BG)
    al = fig_a.add_subplot(1, 2, 1, projection='polar')
    ar = fig_a.add_subplot(1, 2, 2, projection='polar')
    al.set_facecolor(BG); ar.set_facecolor(BG)

    tb2 = sa_b[frame] % 1.0
    ts2 = sa_s[frame] % 1.0
    rb2 = ra_b[frame]
    rs2 = ra_s[frame]

    # LEFT — blackout: rainbow chaos
    for j in range(N):
        al.scatter([2*np.pi*tb2[j]], [0.8], color=plt.cm.hsv(tb2[j]), s=50, alpha=0.8)
    al.annotate('', xy=(np.angle(np.mean(np.exp(2j*np.pi*tb2))), max(rb2*0.6,0.05)), xytext=(0,0),
        arrowprops=dict(arrowstyle='->', color='#ff3344', lw=3.5))
    st_b  = '✗  BLACKOUT' if rb2 < 0.37 else 'destabilizing...'
    col_b = '#ff3344' if rb2 < 0.37 else '#ff8844'
    al.set_title(f'K/K_c = {K_blackout/K_c:.1f}   {st_b}\nr = {rb2:.3f}\nRenewable-heavy grid',
                 color=col_b, fontsize=9, fontweight='bold')
    al.tick_params(colors='#333')

    # RIGHT — stable: green cluster
    mcp3 = np.angle(np.mean(np.exp(2j*np.pi*ts2))) / (2*np.pi) % 1.0
    dc3  = np.minimum(np.abs(ts2 - mcp3), 1 - np.abs(ts2 - mcp3))
    for j in range(N):
        ar.scatter([2*np.pi*ts2[j]], [0.8],
            color=plt.cm.YlGn(0.4 + 0.6*(1-np.clip(dc3[j]*4,0,1))), s=50, alpha=0.85)
    ar.annotate('', xy=(np.angle(np.mean(np.exp(2j*np.pi*ts2))), max(rs2*0.6,0.05)), xytext=(0,0),
        arrowprops=dict(arrowstyle='->', color='#33ff99', lw=3.5))
    st_s  = '✓  STABLE' if rs2 > 0.7 else 'synchronizing...'
    col_s = '#33ff99' if rs2 > 0.7 else '#aaffcc'
    ar.set_title(f'K/K_c = {K_stable/K_c:.1f}   {st_s}\nr = {rs2:.3f}\nTraditional grid',
                 color=col_s, fontsize=9, fontweight='bold')
    ar.tick_params(colors='#333')

    fig_a.suptitle(
        f'Power Grid Blackout  |  {N} generators  |  European 50Hz grid\n'
        f'Left r={rb2:.3f} (renewable)   |   Right r={rs2:.3f} (traditional)   |   '
        f'K_c = 2·sigma_omega/pi',
        color='white', fontsize=9, fontweight='bold')

ani = animation.FuncAnimation(fig_a, draw_frame, frames=NF, interval=80, blit=False)
gif_path = SAVE_PATH + 'power_grid_blackout_phase.gif'
ani.save(gif_path, writer='pillow', fps=15, dpi=100, savefig_kwargs={'facecolor': BG})
print(f"Saved: {gif_path}")
plt.close()

if IN_COLAB:
    from google.colab import files
    files.download(gif_path)
    files.download(png_path)

print()
print("=" * 60)
print(f"  r_stable   = {r_s[-1]:.3f}  (K/K_c={K_stable/K_c:.1f}) STABLE")
print(f"  r_blackout = {r_b[-1]:.3f}  (K/K_c={K_blackout/K_c:.1f}) BLACKOUT")
print(f"  Cascade at t = {t_cascade:.1f}")
