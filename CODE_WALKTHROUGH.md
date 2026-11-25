# Code walkthrough

A line-by-line explanation of `Quantum_Identity_Authentication_Qiskit_v2.ipynb`, written for a reader
who has never seen quantum computing before. Part 1 teaches the physics you need and nothing more.
Part 2 explains what the protocol is trying to do. Part 3 lists the conventions the code relies on.
Part 4 goes through every code cell, line by line.

If you only want the code, skip to Part 4, but read Part 3 first. Most of the confusing lines in this
notebook are confusing because of bit-ordering conventions, and Part 3 is where those are pinned down.

## Contents

- [Part 1. The physics, from zero](#part-1-the-physics-from-zero)
- [Part 2. What the protocol does](#part-2-what-the-protocol-does)
- [Part 3. Conventions used in the code](#part-3-conventions-used-in-the-code)
- [Part 4. The code, cell by cell](#part-4-the-code-cell-by-cell)
- [Part 5. Glossary](#part-5-glossary)

---

# Part 1. The physics, from zero

## 1.1 A bit, and then a qubit

A classical **bit** is either 0 or 1. You can look at it as often as you like and it stays what it was.

A **qubit** is the quantum version. Its two definite values are written `|0>` and `|1>`. The brackets are
just notation meaning "the quantum state called 0". Where a qubit differs from a bit is that it can also
sit in a blend of the two, called a **superposition**, written like this:

```
(|0> + |1>) / sqrt(2)
```

The `/sqrt(2)` is bookkeeping that keeps the probabilities adding to 1; ignore it for now. What matters is
that this qubit is not 0, and not 1, and not secretly one of them either. It genuinely has no value for
that question until you ask.

## 1.2 Measurement destroys the blend

**Measuring** a qubit means forcing it to answer "are you 0 or 1?". The blend above answers 0 half the
time and 1 half the time, at random. And here is the part that makes the whole protocol work:

> After you measure, the qubit *stays* at the answer it gave. The blend is gone for good.

So measurement is not like reading a bit. It is destructive. You cannot peek at a qubit and put it back
the way it was. This is the entire basis of quantum eavesdropping detection: an eavesdropper who looks at
a qubit leaves a mark, and the legitimate parties can go looking for that mark.

## 1.3 Two different questions you can ask: the Z basis and the X basis

A qubit can be asked more than one kind of question. The code only ever uses two.

**The Z question** (also called the computational basis, or `{|0>, |1>}`) asks "are you 0 or 1?".

**The X question** (also called the Hadamard basis, or `{|+>, |->}`) asks "are you plus or minus?", where

```
|+> = (|0> + |1>) / sqrt(2)
|-> = (|0> - |1>) / sqrt(2)
```

Note the minus sign in `|->`. Those two states are blends from the Z point of view, but they are the
definite answers from the X point of view. It works symmetrically: `|0>` and `|1>` are blends from the X
point of view.

This gives the rule that everything else depends on:

| The qubit is | Ask the Z question | Ask the X question |
|---|---|---|
| `\|0>` | always 0 | random, 50/50 |
| `\|1>` | always 1 | random, 50/50 |
| `\|+>` | random, 50/50 | always + |
| `\|->` | random, 50/50 | always − |

**Why this matters.** Suppose an eavesdropper intercepts a qubit, asks the Z question, and passes on
whatever she got. If the legitimate receiver was also going to ask the Z question, no harm done. But if
the receiver was going to ask the X question, he now gets a coin flip where he should have got a certain
answer. He will sometimes notice. That is the detection mechanism, and the factor of 1/2 in the table is
exactly where the protocol's "1/4 error rate" comes from later on.

## 1.4 Gates: reversible operations on qubits

A **gate** changes a qubit's state without measuring it, so nothing is destroyed and everything is
reversible. The notebook uses five.

| Gate | What it does | Plain description |
|---|---|---|
| `X` | `\|0> <-> \|1>` | a flip, the quantum NOT |
| `Z` | `\|0> -> \|0>`, `\|1> -> -\|1>` | attaches a minus sign to `\|1>` |
| `Y` | a flip *and* a sign change | combination of the two above |
| `H` (Hadamard) | `\|0> -> \|+>`, `\|1> -> \|->` | swaps the Z world and the X world |
| `CNOT` | flips a *target* qubit if a *control* qubit is 1 | the one gate that acts on two qubits |

Two notes that will come up repeatedly.

`Z` looks like it does nothing, because a minus sign in front of a state does not change any measurement
probability on its own. But once a qubit is entangled with another one (next section), that minus sign
becomes detectable. This is why `Z` is usable as a signal in the protocol.

`H` is how the code asks the X question using only Z-question hardware. Applying `H` and then measuring
in Z is exactly the same thing as measuring in X. You will see `qc.h(...)` immediately before a
measurement several times, and that is always what it means.

## 1.5 Entanglement and the four Bell states

Prepare two qubits together and you can reach states that cannot be described as "qubit 1 is in this
state and qubit 2 is in that state". They only have a joint description. They are **entangled**.

The most useful example:

```
(|00> + |11>) / sqrt(2)
```

Read `|00>` as "first qubit 0, second qubit 0". So this state says: both qubits are 0, or both are 1, with
equal chance, and *never* one of each. Measure either qubit alone and you get a coin flip. Measure both
and the two answers always agree. The randomness is shared.

There are exactly four such maximally entangled two-qubit states, called the **Bell states**. The paper
names them `U` and `W`; the standard physics names are `Phi` and `Psi`. The notebook keeps the paper's
names:

| Code | Paper | Standard | Definition |
|---|---|---|---|
| `U+` | `U+` | `Phi+` | `(\|00> + \|11>) / sqrt(2)` |
| `U-` | `U−` | `Phi−` | `(\|00> − \|11>) / sqrt(2)` |
| `W+` | `W+` | `Psi+` | `(\|01> + \|10>) / sqrt(2)` |
| `W-` | `W−` | `Psi−` | `(\|01> − \|10>) / sqrt(2)` |

The four differ only in a sign and in whether the two halves agree or disagree. That tiny amount of
structure is what carries the key.

Here is the table that the notebook verifies against the paper's Table 3. It says, for each Bell state,
whether the two measurement results come out **the same** or **opposite**, for each of the two questions:

| Bell state | Z question | X question |
|---|---|---|
| `U+` | same | same |
| `U-` | same | opposite |
| `W+` | opposite | same |
| `W-` | opposite | opposite |

Two things to notice. First, the four rows are all different, so if you know which question was asked and
you see the two results, you learn something about which Bell state it was. Second, the Z column alone
cannot tell `U+` from `U-`. You need the X question for that. This is why the protocol picks the question
at random: an eavesdropper who always asks the Z question is invisible in the Z column and conspicuous in
the X column.

## 1.6 Bell measurement: asking "which of the four?"

Measuring the two qubits separately gives you a row of the table above, which is not quite the same as
identifying the state. A **Bell measurement** is a joint measurement that answers "which of the four Bell
states is this?" directly, giving one of four answers.

The trick for building one: the four Bell states are made from `|00>`, `|01>`, `|10>`, `|11>` by a short
circuit, and that circuit is reversible. So to identify a Bell state, run the preparation circuit
*backwards* and then measure both qubits in the ordinary Z way. The two bits you get name the state. You
will see this in the code as `bell_transform`, which is literally `bell_prep` with the steps reversed.

## 1.7 No-cloning: why Eve cannot just copy

A classical eavesdropper copies the message and forwards the original, leaving no trace. Quantum
mechanics forbids this. There is no machine that takes an unknown qubit and produces two identical
copies. This is the **no-cloning theorem**.

So an attacker has only bad options. She can measure, which destroys the blend and leaves detectable
damage. Or she can entangle the passing qubit with a private qubit of her own, hold onto hers, and measure
it later. That second strategy is the **Entanglement-Measure attack** of the paper's Section 4.2.3, and
Section 7 of the notebook shows that the more information it extracts, the more damage it does, with no
setting where she gets information for free.

## 1.8 The four extra ideas needed for Section 7

Sections 7.2 and 7.3 of the notebook use four pieces of machinery that are harder than the rest. Here is
the minimum to read that code.

**Density matrix.** A square table of numbers describing a quantum state. It can describe states that a
simple `|...>` cannot, in particular "it is one of these states but I do not know which". Needed because
Eve genuinely does not know which Bell state was sent.

**Partial trace.** Given the density matrix of several qubits, the partial trace throws away some of them
and returns the best possible description of the rest. "Trace out Charlie and the travelling qubit" means
"work out what Eve's private qubits look like to Eve, who cannot see the others". In the code this is
`partial_trace(state, [list of qubits to discard])`.

**Von Neumann entropy, and the Holevo quantity.** Entropy measures how uncertain a state is, in bits. The
**Holevo quantity**, written `chi`, combines a few entropies into an upper bound on how many bits of
classical information *any* measurement whatsoever could extract from a quantum state. It is the right
tool for "how much of the key did Eve learn", because it does not require guessing what measurement she
performs. `chi = 0` means she learned nothing and no cleverness will help. In this notebook a key symbol
is 2 bits, so `chi` runs from 0 to 2.

**Pauli transfer matrix.** A 4x4 table summarising what a process does to a single qubit. Only the
diagonal matters here, four numbers usually written `lambda_I, lambda_X, lambda_Y, lambda_Z`. Each is
between −1 and 1, and each says how much of one kind of information survives. A value of 1 means
untouched, 0 means destroyed. `[1, 1, 1, 1]` is "nothing happened", so an attack with that signature
cannot have learned anything. `CNOT` gives `[1, 0, 0, 1]`, meaning Z-type information passed through
intact while X-type information was wiped out, which is exactly the fingerprint of "someone asked the Z
question".

---

# Part 2. What the protocol does

## 2.1 The goal

Hospitals share patient records on a blockchain. Before a hospital accepts a record published by Alice,
it wants proof that the publisher really is Alice. Classically you would use a digital signature, but
those rest on mathematics a future quantum computer could break. The paper replaces the signature with a
physical protocol.

Every pair of nodes already shares a secret key, 20 bits in the paper's example. Alice proves she is
Alice by demonstrating that she knows the key, and she does it by encoding the key into qubits rather
than sending it. Charlie is the verifier.

## 2.2 The four steps, in plain language

**Step 1. Charlie prepares the pairs.** He builds `N/2 + p` Bell pairs, picking each state at random from
the four, and writes down what he picked. With `N = 20` and `p = 10` that is 20 pairs. He will need
`N/2 = 10` of them to carry the key, since each pair carries 2 bits, and the extra `p = 10` are sacrificed
to eavesdropper checks.

**Step 2. He sends one half of each pair and they check the forward channel.** Charlie keeps the first
qubit of every pair and sends the second to Alice. Alice picks `p/2 = 5` of the arriving qubits at random,
picks the Z or X question at random for each, measures, and announces publicly what she asked and what she
got. Charlie measures his matching halves with the same questions and compares. For an undisturbed pair
the two results must follow the same/opposite table from Part 1.5. If too many disagree, somebody
interfered on the way out, and the run is abandoned.

**Step 3. Alice encodes the key.** Of the qubits she has left, Alice picks `N/2 = 10` and applies one of
four gates to each, chosen by the next two bits of the key:

| Key bits | Gate (paper's Eq. 20) |
|---|---|
| `00` | `I`, do nothing |
| `01` | `Z` |
| `10` | `X` |
| `11` | `iY` |

Applying a gate to one half of a Bell pair turns it into a different Bell pair. That is the encoding.
Alice sends everything back to Charlie. The remaining `p/2 = 5` untouched qubits travel back too, and
serve as the second check.

**Step 4. Charlie checks and decodes.** Charlie now holds both halves of every pair again.

- 4a. On the 5 untouched pairs he runs a Bell measurement. Each must come back as the state he originally
  prepared. If too many do not, somebody interfered on the return trip, and the run is abandoned.
- 4b. On the 10 key pairs he runs a Bell measurement. He knows what he prepared and he sees what came
  back, so he can read off which gate Alice applied, which gives him 2 key bits per pair. He compares the
  recovered 20 bits with his copy of the key. If they agree, Alice is authenticated.

## 2.3 Why it is secure, and the three claims that fail

An eavesdropper needs to know both what went out and what came back in order to work out which gate was
applied. Learning either one requires interacting with a qubit, and by Part 1.7 that cannot be done
invisibly. The two checks in Steps 2 and 4a exist to catch the damage.

The paper then makes three claims that the notebook tests and finds wanting. All three are quantitative
rather than conceptual; the protocol's core idea is sound.

1. *"Eve will definitely be detected."* With only 5 check samples, a Z-question eavesdropper on the
   forward channel escapes the forward check about 24% of the time, because each sample catches her with
   probability 1/4 and `(3/4)^5 = 0.237`.
2. *"She will not be able to obtain any key information."* She gets the first bit of every key symbol with
   certainty. Section 9.1 of the notebook demonstrates it. Being detected and learning nothing are two
   different claims, and only the first survives.
3. *"Charlie has only a 1/4 probability of getting the correct Bell state"* under the
   Entanglement-Measure attack. The true figure is 1 for an attack that passes the paper's own
   invisibility condition, and 1/2 for a full `CNOT`. 1/4 is the average over random attacks, which is
   just the random-guess level.

---

# Part 3. Conventions used in the code

These five conventions explain most of the otherwise cryptic lines. They were each verified by running
them, not assumed.

**3.1 Four qubits per pair.** Every pair gets its own small circuit with four qubits:

```python
C, T, EF, ER = 0, 1, 2, 3
```

- `C` = Charlie's half, which never travels.
- `T` = the travelling half, which goes to Alice and comes back. Everything happens to this one.
- `EF` = a spare qubit for Eve to use on the forward trip.
- `ER` = a spare qubit for Eve to use on the return trip.

`EF` and `ER` sit idle unless an attack is switched on.

**3.2 Bell states are numbered 0 to 3.** `BELL_LABELS = ["U+", "U-", "W+", "W-"]`. After a Bell
measurement, the two result bits combine into the index as `bits[0] + 2 * bits[1]`, where `bits[0]` came
from `C` and `bits[1]` from `T`. Verified: `U+` gives index 0, `U-` gives 1, `W+` gives 2, `W-` gives 3.

**3.3 Qiskit is little-endian.** This is the single biggest source of confusion. In a bitstring printed by
Qiskit, **qubit 0 is on the right**. Setting qubit 1 to one and printing gives `'10'`, not `'01'`. So in a
two-character string `s` describing qubits 1 and 0, `s[0]` is qubit 1 and `s[1]` is qubit 0. The code
indexes backwards from the end for this reason, which is why you see `tok[-1]` and `tok[-2]`.

**3.4 The first qubit you name is the matrix's low bit.** When the code applies a 4x4 matrix to two
qubits with `qc.unitary(U, [anc, T])`, the matrix's row and column index is

```
index = 2 * (value of T) + (value of anc)
```

So `anc` is the low bit because it was named first. The same rule applies to
`Statevector(...).probabilities([C, T])`, where the resulting array is indexed `bits[C] + 2 * bits[T]`.
Both were verified directly.

**3.5 Classical registers print in reverse order of declaration.** Each pair circuit declares three
classical registers in the order `ef`, `er`, `out`. Qiskit prints them backwards, space separated, so a
result string looks like `'10 0 1'` meaning `out = 10`, `er = 0`, `ef = 1`. The code asserts this rather
than trusting it, because the behaviour is undocumented.

**3.6 One more thing about `iY`.** The paper's Eq. 20 uses the matrix `[[0, 1], [-1, 0]]`, which equals
`i` times the Pauli `Y`. Qiskit's `y` gate is plain `Y`, so it differs from the paper's gate by an overall
factor of `-i`. An overall factor like that has no physical effect and no measurement can see it, so the
code uses `y` and notes why. Verified: `i * Y == [[0, 1], [-1, 0]]`.

---

# Part 4. The code, cell by cell

Cell numbers are the notebook's own, so they skip the markdown cells in between.

## Cell 1. Imports and the simulator

```python
%pip install -q "qiskit>=1.0,<3" "qiskit-aer>=0.14,<1" numpy pandas
import numpy as np
import pandas as pd
from dataclasses import dataclass, field
from qiskit import QuantumCircuit, QuantumRegister, ClassicalRegister
from qiskit.quantum_info import Statevector, Operator, DensityMatrix, partial_trace, entropy, random_unitary
from qiskit_aer import AerSimulator
try:
    display
except NameError:
    display = print

backend = AerSimulator()
print("Qiskit and Aer are ready.")
```

**Line 1.** Installs the packages. The `<3` and `<1` upper bounds stop a future major release from
silently changing behaviour. `-q` keeps the output quiet. The `%` makes it a notebook magic, so it installs
into whatever Python is running this notebook rather than some other one.

**Lines 2–3.** `numpy` for arrays and arithmetic, `pandas` for the result tables.

**Line 4.** `dataclass` and `field` are used in Cell 11 to make a small record type without writing an
`__init__`.

**Line 5.** The circuit-building classes. `QuantumCircuit` is the circuit itself; `QuantumRegister` and
`ClassicalRegister` are named groups of qubits and of classical bits.

**Line 6.** The exact-mathematics toolkit, as opposed to simulation.
- `Statevector` computes a circuit's state exactly, with no random sampling.
- `DensityMatrix` is the more general state description from Part 1.8.
- `partial_trace` discards qubits you cannot see.
- `entropy` is the von Neumann entropy.
- `random_unitary` generates a random valid quantum operation, used to stand in for "some arbitrary attack".
- `Operator` is imported but never used. Harmless.

**Line 7.** `AerSimulator` is the simulator that mimics a real device, including the random measurement
outcomes. The split in this notebook is deliberate: `Statevector` answers "what is the exact
probability", Aer answers "what happens when you actually run it".

**Lines 8–11.** `display` renders pandas tables nicely, but it only exists inside a notebook. The `try`
checks whether the name exists and falls back to plain `print` if not, so the file also runs as a script.

**Line 13.** Creates the simulator once and reuses it everywhere.

## Cell 3. Protocol parameters

```python
N = 20                  # shared-key length in bits
p = 10                  # overhead parameter: p/2 forward-check + p/2 return-check pairs
ERROR_THRESHOLD = 0.0   # Steps 2 and 4a: a check fails if its error rate is greater than this
KEY_THRESHOLD   = 0.0   # Step 4: allowed symbol error rate in key', the paper's "margin of error"
assert N > 0 and N % 2 == 0 and p > 0 and p % 2 == 0
print(f"Initial pairs={N//2 + p}; forward-check={p//2}; return-check={p//2}; key-carrying={N//2}")
print("The paper gives no numeric value for either threshold; both default to 0 here.")
```

**Line 1.** The key is 20 bits, matching the paper's example.

**Line 2.** `p` is the sacrificial overhead. Half of it goes to the forward check, half to the return
check, so `p = 10` buys 5 samples for each.

**Line 3.** How many check errors to tolerate, as a rate. At `0.0`, a single bad sample out of 5 aborts
the run. The paper never gives a number, so 0 is the strictest honest reading.

**Line 4.** The separate tolerance for the final key comparison. The paper's Step 4 says Charlie accepts
"within a certain margin of error", so this is a rate on the recovered symbols, not an exact match. It is
a distinct knob from line 3 because it trades off differently: loosening it survives more channel noise but
also admits an impostor who only knows part of the key. Cells 32 and 34 measure both sides of that.

**Line 5.** Sanity checks. `N` must be even because each pair carries 2 bits, so `N/2` must be a whole
number of pairs. `p` must be even because it splits in half between the two checks.

**Line 6.** `N//2 + p` is `10 + 10 = 20` pairs, split 5 / 5 / 10. `//` is integer division.

## Cell 5. The building blocks

This is the engine of the notebook. Nothing here prints anything; it defines the functions everything
else calls.

### Constants

```python
C, T, EF, ER = 0, 1, 2, 3
BELL_LABELS = ["U+", "U-", "W+", "W-"]          # index = c0 + 2*c1 of the Bell-transform measurement
```

**Line 1.** The qubit numbering from Part 3.1. Using names instead of bare numbers is why later lines read
`qc.h(q[T])` rather than `qc.h(q[1])`.

**Line 2.** The Bell state ordering from Part 3.2.

### Preparing and identifying Bell states

```python
def bell_prep(qc, c, t, label):
    qc.h(c); qc.cx(c, t)
    if label in ("U-", "W-"): qc.z(c)
    if label in ("W+", "W-"): qc.x(t)
```

Builds any of the four Bell states from two fresh `|0>` qubits.

**Line 2.** `qc.h(c)` puts the first qubit into `(|0> + |1>)/sqrt(2)`. Then `qc.cx(c, t)` applies CNOT
with `c` as control and `t` as target, which flips `t` only in the half of the superposition where `c` is
1. The result is `(|00> + |11>)/sqrt(2)`, which is `U+`. These two gates are the standard
entanglement-creating pair.

**Line 3.** For the two "minus" states, `qc.z(c)` puts a minus sign on the `|1>` half, turning
`(|00> + |11>)` into `(|00> − |11>)`. So `U+` becomes `U-`.

**Line 4.** For the two `W` states, `qc.x(t)` flips the second qubit, turning `|00>` into `|01>` and
`|11>` into `|10>`. So `U+` becomes `W+`, and `U-` becomes `W-`.

The two `if`s are independent, which is how four combinations come out of two lines.

```python
def bell_transform(qc, c, t):                    # inverse of bell_prep: Bell basis -> computational basis
    qc.cx(c, t); qc.h(c)
```

The Bell measurement from Part 1.6, minus the measuring. It is `bell_prep` run backwards: `h` then `cx`
becomes `cx` then `h`. Both gates are their own inverse, so no other change is needed. Feed it a Bell state
and it produces a plain two-bit state, which the caller then measures normally. Verified in Part 3.2 that
the resulting bits index `BELL_LABELS` correctly.

### The key gates, Eq. 20

```python
KEY_GATES = {"00": None, "01": "z", "10": "x", "11": "y"}     # Eq. (20)
def key_gate(qc, qubit, symbol):
    g = KEY_GATES[symbol]
    if g: getattr(qc, g)(qubit)
```

**Line 1.** The paper's Eq. 20 as a lookup table. `None` for `00` because the identity gate means "do
nothing", and there is no point adding a gate that does nothing. `"y"` stands in for the paper's `iY`, for
the global-phase reason in Part 3.6.

**Line 3.** Looks up the gate name for this 2-bit symbol.

**Line 4.** `if g:` skips the `None` case. `getattr(qc, "z")` fetches the circuit's `z` method by name, and
the trailing `(qubit)` calls it. This is just a way to turn the string `"z"` into the call `qc.z(qubit)`
without a chain of `if`s.

### Key handling

```python
def key_symbols(key):
    if len(key) % 2 or set(key) - {"0", "1"}: raise ValueError("Expected even-length binary key.")
    return [key[i:i+2] for i in range(0, len(key), 2)]
```

Chops a key string into 2-bit symbols: `"1010..."` becomes `["10", "10", ...]`.

**Line 2.** Two guards in one condition. `len(key) % 2` is non-zero, hence truthy, for odd lengths.
`set(key) - {"0", "1"}` is the set of characters that are not 0 or 1, which is empty, hence falsy, for a
valid binary string. A single stray character makes it non-empty and triggers the error.

**Line 3.** `range(0, len(key), 2)` steps in twos, and `key[i:i+2]` takes the pair starting there. A
20-character key yields 10 symbols.

```python
def random_key(n, rng):
    return "".join(rng.choice(["0", "1"], size=n).tolist())
```

Generates `n` random bits as a string. `rng` is a numpy random generator passed in rather than created
here, which is what makes every experiment in the notebook reproducible. `.tolist()` converts numpy string
objects to plain Python strings so `"".join` works cleanly.

### The channel: where attacks and noise live

```python
def apply_channel(qc, channel, anc, eve_bit, rng=None):
    """Send the travelling qubit T through a channel or attacker.

    Returns the basis of any measurement Eve made on T ("Z" or "X"), else None, so the caller
    knows whether the bit sitting in `eve_bit` is a real reading or just an untouched zero.
    """
```

Everything that happens to a qubit in transit goes through this one function. The travelling qubit passes
through it twice per round trip, once outbound and once inbound. The arguments: `qc` is the circuit being
built, `channel` describes what happens, `anc` is Eve's spare qubit for this direction, `eve_bit` is the
classical bit where her measurement result is stored, and `rng` is the random generator.

The return value matters. If Eve never measured, `eve_bit` is still 0, and that 0 is not a reading. The
returned basis string lets the caller tell a genuine 0 from an untouched one.

```python
    if channel is None: return None
    rng = np.random.default_rng() if rng is None else rng     # every channel below needs randomness
    kind = channel if isinstance(channel, str) else channel[0]
```

**Line 1.** `None` means an ideal channel. Return immediately and add nothing to the circuit.

**Line 2.** Every remaining branch needs randomness, so supply a generator if the caller forgot. Without
this line, calling with a random channel and no `rng` raises `AttributeError: 'NoneType' object has no
attribute 'random'`.

**Line 3.** Channels are specified either as a bare string like `"imr_z"` or as a tuple like
`("depol", 0.5)`. This line extracts the kind either way.

```python
    if kind in ("imr_z", "imr_x", "imr_rand"):
        basis = {"imr_z": "Z", "imr_x": "X"}.get(kind) or str(rng.choice(["Z", "X"]))
        if basis == "X": qc.h(T)
        qc.measure(T, eve_bit)                    # Eve measures, the collapsed qubit is the one she resends
        if basis == "X": qc.h(T)
        return basis
```

The **intercept-measure-resend** attack, the paper's Section 4.2.2. Eve grabs the qubit, measures it, and
sends on whatever she is left with.

**Line 2.** Picks the question Eve asks. `.get(kind)` returns `"Z"` or `"X"` for the fixed variants, and
`None` for `"imr_rand"`; `or` then falls through to a random choice. A compact way to write "fixed if
specified, random otherwise".

**Lines 3–5.** To ask the X question using Z-only hardware, sandwich the measurement between two `H`
gates, as explained in Part 1.4. The second `H` undoes the first, so the qubit Eve forwards is expressed
back in the original frame. For the Z question both `if`s are skipped and it is a bare measurement.

Note what is *not* here: no re-preparation step. The paper says Eve measures and then prepares a fresh
qubit to send on. Measuring already leaves the qubit in exactly the state she measured, so forwarding that
same qubit is physically identical to preparing a copy of it. One `qc.measure` captures the whole attack.

```python
    if kind == "depol":
        if rng.random() < channel[1]:
            getattr(qc, str(rng.choice(["x", "y", "z"])))(T)
        return None
```

**Depolarising noise**, which models an imperfect channel rather than an attacker. `channel[1]` is the
probability `q` of something going wrong. With probability `q`, one of `X`, `Y`, `Z` is applied at random,
which is the standard way to represent "the qubit got randomly scrambled". Returns `None` because noise is
not Eve and reads nothing.

```python
    if kind == "em":
        qc.unitary(channel[1], [anc, T])          # qargs are little-endian: matrix index = 2*t + a
        return None
```

The **Entanglement-Measure** attack, the paper's Section 4.2.3. `channel[1]` is a 4x4 matrix describing a
joint operation on Eve's spare qubit and the travelling qubit. She does not measure now; she entangles and
waits. `[anc, T]` makes `anc` the matrix's low bit, per Part 3.4.

```python
    if kind == "sparse":
        # ("sparse", f, U): Eve applies the strong unitary U to each particle with probability f.
        if rng.random() < channel[1]:
            qc.unitary(channel[2], [anc, T])
        return None
    if kind == "sparse_imr":
        # ("sparse_imr", f): Z-basis intercept-measure-resend on a fraction f of the particles.
        if rng.random() < channel[1]:
            qc.measure(T, eve_bit)
            return "Z"
        return None
    raise ValueError(f"unknown channel {channel!r}")
```

Two **selective** attacks. Instead of attacking weakly everywhere, Eve attacks at full strength on a random
fraction `f` of the qubits and leaves the rest alone. This matters because it is a genuinely different
trade-off: with `f = 0.05` she survives both checks about 79% of the time, which no weak-everywhere attack
manages. Cells 23 and 30 measure them.

Note `sparse_imr` returns `"Z"` only when it actually fired. On the other qubits it returns `None`, and
Cell 30 relies on that to know which readings are real.

The final `raise` catches typos in channel names instead of silently doing nothing.

### Building one pair's circuit

```python
def pair_circuit(initial, role, fwd=None, ret=None, symbol=None, basis=None, rng=None):
    # One Bell pair through the whole protocol. role: 'fwd' (forward check), 'ret' (return check) or 'auth' (key pair).
    q = QuantumRegister(4, "q")
    ef, er, out = ClassicalRegister(1, "ef"), ClassicalRegister(1, "er"), ClassicalRegister(2, "out")
    qc = QuantumCircuit(q, ef, er, out)
```

One Bell pair's entire journey as a single circuit. `role` decides which of the three jobs this pair does.

**Lines 3–5.** Four qubits, and three classical registers: 1 bit for Eve's forward reading, 1 for her
return reading, 2 for the protocol's own result. Keeping them separate is what makes Part 3.5's parsing
possible. The declaration order here is what determines the print order later.

```python
    bell_prep(qc, q[C], q[T], initial)
    eve_fwd = apply_channel(qc, fwd, q[EF], ef[0], rng)             # Step 2: S2 travels to Alice
    eve_ret = None
```

**Line 1.** Step 1: Charlie makes the pair in the requested state.

**Line 2.** The outbound trip. Any forward attack happens here, after preparation and before Alice touches
anything.

```python
    if role == "fwd":
        if basis == "X": qc.h(q[T]); qc.h(q[C])
        qc.measure(q[T], out[0]); qc.measure(q[C], out[1])         # out0 = Alice, out1 = Charlie
```

The **forward-check** branch. This pair is consumed in Step 2 and never comes back.

**Line 2.** If the X question was chosen, apply `H` to *both* halves before measuring. Both parties must ask
the same question for the comparison to mean anything.

**Line 3.** Alice's result goes to `out[0]`, Charlie's to `out[1]`. Physically these are two separate
people measuring at two different times; putting them in one circuit gives identical statistics and makes
the simulation far simpler.

```python
    else:
        if role == "auth": key_gate(qc, q[T], symbol)               # Step 3
        eve_ret = apply_channel(qc, ret, q[ER], er[0], rng)         # return channel
        bell_transform(qc, q[C], q[T])
        qc.measure(q[C], out[0]); qc.measure(q[T], out[1])         # Step 4: Bell measurement
```

The **round-trip** branch, covering both the return-check pairs and the key-carrying pairs. The only
difference between them is line 2.

**Line 2.** Step 3: only key pairs get a gate. Return-check pairs are deliberately left untouched, which is
what lets Charlie compare against the state he prepared.

**Line 3.** The inbound trip, after Alice's gate. Order matters: an attacker on this leg sees the encoded
qubit.

**Lines 4–5.** Step 4: the Bell measurement from Part 1.6. `bell_transform` undoes the entanglement, then
both qubits are measured. `C` to `out[0]` and `T` to `out[1]` so the bits index `BELL_LABELS` as in
Part 3.2.

```python
    qc.metadata = {"eve_fwd_basis": eve_fwd, "eve_ret_basis": eve_ret}
    return qc
```

Records which legs Eve actually measured on. `metadata` is a free-form slot Qiskit provides on circuits;
attaching it here means the information travels with the circuit and can be read back after the run. This
is how Cell 30 knows which of Eve's bits are genuine readings.

### Running a batch of circuits

```python
def run_circuits_full(circs, seed):
    """Run one batch and keep everything, including the bits Eve read.

    Aer prints the classical registers in reverse declaration order, so the memory string is
    'out er ef'. That ordering is undocumented, hence the assertion.
    """
    res = backend.run(circs, shots=1, memory=True, seed_simulator=int(seed)).result()
```

**Line 6.** Runs a whole list of circuits in one call, which is far faster than one call each.
- `shots=1` because each pair is used once in a real protocol run, not repeated.
- `memory=True` asks for the individual outcome rather than aggregate counts, which `shots=1` requires.
- `seed_simulator` makes the run reproducible. Aer advances the seed per circuit internally, so the
  circuits in one batch do not come out correlated. This was checked by running 20 identical coin-flip
  circuits in one batch and confirming the results varied.

```python
    rows = []
    for i, qc in enumerate(circs):
        toks = res.get_memory(i)[0].split()
        assert len(toks) == 3 and len(toks[0]) == 2, f"expected 'out er ef', got {toks!r}"
        out, er, ef = toks
        md = qc.metadata or {}
```

**Line 3.** `get_memory(i)` returns a list of outcome strings for circuit `i`; with one shot there is one
entry, hence `[0]`. `.split()` breaks it at the spaces into the three registers.

**Line 4.** Guards the convention from Part 3.5. Three registers, and the first is the 2-bit `out`. If a
future Qiskit changes the print order this fails loudly instead of returning silently wrong numbers.

**Line 5.** Unpacks in the printed order, which is the reverse of the declared order.

**Line 6.** `or {}` guards against `metadata` being `None`.

```python
        rows.append({"bits": (int(out[-1]), int(out[-2])),                 # (bit0, bit1)
                     "eve_fwd": int(ef) if md.get("eve_fwd_basis") else None,
                     "eve_ret": int(er) if md.get("eve_ret_basis") else None,
                     "eve_fwd_basis": md.get("eve_fwd_basis"),
                     "eve_ret_basis": md.get("eve_ret_basis")})
    return rows
```

**Line 1.** `out[-1]` is the last character, which is `out[0]`, because of the little-endian printing in
Part 3.3. `out[-2]` is `out[1]`. Reading from the end is the safe way to do this.

**Lines 2–3.** Converts Eve's bit to an integer only if she actually measured on that leg. The `if`
condition exploits truthiness: `"Z"` and `"X"` are truthy, `None` is falsy. Without this, an untouched 0
would masquerade as a measurement result of 0 and corrupt the leakage statistics in Cell 30.

```python
def run_circuits(circs, seed):
    return [row["bits"] for row in run_circuits_full(circs, seed)]
```

A convenience wrapper for the many callers that only want the protocol outcome. Keeping it means adding
Eve's bits did not require changing any existing call site.

```python
def bell_from_bits(bits):                                           # (c0, c1) -> Bell label
    return BELL_LABELS[bits[0] + 2 * bits[1]]
```

Turns the two measured bits into a Bell state name, using the indexing from Part 3.2.

## Cell 7. Checking the paper's Table 3

The point of this cell is to confirm that the paper's Table 3 is physically correct, by computing the same
table from the circuits and comparing. If the paper had a typo here, everything downstream would be wrong.

```python
TABLE3 = {   # paper Table 3
    ("U+", "Z"): {"00", "11"}, ("U-", "Z"): {"00", "11"},
    ("W+", "Z"): {"01", "10"}, ("W-", "Z"): {"01", "10"},
    ("U+", "X"): {"++", "--"}, ("U-", "X"): {"+-", "-+"},
    ("W+", "X"): {"++", "--"}, ("W-", "X"): {"+-", "-+"},
}
```

The paper's table, typed in by hand. Each entry maps a (Bell state, question) pair to the set of results
that are allowed to occur. Read `("U+", "Z"): {"00", "11"}` as: if the state was `U+` and both asked the Z
question, the only possible answers are both-0 or both-1. Each set has two members because which of the two
you get is random; what is fixed is that the other two combinations never appear.

```python
RELATION = {k: ("same" if len(set(next(iter(v)))) == 1 else "opposite") for k, v in TABLE3.items()}
```

Boils each entry down to a single word, `"same"` or `"opposite"`, because that is all the protocol
actually compares.

Working from the inside out, with `v = {"01", "10"}`:
- `iter(v)` starts walking the set and `next(...)` takes one element, say `"01"`. Which one is unspecified,
  but it does not matter here: both members of every set give the same answer.
- `set("01")` is `{"0", "1"}`, the distinct characters.
- `len(...) == 1` asks whether both characters were identical. For `"01"` the length is 2, so the result is
  `"opposite"`. For `"00"` the set is `{"0"}`, length 1, so `"same"`.

The whole thing is a dict comprehension, so this happens for all eight entries.

```python
for (bell, basis), allowed in TABLE3.items():
    qc = QuantumCircuit(2); bell_prep(qc, 0, 1, bell)
    if basis == "X": qc.h(0); qc.h(1)
```

Now rebuild each case from scratch and check it. The loop unpacks the dictionary key into `bell` and
`basis` directly in the `for` statement.

**Line 2.** A plain two-qubit circuit, no attack qubits needed, prepared in the Bell state under test.

**Line 3.** Apply `H` to both qubits for the X question. Note there is no `measure` call anywhere in this
loop, because the next line computes probabilities exactly rather than sampling.

```python
    seen = set()
    for key, pr in Statevector(qc).probabilities_dict().items():    # key = 'q1 q0' = 'Alice Charlie'
        if pr > 1e-9:
            sym = "01" if basis == "Z" else "+-"
            seen.add(sym[int(key[1])] + sym[int(key[0])])            # Charlie + Alice
    assert seen == allowed, (bell, basis, seen, allowed)
```

**Line 2.** `Statevector(qc)` works out the exact state, and `probabilities_dict()` returns every outcome
with its exact probability, as in `{'00': 0.5, '11': 0.5}`. Being exact, this covers outcomes that a
sampled run might miss by chance.

**Line 3.** Keeps only outcomes that genuinely occur. The `1e-9` threshold rather than `> 0` guards against
floating-point dust like `3e-17`.

**Line 4.** Builds a two-character translation string. In the Z basis, bit 0 should print as `"0"` and bit 1
as `"1"`, so `sym = "01"`. In the X basis the same bits mean plus and minus, so `sym = "+-"`. Then
`sym[0]` and `sym[1]` convert a bit into the right symbol.

**Line 5.** The ordering line, and the one worth slowing down on. `key` is a Qiskit bitstring, so by
Part 3.3 `key[0]` is qubit 1 and `key[1]` is qubit 0. Qubit 0 is Charlie, qubit 1 is Alice. The paper's
table lists Charlie first, so the code emits `sym[key[1]]` then `sym[key[0]]`, which is Charlie then Alice.
Getting this backwards would still pass for the symmetric rows and fail for the others, which is precisely
why this check is worth running.

**Line 6.** Compares the computed set with the paper's. `assert X, extra` prints `extra` on failure, so a
mismatch would name the case and show both sets rather than just saying "assertion failed".

```python
# Bell measurement identifies each prepared state
for lab in BELL_LABELS:
    assert bell_from_bits(run_circuits([pair_circuit(lab, "ret")], 1)[0]) == lab
print("Bell measurement identifies all four Bell states.")
```

A second, independent check: does the Bell measurement actually work? Prepare each of the four states, run
a full round trip with no gate and no attack, and confirm the measurement names the state you started
with. Reading the nesting outward: build the circuit, wrap it in a one-element list, run it with seed 1,
take result `[0]`, convert the bits to a label, compare. Role `"ret"` is used because that is the branch
that ends in a Bell measurement, and it applies no key gate.

## Cell 9. Building the decoder

Charlie needs to invert Alice's gate: he knows what he prepared and what came back, and from those two
facts he must recover the 2 key bits. This cell builds that lookup table from first principles rather than
hard-coding it from the paper.

```python
DECODE = {}
for initial in BELL_LABELS:
    DECODE[initial] = {}
    for symbol in KEY_GATES:
        qc = QuantumCircuit(2); bell_prep(qc, 0, 1, initial); key_gate(qc, 1, symbol); bell_transform(qc, 0, 1)
```

**Line 4.** Iterating a dict yields its keys, so `symbol` runs over `"00"`, `"01"`, `"10"`, `"11"`.

**Line 5.** All 16 combinations of starting state and gate. The three calls are the protocol in miniature:
prepare, encode on qubit 1 which is the travelling half, then undo the entanglement ready for measurement.

```python
        pr = Statevector(qc).probabilities([0, 1])
        k = int(np.argmax(pr)); assert abs(pr[k] - 1) < 1e-12
        DECODE[initial][BELL_LABELS[k]] = symbol
```

**Line 1.** The exact outcome probabilities, as an array of four numbers. Per Part 3.4, naming `[0, 1]`
makes qubit 0 the low bit, so the array index is `bits[0] + 2*bits[1]`, matching `BELL_LABELS`.

**Line 2.** `np.argmax` finds the most likely outcome. The assertion then checks that its probability is 1
to within rounding, which verifies something important: the result is **deterministic**. Charlie is not
guessing. Given what he prepared, each gate produces exactly one possible Bell state. If this assertion
ever failed, the protocol would not work at all.

**Line 3.** Stores the mapping in the direction Charlie needs it: given the starting state, the observed
state tells you the symbol. Note the key is the *observed* state and the value is the symbol, which is the
inverse of the direction the loop computed it in.

```python
    assert len(DECODE[initial]) == 4
    print(initial, "->", DECODE[initial])
```

Checks that the four gates produced four *different* Bell states. If two gates landed on the same state,
this dict would have fewer than 4 entries and the code could not tell those two key symbols apart. The
printed output confirms it, for example `U+ -> {'U+': '00', 'U-': '01', 'W+': '10', 'W-': '11'}`.

```python
# confirm with Aer sampling: all 16 cases
for initial in BELL_LABELS:
    for symbol in KEY_GATES:
        got = bell_from_bits(run_circuits([pair_circuit(initial, "auth", symbol=symbol)], 7)[0])
        assert DECODE[initial][got] == symbol
print("All 16 encode/decode cases confirmed on Aer.")
```

The same 16 cases again, but through the real pipeline: the actual `pair_circuit` function, actually run on
the simulator, with actual measurements. The first loop proved the mathematics; this proves the
implementation agrees with the mathematics. If `pair_circuit` had a wiring bug, for example measuring `C`
and `T` into the wrong classical bits, the first loop would still pass and this one would fail.

## Cell 11. The full protocol

The centrepiece. One call to `authenticate` is one complete run of the paper's Algorithm 1.

### The result record

```python
@dataclass
class AuthResult:
    accepted: bool
    reason: str
    forward_error: float
    return_error: float
    key_error: float
    original_key: str
    recovered_key: object
    forward_records: list = field(default_factory=list)
    return_records: list = field(default_factory=list)
    eve_records: list = field(default_factory=list)
```

`@dataclass` is a Python shortcut: list the fields with their types and Python writes the constructor and
the printing for you. The fields:

- `accepted` — the verdict. Did Charlie authenticate Alice?
- `reason` — a human-readable explanation, also used as a machine-readable tag later.
- `forward_error`, `return_error` — the measured error rates of the two eavesdropping checks.
- `key_error` — the fraction of key symbols Charlie recovered wrongly.
- `original_key`, `recovered_key` — Charlie's copy, and what he read off the qubits.
- `forward_records`, `return_records` — per-pair detail, used to render the tables.
- `eve_records` — per-key-pair detail of what Eve got to see.

`field(default_factory=list)` is required rather than `= []` because a plain `[]` default would be a single
list shared by every instance ever created, a classic Python trap.

```python
    @property
    def checks_passed(self):
        """Eve stayed under both eavesdropping thresholds, i.e. raised no alarm in Step 2 or Step 4a.

        This is the metric that matters for Eve, not `accepted`: her goal in Sections 4.2.2 and
        4.2.3 is to steal the key, and losing the final key comparison costs her nothing.
        """
        return self.reason not in ("Forward-channel check failed", "Return-channel check failed")
```

A computed field. `@property` means you write `result.checks_passed` with no parentheses, and the body runs
each time.

The idea behind it is worth understanding, because it is the main analytical point of the notebook. There
are two very different questions you can ask about an attack:

- Did Alice get authenticated? That is `accepted`.
- Did anyone *notice* Eve? That is `checks_passed`.

Eve does not care about the first. Her goal is to learn the key so she can impersonate Alice later. If she
corrupts this run and Charlie rejects Alice, Eve loses nothing; the key is unchanged and she can try again.
So the honest measure of her success is whether she escaped both alarms, and that is what this property
reports. Tables in later cells show both columns side by side.

### Validation

```python
def authenticate(shared_key, N, p, threshold=0.0, key_threshold=0.0, rng=None,
                 forward_channel=None, return_channel=None, alice_key=None):
    rng = np.random.default_rng() if rng is None else rng
    if N % 2 or p % 2 or N <= 0 or p <= 0 or len(shared_key) != N:
        raise ValueError("N and p must be positive even numbers and len(shared_key) must equal N.")
    if set(shared_key) - {"0", "1"}: raise ValueError("Key must be binary.")
    alice_key = shared_key if alice_key is None else alice_key
    if len(alice_key) != N or set(alice_key) - {"0", "1"}: raise ValueError("Alice's key must be N binary bits.")
```

The two channel arguments default to `None`, so a plain call simulates an honest run.

**Line 6.** This is the line that makes impostor testing possible. `shared_key` is *Charlie's* copy and
`alice_key` is what the party claiming to be Alice actually knows. Normally they are the same string. Pass
a different `alice_key` and you are simulating someone who does not know the real key, which is what
Cell 34 does.

### Setting up the pairs

```python
    total = N // 2 + p
    initial = [str(rng.choice(BELL_LABELS)) for _ in range(total)]                    # Step 1
    perm = rng.permutation(total)
    fwd_idx  = sorted(int(i) for i in perm[:p//2])
    ret_idx  = sorted(int(i) for i in perm[p//2:p])
    auth_idx = sorted(int(i) for i in perm[p:])
```

**Line 1.** 20 pairs for the default parameters.

**Line 2.** Step 1: pick each pair's state at random and remember it. `str(...)` converts numpy's string
type to a plain one so later dictionary lookups match. `_` is the conventional name for a loop variable
you do not use.

**Line 3.** `rng.permutation(20)` returns the numbers 0 to 19 in random order. This is how roles get
assigned.

**Lines 4–6.** Slice the shuffled list into three groups: the first 5 become forward-check pairs, the next
5 become return-check pairs, the remaining 10 carry the key. Because the list was shuffled, the roles land
on random positions, which is what the paper specifies. An attacker must not be able to predict which pairs
are being checked.

`sorted(...)` puts each group back into position order, purely so the printed tables read sensibly.
`int(i)` converts numpy integers to Python ones so they work as dict keys consistently.

```python
    assert len(auth_idx) == N // 2 == len(key_symbols(alice_key))                     # zip() below truncates silently
    syms = dict(zip(auth_idx, key_symbols(alice_key)))
    bases = {i: str(rng.choice(["Z", "X"])) for i in fwd_idx}
```

**Line 1.** A chained comparison, equivalent to checking both equalities. The comment says why it exists:
`zip` stops at the shorter of its inputs without complaining, so if the counts ever disagreed, the next
line would quietly encode only part of the key and the bug would look like a protocol failure rather than a
coding error.

**Line 2.** Pairs up each key-carrying position with the key symbol it will encode. `zip` interleaves the
two lists, and `dict` turns the pairs into a lookup from position to symbol.

**Line 3.** Picks the Z or X question independently for each forward-check pair, as Step 2 requires.

### Building and running the batch

```python
    order, circs = [], []
    for i in fwd_idx:  order.append(("fwd", i));  circs.append(pair_circuit(initial[i], "fwd", forward_channel, None, basis=bases[i], rng=rng))
    for i in ret_idx:  order.append(("ret", i));  circs.append(pair_circuit(initial[i], "ret", forward_channel, return_channel, rng=rng))
    for i in auth_idx: order.append(("auth", i)); circs.append(pair_circuit(initial[i], "auth", forward_channel, return_channel, symbol=syms[i], rng=rng))
    rows = run_circuits_full(circs, rng.integers(2**31 - 1))
```

Three loops build all 20 circuits into one list, with `order` recording what each one is so the results can
be sorted out afterwards.

The differences between the three lines are the whole protocol:
- Forward-check pairs get the forward channel and `None` for the return channel, because they never come
  back. They get a `basis`.
- Return-check pairs get both channels but no `symbol`, because no gate is applied.
- Key pairs get both channels and a `symbol`.

Note that `forward_channel` is given to all three. The attack applies to every qubit in that direction,
including the check qubits. That is the honest way to simulate it: Eve cannot tell which qubits are
checks, so she must attack indiscriminately.

**Line 5.** One simulator call for all 20 circuits. `rng.integers(...)` draws the seed from the run's own
generator, so a given `rng` reproduces the whole run exactly.

### Sorting out the results

```python
    f_rec, r_rec, e_rec, recovered = [], [], [], []
    for (role, i), row in zip(order, rows):
        bits = row["bits"]
```

Walks the results alongside `order`, so each result is interpreted according to what its circuit was for.

```python
        if role == "fwd":
            alice, charlie = bits
            obs = "same" if alice == charlie else "opposite"
            f_rec.append({"pair": i, "bell": initial[i], "basis": bases[i], "Alice": alice, "Charlie": charlie,
                          "observed": obs, "expected": RELATION[(initial[i], bases[i])],
                          "error": obs != RELATION[(initial[i], bases[i])]})
```

Step 2's comparison.

**Line 2.** `bits` is `(bit0, bit1)`, and `pair_circuit` wrote Alice to `out[0]` and Charlie to `out[1]`, so
this unpacking is correct. Getting it the wrong way round would not change `same`/`opposite`, which is a
latent trap; the Cell 7 check is what guards against it.

**Line 3.** Reduces the two results to the one fact the protocol compares.

**Lines 4–6.** Records everything needed for the printed table, and computes `error` by looking up what
`RELATION` says *should* have happened and comparing. An error here means the forward trip disturbed the
pair.

```python
        elif role == "ret":
            m = bell_from_bits(bits)
            r_rec.append({"pair": i, "initial": initial[i], "measured": m, "error": m != initial[i]})
```

Step 4a. Name the Bell state that came back and compare it with what was prepared. These pairs had no gate
applied, so anything other than the original state is damage.

```python
        else:
            recovered.append(DECODE[initial[i]][bell_from_bits(bits)])
            e_rec.append({"pair": i, "symbol": syms[i],                      # what Eve got to see
                          "eve_fwd": row["eve_fwd"], "eve_fwd_basis": row["eve_fwd_basis"],
                          "eve_ret": row["eve_ret"], "eve_ret_basis": row["eve_ret_basis"]})
```

Step 4b, the actual decoding. **Line 2** is the heart of the protocol: `DECODE[what I prepared][what I
received]` gives the 2 key bits. Both pieces of knowledge are required, which is exactly why an
eavesdropper needs to attack both directions to learn anything.

**Lines 3–5.** Records what Eve saw on this pair, for the leakage analysis in Cell 30. Note this is
recorded whether or not she gets caught, which is the point: leakage and detection are tracked separately.

### The three decisions

```python
    f_err = float(np.mean([r["error"] for r in f_rec]))
    if f_err > threshold:                                                           # Step 2 outcome
        return AuthResult(False, "Forward-channel check failed", f_err, np.nan, np.nan,
                          shared_key, None, f_rec, [], e_rec)
```

**Line 1.** The forward error rate. `np.mean` over booleans treats `True` as 1, so this is the fraction of
the 5 samples that went wrong. With 5 samples the possible values are 0, 0.2, 0.4, 0.6, 0.8, 1.0.

**Line 2.** Abort if it exceeds the tolerance. The paper words this as "if the error rate is less than
threshold, proceed", which read literally would abort even at exactly the threshold; this code accepts
equality, which is the sensible reading and the only one that works at threshold 0.

**Line 3.** Returns early. `np.nan`, "not a number", marks the two later statistics as never computed
rather than as zero. The empty list is for the return records, which were never evaluated. `e_rec` is
still passed, because Eve's readings happened regardless.

```python
    r_err = float(np.mean([r["error"] for r in r_rec]))
    if r_err > threshold:                                                           # Step 4a
        return AuthResult(False, "Return-channel check failed", f_err, r_err, np.nan,
                          shared_key, None, f_rec, r_rec, e_rec)
```

The same structure for Step 4a. Note the sequencing: the forward check must pass before the return check is
even looked at, matching Algorithm 1, and it is also why a forward failure leaves `return_error` as `nan`.

```python
    # Step 4b. The paper compares key' with key "within a certain margin of error", so this is a
    # rate test on the recovered symbols. One symbol = one Bell measurement = the granularity
    # at which a disturbed pair goes wrong.
    rec_key = "".join(recovered)
    k_err = float(np.mean([a != b for a, b in zip(recovered, key_symbols(shared_key))]))
    ok = k_err <= key_threshold
```

**Line 4.** Joins the 10 recovered symbols into one 20-character string, for display.

**Line 5.** The error rate, counted in **symbols** rather than bits. The comment explains the choice: one
symbol corresponds to one Bell measurement on one pair, so a single disturbed pair spoils exactly one
symbol. Counting in symbols therefore measures how many pairs went wrong, which is the physically
meaningful quantity.

**Line 6.** The verdict. `<=` rather than `<` so that `key_threshold = 0` means "no errors allowed", which
is what a reader expects.

```python
    if ok:
        reason = "Key matched; Alice authenticated" if k_err == 0 else "Key within tolerance; Alice authenticated"
    else:
        reason = "Recovered key mismatch; reject Alice"
    return AuthResult(ok, reason, f_err, r_err, k_err, shared_key, rec_key, f_rec, r_rec, e_rec)
```

Three distinct outcomes rather than two, so that "perfect" and "accepted with some errors" can be
distinguished in the statistics. Cell 32 uses exactly this distinction to show which test is doing the
rejecting.

### Running it once

```python
master = np.random.default_rng(2026)
shared_key = random_key(N, master)
honest = authenticate(shared_key, N, p, ERROR_THRESHOLD, KEY_THRESHOLD, rng=np.random.default_rng(101))
print("Honest run:", honest.accepted, "|", honest.reason)
print("Original key: ", honest.original_key)
print("Recovered key:", honest.recovered_key)
print(f"Forward error: {honest.forward_error} | Return error: {honest.return_error} | Key symbol error: {honest.key_error}")

print("\nForward-check records:"); display(pd.DataFrame(honest.forward_records))
print("Return-check records:");  display(pd.DataFrame(honest.return_records))
```

**Lines 1–2.** A 20-bit key, generated from a fixed seed so the notebook reads the same every time.

**Line 3.** One honest run, no channels, so no attack and no noise. A *separate* fixed seed is used for the
protocol run, keeping key generation and protocol randomness independent.

**Lines 9–10.** `pd.DataFrame` turns a list of dicts into a table, with the dict keys as columns.
Expected output: accepted, both error rates 0, and the recovered key identical to the original. Any other
result would mean the implementation is broken, so this doubles as a smoke test.

## Cell 13. Replaying the paper's worked example

The paper's Section 3.3 works through a concrete run by hand and states the answers. This cell reproduces
it on the simulator. It is the strongest single test in the notebook, because the expected output comes
from the authors rather than from this code.

```python
key_AB = "10101100010101011110"
paper_ops      = ["U10","U10","U11","U00","U01","U01","U01","U01","U11","U10"]
paper_outcomes = ["W+","W+","W-","U+","U-","U-","U-","U-","W-","W+"]       # Psi+,Psi+,Psi-,Phi+,Phi-,Phi-,Phi-,Phi-,Psi-,Psi+
assert ["U" + s for s in key_symbols(key_AB)] == paper_ops
```

**Lines 1–3.** The paper's key, the sequence of gates it says node A applies, and the ten Bell states it
says node B measures. All three are transcribed from the paper.

**Line 4.** Checks the transcription before trusting it. Splitting the key into symbols gives
`["10", "10", "11", "00", ...]`; prefixing each with `"U"` gives `["U10", "U10", "U11", "U00", ...]`, which
should equal the paper's list. This catches a typo in either the key or the gate list.

```python
fw = run_circuits([pair_circuit("U+", "fwd", basis=b) for b in ["Z", "X", "Z", "X", "Z"]], 5)
assert all(a == c for a, c in fw)                                            # U+ is 'same' in both bases (Table 3)
```

The paper's forward check: particles 2, 5, 8, 12 and 17, measured in the bases Z, X, Z, X, Z, with every
pair prepared as `U+`.

**Line 2.** By the table in Part 1.5, `U+` reads "same" in *both* questions, so Alice's and Charlie's
results must agree in all five cases regardless of which basis was picked. `all(...)` checks every one.
Each element of `fw` is a `(bit0, bit1)` tuple, unpacked as `(a, c)` for Alice and Charlie.

```python
assert all(bell_from_bits(b) == "U+" for b in run_circuits([pair_circuit("U+", "ret") for _ in range(5)], 6))
```

The paper's return check: particles 4, 7, 10, 15 and 18 come back and node B measures `U+` on all of them.
Five untouched round trips, each of which must return the state it started in.

```python
outs = [bell_from_bits(b) for b in run_circuits([pair_circuit("U+", "auth", symbol=s) for s in key_symbols(key_AB)], 8)]
assert outs == paper_outcomes, outs
rec = "".join(DECODE["U+"][o] for o in outs)
assert rec == key_AB
print("Paper example reproduced on Aer: ops, Bell outcomes and recovered key", rec, "all match.")
```

**Line 1.** The ten key-carrying pairs. Builds one circuit per key symbol, runs them all, and names the
resulting Bell states.

**Line 2.** Compares against the paper's list, in order. This is the real test. If the gate table, the
Bell preparation, the measurement wiring or the bit ordering were wrong anywhere, this line would fail.

**Lines 3–4.** The round trip: decode the ten observed states back into key bits and confirm you recover
the key you started from. Line 2 checks agreement with the paper; line 4 checks internal consistency.

For a sanity check you can do by hand, start from `U+ = (|00> + |11>)/sqrt(2)` and apply the gate to the
second qubit:
- `U10 = X` flips the second qubit, giving `(|01> + |10>)/sqrt(2) = W+`. The paper's first outcome is `W+`.
- `U01 = Z` puts a minus on `|1>`, giving `(|00> − |11>)/sqrt(2) = U-`. The paper's fifth outcome is `U-`.
- `U00 = I` changes nothing, so `U+`. The paper's fourth outcome is `U+`.

## Cell 15. The Entanglement-Measure attack and Eq. 22–23

This cell addresses the paper's Section 4.2.3. The claim being tested: an attack that is *invisible* to the
Step 2 check necessarily learns *nothing*. If that holds, the attack is useless, and the paper derives it
algebraically. This cell checks it computationally.

### Candidate attacks

```python
def ctrl_ry(theta):
    # Eve rotates her ancilla by Ry(theta) iff the travelling qubit is |1>. theta=0: no attack, theta=pi: full copy.
    c, s = np.cos(theta / 2), np.sin(theta / 2)
    return np.array([[1, 0, 0, 0], [0, 1, 0, 0], [0, 0, c, -s], [0, 0, s, c]], complex)
```

A family of attacks with a dial on them, which is what makes the trade-off measurable.

Reading the matrix with the Part 3.4 convention, index `2*t + a`: rows and columns 0 and 1 are the cases
where the travelling qubit is 0, and rows and columns 2 and 3 are where it is 1. The top-left 2x2 block is
the identity, so **when the travelling qubit is 0, nothing happens to Eve's qubit**. The bottom-right block
is a rotation by `theta`, so **when the travelling qubit is 1, Eve's qubit turns by `theta`**.

That is a partial copy. Eve's qubit ends up tilted by an amount that depends on the travelling qubit's
value, so measuring hers tells her something. How much depends on `theta`:
- `theta = 0` gives the identity. No attack, no information.
- `theta = pi` gives `c = 0, s = 1`, which moves her qubit all the way from `|0>` to `|1>`. A full copy of
  the Z value.
- In between, a partial copy, with correspondingly partial information and partial damage.

```python
CNOT = np.array([[1, 0, 0, 0], [0, 1, 0, 0], [0, 0, 0, 1], [0, 0, 1, 0]], complex)
I2 = np.eye(2, dtype=complex)

def haar(d, seed):
    return random_unitary(d, seed=seed).data
```

`CNOT` written out as a matrix, which is `ctrl_ry(pi)` up to a sign and represents the strongest
copy-style attack. `I2` is the 2x2 identity.

`haar` generates a random valid quantum operation of size `d`, uniformly over all of them. The name refers
to the Haar measure, the mathematically natural uniform distribution on such operations. It is used to
represent "an arbitrary attack nobody designed", so that conclusions do not depend on having picked
convenient examples. `.data` extracts the raw numpy array, and `seed` makes it reproducible.

```python
def eq21_coefficients(U):
    # Matrix index = 2*t + a (t = travelling qubit, a = ancilla); ancilla starts in |0>.
    o0, o1 = U[:, 0].reshape(2, 2), U[:, 2].reshape(2, 2)      # U|0>|0>, U|1>|0>  as [t, a]
    return {"a0|E0>": o0[0], "b0|E1>": o0[1], "a1|E2>": o1[0], "b1|E3>": o1[1]}
```

Extracts the paper's Eq. 21 coefficients from a matrix, so its notation can be matched against the code.
The paper writes the attack as

```
U|0>|E> = a0|0>|E0> + b0|1>|E1>
U|1>|E> = a1|0>|E2> + b1|1>|E3>
```

meaning: send in a travelling qubit of 0 and you get a mixture of "travelling qubit still 0, Eve's qubit in
some state `E0`" and "travelling qubit flipped to 1, Eve's qubit in some state `E1`".

**Line 3.** A matrix column is what the operation does to one input. Eve's qubit always starts at 0, so
only two columns are relevant: index `2*0+0 = 0` for an incoming 0, and index `2*1+0 = 2` for an incoming
1. `reshape(2, 2)` splits each 4-element column into a 2x2 grid indexed `[t][a]`.

**Line 4.** Row `[0]` of that grid is the part where the travelling qubit came out as 0, which is the
paper's `a0|E0>` term. Row `[1]` is where it came out as 1, the `b0|E1>` term. Same for the other column.

### The invisibility test, Eq. 22

```python
def passes_eq22(U, tol=1e-9):
    # Qubit 0 = ancilla, qubit 1 = travelling qubit. Eq. (22): reduced state of T must equal the input state.
    for prep in (lambda qc: None, lambda qc: qc.x(1), lambda qc: qc.h(1), lambda qc: (qc.x(1), qc.h(1))):
        ref = QuantumCircuit(2); prep(ref)
        qc = QuantumCircuit(2); prep(qc); qc.unitary(U, [0, 1])
        rho_in = partial_trace(Statevector(ref), [0]).data
        rho_out = partial_trace(Statevector(qc), [0]).data
        if not np.allclose(rho_in, rho_out, atol=tol): return False
    return True
```

Eq. 22 is the paper's condition for an attack to be invisible in Step 2. In words: after the attack, the
travelling qubit must look exactly as it did before, for each of the four states `|0>`, `|1>`, `|+>`,
`|->`. If it does, no check Alice performs can reveal anything.

**Line 3.** Four small functions, one per test state, each written with `lambda` which is Python's way of
defining a one-line unnamed function.
- `lambda qc: None` does nothing, leaving `|0>`.
- `qc.x(1)` flips to `|1>`.
- `qc.h(1)` gives `|+>`.
- `(qc.x(1), qc.h(1))` runs both, giving `H|1> = |->`. It is written as a tuple only because a lambda
  cannot contain two statements; the tuple is discarded and the side effects are what count.

**Line 4.** A reference circuit with the test state and no attack.

**Line 5.** The same test state, then the attack. `[0, 1]` makes qubit 0 Eve's, matching the convention
used by `eq21_coefficients`.

**Lines 6–7.** `partial_trace(..., [0])` throws away Eve's qubit and returns the description of the
travelling qubit alone. This is the formal meaning of "what Alice sees". `.data` gets the numbers out.

**Line 8.** `np.allclose` compares with a tolerance, which is necessary because these are floating-point
numbers and exact equality would fail on rounding. Any one state failing is enough to make the attack
visible, so it returns `False` immediately.

```python
candidates = {
    "no attack (identity)":                  np.eye(4, dtype=complex),
    "Eve acts only on her ancilla (I x V)":  np.kron(I2, haar(2, 11)),
    "controlled-Ry(pi/4)":                   ctrl_ry(np.pi / 4),
    "controlled-Ry(pi/2)":                   ctrl_ry(np.pi / 2),
    "CNOT (full copy of Z value)":           CNOT,
    "Haar-random 4x4":                       haar(4, 12),
}
print("Eq. (22) satisfied?  (True = invisible to the Step-2 disturbance test)")
for name, U in candidates.items():
    print(f"  {name:40s} {passes_eq22(U)}")
```

Six attacks spanning the range from harmless to aggressive.

`np.kron(I2, haar(2, 11))` deserves a note. `kron` is the tensor product, the operation that combines
single-qubit matrices into a two-qubit one. With `I2` on the travelling qubit and a random operation on
Eve's, this represents Eve fiddling with her own qubit and never touching the message at all. It is
obviously useless as an attack, and it is included as the control case: it is the shape of attack that
Eq. 22 *permits*.

The printed output shows only the identity and that do-nothing case pass. Every attack that reaches toward
the message, including the mildest `ctrl_ry(pi/4)`, is in principle visible.

### Eq. 23, and why it cannot fail

```python
PAULI = [np.eye(2, dtype=complex), np.array([[0, 1], [1, 0]], complex),
         np.array([[0, -1j], [1j, 0]]), np.array([[1, 0], [0, -1]], complex)]
```

The four Pauli matrices in the order `I`, `X`, `Y`, `Z`. These four are a basis, meaning any single-qubit
operation can be written as a combination of them, which is what makes the next function possible. `1j` is
Python's imaginary unit.

```python
def pauli_transfer(U):
    """Pauli transfer matrix of Lambda(rho) = tr_anc(U (rho_T (x) |0><0|_anc) U*), i.e. M[k,j] in
    Lambda(P_j) = sum_k M[k,j] P_k. Qubit 0 = ancilla, qubit 1 = T, so the index is 2*t + a."""
    M = np.zeros((4, 4))
    for j, Pj in enumerate(PAULI):
        rho = np.zeros((4, 4), complex)
        for t1 in range(2):
            for t2 in range(2):
                rho[2 * t1, 2 * t2] = Pj[t1, t2]                   # ancilla in |0><0|
        out = U @ rho @ U.conj().T
        red = np.array([[out[0, 0] + out[1, 1], out[0, 2] + out[1, 3]],
                        [out[2, 0] + out[3, 1], out[2, 2] + out[3, 3]]])   # trace out the ancilla
        for k, Pk in enumerate(PAULI):
            M[k, j] = np.real(np.trace(Pk.conj().T @ red) / 2)
    return M
```

Computes the summary described in Part 1.8: feed each Pauli in, see what comes out, tabulate.

**Lines 5–8.** Builds the input. `rho[2*t1, 2*t2] = Pj[t1, t2]` places the Pauli into the travelling
qubit's slots while leaving Eve's qubit at 0, because only entries with `a = 0` are filled. This is the
tensor product `Pj ⊗ |0><0|` written out by hand in the `2t + a` indexing.

The Paulis `X`, `Y`, `Z` are not themselves valid states, since they have negative eigenvalues. That is
fine: the operation is linear, so feeding in a basis element and reading off the output is legitimate even
though that basis element is not physically preparable on its own.

**Line 9.** Applies the operation. `U rho U-dagger` is the standard way an operation acts on a density
matrix. `@` is matrix multiplication and `.conj().T` is the conjugate transpose.

**Lines 10–11.** Traces out Eve's qubit by hand. The entry `red[t1][t2]` sums over both of Eve's values:
`out[2*t1 + 0, 2*t2 + 0] + out[2*t1 + 1, 2*t2 + 1]`. Writing out `red[0][0] = out[0,0] + out[1,1]` and
`red[0][1] = out[0,2] + out[1,3]` matches the code.

**Lines 12–13.** Projects the result back onto the Pauli basis. `trace(Pk-dagger @ red) / 2` extracts the
`Pk` component, which is the standard formula. `np.real` discards numerically-zero imaginary parts.

```python
def is_product_after(U, tol=1e-9):
    # Eq. (23): U|u>|E> = a0|u>|E0>, so each input column factorises. Rank 1 <=> product state.
    return all(np.linalg.matrix_rank(U[:, col].reshape(2, 2), tol=tol) == 1 for col in (0, 2))
```

Tests the paper's Eq. 23 conclusion: after an invisible attack, the travelling qubit and Eve's qubit end up
**unentangled**, which means her qubit carries no information about the message.

Columns 0 and 2 are again the two relevant inputs. Reshaped to 2x2, the question "is this state a product"
becomes the question "does this matrix have rank 1", which is a standard linear-algebra fact: a 2x2 matrix
factorises into an outer product of two vectors exactly when its rank is 1. `np.linalg.matrix_rank` with a
tolerance handles the floating-point side.

```python
pool = ([np.eye(4, dtype=complex)]
        + [haar(4, s) for s in range(300)]                     # unconstrained attacks
        + [np.kron(I2, haar(2, 1000 + s)) for s in range(300)])  # attacks that never touch T
eq22_pool = [U for U in pool if passes_eq22(U, tol=1e-8)]
assert eq22_pool, "no candidate satisfied Eq. (22)"
assert all(is_product_after(U) for U in eq22_pool)
assert all(np.allclose(pauli_transfer(U), np.eye(4), atol=1e-7) for U in eq22_pool)
```

**Lines 1–3.** 601 candidate attacks: the identity, 300 completely arbitrary ones, and 300 of the
do-nothing shape. The second group is there to give an Eq. 22-passing attack a chance to exist; the first
group is there to give a counterexample a chance to be found.

**Line 4.** Filters down to the ones that are invisible. In practice 301 survive, namely the identity and
the 300 do-nothing attacks. Not one of the 300 arbitrary attacks is invisible.

**Line 5.** Guards against a vacuous result. Without it, a bug that made `passes_eq22` always return
`False` would leave an empty list, and the two assertions below would pass trivially on nothing.

**Lines 6–7.** The actual claims. Every invisible attack leaves the two qubits unentangled, and every one
has the Pauli signature `[1, 1, 1, 1]`, meaning the travelling qubit passed through completely untouched.
An attack that touched nothing cannot have learned anything.

```python
print(f"\nEq. (23) over a pool: {len(eq22_pool)}/{len(pool)} candidate unitaries satisfy Eq. (22), and every")
print("one of them leaves Eve's ancilla in a product state with T and induces the identity channel on T.")
print("\nThat is forced, not luck. Eq. (22) pins three columns of the Pauli transfer matrix: Lambda fixes")
print("I, X and Z. It says nothing about Y, because |0>,|1>,|+>,|-> only span I, X and Z. But a Pauli-")
print("diagonal map with lambda_X = lambda_Z = 1 is completely positive only if 1 - lambda_Z >= |lambda_X - lambda_Y|,")
print("which forces lambda_Y = 1 too. So no physical attack satisfies Eq. (22) and still leaks, and the")
print("paper's Section 4.2.3 derivation for (Step 2) holds in general. Section 7.3 confirms it with chi = 0.")
```

Searching 601 attacks and finding no counterexample is suggestive but not a proof. These lines give the
proof, and the argument is worth following because it closes a real gap.

Eq. 22 constrains the attack using four states: `|0>`, `|1>`, `|+>`, `|->`. In Pauli terms those four pin
down the `I`, `X` and `Z` behaviour, but they say **nothing about `Y`**. So on the face of it, an attack
could satisfy Eq. 22 and still disturb `Y`-type information, which would mean it learned something. The
paper does not address this.

The gap closes because of **complete positivity**, the requirement that any physical process map valid
states to valid states. For a process with Pauli signature `[1, lambda_X, lambda_Y, lambda_Z]` that
requirement includes the inequality

```
1 - lambda_Z >= |lambda_X - lambda_Y|
```

Set `lambda_X = lambda_Z = 1`, as Eq. 22 forces. The left side becomes 0, so the right side must be 0 too,
which means `lambda_Y = 1`. The `Y` behaviour was never free. Any attack satisfying Eq. 22 is the identity,
learns nothing, and the paper's conclusion holds for every physically realisable attack rather than just
the 601 tested.

```python
print("\nPauli transfer diagonal (lambda_I, lambda_X, lambda_Y, lambda_Z) per candidate:")
for name, U in candidates.items():
    print(f"  {name:40s} {np.round(np.diag(pauli_transfer(U)), 4)}   eq22={passes_eq22(U)}")
print("\nCNOT coefficients (Eq. 21):", {k: np.round(v, 3).tolist() for k, v in eq21_coefficients(CNOT).items()})
```

Prints the four-number signature for each candidate, which makes the mechanism visible at a glance:

- identity and the do-nothing attack give `[1, 1, 1, 1]`, nothing touched.
- `ctrl_ry(pi/2)` gives `[1, 0.7071, 0.7071, 1]`. Z-type information intact, X and Y partly degraded. That
  `0.7071` is `cos(pi/4)`, and it is the same number that drives the error rate in Cell 17.
- `CNOT` gives `[1, 0, 0, 1]`. Z-type information perfectly preserved, X and Y completely destroyed. This
  is the fingerprint of someone asking the Z question.

The last line shows `CNOT` expressed in the paper's Eq. 21 notation: `a0|E0>` is `[1, 0]` and `b1|E3>` is
`[0, 1]`, with the other two coefficients zero, which is exactly the paper's `b0 = a1 = 0` case but with
`E0` and `E3` *different*. The paper's Eq. 23 additionally requires `a0|E0> = b1|E3>`, and `CNOT` violates
that, which is why it leaks and why it is detectable.

## Cell 17. How often the forward check catches an EM attack

Cell 15 showed that any informative attack is *in principle* visible. This cell measures how visible, by
running the attack many times and counting errors.

```python
def forward_error_rate(U, n=800, seed=1):
    rr = np.random.default_rng(seed)
    labs = [str(rr.choice(BELL_LABELS)) for _ in range(n)]
    bas = [str(rr.choice(["Z", "X"])) for _ in range(n)]
    outs = run_circuits([pair_circuit(l, "fwd", ("em", U), None, basis=b, rng=rr) for l, b in zip(labs, bas)], seed)
    return float(np.mean([("same" if a == c else "opposite") != RELATION[(l, b)] for (a, c), l, b in zip(outs, labs, bas)]))
```

**Lines 3–4.** 800 random Bell states and 800 random questions, mirroring what the protocol does.

**Line 5.** 800 forward-check circuits, each with the EM attack on the forward leg and nothing on the
return leg, since forward-check pairs never return. All 800 run in one batch.

**Line 6.** The error rate. For each result, reduce it to `same`/`opposite`, compare with what `RELATION`
says it should have been, and average the mismatches. `zip` walks the three lists together.

```python
print(f"{'theta':>8} | {'simulated':>9} | {'theory (1-cos(t/2))/4':>22}")
for th in [0, np.pi/4, np.pi/2, 3*np.pi/4, np.pi]:
    sim, th_val = forward_error_rate(ctrl_ry(th)), (1 - np.cos(th / 2)) / 4
    print(f"{th:8.3f} | {sim:9.3f} | {th_val:22.3f}")
    assert abs(sim - th_val) < 0.05
```

Sweeps the attack strength from none to full and compares simulation against theory.

Where the formula comes from, using the Pauli signature from Cell 15. For `ctrl_ry(theta)` the signature
is `[1, cos(theta/2), cos(theta/2), 1]`.

- In the **Z** question, `lambda_Z = 1` means Z-type information is untouched, so the error rate is **0**.
  An attack of this shape is completely invisible to a Z-basis check.
- In the **X** question, `lambda_X = cos(theta/2)`, and a surviving fraction `lambda` corresponds to an
  error probability of `(1 - lambda)/2`. So the X error rate is `(1 - cos(theta/2))/2`.
- The protocol picks Z or X with equal probability, so the average is half of that: `(1 - cos(theta/2))/4`.

The printed table confirms it, ending at `theta = pi` where the error rate is 0.25. That is the same 1/4
the paper quotes for the intercept-resend attack, which makes sense: a full `CNOT` copies the Z value
exactly as a Z measurement would.

**Line 5.** The tolerance is 0.05 because this is sampled data. At the worst case the standard error is
about 0.015, so 0.05 is roughly three standard errors, loose enough not to fail by chance and tight enough
to catch a real discrepancy.

## Cell 19. Testing the paper's 1/4 claim for the return channel

The paper's Section 4.2.3 says that under an EM attack on the return leg, "Charlie has only a 1/4
probability of getting the correct Bell state". This cell computes the real number.

```python
def p_correct_return(U):
    vals = []
    for b in BELL_LABELS:
        qc = QuantumCircuit(4); bell_prep(qc, C, T, b); qc.unitary(U, [ER, T]); bell_transform(qc, C, T)
        vals.append(Statevector(qc).probabilities([C, T])[BELL_LABELS.index(b)])
    return float(np.mean(vals))
```

**Line 4.** Prepare a Bell state, attack the travelling qubit on its way back, then have Charlie measure.
No `qc.measure` call: the next line computes the exact probability instead of sampling, so the answer has
no statistical error at all.

**Line 5.** `probabilities([C, T])` gives all four outcomes; `BELL_LABELS.index(b)` is the index of the
state Charlie prepared. So this reads off the exact probability that Charlie sees the correct state. The
Part 3.4 convention is what makes the index line up.

**Line 6.** Averages over the four possible starting states, since Eve does not get to choose which one
Charlie prepared.

```python
print(f"{'attack':40s} P(correct Bell state)")
for name, U in candidates.items():
    print(f"  {name:38s} {p_correct_return(U):.3f}")
for th in [np.pi/4, np.pi/2]:
    assert abs(p_correct_return(ctrl_ry(th)) - (1 + np.cos(th/2)) / 2) < 1e-9
```

**Lines 4–5.** Checks the family against its closed form `(1 + cos(theta/2))/2`. The tolerance is `1e-9`
rather than Cell 17's `0.05` because these numbers are exact, so only floating-point rounding is at play.

```python
hv = [p_correct_return(haar(4, s)) for s in range(500)]
print(f"  {'Haar-random (mean of 500)':38s} {np.mean(hv):.3f}   (min {min(hv):.3f}, max {max(hv):.3f})")
print("\nCNOT (strongest copy-style attack) gives 1/2. The Haar-random average is about 1/4, the random-guess level;")
print("the paper's 1/4 matches that average, not a specific attack.")
```

500 arbitrary attacks, to see where the 1/4 actually comes from.

The results settle the question:
- Attacks that satisfy Eq. 22 give **1.000**. Charlie always gets the right state, which is consistent:
  these attacks do nothing, so there is nothing to detect and nothing to learn.
- `CNOT`, the strongest copy-style attack, gives **0.500**.
- The 500 random attacks average **0.252**, with a spread from 0.005 to 0.663.

So 1/4 is not the figure for any particular attack. It is the average over arbitrary attacks, and it
coincides with 1/4 because one correct answer out of four possibilities is the random-guess level. The
paper's claim is not wrong so much as mislabelled: an attacker who actually wants information gets 1/2,
not 1/4, so the protocol's detection power against a real attack is weaker than stated.

## Cell 21. How many key bits Eve can extract

Detection is one question. This cell answers the other one: supposing Eve is not caught, how much of the
key does she hold? The Holevo quantity from Part 1.8 answers it without needing to guess her strategy.

```python
def rho_E(initial, symbol, fwd_U, ret_U):
    qc = QuantumCircuit(4); bell_prep(qc, C, T, initial)
    if fwd_U is not None: qc.unitary(fwd_U, [EF, T])
    key_gate(qc, T, symbol)
    qc.unitary(ret_U, [ER, T])
    return partial_trace(Statevector(qc), [C, T]).data
```

Computes the state of Eve's two private qubits after a complete round trip.

**Lines 2–5.** The protocol in order: prepare, attack outbound into `EF`, Alice's key gate, attack inbound
into `ER`. The `if` on line 3 allows a return-only attack. Line 5 has no guard, so `ret_U` must always be
supplied.

**Line 6.** Discards `C` and `T`, the two qubits Charlie ends up with, leaving `EF` and `ER`. This is the
formal statement of "everything Eve holds, and nothing else". Her two attack qubits are kept together
because she can measure them jointly, which is more powerful than one at a time.

```python
def holevo(fwd_U, ret_U):
    rhos = [np.mean([rho_E(b, s, fwd_U, ret_U) for b in BELL_LABELS], axis=0) for s in KEY_GATES]
    S = lambda r: entropy(DensityMatrix(r), base=2)
    return float(S(np.mean(rhos, axis=0)) - np.mean([S(r) for r in rhos]))
```

**Line 2.** For each of the four key symbols, average Eve's state over the four possible starting Bell
states. Averaging density matrices is the correct way to express "it was one of these four and Eve does not
know which", and it is where Charlie's randomisation in Step 1 enters the calculation. The result is four
density matrices, one per symbol.

**Line 3.** Entropy in bits. `base=2` is what makes the unit a bit rather than a nat.

**Line 4.** The Holevo formula: the entropy of the overall average, minus the average of the individual
entropies. Intuitively, the first term is Eve's total uncertainty and the second is the part of it that is
noise rather than signal; the difference is the part that actually tracks the key.

```python
print("Holevo information about the 2-bit key symbol (bits, max 2):")
print(f"  return channel only, CNOT                     : {holevo(None, CNOT):.4f}")
print(f"  return channel only, Haar-random              : {holevo(None, haar(4, 5)):.4f}")
print(f"  both channels, CNOT + CNOT                    : {holevo(CNOT, CNOT):.4f}")
print(f"  both channels, ctrl-Ry(pi/2) x2               : {holevo(ctrl_ry(np.pi/2), ctrl_ry(np.pi/2)):.4f}")
print(f"  Eq.(22)-safe forward + CNOT return            : {holevo(np.kron(I2, haar(2, 6)), CNOT):.4f}")
assert abs(holevo(CNOT, CNOT) - 1.0) < 1e-6 and abs(holevo(None, CNOT)) < 1e-6
```

The results are the most informative output in the notebook:

| Attack | chi, bits out of 2 |
|---|---|
| return channel only, `CNOT` | **0.0000** |
| return channel only, random | **0.0000** |
| both channels, `CNOT` and `CNOT` | **1.0000** |
| both channels, `ctrl_ry(pi/2)` twice | 0.3905 |
| Eq. 22-safe forward, `CNOT` return | **0.0000** |

Attacking only the return channel yields **exactly nothing**, no matter how hard Eve attacks. The reason
is worth stating plainly: Alice's gate changes the Bell *state*, but a single qubit of an entangled pair
looks completely random on its own regardless of which Bell state it belongs to. Since Charlie's half is
beyond Eve's reach, there is nothing in the returning qubit to read. She has to learn the starting state
from the forward leg and the finishing state from the return leg, and only the combination of the two tells
her anything. The last row makes the same point from the other side: a forward attack that satisfies
Eq. 22 is useless even when paired with a maximally aggressive return attack.

With both channels fully attacked she gets 1 bit out of 2. That is the Z-type bit, consistent with `CNOT`'s
`[1, 0, 0, 1]` signature in Cell 15: she learns everything about Z and nothing about X.

```python
# Eq. (22)-compliant attacks must leak nothing. Check the whole pool from Section 7, not one example.
chis = [holevo(U, U) for U in eq22_pool[:40]]
assert max(abs(c) for c in chis) < 1e-9, f"an Eq.(22)-safe attack leaked {max(chis)} bits"
print(f"\nAll {len(chis)} Eq.(22)-compliant attacks applied to both channels give chi = 0 bits exactly.")
print("Note which channel matters: return-only CNOT also gives 0, because with the initial Bell state")
print("unknown the returning qubit is maximally mixed. Eve needs both channels to learn anything.")
```

Ties Cell 15 and this cell together. Cell 15 showed Eq. 22-compliant attacks leave the qubit untouched;
this confirms they also extract zero bits, measured independently. 40 of the 301 are used rather than all
301 because each `holevo` call runs 16 simulations and 40 is enough to be convincing while keeping the
notebook fast.

## Cell 23. The full trade-off: information against detection

Cells 17, 19 and 21 each measured one piece. This cell runs complete authentications and puts the pieces in
one table, so the trade-off Eve faces is visible.

```python
def acceptance(trials, base=70_000, **kw):
    """Returns (final accept rate, forward-check pass rate, both-checks pass rate).

    The last one is Eve's metric: if both checks pass she was never noticed, whatever the
    final key comparison says.
    """
    acc = fwd_pass = both_pass = 0
    for t in range(trials):
        rr = np.random.default_rng(base + t)
        r = authenticate(random_key(N, rr), N, p, ERROR_THRESHOLD, KEY_THRESHOLD, rng=rr, **kw)
        acc += r.accepted
        fwd_pass += (r.reason != "Forward-channel check failed")
        both_pass += r.checks_passed
    return acc / trials, fwd_pass / trials, both_pass / trials
```

Runs many authentications under the same conditions and reports three rates.

**Line 7.** `**kw` collects any extra keyword arguments and forwards them to `authenticate`, so the same
function works for every attack without knowing their names.

**Line 9.** A fresh generator per trial, seeded `base + t`. Using separate `base` values for different
experiments keeps them independent; numpy's generator spreads sequential seeds properly, so consecutive
integers do not produce correlated streams.

**Lines 11–13.** The three counters. Adding a boolean to an integer works in Python because `True` is 1.
- `acc` counts full acceptances.
- `fwd_pass` counts runs that got past Step 2.
- `both_pass` counts runs where neither alarm fired, which is Eve's success metric.

```python
TR = 100
print(f"{'theta':>7} | {'Holevo (bits/symbol)':>20} | {'fwd check pass':>14} | {'both checks pass':>16} | {'FINAL ACCEPT':>12}")
for th in [0, np.pi/8, np.pi/4, np.pi/2, 3*np.pi/4, np.pi]:
    U = ctrl_ry(th)
    a, f, bp = acceptance(TR, forward_channel=("em", U), return_channel=("em", U))
    print(f"{th:7.3f} | {holevo(U, U):20.4f} | {f:14.3f} | {bp:16.3f} | {a:12.3f}")
```

The core sweep: dial the attack from nothing to full, attacking both channels, and watch information and
detection move together.

Reading the output, the trade-off is strict. At `theta = pi/8` Eve gets 0.02 bits per symbol and slips past
both checks 88% of the time. At `theta = pi` she gets the full 1 bit but is caught 97% of the time. There is
no setting where she gets meaningful information cheaply, which is the protocol working as intended.

```python
print(f"\nOther EM variants ({TR} trials):")
for label, kw in [("return-only CNOT",                    dict(return_channel=("em", CNOT))),
                  ("Eq.(22)-safe forward + CNOT return",  dict(forward_channel=("em", np.kron(I2, haar(2, 21))), return_channel=("em", CNOT))),
                  ("Haar-random on both channels",        dict(forward_channel=("em", haar(4, 22)), return_channel=("em", haar(4, 23))))]:
    a, f, bp = acceptance(TR, **kw)
    print(f"  {label:38s} fwd pass={f:.3f}  both checks pass={bp:.3f}  FINAL ACCEPT={a:.3f}")
```

Three specific strategies, each making a point. The first two pass the forward check 100% of the time,
because neither disturbs the forward leg, and both still get caught by the return check. Combined with
Cell 21, where both scored `chi = 0`, these are the worst possible strategies: all of the risk and none of
the information.

```python
# A weak attack on every particle is not Eve's only option. A strong attack on a few particles
# trades detection for information along a different curve, and the ctrl-Ry sweep above never
# reaches it: Eve keeps the full CNOT but fires it on a fraction f of the particles.
print(f"\nSparse CNOT ({TR} trials): full-strength copying on a fraction f of particles only")
print(f"  {'f':>5} | {'fwd check pass':>14} | {'both checks pass':>16} | {'FINAL ACCEPT':>12} | {'pairs hit both ways':>19}")
for f_att in [0.05, 0.1, 0.2, 0.4, 1.0]:
    ch = ("sparse", f_att, CNOT)
    a, f, bp = acceptance(TR, base=71_000, forward_channel=ch, return_channel=ch)
    print(f"  {f_att:5.2f} | {f:14.3f} | {bp:16.3f} | {a:12.3f} | {f_att**2 * (N//2):19.2f}")
```

A different attack shape, and the reason it is here is that the sweep above cannot reach it. Instead of
attacking weakly everywhere, Eve attacks at full strength on a random fraction `f` and leaves the rest
alone.

This matters because the two strategies sit on different curves. At `f = 0.05` she escapes both checks
**79%** of the time, where the weak-everywhere attack never does better than 88% while getting almost
nothing. She gets less per run but she survives, and survival is what lets a leak accumulate.

**Line 10.** `f_att**2 * (N//2)` is the expected number of key pairs she hits in *both* directions, which
by Cell 21 is the only kind that teaches her anything. Each pair needs an independent hit on each leg, so
the probability is `f` squared, times the 10 key pairs. At `f = 0.1` that is 0.1 pairs per run, which is
small but not zero.

## Cell 26. Negative tests

A short cell that checks the protocol rejects things it should reject. Without tests like these, an
implementation that accepted everything would look like it worked.

```python
wrong_key = ("1" if shared_key[0] == "0" else "0") + shared_key[1:]
```

Flips the first bit of the key and keeps the rest. The conditional expression reads "1 if the first
character was 0, otherwise 0", and `shared_key[1:]` is everything from position 1 onwards. The result
differs from the real key in exactly one bit, which is the hardest case to detect.

```python
tests = {
    "Wrong key":              dict(alice_key=wrong_key, seed=202),
    "Forward IMR (Z)":        dict(forward_channel="imr_z", seed=303),
    "Return IMR (Z)":         dict(return_channel="imr_z", seed=404),
    "Return depol q=0.5":     dict(return_channel=("depol", 0.5), seed=505),
    "EM CNOT, both channels": dict(forward_channel=("em", CNOT), return_channel=("em", CNOT), seed=606),
}
for name, kw in tests.items():
    seed = kw.pop("seed")
    r = authenticate(shared_key, N, p, ERROR_THRESHOLD, KEY_THRESHOLD, rng=np.random.default_rng(seed), **kw)
    print(f"{name:24s} accepted={r.accepted!s:5} | checks_passed={r.checks_passed!s:5} | {r.reason}")
    print(f"{'':24s} fwd_err={r.forward_error:.2f}  ret_err={r.return_error}  key_err={r.key_error}")
```

Five scenarios, each of which must be rejected, with a fixed seed each so the output is stable.

**Line 9.** `kw.pop("seed")` removes the seed from the dictionary and returns it, leaving only the
arguments `authenticate` actually accepts. Note this mutates `tests`, so re-running this cell alone without
re-running the definition would fail. Running the whole notebook in order, as intended, is fine.

**Line 11.** `!s` forces string conversion before the width padding is applied, which is needed because
`:5` cannot pad a boolean directly.

The expected output, and why each case is interesting:

- **Wrong key** passes both eavesdropping checks and fails only the key comparison, with `key_err = 0.1`.
  One wrong bit spoils exactly one of the ten symbols, so 0.1 is correct. This also shows the two
  mechanisms are independent: an impostor who knows nothing about the channel is caught by the key test
  alone.
- **Forward IMR** shows `fwd_err = 0.00` yet is still rejected, by the return check. Eve got lucky on all
  five forward samples, which happens about 24% of the time, and was caught downstream. A good
  demonstration that the forward check alone is not sufficient.
- **Return IMR**, **depolarising noise** and **EM CNOT** are all caught by the return check.

Note that `key_err` is `nan` whenever a check fails, because Charlie abandoned the run and never got as far
as decoding.

## Cell 28. Detection statistics against theory

Checks that the measured detection rates match the paper's analysis, and quantifies how often the checks
miss.

```python
n = 1000; rr = np.random.default_rng(1)
labs = [str(rr.choice(BELL_LABELS)) for _ in range(n)]; bas = [str(rr.choice(["Z", "X"])) for _ in range(n)]
outs = run_circuits([pair_circuit(l, "fwd", "imr_z", None, basis=b, rng=rr) for l, b in zip(labs, bas)], 3)
errs = [("same" if a == c else "opposite") != RELATION[(l, b)] for (a, c), l, b in zip(outs, labs, bas)]
print("Per-sample forward error under IMR(Z): measured", round(float(np.mean(errs)), 3), "(theory 0.25)")
print(f"Theory: P(forward check misses IMR) = (3/4)^{p//2} = {0.75**(p//2):.3f}")
```

**Lines 1–4.** 1000 forward-check samples under a Z-basis intercept-resend attack, same structure as
Cell 17.

**Line 5.** The measured rate, about 0.263 against a theoretical 0.25.

Where 1/4 comes from, from Part 1.3. Eve measures in Z. If Alice happens to ask the Z question, Eve's
interference is invisible, because she measured in the same basis the check uses. That is half the time.
If Alice asks the X question, Eve has destroyed the X-type correlation, so Alice and Charlie get
independent coin flips and disagree half the time. Multiply: `1/2 x 1/2 = 1/4`.

**Line 6.** The consequence, and the paper's first overstatement. Each sample catches Eve with probability
1/4, so it misses with probability 3/4, and five independent samples all miss with probability
`(3/4)^5 = 0.237`. The paper says Eve "will definitely be detected" by this check. She escapes it almost a
quarter of the time.

```python
# The 1/2 return-channel rate was prose only; measure it. A Z measurement anywhere on the round trip
# collapses the pair, so Charlie's Bell measurement lands on the right state only half the time.
outs_r = run_circuits([pair_circuit(l, "ret", None, "imr_z", rng=rr) for l in labs], 4)
outs_f = run_circuits([pair_circuit(l, "ret", "imr_z", None, rng=rr) for l in labs], 5)
e_r = float(np.mean([bell_from_bits(b) != l for b, l in zip(outs_r, labs)]))
e_f = float(np.mean([bell_from_bits(b) != l for b, l in zip(outs_f, labs)]))
print(f"Per-sample return error under IMR(Z) on the return leg:  measured {e_r:.3f} (theory 0.50)")
print(f"Per-sample return error under IMR(Z) on the forward leg: measured {e_f:.3f} (theory 0.50)")
assert abs(e_r - 0.5) < 0.05 and abs(e_f - 0.5) < 0.05
print(f"Theory: P(return check misses IMR)  = (1/2)^{p//2} = {0.5**(p//2):.3f}\n")
```

The return check's detection rate. Both cases are measured, attack on the way back and attack on the way
out, and both come out at 0.5.

Why 1/2: once Eve measures either half of an entangled pair in the Z basis, the pair is no longer
entangled. Charlie's Bell measurement on a non-entangled pair cannot return a definite Bell state; it lands
on one of two possibilities with equal probability, one of which happens to be the original. Hence a 1/2
error rate regardless of which leg she attacked.

This check is much stronger than the forward one: it misses only `(1/2)^5 = 0.031` of the time. That
asymmetry explains Cell 26, where several attacks sailed through the forward check and were stopped here.

```python
for label, kw in [("Honest", {}),
                  ("Forward IMR (Z)", dict(forward_channel="imr_z")),
                  ("Forward IMR (random basis)", dict(forward_channel="imr_rand")),
                  ("Return IMR (Z)", dict(return_channel="imr_z")),
                  ("Wrong key", dict(alice_key="0" * N))]:
    a, f, bp = acceptance(TR, base=10_000, **kw)
    print(f"{label:28s} forward-check pass={f:6.3f}   both checks pass={bp:6.3f}   FINAL ACCEPT={a:6.3f}")
```

Full runs of the same scenarios. The honest row gives 1.000 across the board, confirming the protocol
accepts legitimate users with no false rejections in the noiseless case. The forward IMR rows show
`forward-check pass` around 0.22 to 0.29, matching the predicted 0.237 from line 6 above. Every attack ends
at `FINAL ACCEPT = 0.000`.

The `"Wrong key"` row uses `"0" * N`, a key of twenty zeros, representing an impostor who knows nothing. It
passes both eavesdropping checks, since it attacks nothing, and fails the key comparison, which is exactly
the intended behaviour.

## Cell 30. What an eavesdropper actually learns

The paper's Section 4.2.2 concludes that an intercept-resend attacker "will not be able to obtain any key
information". This cell tests that claim and finds it false.

Eve's inference is simple enough to state in one sentence. She measures the travelling qubit in the Z basis
on the way out and again on the way back. Of Alice's four gates, `I` and `Z` leave a Z value unchanged,
while `X` and `iY` flip it. So **if the two readings differ, the symbol's first bit is 1**, and if they
agree, it is 0. No guessing, no statistics.

| Key symbol | Gate | Flips the Z value? | First bit |
|---|---|---|---|
| `00` | `I` | no | 0 |
| `01` | `Z` | no | 0 |
| `10` | `X` | yes | 1 |
| `11` | `iY` | yes | 1 |

The "flips" column and the "first bit" column are identical, which is the leak.

```python
def imr_leak(trials, forward_channel, return_channel, base, label):
    """Score what Eve reads from her own IMR bits across full authentications."""
    seen = hits = undetected_bits = n_pass = 0
    for t in range(trials):
        rr = np.random.default_rng(base + t)
        r = authenticate(random_key(N, rr), N, p, ERROR_THRESHOLD, KEY_THRESHOLD, rng=rr,
                         forward_channel=forward_channel, return_channel=return_channel)
```

Runs complete authentications and scores Eve's guesses against the truth. Four counters:
- `seen` — key pairs where she got a reading on both legs, so she can make a guess at all.
- `hits` — guesses that were right.
- `undetected_bits` — bits read during runs where no alarm fired.
- `n_pass` — runs where no alarm fired.

```python
        got = 0
        for e in r.eve_records:                       # key-carrying pairs only
            if e["eve_fwd_basis"] == "Z" and e["eve_ret_basis"] == "Z":
                guess = "1" if e["eve_fwd"] != e["eve_ret"] else "0"
                seen += 1
                hits += (guess == e["symbol"][0])     # first bit of the 2-bit symbol
                got += 1
```

**Line 3.** Only pairs where she measured in Z on *both* legs. This is where the basis strings recorded
back in `pair_circuit` earn their place: without them a `0` from an untouched register would be
indistinguishable from a genuine reading of `0`, and the accuracy figure would be meaningless.

**Line 4.** The inference from the table above, in one line.

**Line 6.** Scores it. `e["symbol"][0]` is the first character of the 2-bit symbol. Adding the boolean
increments on a correct guess.

```python
        n_pass += r.checks_passed
        undetected_bits += got if r.checks_passed else 0
    return {"attack": label,
            "P(both checks pass)": n_pass / trials,
            "Eve bit accuracy": (hits / seen) if seen else float("nan"),
            "key bits read / run": seen / trials,
            "key bits read / run, undetected": undetected_bits / trials}
```

**Line 2.** Counts bits only from runs where nobody noticed. This is the figure that matters for a reused
key, since bits read during a run that triggered an alarm are at least bits the operator could react to.

**Line 6.** `if seen else float("nan")` guards against dividing by zero when an attack never produced a
usable reading.

```python
TRIALS = 300
rows = [imr_leak(TRIALS, "imr_z", "imr_z", 20_000, "full IMR(Z), both channels")]
for f_att in [0.1, 0.2, 0.4]:
    ch = ("sparse_imr", f_att)
    rows.append(imr_leak(TRIALS, ch, ch, 21_000, f"sparse IMR(Z), f={f_att}"))
leak = pd.DataFrame(rows).set_index("attack")
display(leak.round(4))
```

Four variants: full-strength everywhere, then selective at three densities.

The table that comes out:

| attack | P(both checks pass) | Eve bit accuracy | bits read / run | bits read / run, undetected |
|---|---|---|---|---|
| full IMR(Z), both channels | 0.003 | **1.000** | 10.0 | 0.033 |
| sparse IMR(Z), f=0.1 | 0.507 | **1.000** | 0.107 | 0.053 |
| sparse IMR(Z), f=0.2 | 0.260 | **1.000** | 0.433 | 0.100 |
| sparse IMR(Z), f=0.4 | 0.083 | **1.000** | 1.857 | 0.123 |

Accuracy is 1.000 in every row. The paper's claim that she obtains no key information is wrong. What is
true is the other claim: at full strength she is caught 99.7% of the time. Detection works; ignorance was
never the mechanism.

```python
acc = leak["Eve bit accuracy"].dropna()
assert (acc > 0.999).all(), f"Eve's inference should be exact, got {acc.to_dict()}"
```

Asserts the result rather than leaving it as a printed number, so that if a future change broke the
inference the notebook would fail instead of quietly reporting a lower figure. `.dropna()` removes any
`nan` rows.

```python
print("\nRuns needed to read a given key symbol's first bit while never raising an alarm,")
print("assuming the same static key is reused (coupon-collector style, 99% confidence):")
for row in rows:
    per_run = row["key bits read / run, undetected"] / (N // 2)      # per-symbol capture rate
    if per_run > 0:
        n_runs = np.log(0.01) / np.log(1 - min(per_run, 0.999))
        print(f"  {row['attack']:30s} {n_runs:8.0f} authentications")
    else:
        print(f"  {row['attack']:30s}   never undetected in {TRIALS} trials")
```

Converts a per-run leak into a time-to-compromise, which is the figure that actually matters.

**Line 4.** Undetected bits per run, divided by the 10 symbols, gives the chance that one specific symbol
gets captured undetected in one run.

**Line 6.** Standard waiting-time arithmetic. If a given run captures a symbol with probability `per_run`,
then after `n` runs it is still uncaptured with probability `(1 - per_run)^n`. Setting that to 0.01 and
solving gives `n = log(0.01) / log(1 - per_run)`. The `min(..., 0.999)` guards against a logarithm of zero.

The answers range from 371 to 1379 authentications. Whether that is alarming depends on a detail the paper
leaves open: Section 3.2 gives each pair of nodes **one** key, and every node authenticates every published
record, so the number of authentications is large. The paper specifies no re-keying after a failed check,
only that "the communication is abandoned", so Eve can keep trying indefinitely against an unchanged
target. The final two printed lines say exactly this.

## Cell 32. Honest users under noise

Everything so far has been about attackers. This cell asks the opposite question: how much channel noise
can an honest Alice survive? A protocol that rejects legitimate users is useless no matter how secure it
is.

```python
print("Honest Alice, depolarising noise q on every particle in both directions.")
print("The Step 2/4a check threshold and the Step 4 key tolerance are independent knobs.\n")
print(f"{'q':>6} | {'thr=0 key=0':>12} | {'thr=.2 key=0':>13} | {'thr=.2 key=.2':>14} | {'thr=.2 key=.4':>14}")
GRID = [(0.0, 0.0), (0.2, 0.0), (0.2, 0.2), (0.2, 0.4)]
for q in [0.0, 0.02, 0.05, 0.1]:
    row = []
    for thr, kthr in GRID:
        acc = 0
        for t in range(80):
            rr = np.random.default_rng(50_000 + t)
            acc += authenticate(random_key(N, rr), N, p, thr, kthr, rng=rr,
                                forward_channel=("depol", q), return_channel=("depol", q)).accepted
        row.append(acc / 80)
    print(f"{q:6.2f} | {row[0]:12.3f} | {row[1]:13.3f} | {row[2]:14.3f} | {row[3]:14.3f}")
```

Three nested loops: noise level, then threshold pair, then 80 trials.

**Line 4.** The four threshold combinations being compared. Each is `(check threshold, key tolerance)`. The
first entry is the strict default; the others loosen one or both.

**Lines 11–12.** No attack channel here, just `("depol", q)` noise in both directions, representing an
imperfect fibre rather than an adversary.

The resulting table:

| q | thr=0 key=0 | thr=.2 key=0 | thr=.2 key=.2 | thr=.2 key=.4 |
|---|---|---|---|---|
| 0.00 | 1.000 | 1.000 | 1.000 | 1.000 |
| 0.02 | 0.425 | 0.625 | **0.988** | 0.988 |
| 0.05 | 0.150 | 0.275 | 0.863 | 0.900 |
| 0.10 | 0.025 | 0.087 | 0.550 | 0.662 |

Compare columns 2 and 3. Both have the same check threshold of 0.2; the only difference is the key
tolerance. Loosening the check threshold alone took acceptance from 0.425 to 0.625 at `q = 0.02`, while
loosening the key tolerance took it to 0.988. The key comparison, not the eavesdropping checks, is what
sets the noise budget. That is not obvious from the protocol description, and it is the reason the key
tolerance exists as a separate parameter.

```python
# Which test is actually doing the rejecting?
print("\nRejection reasons at q=0.02, by threshold pair:")
import collections
for thr, kthr in GRID:
    reasons = collections.Counter()
    for t in range(300):
        rr = np.random.default_rng(900_000 + t)
        r = authenticate(random_key(N, rr), N, p, thr, kthr, rng=rr,
                         forward_channel=("depol", 0.02), return_channel=("depol", 0.02))
        reasons[r.reason.split(";")[0]] += 1
    print(f"  thr={thr}, key={kthr}: " + ", ".join(f"{k}={v}" for k, v in reasons.most_common()))
```

The explanation behind the table. Instead of counting acceptances, this counts *why* each run ended as it
did.

**Line 3.** `collections.Counter` is a dictionary that counts things. Importing mid-notebook is untidy but
harmless.

**Line 10.** `r.reason.split(";")[0]` keeps the part before the semicolon, turning
`"Key matched; Alice authenticated"` into `"Key matched"`. The check-failure messages contain no semicolon
and pass through unchanged. This is what the `reason` strings were designed for.

**Line 11.** `.most_common()` sorts by count, so the dominant reason appears first.

The output makes the point sharply. At `thr=0.2, key=0.0`, out of 300 runs: 192 accepted, **103 rejected on
the key**, and only 5 on a channel check. The eavesdropping checks have essentially stopped firing, and the
key comparison is rejecting a third of honest users by itself.

```python
print("\nOnce the check threshold is loose, neither eavesdropping check fires and the key comparison is")
print("the only test rejecting honest users, so the Step 4 tolerance is what sets the noise budget.")
```

The conclusion in two lines.

## Cell 34. What the key tolerance costs

Cell 32 showed a loose key tolerance is necessary for noise resilience. This cell shows what you pay for
it. The two cells have to be read together; either one alone gives the wrong impression.

```python
shared_imp = random_key(N, np.random.default_rng(5))
TOLS = (0.0, 0.1, 0.2, 0.4)
print("Impostor who knows all but k of the N key bits; eavesdropping checks at threshold 0.2.")
print(f"{'wrong bits':>10} | " + " | ".join(f"{'key tol=' + str(kt):>12}" for kt in TOLS))
```

**Line 1.** A fixed target key for the impostor to attack.

**Line 4.** Builds the header by joining one label per tolerance. `str.join` with a generator is the
idiomatic way to assemble a row like this.

```python
for wrong in [1, 2, 4]:
    row = []
    for kt in TOLS:
        acc = 0
        for t in range(200):
            rr = np.random.default_rng(600_000 + t)
            guess = list(shared_imp)
            for j in rr.choice(N, size=wrong, replace=False):
                guess[j] = "1" if guess[j] == "0" else "0"
            acc += authenticate(shared_imp, N, p, 0.2, kt, rng=rr, alice_key="".join(guess)).accepted
        row.append(acc / 200)
    print(f"{wrong:10d} | " + " | ".join(f"{v:12.3f}" for v in row))
```

**Line 7.** `list(...)` turns the key string into a list of characters, because strings cannot be modified
in place.

**Line 8.** `rr.choice(N, size=wrong, replace=False)` picks that many *distinct* positions. Without
`replace=False` the same position could be chosen twice, and flipping it twice would undo the change.

**Line 9.** Flips the chosen bits, producing an impostor's key that differs from the real one in exactly
`wrong` places.

**Line 10.** The key line. `shared_imp` is passed as Charlie's key and the corrupted string as
`alice_key`, so Charlie is testing a claimant who *almost* knows the secret. No channel attack is involved;
the impostor is simply guessing.

The result:

| wrong bits | key tol=0.0 | key tol=0.1 | key tol=0.2 | key tol=0.4 |
|---|---|---|---|---|
| 1 | 0.000 | **1.000** | 1.000 | 1.000 |
| 2 | 0.000 | 0.060 | **1.000** | 1.000 |
| 4 | 0.000 | 0.000 | 0.005 | **1.000** |

At a tolerance of 0.1, an impostor who has 19 of the 20 key bits is accepted **every single time**. Put
beside Cell 32, where that same tolerance was needed to keep honest users above 98% at modest noise, the
dilemma is plain.

```python
print("\nk wrong bits corrupt at most k of the N/2 symbols, so any tolerance >= k/(N/2) admits the")
print("impostor. Choosing it is a real design decision the paper leaves open: too tight and honest")
print("nodes fail under channel noise, too loose and partial key knowledge is enough to authenticate.")
```

The arithmetic behind the table. `k` wrong bits spoil at most `k` of the 10 symbols, giving a symbol error
rate of at most `k/10`. Any tolerance at or above that lets the impostor through. One wrong bit gives 0.1,
so a tolerance of 0.1 admits him.

The paper asks for a tolerance in Step 4 but never says how big, and this is why that omission matters.
The two tables bracket the decision from both sides, which is as far as this notebook can take it; picking
a number requires knowing the real channel's error rate and the acceptable impersonation risk.

## Cell 36. Printing the circuits

```python
rr = np.random.default_rng(0)
print("Honest key pair (U+, key symbol 01):")
print(pair_circuit("U+", "auth", symbol="01").draw(output="text", fold=120))
print("\nForward-check pair (W-, X basis):")
print(pair_circuit("W-", "fwd", basis="X").draw(output="text", fold=120))
print("\nIntercept-resend (Z) on both channels, key pair:")
print(pair_circuit("U+", "auth", "imr_z", "imr_z", symbol="01", rng=rr).draw(output="text", fold=120))
print("\nEM attack (CNOT) on both channels, key pair:")
print(pair_circuit("U+", "auth", ("em", CNOT), ("em", CNOT), symbol="01", rng=rr).draw(output="text", fold=120))
```

Four representative circuits as ASCII diagrams, so you can see the gate sequences the rest of the notebook
has been describing. `.draw(output="text")` renders the circuit; `fold=120` wraps at 120 characters.

How to read the diagrams: each horizontal line is a qubit, time flows left to right, and `q_0` to `q_3` are
`C`, `T`, `EF`, `ER`. The three double lines at the bottom are the classical registers `ef`, `er`, `out`.
`M` is a measurement, and the vertical line from an `M` down to a register shows where the bit is stored.
A `■` connected to a box by a vertical line is a controlled gate, with the `■` on the control qubit.

Taking the first diagram, the honest key pair:

```
       ┌───┐               ┌───┐┌─┐
  q_0: ┤ H ├──■─────────■──┤ H ├┤M├
       └───┘┌─┴─┐┌───┐┌─┴─┐└┬─┬┘└╥┘
  q_1: ─────┤ X ├┤ Z ├┤ X ├─┤M├──╫─
```

Left to right: `H` on `q_0` then CNOT onto `q_1` is `bell_prep` making `U+`. The `Z` on `q_1` is Alice's
key gate for symbol `01`. The second CNOT and the second `H` are `bell_transform`. Then both qubits are
measured. `q_2` and `q_3` stay empty because there is no attack. Four of the protocol's steps are visible
in six gates.

In the third diagram you can see two extra `M` symbols appear on `q_1` mid-circuit, before and after the
`Z`. Those are Eve's two intercept-resend measurements, and they are exactly the two readings Cell 30 uses
to deduce the key bit.

---

# Part 5. Glossary

| Term | Meaning |
|---|---|
| **Aer** | Qiskit's simulator. Mimics a real device including random measurement outcomes. |
| **ancilla** | A spare working qubit. Here, the qubits Eve uses to hold stolen information. |
| **basis** | Which question you ask a qubit. This notebook uses only the Z basis (`0` or `1`) and the X basis (`+` or `−`). |
| **Bell measurement** | A joint measurement on two qubits that answers "which of the four Bell states is this?". Implemented by running the preparation circuit backwards, then measuring. |
| **Bell state** | One of the four maximally entangled two-qubit states. Called `U+`, `U-`, `W+`, `W-` in the paper and in this code. |
| **CNOT** | Controlled-NOT. Flips a target qubit if the control qubit is 1. The standard two-qubit gate. |
| **complete positivity** | The requirement that a physical process turn valid states into valid states. Used in Cell 15 to close a gap in the paper's argument. |
| **density matrix** | A general description of a quantum state, able to express "it is one of these but I do not know which". |
| **depolarising noise** | A standard noise model: with probability `q`, apply a random one of `X`, `Y`, `Z`. |
| **EM attack** | Entanglement-Measure. Eve entangles the passing qubit with her own and measures hers later. The paper's Section 4.2.3. |
| **entanglement** | Two or more qubits with only a joint description, no separate individual ones. Their measurement results are correlated. |
| **gate** | A reversible operation on qubits. Unlike measurement, it destroys nothing. |
| **Haar-random** | Uniformly random over all valid quantum operations. Used to represent an arbitrary attack. |
| **Hadamard (`H`)** | The gate that converts between the Z basis and the X basis. |
| **Holevo quantity (`chi`)** | An upper bound on the classical bits any measurement could extract from a quantum state. `chi = 0` means no information is there to find. |
| **IMR attack** | Intercept-Measure-Resend. Eve measures the qubit and forwards the result. The paper's Section 4.2.2. |
| **little-endian** | Qiskit's bit ordering: qubit 0 is the rightmost character of a printed bitstring. |
| **measurement** | Forcing a qubit to answer. Irreversible; it destroys any superposition. |
| **no-cloning** | The theorem that an unknown qubit cannot be copied. The reason eavesdropping leaves a trace. |
| **partial trace** | Discarding some qubits and keeping the best description of the rest. Formalises "what can Eve see". |
| **Pauli matrices** | The four matrices `I`, `X`, `Y`, `Z`. A basis for all single-qubit operations. |
| **Pauli transfer matrix** | A 4x4 summary of what a process does to a qubit. The diagonal `[1,1,1,1]` means nothing happened. |
| **statevector** | The exact mathematical description of a circuit's state, computed without sampling. |
| **superposition** | A blend of `\|0>` and `\|1>` that has no definite value until measured. |
| **unitary** | A valid reversible quantum operation, represented as a matrix. |
| **von Neumann entropy** | Uncertainty of a quantum state, in bits. Input to the Holevo quantity. |

## Where to go next

- `README.md` for the commands to run the notebook.
- The notebook's Section 11 maps each function to the equation or step in the paper it implements.
- The notebook's Section 12 lists the limitations, including the two protocol features this simulation
  does not model.
