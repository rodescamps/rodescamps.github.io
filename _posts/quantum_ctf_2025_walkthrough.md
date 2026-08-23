---
title: "DEF CON Quantum CTF 2025: Challenge Walkthroughs"
date: 2026-08-12
categories:
  - CTF
tags:
  - ctf
  - quantum
  - qkd
  - quantum-computing
excerpt: "Walkthrough of the two challenges I created for the Quantum Village CTF 2025: The Shapiro Incident and Navigation Log Extract."
---

Last year, I had the chance to participate in the Quantum Village CTF at DEF CON 32. What initially started as a way to discover a field I barely knew ended with a third place in the competition and, more importantly, a lot of new things to learn around quantum computing and quantum security. I wrote a [walkthrough of that experience](/Quantum-CTF-2024/) afterwards.

For DEF CON 33 in 2025, I had the opportunity to look at the CTF from the other side and create two challenges myself.

I wanted to avoid challenges where the solution was simply to recognise a formula and apply it. Both challenges therefore hide the quantum concept behind another layer: the first one looks more like a small forensic investigation around a BB84 implementation, while the second one hides bytes inside directions reconstructed from quantum measurements.

This post walks through both challenges and their intended solutions.

# The Shapiro Incident

## Description

> Alice and Bob have a problem.
>
> A review of their BB84 experiment showed no obvious increase in the error rate, yet Professor Sylvia N. Shapiro somehow managed to recover information from the exchange.
>
> The lab was using an aging optical setup, and several diagnostic channels had been left enabled during the experiment. Alice and Bob recovered the logical exchange together with a few instrument traces, but they do not know which one could have leaked useful information.
>
> Can you determine what Professor Shapiro was able to observe and reconstruct the data she recovered?
>
> The recovered bytes contain the flag.

The following files were provided:

- `exchange_obf.csv`: the BB84 exchange, with Alice and Bob's bases and values;
- `charge_histogram.npy`: analogue detector measurements associated with the pulses;
- `timing_jitter.npy`: timing measurements;
- `detector_temp.csv`: a temperature log.

There is deliberately no ciphertext in this challenge. The information that Eve could recover directly forms the flag.

## Walkthrough

At first sight, the CSV mostly looks like a normal BB84 exchange:

```text
pid,bA,vA,bB,vB
0,Z,0,X,1
1,X,1,X,1
2,Z,0,Z,0
...
```

The first obvious BB84 operation is the sifting step. Alice and Bob only keep measurements for which they used the same basis:

```python
same_basis = df["bA"] == df["bB"]
```

Doing only this, however, does not produce anything useful. The important part of the challenge is to understand why Professor Shapiro could know **some** of these bits without disturbing the exchange.

The other files are the main clue.

### Looking at the lab measurements

The two `.npy` files are simply NumPy arrays. They can be loaded directly:

```python
import numpy as np

charge = np.load("charge_histogram.npy")
jitter = np.load("timing_jitter.npy")

print(charge.shape)
print(jitter.shape)
```

The arrays have one entry per pulse, in the same order as `pid` in the CSV. Although the charge file is named `charge_histogram.npy`, it contains the individual measurements; we can build the histogram ourselves.

For example:

```python
import numpy as np
import matplotlib.pyplot as plt

charge = np.load("charge_histogram.npy")

plt.hist(charge, bins=100)
plt.xlabel("Charge (a.u.)")
plt.ylabel("Occurrences")
plt.show()
```

The result is much more interesting than the timing or temperature traces: there are two very distinct populations.

One group is centred around a low charge, while a second group is centred much higher.

At this point, the challenge turns back into quantum security.

### From charge to photon number

A practical BB84 implementation does not necessarily emit perfect single photons. Weak coherent optical pulses can contain zero, one, or multiple photons. The existence of multi-photon pulses is exactly what makes **Photon Number Splitting**, or PNS, attacks interesting.

The simplified model used for the challenge is that the analogue charge gives us a way to distinguish the two populations: the low-charge events represent single-photon pulses and the high-charge events represent multi-photon pulses.

This is intentionally exaggerated to make the forensic clue visible. It should not be understood as "every single-photon detector outputs twice the charge when two photons arrive". Conventional SPADs operated in Geiger mode generally behave as threshold detectors. On the other hand, photon-number-resolving detector architectures, including SNSPD-based approaches, can extract photon-number information from properties such as pulse amplitude, rising edge or spatially multiplexed responses.

The important point for the challenge is simply:

```text
high analogue response -> likely multi-photon pulse
```

Plotting the values makes the separation clear enough that a threshold around `0.9` safely separates the two artificial populations:

```python
multi = charge > 0.9
```

### Why multi-photon pulses matter

Suppose Alice emits two photons carrying the same BB84 state.

Eve can, in the idealised PNS model:

