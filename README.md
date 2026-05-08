# QuantumEncryptionSim
This repo tries to explain how the post quantum encryption works to the lowest levels of math to how will the deployment be like, featuring all the quirks and hidden facts of this amazing new secure  future-proof technology and it's implementations. As this repo can help you to get started with the quantum security world



# Deep Dive


# Proton’s Post-Quantum Cryptography: Architecture and Mathematics

## Part 1: The Core Threat and Architectural Strategy

### The Core Threat: "Store Now, Decrypt Later"
Classical public-key cryptography—specifically RSA and Elliptic Curve Cryptography (ECC)—relies on the computational difficulty of integer factorization and discrete logarithms. While these are virtually impossible for classical computers to crack, quantum computers running **Shor’s algorithm** can theoretically solve them with ease.

While a quantum computer capable of breaking a 600-digit encryption key does not yet exist, the immediate danger is the **"store now, decrypt later"** attack. Adversaries harvest and store encrypted traffic today, wait until large-scale quantum computers are viable, and decrypt it later.

### Proton’s Hybrid Implementation Strategy
Proton is not abruptly discarding current encryption standards. Instead, they use a **hybrid approach**, running classical cryptography in parallel with post-quantum cryptography (PQC). For an attacker to compromise a message, they must break *both* the classical math and the new quantum-resistant math.

#### 1. Encryption and Key Exchange: CRYSTALS-Kyber + X25519
For encrypting data and exchanging keys, Proton combines the classical **X25519** curve with **CRYSTALS-Kyber** (recently standardized by NIST as ML-KEM).
* **The Technical Math:** Kyber is a Key Encapsulation Mechanism (KEM) built on the **Module Learning With Errors (MLWE)** problem over lattices. In lattice-based cryptography, it is mathematically straightforward to traverse a multi-dimensional grid if you have the "good" (short) basis vectors (the private key), but computationally intractable to find a target point if you only have the "bad" (long) basis vectors (the public key) along with injected noise.

#### 2. Digital Signatures: CRYSTALS-Dilithium + Ed25519
To ensure the authenticity and integrity of emails, Proton uses the classical **Ed25519** combined with **CRYSTALS-Dilithium** (standardized by NIST as ML-DSA).
* **The Technical Math:** Dilithium is also a lattice-based algorithm, operating on a framework known as "Fiat-Shamir with aborts." Traditional lattice signature schemes accidentally leak data about the secret key over time. Dilithium uses a **rejection sampling** technique that ensures the signature is completely independent of the secret key, preventing side-channel data leaks.

### Why Kyber and Dilithium? (The Mobile & Hardware Context)
When NIST evaluated post-quantum algorithms, they compared lattice-based schemes (Kyber, Dilithium) against hash-based schemes (SPHINCS+).
* **The Mobile Bottleneck:** PQC comes with a severe hardware cost: larger keys and heavier ciphertexts. Hash-based algorithms require massive computational overhead, causing mobile CPUs to heat up, trigger thermal throttling, and slow down the device.
* **The Lattice Advantage:** Lattice-based algorithms strike the best balance. They require vastly less CPU time to generate and verify signatures or encapsulate keys, ensuring fast encryption without draining the battery.

### Protecting Old Data: Symmetric Re-encryption
Public-key encryption is inefficient for long-term storage. To protect older emails at rest, Proton symmetrically re-encrypts messages using a key derived from the user's password.
* **The Quantum Threat to AES:** Symmetric cryptography (like AES) is not fundamentally broken by quantum computers. A quantum algorithm known as **Grover's algorithm** can theoretically accelerate brute-force searches, effectively halving the security strength of a symmetric key. 
* **The Result:** An AES-256 key simply degrades to the strength of an AES-128 key—which is still entirely secure against quantum attacks.

---

## Part 2: The Deep Mathematical Foundation and Engineering

Post-quantum cryptography relies on a completely different branch of mathematics: **Lattice-based cryptography**.

### 1. The Foundation: Learning With Errors (LWE)
Both Kyber and Dilithium are built on **Module Learning With Errors (MLWE)**. To understand MLWE, we must look at standard LWE. 

Imagine a system of linear equations. If given a public matrix $A$ and a result vector $b$, finding the secret vector $s$ is trivial using Gaussian elimination:
$$b = A \cdot s \pmod q$$

