# get_D() and get_y() in Curve StableSwap

Two of the most math-heavy functions in Curve V1 are `get_D()` and `get_y()`.

In Curve V1 (StableSwap), `D` behaves similarly to `k` in Uniswap V2 — the larger `D` is, the “further out” the price curve will be. `D` changes — and needs to be recomputed — after liquidity is added or removed, or a fee changes the pool balance.

If a curve pool holds two tokens, `x` and `y`, the StableSwap invariant is

$$
4A(x + y) + D = 4AD+\frac{D^3}{4xy}
$$

The “amplification factor” `A`, for our purposes, can be treated as a constant.

The function `get_y()` is used during a swap. Similar to `k` in Uniswap V2, `D` must be held constant during a swap (ignoring fees). Specifically, given a new value for `x`, it computes the value of `y` that keeps the equation balanced.

Curve can hold more than 2 tokens in the pool (e.g. 3pool holds USDT, USDC, and DAI). In this context, `get_y()` means change the balance of a particular token `x`, hold the other balances constant, and compute what the new balance of a selected token must be to keep the invariant balanced.

The invariant for `n` tokens is:

$$
An^nS + D = ADn^n+\frac{D^{n+1}}{n^nP}
$$

where, $S$ is the sum of the balances of the tokens ($x_0 + x_1 + … + x_n$) and $P$ is the product of the balances ($x_0x_1...x_n)$ where $x_i$ is the balance of token `i`.

In the whitepaper, $S$ is written as $\sum x_i$ and $P$ is written as $\prod x_i$. The whitepaper equation is replicated below:

$$
An^n\sum x_i + D=ADn^n+\frac{D^{n+1}}{n^n\prod x_i}
$$

We will use $S$ and $P$ instead of the sum and product notation.

This article shows algebraically step-by-step how the code for `get_D()` and `get_y()` are derived from the StableSwap invariant.

We assume that the pools can hold an arbitary number $n$ tokens, so the formulas will reflect that.

## Computing $D$ with `get_D()`

In `get_D()` we are presented with a set of balances `x_0, x_1, ..., x_n` and we are to compute `D`.

It is not possible to algebraically solve

$$
An^nS + D = ADn^n+\frac{D^{n+1}}{n^nP}
$$

for $D$. Instead, we need to apply Newton’s method to solve it numerically. To do so, we create a function $f(D)$ which is 0 when the equation is balanced.

$$
f(D)=\frac{D^{n+1}}{n^nP}+An^nD-D-An^nS
$$

and we compute the derivative $f'(D)$ with respect to $D$ as:

$$
f'(D) = \frac{(n+1)D^n}{n^nP}+An^n-1
$$

We can iteratively solve for $D$ using:

$$
D_\text{next}=D-\frac{f(D)}{f'(D)}
$$

It will be helpful to express $f'(D)$ with $D$ in the denominator. First we multiply the top and bottom of the left fraction by $D$:

$$
f'(D) = \frac{\frac{(n+1)D^{n+1}}{n^nP}}{D}+An^n-1
$$

And then combine $f'(D)$ into a single fraction:{

$$
f'(D) = \frac{\frac{(n+1)D^{n+1}}{n^nP}}{D}+\frac{(An^n-1)D}{D}=\frac{\frac{(n+1)D^{n+1}}{n^nP}+(An^n-1)D}{D}
$$

We can re-write Newton’s method to have a common denominator:

$$
D_\text{next}=D-\frac{f(D)}{f'(D)}=\frac{Df'(D)}{f'(D)}-\frac{f(D)}{f'(D)}=\frac{\color{violet}{D}\color{red}{f'(D)}-\color{green}{f(D)}}{\color{red}{f'(D)}}
$$

By substituting the $f(D)$ and $f'(D)$ from earlier into the re-written Newton’s method formula we get:

$$
=\frac{\color{violet}{D}\color{red}{\frac{(n+1)\frac{D^{n+1}}{n^nP}+(An^n-1)D}{D}}-\color{green}{(\frac{D^{n+1}}{n^nP}+An^nD-D-An^nS)}}{\color{red}{\frac{(n+1)\frac{D^{n+1}}{n^nP}+(An^n-1)D}{D}}}
$$

Since we re-arranged $f'(D)$ to have $D$ in the denominator, the $\color{violet}D\color{red}f'(D)$ term will cancel nicely:

$$
=\frac{(n+1)\frac{D^{n+1}}{n^nP}+(An^n-1)D-\color{green}{(\frac{D^{n+1}}{n^nP}+An^nD-D-An^nS)}}{\frac{(n+1)\frac{D^{n+1}}{n^nP}+(An^n-1)D}{D}}
$$

Distribute all the terms to remove the parenthesis in the numerator:

$$
=\frac{\frac{D^{n+1}}{n^nP}+\frac{nD^{n+1}}{n^nP}+An^nD-D-\frac{D^{n+1}}{n^nP}-An^nD+D+An^nS}{\frac{(n+1)\frac{D^{n+1}}{n^nP}+(An^n-1)D}{D}}
$$

This leads to a lot of cancellations

$$
=\frac{\cancel{\frac{D^{n+1}}{n^nP}}+\frac{nD^{n+1}}{n^nP}+\cancel{An^nD}-\cancel{D}-\cancel{\frac{D^{n+1}}{n^nP}}-\cancel{An^nD}+\cancel{D}+An^nS}{\frac{(n+1)\frac{D^{n+1}}{n^nP}+(An^n-1)D}{D}}\\
$$

$$
=\frac{\frac{nD^{n+1}}{n^nP}+An^nS}{\frac{(n+1)\frac{D^{n+1}}{n^nP}+(An^n-1)D}{D}}
$$

We multiply the numerator and denominator by $D$

$$
=\frac{(An^nS+n\frac{D^{n+1}}{n^nP})D}{(An^n-1)D+(n+1)\frac{D^{n+1}}{n^nP}}
$$

If we define $D_p$ as

$$
D_p=\frac{D^{n+1}}{n^nP}
$$

and substitute $D_p$ we get

$$
D_\text{next}=\frac{(An^nS+D_pn)D}{(An^n-1)D+(n+1)D_p}
$$

### Comparison to the original source code

This matches exactly what is in the [Vyper code](https://github.com/curvefi/curve-contract/blob/master/contracts/pools/3pool/StableSwap3Pool.vy#L210):

![Screenshot 2024-07-25 at 2.16.30 PM.png](get_D()%20and%20get_y()%20in%20Curve%20StableSwap%20815a4ed4178948048d80000407b65553/Screenshot_2024-07-25_at_2.16.30_PM.png)

The variable $D_p$ was defined as:

```python
D_P: uint256 = D # D_P = S

for _x in xp:
    D_P = D_P * D / (_x * N_COINS)
```

`xp` is the number of tokens, so the loop will run `n` times. Therefore, we have $D$ multiplied by itself `n` times in the denominator

$$
D_p=\frac{D^{n+1}}{n^n\prod_{i=1}^nx_i}
$$

## Computing `y` with `get_y()`

The idea is we force one of the $x_i$ to take on a new value (the code calls this `x`) and calculate the correct value for another $x_j$ (where $i \neq j)$ such that the equation stays balanced. The balance of the other tokens remains unchanged. $x_j$ is referred to as $y$.

Although a StableSwap pool could have multiple tokens, it is only possible to exchange two of those tokens at a time. `get_y()`.

Again we have the same invariant

$$
An^nS + D = ADn^n+\frac{D^{n+1}}{n^nP}
$$

$D$, $A$, and $n$ are fixed, but we will be changing two of the values in $S$ and $P$

$$
\begin{align*}
S &= x_0+x_1+...+x_n\\
P &= x_0x_1...x_n
\end{align*}
$$

Therefore, we need to adjust the formula a bit, since $S$ and $P$ contain the values we are computing for.

$S'$ will be the sum of all the balances *except* the new balance of token $x_i$ that we are trying to solve for, and $P$ will be the product of the balances of all the tokens, *except* for the one we are trying to solve for.

In other words,

$$
\begin{align*}
S &= S'+y \\
P &= P'y
\end{align*}
$$

To stay consistent with the code, we will call the token whose new balance we are trying to compute $y$.

The formula then becomes

$$
An^n(S'+y) + D = ADn^n+\frac{D^{n+1}}{n^nP'y}
$$

Again, we derive an $f(y)$ which is 0 when the equation is balanced, and its derivative:

$$
\begin{align*}f(y) &= ADn^n+\frac{D^{n+1}}{n^nP'y}-An^n(S'+y) - D\\f'(y)&=\frac{-D^{n+1}}{n^nP'y^2}-An^n\end{align*}
$$

Here is the formula for Newton’s method again:

$$
y_\text{next}=y-\frac{f(y)}{f'(y)}
$$

After substiuting $f(y)$ and $f'(y)$ into Newton’s method we get:

$$
y_\text{next}=y-\frac{ADn^n+\frac{D^{n+1}}{n^nP'y}-An^n(S'+y) - D}{\frac{-D^{n+1}}{n^nP'y^2}-An^n}
$$

$$
y_\text{next}=y+\frac{ADn^n+\frac{D^{n+1}}{n^nP'y}-An^n(S'+y) - D}{\frac{D^{n+1}}{n^nP'y^2}+An^n}
$$

$$
y_\text{next}=y\frac{\frac{D^{n+1}}{n^nP'y^2}+An^n}{\frac{D^{n+1}}{n^nP'y^2}+An^n}+\frac{ADn^n+\frac{D^{n+1}}{n^nP'y}-An^n(S'+y) - D}{\frac{D^{n+1}}{n^nP'y^2}+An^n}
$$

$$
y_\text{next}=\frac{\frac{D^{n+1}}{n^nP'y}+An^ny}{\frac{D^{n+1}}{n^nP'y^2}+An^n}+\frac{ADn^n+\frac{D^{n+1}}{n^nP'y}-An^nS'-An^ny - D}{\frac{D^{n+1}}{n^nP'y^2}+An^n}
$$

$$
y_\text{next}=\frac{ADn^n+2\frac{D^{n+1}}{n^nP'y}-An^nS' - D}{\frac{D^{n+1}}{n^nP'y^2}+An^n}
$$

It might seem like the equation cannot be simplified further, but if we revist our original invariant

$$
An^n(S'+y) + D = ADn^n+\frac{D^{n+1}}{n^nP'y}
$$

We can solve for $ADn^n$ we get

$$
ADn^n=\boxed{-\frac{D^{n+1}}{n^nP'y}+An^n(S'+y) + D}
$$

And then if we substitute $ADn^n$ into the numerator, we get

$$
y_\text{next}=\frac{\boxed{(-\frac{D^{n+1}}{n^nP'y}+An^n(S'+y) + D)}+2\frac{D^{n+1}}{n^nP'y}-An^nS' - D}{\frac{D^{n+1}}{n^nP'y^2}+An^n}
$$

A lot of cancellation happens

$$
y_\text{next}=\frac{-\frac{D^{n+1}}{n^nP'y}+\cancel{An^nS'}+An^ny + \cancel{D}+\cancel{2}\frac{D^{n+1}}{n^nP'y}-\cancel{An^nS'} - \cancel{D}}{\frac{D^{n+1}}{n^nP'y^2}+An^n}
$$

And we are left with a much smaller equation

$$
y_\text{next}=\frac{An^ny +\frac{D^{n+1}}{n^nP'y}}{\frac{D^{n+1}}{n^nP'y^2}+An^n}
$$

We multiply the top and bottom by $Y/(An^n)$

$$
y_\text{next}=\frac{(An^ny +\frac{D^{n+1}}{n^nP'y})\frac{y}{An^n}}{(\frac{D^{n+1}}{n^nP'y^2}+An^n)\frac{y}{An^n}}
$$

$$
y_\text{next}=\frac{(y^2 +\frac{D^{n+1}}{n^nP'An^n})}{\frac{D^{n+1}}{n^nP'y}\frac{1}{An^n}+y}
$$

Going back to our invariant, we can solve for the fractional term in the denominator:

$$
\frac{D^{n+1}}{n^nP'y}=\boxed{An^n(S'+y) + D -ADn^n}
$$

$$
y_\text{next}=\frac{(y^2 +\frac{D^{n+1}}{n^nP'An^n})}{\boxed{\frac{D^{n+1}}{n^nP'y}}\frac{1}{An^n}+y}
$$

$$
y_\text{next}=\frac{y^2 +\frac{D^{n+1}}{n^nP'An^n}}{\boxed{(An^n(S'+y) + D -ADn^n)}\frac{1}{An^n}+y}
$$

Then we can distribute and simplify the denominator

$$
y_\text{next}=\frac{y^2 +\frac{D^{n+1}}{n^nP'An^n}}{((S'+y) + \frac{D}{An^n} -D)+y}
$$

$$
y_\text{next}=\frac{y^2 +\frac{D^{n+1}}{n^nP'An^n}}{(S'+y + \frac{D}{An^n} -D)+y}
$$

$$
y_\text{next}=\frac{y^2 +\frac{D^{n+1}}{n^nP'An^n}}{2y+S' + \frac{D}{An^n} -D}
$$

In the original code, Curve defines additional variables:

$$
c = \frac{D^{n+1}}{n^nP'An^n}
$$

$$
b = S' + D/An^n
$$

After substitution into the formula, we get:

$$
y_\text{next}=\frac{y^2 +c}{2y+b -D}
$$

## Comparison to the original source code

This matches the Curve code exactly, see the purple box below:

![Screenshot 2024-07-25 at 1.09.45 PM.png](get_D()%20and%20get_y()%20in%20Curve%20StableSwap%20815a4ed4178948048d80000407b65553/Screenshot_2024-07-25_at_1.09.45_PM.png)

## Mismatch between Ann and An**ⁿ**

Rather confusingly, the Curve whitepaper uses the invariant $An^n$ but the codebase uses $Ann$. The reason for this discrepancy is that the codebase stores $A$ as $An^{n-1}$. Since $n$ is fixed at deployment time, precomputing $n^{n-1}$ allows the code to avoid computing an exponent on-chain, which is more costly operation.

## Summary

The core invariant of Curve does not allow for the variables $D$ or $x_i$ to be solve for symbolically. Instead, the terms must be solved for numerically.

One takeway from this exercise is that good algebraic manipulation is a very effective [gas optimization](rareskills.io/post/gas-optimization) technique. The Curve developers were able to compute a Newton’s method formula that is much smaller than naively plugging in $f$ and it’s derivative and leaving it at that.

## Citations and Acknowledgements

The following resources were consulted in writing this article:

- StableSwap - efficient mechanism for Stablecoin liquidity, Michael Egorov, [https://resources.curve.fi/pdf/curve-stableswap.pdf](https://resources.curve.fi/pdf/curve-stableswap.pdf)

- Understanding the Curve AMM, Part -1: StableSwap Invariant, Atul Agarwal [https://atulagarwal.dev/posts/curveamm/stableswap/](https://atulagarwal.dev/posts/curveamm/stableswap/)

- Curve Finance Discord, “chanho”

[https://discord.com/channels/729808684359876718/729812922649542758/1126630568004698132](https://discord.com/channels/729808684359876718/729812922649542758/1126630568004698132)

- Curve - Code Explaind - get_y() | DeFi, Smart Contract Programmer [https://www.youtube.com/watch?v=jAhKbxoeskQ](https://www.youtube.com/watch?v=jAhKbxoeskQ)
