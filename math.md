Here is the mathematical foundation behind the Noetic DNA Cryptography Standard v1.0 implementation, covering each component from encoding to error correction.

---

## 📐 Mathematical Foundations of Noetic DNA Cryptography

### 1. Key-Dependent Nucleotide Mapping

The core of the encoding protocol is a bijective mapping between 2-bit pairs and the four DNA nucleotides $\Sigma = \{A, C, G, T\}$. The mapping is derived deterministically from a cryptographic key $K \in \{0,1\}^{256}$.

**Key-Dependent Permutation**:
Let $H(K) = \text{SHA-256}(K) = (h_0, h_1, \ldots, h_{31}) \in \{0,1\}^{256}$. We use the first 4 bytes as entropy to generate a permutation $\pi \in S_4$ over the set of nucleotides.

The Fisher-Yates shuffle produces a uniformly random permutation:
$$\pi = \text{FisherYates}([A, C, G, T], \text{seed}=H(K))$$

The mapping is then fixed:
$$\mathcal{M}: \{00, 01, 10, 11\} \to \Sigma, \quad \mathcal{M}(b_1b_2) = \pi[\text{int}(b_1b_2)]$$

**Entropy Preservation**:
The mapping preserves entropy: $H(\mathcal{M}(B)) = H(B)$ for uniformly random bits $B$. The key space has $4! = 24$ possible mappings, providing $\log_2(24) \approx 4.58$ bits of key-dependent obscurity per mapping, which is combined with the larger encryption key.

```python
# Code mapping
self.mapping = derive_mapping_from_key(key)  # Returns list of (bit_pair, base)
self.bit_to_base = {bits: base for bits, base in self.mapping}
```

---

### 2. Biochemical Constraints

DNA synthesis and sequencing impose physical constraints on sequence composition.

**GC Content**:
The fraction of G and C nucleotides must be balanced for stability:
$$\text{GC} = \frac{| \{i : s_i \in \{G, C\} \} |}{|s|} \in [0.4, 0.6]$$

**Homopolymer Run Length**:
Consecutive identical bases cause sequencing errors. The maximum run length is constrained:
$$\max_{i} \{ k : s_i = s_{i+1} = \cdots = s_{i+k-1} \} \leq 4$$

**Constraint Satisfaction via Constrained Coding**:
The encoder applies a constrained coding scheme that maps unconstrained bit sequences to sequences satisfying both GC and homopolymer constraints. In our prototype, we use simple post-processing correction:

```python
def _break_homopolymers(self, sequence: str) -> str:
    # Insert a different base after runs of length max_homopolymer
    if run == self.max_homopolymer:
        alt = next(b for b in "ACGT" if b != current)
        result.append(alt)
```

---

### 3. Reed-Solomon Error Correction

DNA synthesis and sequencing introduce substitution, insertion, and deletion errors. We use Reed-Solomon (RS) codes over the finite field $\mathbb{F}_{256} = \text{GF}(2^8)$.

**Finite Field Arithmetic**:
Elements are bytes (0-255). Addition is XOR, multiplication is modulo an irreducible polynomial (typically $x^8 + x^4 + x^3 + x^2 + 1$).

**RS Encoding**:
For an RS$(n, k)$ code with $n = 255$, $k = 223$, and $2t = n - k = 32$ parity symbols:
- Message: $\mathbf{m} = (m_0, \ldots, m_{k-1}) \in \mathbb{F}_{256}^k$
- Generator polynomial: $g(x) = \prod_{i=0}^{2t-1} (x - \alpha^i)$ where $\alpha$ is a primitive element of $\mathbb{F}_{256}$.
- Codeword: $c(x) = m(x) \cdot x^{2t} + r(x)$ where $r(x) = m(x) \cdot x^{2t} \bmod g(x)$.

**RS Decoding (Berlekamp-Massey)**:
Given received word $\mathbf{r} = \mathbf{c} + \mathbf{e}$:
1. Compute syndromes $S_j = r(\alpha^j)$ for $j = 0, \ldots, 2t-1$.
2. Find error locator polynomial $\Lambda(x)$ via Berlekamp-Massey algorithm.
3. Find error positions (roots of $\Lambda(x)$) via Chien search.
4. Compute error magnitudes via Forney's algorithm.
5. Correct: $\hat{c}_i = r_i - e_i$.

Our implementation uses `reedsolo.RSCodec(32)` which can correct up to 16 byte errors per 255-byte block.

```python
self.ecc = ReedSolomonECC(nsym=32)
protected_data = self.ecc.encode(data)
original_data = self.ecc.decode(protected_data)
```

---

### 4. HKDF Key Derivation

We use HMAC-based Key Derivation Function (HKDF) to derive multiple independent keys from a single master entropy source.

