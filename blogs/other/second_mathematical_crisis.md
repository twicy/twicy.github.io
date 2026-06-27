---
layout: page
permalink: /blogs/other/second_mathematical_crisis.html
title: The Second Mathematical Crisis
---

# The Second Mathematical Crisis

## Special Thank You

This blog is an almost identical replay of this educatioal video serie(s):

- [【数学史】数学危机（上）——无穷小量的幽灵与分析学的严谨化](https://www.bilibili.com/video/BV1ZnqDBhEvm?spm_id_from=333.788.videopod.sections&vd_source=b676aa6b65a2aa07f663041555254817)

I personally find them extremely interesting, and wish I had watched them earlier when I was in college.

## Before Introduction

Long before the debute of the second mathematical crisis, ancient Greek philosopher Zeno of Elea presented the famous "Zeno's paradoxes"

> In a race, the quickest runner can never over­take the slowest, since the pursuer must first reach the point whence the pursued started, so that the slower must always hold a lead.

If we quantize the problem to, Achilles (the quickest runner) moves at **10 m/s** and the tortoise (the slowest) moves at **1 m/s**, and the tortoise was **9 m ahead** of Achilles. Zeno is saying, the accumulated time that Achilles spent reaching the point the tortoise was, which can be calculated below

$$
t = 0.9 + 0.09 + 0.009 + ...
$$

is going to extend forever and never ends.

> Is $0.999\ldots$ ending somewhere?

That seemlingly easy question has bothered the top minds of the entire globe for more than 150 years.

## Introduction

The invention of calculus by Newton and Leibniz revolutionized mathematics. It just works!

$$
(x^2)^{\prime} = 2x, (x^3)^{\prime} = 3x^2
$$

Yet the foundations of calculus were surprisingly shaky. Mathematicians routinely manipulated **infinitesimals**—quantities smaller than any measurable number, yet somehow not zero. This worked extraordinarily well in practice, but philosophers questioned whether the arguments were logically sound.

This debate eventually led to the rigorous foundations of modern analysis during the nineteenth century.

## 1. George Berkeley (1685–1753)

![The Analyst](/blogs/images/other/The_Analyst_(book_cover).jpg)


> *"The Analyst; or, A Discourse Addressed to an Infidel Mathematician"* (1734)

George Berkeley is better known as the philosopher of **Idealism**, summarized by the famous Latin phrase

> **Esse est percipi**
>
> ("To be is to be perceived.")

Ironically, although remembered primarily as a philosopher and bishop, Berkeley understood contemporary mathematics remarkably well. His famous criticism was aimed not at the results of calculus, but at its logical foundations.

A famous quotation:

> **"And what are these Fluxions? The Velocities of evanescent Increments. And what are these same evanescent Increments? They are neither finite Quantities, nor Quantities infinitely small, nor yet nothing. May we not call them the ghosts of departed quantities?"**

This "ghosts of departed quantities" became one of the most famous criticisms in the history of mathematics.

### Example 1

Suppose

$$
y=x^2.
$$

Take two nearby points

$$
x,\qquad x+\Delta x.
$$

Then

$$
\frac{(x+\Delta x)^2-x^2}{(x + \Delta x) - x} = \frac{2x\Delta x + \Delta x^2}{\Delta x} = 2x+\Delta x = 2x.
$$

At first,

* $\Delta x$ is treated as an ordinary nonzero number so division is allowed.

Later,

* we let $\Delta x=0$.

The simple question Berkeley asked was:

> Is $\Delta x$ zero or is it not?

- If it is zero, division by it is impossible.
- If it is nonzero, why may we simply discard it?

He added that:

![The Analyst P93](/blogs/images/other/The_Analyst_P93.jpg)

> Qu. 63. Whether such Mathematicians as cry out against Mysteries, have ever examined their own Principles?

> Qu. 64. Whether Mathematicians, who are so delicate in religious Points, are strictly scrupulous in their own Science? Whether they do not submit to Authority, take things upon Trust, and believe Points inconceivable? Whether they have not their Mysteries, and what is more, their Repugnancies and Contradictions?

## 2. Thomas Bayes (1701–1761)

Today Bayes is famous for Bayes' theorem
$$
P(A|B) = \frac{P(B|A)P(A)}{P(B)}
$$
but during his lifetime he also wrote in defense of Newtonian calculus. He is was also elected a Fellow of the Royal Society.

![Fellow of the Royal Society](/blogs/images/other/Coat_of_arms_of_the_Royal_Society.svg.png)

His two principal mathematical AND theological works include

* *An Introduction to the Doctrine of Fluxions* (1736)
* *Divine Benevolence* (1731)

Bayes argued that infinitesimals should be viewed as **arbitrarily small quantities**, not mysterious objects.

However, Berkeley's criticism remained:

> "Arbitrarily small" is still **not zero**.

Merely making something smaller does not explain why it may suddenly be treated as zero. The following example would be a even better illustration of the issue.

### Example 2

$$
0.99999\ldots =1?
$$

> If "Arbitrarily small" is still **not zero**, they are not equal. No room for ambiguity

We now know that these two terms are equal in the scope of real analysis. But one of the $\cancel{\text{misleading}}$ interesting proofs I myself was taught in middle school was:

Let
$$
S=0.9+0.09+0.009+\cdots.
$$

Then
$$
10S=9+0.9+0.09+\cdots,
$$

so
$$
9S=9,
$$

thus
$$
S=1.
$$

But from another perspective, this could also lead us to a ridiculous conclusion: 
$$
S=9+90+900+\cdots.
$$

Again,

$$
10S=90+900+9000\cdots,
$$

leading to

$$
9S=-9,
$$

so

$$
S=-1.
$$

The key piece here is that many operations valid for finite series do not work for inifinite series.

## 3. Jean le Rond d'Alembert

Converging series:

$$
1 + \frac{1}{2} + \frac{1}{4} + \cdots
$$

There are non-converging series:

$$
1 + 2 + 3 + 4 + \cdots
$$

The first formal way to tell if an infinite series converges is the Ratio Test (d'Alembert's test), named after Jean le Rond d'Alembert

If
$$
L=\lim_{n\to\infty}\left|\frac{a_{n+1}}{a_n}\right|,
$$

then

* $L<1$: converges
* $L>1$: diverges
* $L=1$: inconclusive

Example for $L=1$, the Harmonic series:

$$
a_n=\frac1n
$$

Since

$$
L=\lim_{n\to\infty}\left|\frac{a_{n+1}}{a_n}\right| = 1
$$

the Ratio Test cannot determine convergence for

$$
\sum_{n=1}^{\infty}\frac{1}{n} = 1 + \frac{1}{2} + \frac{1}{3} + \cdots
$$

At this time, we are still lack of a formal definition of infinitesimals and limits.

Another fun fact about d'Alembert is, he worked with Denis Diderot in editting *Encyclopédie*. It was the first encyclopedia to include contributions from many named contributors and the first to describe the mechanical arts.

![Encyclopédie](/blogs/images/other/Encyclopedie_de_D'Alembert_et_Diderot_-_Premiere_Page_-_ENC_1-NA5.jpg)

Instead of trying to define infinitesimals, d'Alembert proposed defining calculus using **limits**.

His article **"Limite"** in the *Encyclopédie* (edited by Diderot and d'Alembert, Le chevalier de Jaucourt, and numerous other contributors) famously states that infinitesimals are unnecessary and should be replaced by limiting processes.

A representative quotation:

> **"One does not pass to the limit; one approaches it as closely as one wishes."**

## 4. Euler and Lagrange

On the other side, there are people who does not care about the definition of infinitesimals at all.

Euler was unconcerned by philosophical objections.

> **"The objection that the analysis of the infinitely small neglects mathematical rigor disappears entirely, since nothing is rejected except what is absolutely nothing."**

Unlike Berkeley, Euler was primarily interested in solving mathematical and physical problems rather than debating philosophical foundations. His work spanned mechanics, astronomy, optics, fluid dynamics, number theory, and engineering.

![Bridges ofKonigsberg](/blogs/images/other/Bridges_of_Konigsberg.png)

To Euler, the remarkable success of infinitesimal methods in producing correct results was itself compelling evidence of their validity. This pragmatic attitude explains why he regarded Berkeley's criticism as a philosophical concern rather than a mathematical obstacle.

![Leonhard Euler](/blogs/images/other/Leonhard_Euler_-_Jakob_Emanuel_Handmann_(Kunstmuseum_Basel).jpg)

> By the way, Euler was a true virtuoso. He made groundbreaking contributions across mathematics, physics, astronomy, and engineering. Even after losing the sight in one eye and later becoming almost completely blind, he continued to produce an extraordinary volume of influential work, **demonstrating remarkable perseverance alongside his unparalleled mathematical genius**.

Joseph-Louis Lagrange attempted to eliminate infinitesimals entirely by rebuilding calculus upon **Taylor's theorem**. He believed that differentiation should not rely on mysterious infinitesimal quantities, but instead on ordinary algebra and infinite polynomial expansions. In his *Théorie des fonctions analytiques* (1797), Lagrange argued that every sufficiently smooth function should be representable by its Taylor series, making infinite series—not infinitesimals—the true foundation of calculus.

### Talyor Series

Brook Taylor was a younger contemporary of Isaac Newton and, like Newton, a Fellow of the Royal Society. In 1712, Taylor served on the Royal Society committee that examined the famous priority dispute between Isaac Newton and Gottfried Wilhelm Leibniz over the invention of calculus. Although Taylor is remembered today for Taylor's theorem, his work was originally motivated by problems in finite differences and interpolation.

![Issac Newton](/blogs/images/other/Issac_Newton_Westminster.jpg)

One of Newton's many remarkable contributions was his generalization of the classical binomial theorem. Instead of restricting the exponent to positive integers, Newton showed that the expansion also works for arbitrary real (and later, even complex) exponents, provided the resulting infinite series converges.

$$
(1+x)^k = \sum_{n=0}^{\infty} \binom{k}{n}x^n = 1 + kx + \frac{k(k-1)}{2!}x^2 + \frac{k(k-1)(k-2)}{3!}x^3 + \cdots
$$
for any $k$ and $|x| < 1$, for convergece

At some point he was sure that all functions (not just binomial) should be able to expand as a polynomial series

$$
f(x) = a_{0} + a_{1}x + a_{2}x^2 + \cdots
$$

Taylor's key contribution was determining the coefficients of this infinite polynomial.

Suppose we wish to approximate a function around the point $x=x_0$.

For a zeroth-order approximation,

$$
f(x)\approx f(x_0).
$$

Including the first-order term gives

$$
f(x)\approx
f(x_0)
+
f'(x_0)(x-x_0).
$$

Including the second-order term,

$$
f(x)\approx
f(x_0)
+
f'(x_0)(x-x_0)
+
\frac{f''(x_0)}{2!}(x-x_0)^2.
$$

The factor $\frac{1}{2!}$ appears because

$$
\frac{d^2}{dx^2}(x-x_0)^2=2!,
$$

so dividing by $2!$ ensures that the second derivative of the polynomial equals $f''(x_0)$.

Continuing this process, we arrive at Taylor's theorem

$$
f(x)=
\sum_{n=0}^{\infty}
\frac{f^{(n)}(x_0)}{n!}(x-x_0)^n.
$$


Colin Maclaurin specialized Taylor's theorem at $x = 0$:

$$
f(x) = \sum_{n=0}^{\infty}
\frac{f^{(n)}(0)}{n!}x^n.
$$

### Example 3

$$
e^x
=
1+x+\frac{x^2}{2!}+\frac{x^3}{3!}+\cdots
$$

Let $x=2$

$$
7.389056\cdots
=
e^2
\approx
1
+
2
+
\frac{2^2}{2!}
+
\frac{2^3}{3!}
+
\frac{2^4}{4!}
=
7.000000\cdots
$$

Adding more terms rapidly improves the approximation.

### Example 4

$$
\sin x
=
x
-
\frac{x^3}{3!}
+
\frac{x^5}{5!}
-
\frac{x^7}{7!}
+\cdots
$$

Let $x=2$

$$
0.909297\cdots
=
\sin2
\approx
2
-
\frac{2^3}{3!}
+
\frac{2^5}{5!}
=
0.933333\cdots
$$

Again, including more terms produces increasingly accurate approximations.

### The problem of Talyor Series

Using Talor Expansion, instead of using $\approx$ or guilty consciencely putting a $=$ here, we can confidently place an $=$ between $0.99999\ldots$ and $1$, because

$$
f(x) = \frac{1}{1 - x}, f^{(1)}(x) = \frac{1}{(1 - x)^2}, f^{(2)}(x) = \frac{2}{(1 - x)^3}, \cdots, f^{(n)}(x) = \frac{n!}{(1 - x)^n}
$$

Therefore $f^{n}(0) = n!$

$$
\frac{1}{1 - x} = f(x) = \sum_{n=0}^{\infty}
\frac{f^{(n)}(0)}{n!}x^n = \sum_{n=0}^{\infty}x^n.
$$
 
Substitute $x = 0.1$,

$$
\frac{1}{1-0.1} = \sum_{n=0}^{\infty}0.1^n
$$

$$
10 = \sum_{n=0}^{\infty}9\times(0.1)^{n}
$$

$$
1 = \sum_{n=1}^{\infty}9\times(0.1)^{n}
$$

This method totally bypassed the definition of infinitesimals, but comes with a deadly inperfection. Taylor series do **not** always work well with some functions. If instead of $0.1$, but $x = 10$ was substituted in the above equation:

$$
-\frac{1}{9}=\frac{1}{1 - 10} = \sum_{n=0}^{\infty}
\frac{f^{(n)}(0)}{n!}10^n = \sum_{n=0}^{\infty}10^n = \ldots11111.
$$

In another example:

$$
\sqrt{1 + x^2} = 1 + \frac{1}{2}x^2 - \frac{1}{8}x^4 + \frac{1}{16}x^6 -\frac{5}{128}x^8+ \frac{7}{256}x^{10} \cdots
$$

Substitute with $x = 2$

$$
\sqrt{1 + 2^2} ?\approx 23
$$

However, if we substitute with $x = \frac{1}{2}$, it works again

$$
1.118033989\cdots =\sqrt{1 + (\frac{1}{2})^2} \approx = 1.118038177\cdots
$$

Lagrange's hope that every sufficiently smooth function could be represented by its Taylor series, died, and thus comes the Lagrange Remainder, appended to Talor's Theorem

$$
f(x) = \sum_{n=0}^{\infty}
\frac{f^{(n)}(x_0)}{n!}(x - x_0)^n + R_{n}(x).
$$

## 5. Augustin-Louis Cauchy

If Berkeley exposed the logical weaknesses of infinitesimals, and Lagrange attempted to replace them with Taylor series, it was Augustin-Louis Cauchy who began rebuilding calculus on rigorous foundations.

Cauchy entered the École Polytechnique only a few years after Joseph-Louis Lagrange had taught there. His landmark textbook, Cours d'analyse (1821), marks the beginning of modern rigorous analysis.

> "When the successive numerical values of the same variable decrease indefinitely, so as to fall below any given number, this variable becomes what is called an infinitely small quantity, or an infinitesimal."

This is very close to the $\varepsilon-\delta$ language we see today.

Also, he was the first to prove Taylor's theorem rigorously, establishing his well-known form of the remainder.

## 6. Karl Weierstrass

Karl Weierstrass has a famous function named after him, Weierstrass Function.

$$
f(x) = \sum_{n = 0}^{\infty}a^n\cos(b^n \pi x)
$$
, where $0<a < 1$ and $b$ is a positive odd integer and $ab > 1 +\frac{3}{2}\pi$

![Weierstrass Function](/blogs/images/other/weienstrass_function.jpg)

It is famous for being continuous everywhere but differentiable nowhere. People tend to think that continuous functions should be smooth after zooming in, but Weierstrass function proves them wrong.

And finally at this moment, as we take off from Leibniz defining continuity as "natura non facit saltus" or "nature makes no jump", we finally arrive at the modern **$\varepsilon-\delta$** definition of a limit, formalized by Weierstrass (building on Cauchy's work), is as follows.

Let $f$ be a function defined on an open interval around $a$ (possibly excluding $a$ itself). We say

$$
\lim_{x\to a}f(x)=L
$$

if and only if

$$
\boxed{
\forall,\varepsilon>0,;
\exists,\delta>0,;
\text{such that whenever }
0<|x-a|<\delta,
\text{ we have }
|f(x)-L|<\varepsilon.
}
$$

# References

1. Berkeley, G. (1734). *The Analyst; or, A Discourse Addressed to an Infidel Mathematician.*
2. Bayes, T. (1736). *An Introduction to the Doctrine of Fluxions.*
3. Bayes, T. (1731). *Divine Benevolence; or, an Attempt to Prove that the Principal End of the Divine Providence and Government is the Happiness of His Creatures.*
4. d'Alembert, J. le Rond. (1751–1772). "Limite," in *Encyclopédie*, edited by Denis Diderot and Jean le Rond d'Alembert.
5. Euler, L. (1755). *Institutiones Calculi Differentialis.*
6. Lagrange, J.-L. (1797). *Théorie des fonctions analytiques.*
7. Cauchy, A.-L. (1821). *Cours d'analyse de l'École Royale Polytechnique.*
8. Weierstrass, K. (1872). "Über continuierliche Funktionen eines reellen Arguments, die für keinen Werth des letzteren einen bestimmten Differentialquotienten besitzen."
9. Grabiner, J. V. (1981). *The Origins of Cauchy's Rigorous Calculus.*
10. Boyer, C. B., & Merzbach, U. C. (2011). *A History of Mathematics* (3rd ed.).
11. Kline, M. (1972). *Mathematical Thought from Ancient to Modern Times.*
12. Stillwell, J. (2010). *Mathematics and Its History* (3rd ed.).