However, if we inject a small amount of intentional, random "noise" or "error" ($e$) into the equation, the problem becomes exponentially harder:
$$b = A \cdot s + e \pmod q$$

* $A$ = Public matrix (a grid of random numbers)
* $s$ = Secret key (a vector of small numbers)
* $e$ = Error vector (very small, randomly generated numbers)
* $b$ = Public key
* $q$ = Prime modulus (math wraps around $q$, like a clock)

Because of the unknown error $e$, standard algebra fails. To find $s$, an attacker must search through a multi-dimensional lattice to find the "closest vector," which is mathematically hard even for a quantum computer. *(Kyber uses "Module" LWE, utilizing matrices of polynomials instead of standard integers to reduce key size).*

### 2. Encryption & Decryption Math (CRYSTALS-Kyber)
When Alice sends an encrypted message to Bob, they execute a Key Encapsulation Mechanism (KEM).

* **Key Generation (Bob):** Generates $A$, $s$, and $e$. He publishes his public key: $(A, b)$ where $b = A \cdot s + e$.
* **Encryption (Alice):** Generates a random vector $r$ and small errors $e_1, e_2$ to hide a shared secret $m$. She computes the ciphertext $(u, v)$:
    $$u = A^T \cdot r + e_1 \pmod q$$
    $$v = b^T \cdot r + e_2 + \text{encode}(m) \pmod q$$
* **Decryption (Bob):** Bob uses his secret key $s$ to strip away the cryptographic layers:
    $$v - s^T \cdot u \approx \text{encode}(m)$$
    Because the errors are explicitly designed to be very small, Bob can mathematically round the resulting value and perfectly recover the shared secret $m$.

### 3. Digital Signatures Math (CRYSTALS-Dilithium)
Dilithium uses **Fiat-Shamir with Aborts** to prevent key leakage.
1.  **Commitment:** Signer generates a random masking vector $y$ and creates a commitment $w$.
2.  **Challenge:** A hash function takes the commitment and message to generate a challenge $c$.
3.  **Attempted Signature:** Signer calculates a potential signature: $z = y + c \cdot s$.
4.  **The Abort Check:** The signer checks if $z$ is "too large" or "too small". If it is, the signature might leak information about $s$. The algorithm aborts, throws $z$ away, and starts over with a new $y$.

### 4. Practical Implementation (The Engineering Reality)

**The Size Problem:**
An X25519 (classical) public key is 32 bytes. A Kyber-768 (quantum) public key is 1,184 bytes. If a PGP packet exceeds the network's Maximum Transmission Unit (MTU, typically 1500 bytes), it fragments, increasing packet loss and latency.

**The Hybrid Key Derivation:**
Proton generates two shared secrets ($K_{classical}$ from X25519 and $K_{quantum}$ from Kyber). They feed both into a Key Derivation Function (KDF):
$$K_{final} = \text{KDF}(K_{classical} \parallel K_{quantum})$$
This creates a final AES key. If a quantum computer breaks Kyber tomorrow, the system is still protected by X25519, and vice versa.

# 🧮 LWE Cryptography Sandbox

Adjust the sliders in the repo HTML file to see how injecting error (noise) masks the secret key in the equation: `b = A * s + e (mod 101)`

**Your Inputs:**
* Secret Key (`s`): `INPUT[slider(minValue(1), maxValue(20)):s]`
* Public Multiplier (`A`): `INPUT[slider(minValue(1), maxValue(50)):A]`
* Error / Noise (`e`): `INPUT[slider(minValue(-5), maxValue(5)):e]`

---
**Live Calculations (Modulo 101):**

**1. Pure Result (No Noise):** If an attacker intercepts this, they can easily reverse the math to find your secret.
`A * s (mod 101)` = **`VIEW[round(({A} * {s}) % 101)]`**

**2. Masked Public Key (`b`):** This is what is actually broadcasted. The tiny error fundamentally breaks the algebra for the attacker.
`A * s + e (mod 101)` = **`VIEW[round(({A} * {s} + {e}) % 101)]`**


# REFERENCES
### https://www.mdpi.com/2079-9292/15/6/1275
### https://proton.me/blog/post-quantum-encryption