**HKDF-Extract**:
$$\text{PRK} = \text{HMAC-Hash}(\text{salt}, \text{IKM})$$
where IKM is the input keying material (master entropy), salt is optional (defaults to zeros).

**HKDF-Expand**:
$$T(0) = \text{empty}$$
$$T(i) = \text{HMAC-Hash}(\text{PRK}, T(i-1) \| \text{info} \| \text{byte}(i))$$
for $i = 1, \ldots, \lceil L / \text{HashLen} \rceil$, where $L$ is the desired output length.

The output is $T(1) \| T(2) \| \ldots$ truncated to $L$ bytes.

```python
hkdf = HKDF(algorithm=hashes.SHA256(), length=length, salt=None, info=info.encode())
return hkdf.derive(self.master_entropy)
```

---

### 5. DNA Steganography

We hide a payload DNA sequence within a carrier sequence using key-dependent insertion positions.

**Combinatorial Hiding**:
Let carrier have length $N_c$ and payload length $N_p$. The number of ways to choose insertion positions is $\binom{N_c + N_p}{N_p}$. With a key $K$, we select a specific combination via pseudo-random permutation.

**Key-Dependent Position Generation**:
$$p_i = H(K \| i) \bmod (N_c + i), \quad i = 1, \ldots, N_p$$
The positions are then sorted to determine insertion indices.

```python
pos = hash_val[i % len(hash_val)] % (len(carrier) + 1)
positions.sort()
result.insert(pos + offset, payload[i])
```

---

### 6. Cryptographic Signatures (ECDSA)

We use Elliptic Curve Digital Signature Algorithm over the NIST P-256 curve (secp256r1).

**Curve Parameters**:
$y^2 = x^3 - 3x + b$ over $\mathbb{F}_p$ with $p = 2^{256} - 2^{224} + 2^{192} + 2^{96} - 1$.
Generator point $G$ of prime order $n$.

**Key Generation**:
Private key $d \in_R [1, n-1]$, public key $Q = d \cdot G$.

**Signing**:
Given message hash $z = \text{SHA-256}(m)$:
1. Select random $k \in [1, n-1]$
2. Compute $R = k \cdot G = (x_R, y_R)$
3. Compute $r = x_R \bmod n$ (if $r = 0$, retry)
4. Compute $s = k^{-1}(z + r \cdot d) \bmod n$ (if $s = 0$, retry)
5. Signature is $(r, s)$

**Verification**:
1. Compute $w = s^{-1} \bmod n$
2. Compute $u_1 = z \cdot w \bmod n$, $u_2 = r \cdot w \bmod n$
3. Compute $P = u_1 \cdot G + u_2 \cdot Q = (x_P, y_P)$
4. Valid if $x_P \equiv r \pmod n$

```python
signature = self._private_key.sign(data, ec.ECDSA(hashes.SHA256()))
self._public_key.verify(signature, data, ec.ECDSA(hashes.SHA256()))
```

---

### 7. Entropy Estimation and One-Time Pad

For the OTP mode, we derive a key of equal length to the message.

**Information-Theoretic Security**:
The OTP ciphertext $C = M \oplus K$ provides perfect secrecy if $K$ is truly random, independent of $M$, and $H(K) \geq H(M)$. The DNA entropy source must provide min-entropy $H_\infty(K) \geq |M|$.

**Min-Entropy Estimation**:
$$H_\infty(X) = -\log_2 \left( \max_{x} P(X = x) \right)$$
For our simulated DNA TRNG, we use cryptographic hashing to extract uniform bits from the entropy source.

```python
def derive_otp_key(self, length: int) -> bytes:
    return self._hkdf_derive("otp", length)
```

---

## 💎 Summary Table

| Component | Mathematical Foundation | Key Formula |
|:---|:---|:---|
| Nucleotide Mapping | Permutation group $S_4$ | $\mathcal{M} = \pi \circ \text{int}$ |
| GC Constraint | Frequency ratio | $\text{GC} \in [0.4, 0.6]$ |
| Homopolymer Limit | Maximum run length | $\max \text{run} \leq 4$ |
| Error Correction | Reed-Solomon over $\mathbb{F}_{256}$ | $c(x) = m(x)x^{2t} + (m(x)x^{2t} \bmod g(x))$ |
| Key Derivation | HKDF with HMAC-SHA256 | $\text{PRK} = \text{HMAC}(salt, IKM)$ |
| Steganography | Combinatorial insertion | $\binom{N_c+N_p}{N_p}$ choices |
| Digital Signature | ECDSA over secp256r1 | $s = k^{-1}(z + rd) \bmod n$ |
| One-Time Pad | Information theory | $H(M \| C) = H(M)$ |

These mathematical foundations ensure that the Noetic DNA Cryptography Standard v1.0 provides strong cryptographic guarantees while respecting the physical constraints of DNA as a storage medium.