1. determine that the pulse contains multiple photons;
2. keep one photon in quantum memory;
3. forward another photon to Bob;
4. wait for the public basis reconciliation;
5. measure her stored photon in the correct basis.

She then learns the bit without performing an intercept/resend measurement in a random basis and therefore without introducing the usual BB84 errors on that pulse.

With a single photon, she cannot do the same: if she measures it before knowing the basis, she risks disturbing the state. This is why the distinction between the two populations in the charge measurements matters.

The title also contained a small additional hint. If we take **Professor**, the middle initial **N.**, and **Shapiro**, we end up with P.N.S.

### Recovering the flag

We now combine the two conditions:

- Alice and Bob used the same basis;
- the pulse belongs to the high-charge population.

The associated Alice bits are the bits Professor Shapiro could recover in the model used by the challenge.

The full solver is short once the vulnerability is identified:

```python
import re
import numpy as np
import pandas as pd

df = pd.read_csv("exchange_obf.csv")
charge = np.load("charge_histogram.npy")

# Multi-photon candidates + BB84 basis reconciliation
mask = (charge > 0.9) & (df["bA"] == df["bB"])

bits = df.loc[mask, "vA"].astype(np.uint8).to_numpy()

# Only complete bytes
bits = bits[:len(bits) // 8 * 8]

data = np.packbits(bits, bitorder="big").tobytes()

print(data)

flag = re.search(rb"qv\{[^}]+\}", data)
if flag:
    print(flag.group().decode())
```

The first bytes give:

```text
qv{photon_number_splitters_never_die}
```

The remaining selected pulses contain random bits, so printing the complete byte stream also produces garbage after the closing brace.

The flag is therefore:

```text
qv{photon_number_splitters_never_die}
```

### A note on the real attack

The challenge deliberately simplifies the full QKD protocol.

A PNS attack does **not** automatically mean that Eve gets the complete final secret key of a properly implemented QKD system. If she only learns a fraction of the sifted key, privacy amplification is precisely supposed to remove that partial knowledge. Modern weak-coherent-pulse BB84 implementations also commonly use **decoy states** to estimate the single-photon contribution and prevent Eve from hiding a PNS strategy inside normal channel loss.

For the CTF, I instead encoded the flag directly into the subset of bits that would be exposed to the PNS attacker. This keeps the focus on recognising the implementation weakness rather than reproducing the complete information-reconciliation and privacy-amplification phases of a QKD stack.

Useful references on this topic are:

