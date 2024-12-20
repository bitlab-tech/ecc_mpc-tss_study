# Threshold Diffie-Hellman Protocol using Shamir's Secret Sharing (SSS) and ECDH

This implementation combines [Shamir's Secret Sharing (SSS)](https://en.wikipedia.org/wiki/Shamir%27s_secret_sharing) with [Elliptic Curve Diffie-Hellman (ECDH)](https://en.wikipedia.org/wiki/Elliptic-curve_Diffie%E2%80%93Hellman) to distribute a private key across multiple participants, allowing them to collaboratively compute the shared secret without any participant knowing the full private key. The key idea here is that the [Lagrange interpolation](https://en.wikipedia.org/wiki/Lagrange_polynomial) on the shares' $y$-coordinates, weighted by their Lagrange coefficients based on $x$, enables the participants to recover the secret key contributions during the ECDH operation.

This POC is partly based on the research paper: [Threshold Diffie — Hellman Protocol](https://www.mathnet.ru/php/archive.phtml?wshow=paper&jrnid=pdma&paperid=536&option_lang=eng) (2021) done by **D.N. Kolegov and Yu.R. Khalniyazova**. A translation can be found [here](translation.pdf).

## Overview of the Protocol

- **Private Key Splitting**: The secret (private key) is divided into several shares using Shamir’s Secret Sharing scheme. Each share corresponds to a point on a polynomial, and at least $t$ shares are required to reconstruct the secret.
  
- **ECDH Key Exchange**: Elliptic Curve Diffie-Hellman is used to establish a shared secret between two parties, Alice and Bob, without revealing their private keys.

### Steps Involved:

1. **Private Key Sharing**:

   The private key is split into multiple shares using a polynomial of degree $t-1$.
   
   Each share corresponds to a pair $(x_i, y_i)$, where:
   $y = f(x) = \text{{secret}} + a_1 \cdot x + a_2 \cdot x^2 + \dots + a_{t-1} \cdot x^{t-1}$
   
   Here, $a_1, a_2, \dots, a_{t-1}$ are random coefficients, and the secret is the constant term of the polynomial.

2. **Lagrange Interpolation**:
   
   To combine the shares, Lagrange interpolation is used. Each share's contribution is scaled by its corresponding Lagrange coefficient:
   
   $\lambda_i = \prod_{\substack{1 \leq j \leq t \\ j \neq i}} \frac{0 - x_j}{x_i - x_j}$
   
   Part $i$ of secret is reconstructed by multiplying share $i$'s $y$-value with its Lagrange coefficient:
   
   $w_i = y_i \cdot \lambda_i$

3. **ECDH Operation**:

    In an ECDH exchange, Alice has a private key $a$ and public key $A = a \cdot G$
  
    Bob has a private key $b$ and public key $B = b \cdot G$.
    
    They can compute a shared secret using:
      - $\text{{Alice computes}}: a \cdot B = a \cdot (b \cdot G)$
      - $\text{{Bob computes}}: b \cdot A = b \cdot (a \cdot G)$
    
    The shared secret is the point $ab \cdot G$ on the elliptic curve.

4. **Threshold-Based ECDH**:

    Alice splits her private key $a$ into $n$ shares with threshold $t$ and performs ECDH with Bob using his public key $B$.

    Since:

    - Shared secret $S= a \cdot B = b \cdot A \ (1)$

    - Alice private key $a = \sum_{i=1}^{t} y_i \cdot \lambda_i \ (2)$

    From $(1)$ and $(2)$ we have:


    - $S = \sum_{i=1}^{t} (y_i \cdot \lambda_i \cdot B) = b \cdot A$

5. **Conclusion**

    In this implementation, each agent holds a share of the private key and computes a partial ECDH operation using their share.

    The Lagrange coefficient $\lambda_i$ is applied to their share, and they compute:

    $S_i = (\lambda_i \cdot y_i) \cdot B$
    Where $B$ is Bob's public key and $y_i$ is the share of Alice's private key.
    
    The final shared secret is obtained by summing all partial results:
    $S = \sum S_i$

## *Note: Finite field arithmetics

  When applying Shamir's Secret Sharing (SSS) to a private key in elliptic curve cryptography (ECC), we perform the secret sharing over the [finite field](https://en.wikipedia.org/wiki/Finite_field) defined by $n$, the order of the base point $G$ of the Elliptic curve in use, rather than the prime field $p$.

  **Reason:**

  The private key in ECC is an integer in the range $[1, n - 1]$, where $n$ is the order of the base point $G$. All cryptographic operations related to private keys (like scalar multiplication) take place modulo $n$, not 
  $p$. This is because the number of distinct points on the elliptic curve that can be generated from multiples of $G$ is limited by $n$, and the private key space is confined to this range.

  Therefore, when applying Shamir's Secret Sharing to a private key, the shares should be elements of the finite field $F_n$, where all operations (such as addition, multiplication) are performed modulo $n$.