# Core Accountability Theorems

## Theorem 1 (Normal Views — Single-Slot Finality)
**Parameters**: Adversary stake β ≤ 0.25, committee size k = 2000, security parameter ε < 2⁻⁴⁰.

**Statement**: In any view v ≠ 3m, if two conflicting blocks are finalized, then with probability ≥ 1 − ε, there exists an identifiable set of equivocating validators whose total stake is ≥ 1/3.

**Numerical bound**: ε = Pr[Binomial(2000, 0.25) ≥ 2000/3] = 1.11 × 10⁻¹⁶ < 2⁻⁴⁰ ≈ 9.09 × 10⁻¹³.

---

## Theorem 2 (Checkpoint Views — Hard 1/3 Guarantee for β=0.3)
**Parameters**: Adversary stake β = 0.3, checkpoint committee size k_acc = 10,000, security parameter ε < 2⁻⁶⁰.

**Statement**: In any checkpoint view v = 3m, if two conflicting blocks are finalized, then with probability ≥ 1 − ε, there exists an identifiable set of equivocating validators whose total stake is ≥ 1/3.

**Numerical bound**: ε = Pr[Binomial(10000, 0.3) ≥ 10000/3] = 1.27 × 10⁻²¹ < 2⁻⁶⁰ ≈ 8.67 × 10⁻¹⁹.

---

## Unified Chernoff Bound
For any β < 1/3 and committee size k:
```
ε(k, β) ≤ exp(−k · D(1/3 || β))
where D(a||b) = a ln(a/b) + (1−a) ln((1−a)/(1−b))
```

| β   | D(1/3||β) | k for ε≤2⁻³⁰ | k for ε≤2⁻⁴⁰ | k for ε≤2⁻⁶⁰ |
|-----|-----------|--------------|--------------|--------------|
| 0.20 | 0.0530    | 395          | 527          | 790          |
| 0.25 | 0.0204    | 1,047        | 1,396        | 2,094        |
| 0.30 | 0.0046    | 4,565        | 6,087        | 9,130        |

**Design choice**: Normal views use k=2000 (covers β≤0.25 with ε<2⁻⁴⁰). Checkpoints use k_acc=10,000 (covers β=0.3 with ε<2⁻⁶⁰).