- G. Brassard, N. Lütkenhaus, T. Mor and B. C. Sanders, [*Limitations on Practical Quantum Cryptography*](https://doi.org/10.1103/PhysRevLett.85.1330), 2000.
- H.-K. Lo, X. Ma and K. Chen, [*Decoy State Quantum Key Distribution*](https://arxiv.org/abs/quant-ph/0411004), 2005.
- T. Schapeler et al., [*How well can superconducting nanowire single-photon detectors resolve photon number?*](https://arxiv.org/abs/2310.12471), 2023.

# Navigation Log Extract

## Description

The second challenge did not mention quantum states explicitly. Instead, it provided the following navigation log:

> The leak swears that every path is fixed in two stages.
>
> First comes the climb — counted out from the equator up toward the pole, counted like the points on a full mariner's compass rose.
>
> Then the turn — enough steps that a careful climber could match the number of teeth on a standard ship's wheel.
>
> Once the heading is set, the course is locked, and the package moves without a word. What we recovered are the final readings from three different instruments, taken just before the cargo was lost.
>
> They say each entry is a piece of a larger whole. Put them together, and you'll see the message they tried to hide.

Twenty blocks followed, containing measurements such as:

```text
Block 0

X: 1 -> 0.160187
Y: 0 -> 0.858212
Y: 1 -> 0.142310
Z: 0 -> 0.574913
Z: 1 -> 0.426103

Block 1

X: 1 -> 0.486279
Z: 0 -> 0.574931
Z: 1 -> 0.427967
Y: 0 -> 0.006345
Y: 1 -> 0.995331
```

The same structure continues until Block 19.

The nautical wording is not just flavour text. It describes how the information stored in each qubit direction must be converted back into a byte.

## Walkthrough

### Reconstructing a qubit from X, Y and Z measurements

A single-qubit state can be represented as a point on the Bloch sphere.

Its Bloch vector is:

```text
r = (x, y, z)
```

where `x`, `y` and `z` are the expectation values of the Pauli X, Y and Z observables.

For the X measurement, for example:

```text
x = P(X=0) - P(X=1)
```

and similarly:

```text
y = P(Y=0) - P(Y=1)
z = P(Z=0) - P(Z=1)
```

Some blocks only give one of the two outcomes for an axis. Since the two probabilities sum to approximately one, we can recover the same expectation value from either side:

```text
x = 2 * P(X=0) - 1
```

or:

```text
x = 1 - 2 * P(X=1)
```

Because the challenge contains a little measurement noise, when both values are available I average the two estimates.

For Block 0 this gives approximately:

```text
x =  0.679626
y =  0.715902
z =  0.148810
```

The norm is already close to one. Normalising it gives:

```text
(0.680801, 0.717140, 0.149067)
```

We can then convert the Cartesian Bloch vector to spherical coordinates:

```python
theta = acos(z)
phi = atan2(y, x) % (2*pi)
```

For Block 0:

```text
theta = 81.43 degrees
phi   = 46.49 degrees
```

The quantum part is now mostly finished. The rest of the solution is hidden in the navigation text.

### "A full mariner's compass rose"

A traditional full compass rose has 32 points.

Thirty-two possibilities require exactly five bits:

```text
2^5 = 32
```

The first part of every encoded byte is therefore a five-bit quantisation of the polar angle `theta`.

The challenge divides the full `[0, pi]` polar range into 32 intervals:

```python
theta_index = floor(theta / pi * 32)
```

For Block 0:

```text
theta/pi * 32 = 14.4759...
theta_index = 14
```

In binary:

```text
14 = 01110
```

### "The turn" and the ship's wheel

The second direction is the azimuth `phi`: the rotation around the sphere.

The wording points to eight directions, which need three bits:

```text
2^3 = 8
```

Here the direction is snapped to the closest of eight headings:

```python
phi_index = round(phi / (2*pi) * 8) % 8
```

For Block 0:

```text
phi/(2*pi) * 8 = 1.033...
phi_index = 1
```

or:

```text
1 = 001
```

We now have exactly eight bits:

```text
01110 001
```

which can also be written as:

```python
byte = (theta_index << 3) | phi_index
```

For Block 0:

```text
(14 << 3) | 1 = 113
```

and ASCII 113 is:

```text
q
```

That is a good indication that the navigation story is leading in the right direction.

### Solving all 20 blocks

The following solver takes a text file containing the full challenge output and reconstructs every byte:

```python
import math
import re
from pathlib import Path

text = Path("navigation_log.txt").read_text()

raw_blocks = re.split(r"Block\s+\d+\s*", text)[1:]


def expectation(values, axis):
    estimates = []

    if (axis, 0) in values:
        estimates.append(2 * values[(axis, 0)] - 1)

    if (axis, 1) in values:
        estimates.append(1 - 2 * values[(axis, 1)])

    return sum(estimates) / len(estimates)


decoded = []

for raw in raw_blocks:
    values = {}

    for axis, bit, probability in re.findall(
        r"([XYZ]):\s*([01])\s*->\s*([0-9.]+)", raw
    ):
        values[(axis, int(bit))] = float(probability)

    x = expectation(values, "X")
    y = expectation(values, "Y")
    z = expectation(values, "Z")

    # Remove the small amount of simulated measurement noise
    norm = math.sqrt(x*x + y*y + z*z)
    x /= norm
    y /= norm
    z /= norm

    theta = math.acos(max(-1.0, min(1.0, z)))
    phi = math.atan2(y, x) % (2 * math.pi)

    # 32 polar intervals -> 5 bits
    theta_index = int(math.floor(theta / math.pi * 32))

    # 8 azimuth headings -> 3 bits
    phi_index = int(round(phi / (2 * math.pi) * 8)) % 8

    decoded.append((theta_index << 3) | phi_index)


print(bytes(decoded).decode())
```

The output is:

```text
qv{QNT0M_T0M0GR4PHY}
```

The flag is therefore:

```text
qv{QNT0M_T0M0GR4PHY}
```

The name is of course a reference to **quantum state tomography**: the X, Y and Z measurements allow us to reconstruct the state of every qubit before translating its direction back into the hidden data.

IBM's [Bloch sphere introduction](https://quantum.cloud.ibm.com/learning/courses/general-formulation-of-quantum-information/density-matrices/bloch-sphere) is a useful reference for the relation between qubit states, Pauli observables and the `(theta, phi)` representation.

# Conclusion

These were the two challenges I created for the 2025 edition of the Quantum Village CTF.

I liked the idea of keeping the quantum part slightly hidden instead of explicitly asking the player to "perform a PNS attack" or "reconstruct the Bloch sphere". In both cases, the first difficulty is therefore to identify what quantum concept is actually represented by the data:

- in **The Shapiro Incident**, an analogue forensic trace reveals which BB84 pulses are interesting to an eavesdropper;
- in **Navigation Log Extract**, apparently unrelated X/Y/Z probabilities become directions on the Bloch sphere, while the navigation story explains how to turn these directions into bytes.

This is also what I find interesting in quantum security challenges. The mathematics is obviously part of it, but the hacking aspect appears when imperfect implementations, measurements and unexpected representations are added around the theory.

The Quantum Village describes its CTF as a way to hack quantum technologies open, and I think these kinds of challenges fit well with that objective: understand the physics just enough to find where the information is hiding, and then treat the rest as a hacking problem.
