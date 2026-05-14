# RSA-CRT PACD Fault Injection Attack

A SageMath implementation of a lattice-based fault injection attack on RSA-CRT
digital signatures, based on Barbu, Grémy, and Lescuyer (IACR ePrint 2024/1125).
The attack exploits partial faults in the intermediate variable sq to recover the
private key via PACD reduction to HNP and LLL lattice reduction.

---

## Academic Transparency & AI Disclosure

**Relationship to the reference paper:** This project is a faithful simplified
implementation of the attack methodology described in Barbu et al. (2024). It does
not introduce new ideas or extend the original work. Every design step follows the
paper directly, targeting RSA-512 instead of RSA-1024–8192, and using SageMath's
standard LLL instead of flatter or BDD with Predicate. The value lies in the
**working end-to-end implementation with quantitative evaluation**, not in novel
contributions.

**What "simplified" means here:** The original paper targets RSA-1024 to RSA-8192
using flatter (Ryan & Heninger, 2023) and BDD with Predicate for lattice reduction.
This implementation uses SageMath's standard LLL on RSA-512. The smaller RSA
dimension compensates for LLL's lower reduction power — the qualitative behavior
is identical, the quantitative parameters differ as expected.

**AI usage:** This project was developed with AI assistance across all stages,
including:

- Understanding the mathematical foundations of RSA-CRT, PACD, HNP, and LLL
- Implementing and debugging the SageMath attack cells
- Structuring the LaTeX report and evaluating experimental results

---

## Project Overview

This project implements and evaluates two levels of fault injection attack against
RSA-CRT signature implementations:

**Level 1 — Bellcore Attack (1997):** A complete fault zeroing sq yields the prime
factor p directly via a single GCD computation. One corrupted signature is sufficient
to recover the full private key.

**Level 2 — PACD Attack (IACR 2024):** A partial fault zeroing only L MSBs of sq
produces signatures that are approximate multiples of q. Collecting t such signatures
and applying LLL lattice reduction recovers p. The paper's key contribution is
reducing PACD to HNP by placing -N on the matrix diagonal instead of -s₀, lowering
the required bits from 32 (CHES 2012) to 7 (IACR 2024).

A multiplicative message blinding countermeasure is also demonstrated, confirming
complete protection against the PACD attack.

---

## Repository Structure

```
rsa-crt-pacd-attack/
├── docs/
│   └── pacd_attack_report.pdf    # LaTeX report — methodology, results, analysis
├── src/
│   └── pacd_attack.ipynb         # SageMath notebook — all 6 attack cells
├── LICENSE
└── README.md
```

---

## Attack Overview

### RSA-CRT and Garner's Formula

RSA-CRT computes the signature as:

```
sp = m^dp mod p
sq = m^dq mod q
s  = sq + q · [(sp - sq) · iq mod p]     ← Garner's Formula
```

The variable sq exists temporarily in device memory — this is the attack target.
All recombination is performed mod p via iq = q⁻¹ mod p, which means any fault
in sq is algebraically tied to p — the attack recovers p, not q.

### Bellcore Attack

```
sq_faulted = 0
gcd(s - s_faulted, N) = p
```

One corrupted signature → full private key recovered instantly.

### PACD Attack — Lattice Construction

For t corrupted signatures with L MSBs zeroed, build a (t+1)×(t+1) matrix:

```
M[0,0]   = 2^ρ          (ρ = nbits(q) - L)
M[0,i]   = s̃ᵢ           (corrupted signatures)
M[i,i]   = -N            (HNP reduction — key innovation vs CHES 2012)
```

LLL finds the short vector v = (2^ρ · p, p·r₁, ..., p·rₜ).
Recovery: `gcd(v[0], N) = p`.

---

## Results

### Success Rate — My Implementation (RSA-512)

| L (bits) | Signatures (t) | Lattice size | Success Rate | Time (s) |
|----------|----------------|--------------|--------------|----------|
| 32       | 10             | 11 × 11      | 100%         | 0.03     |
| 32       | 50             | 51 × 51      | 100%         | 3.60     |
| 7        | 30             | 31 × 31      | 0%           | 0.67     |
| 7        | 50             | 51 × 51      | 20%          | 3.59     |
| 7        | 61             | 62 × 62      | 80%          | 6.58     |
| 7        | 70             | 71 × 71      | 100%         | 9.84     |

### Comparison with Paper (Barbu et al., 2024)

|                      | Paper (2024)               | This implementation      |
|----------------------|----------------------------|--------------------------|
| RSA size             | 1024–8192 bits             | 512 bits                 |
| Min bits (L)         | 7 bits                     | 7 bits ✓                 |
| Signatures (L=7)     | ~2500                      | 61–70                    |
| Algorithm            | flatter + BDD w/ Predicate | SageMath LLL             |
| Hardware             | 64 cores, ~70h             | 1 core (single-threaded) |
| Success rate (L=7)   | 63% / 2500 signatures      | 80–100% / 61–70 sig.     |

The difference in signatures needed (61–70 vs ~2500) reflects the smaller RSA
dimension, not a superior implementation. The target vector norm scales with
q ≈ 2^256 (RSA-512) vs q ≈ 2^512 (RSA-1024), making the problem structurally
easier and standard LLL sufficient.

### Countermeasure Validation

Multiplicative blinding was tested over 20 trials with t=70 and L=7:

```
Success rate with blinding: 0 / 20
→ [CONFIRMED] Multiplicative blinding completely blocks the PACD attack.
```

---

## Running the Project

**Requirements:** SageMath (tested on CoCalc).

Open `src/pacd_attack.ipynb` in CoCalc or any SageMath-compatible environment
and run cells in order:

| Cell | Content |
|------|---------|
| 1    | RSA-CRT key generation and signature verification |
| 2    | Partial fault injection — generate t corrupted signatures |
| 3    | Lattice construction (SDA/HNP) + LLL + factor recovery |
| 4    | Bellcore classical attack — complete fault, single signature |
| 5    | Success rate evaluation across L and t parameters |
| 6    | Multiplicative blinding countermeasure demonstration |

---

## References

- G. Barbu, L. Grémy, and R. Lescuyer, "Revisiting PACD-based Attacks on RSA-CRT,"
  IACR Cryptology ePrint Archive, Report 2024/1125, 2024.
  https://eprint.iacr.org/2024/1125

- P.-A. Fouque et al., "Attacking RSA-CRT Signatures with Faults on Montgomery
  Multiplication," CHES 2012, LNCS vol. 7428, pp. 447–462. Springer, 2012.
  https://link.springer.com/chapter/10.1007/978-3-642-33027-8_26

- A. K. Lenstra, H. W. Lenstra Jr., and L. Lovász, "Factoring Polynomials with
  Rational Coefficients," Mathematische Annalen, 261:515–534, 1982.
  https://doi.org/10.1007/BF01457454

The full project report is available in [`docs/pacd_attack_report.pdf`](./docs/pacd_attack_report.pdf).

---

## Development Notes

This is a 4th-year Cryptology course project, developed with continuous AI assistance
throughout. It is a simplified reproduction of the Barbu et al. (2024) methodology —
the contribution lies in the working end-to-end implementation with quantitative
evaluation on RSA-512, not in novel research.

---

## License

This project is licensed under the MIT License. See the [LICENSE](./LICENSE) file
for details.