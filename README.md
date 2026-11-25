# Quantum Identity Authentication — Qiskit implementation

Circuit-level Qiskit/Aer simulation of the identity-authentication procedure in Qu, Zhang & Zheng,
*"A quantum blockchain-enabled framework for secure private electronic medical records in the Internet
of Medical Things"*, Information Sciences 612 (2022) 942–958 — Section 3.2 / Algorithm 1, with the
attack analysis of Section 4.2.

Scope: the authentication module only. The quantum hash function, the quantum blockchain and BB84/B92
key distribution are not implemented; the shared key is an input.

- Notebook: `Quantum_Identity_Authentication_Qiskit_v2.ipynb`
- Paper: `2022A quantum blockchain-enabled framework ... .pdf`

## Running it

Both commands are run from the project root, and use the venv's `jupyter` directly so no `activate`
step is needed.

```bash
cd /Users/acube/Rajaram/mini-project-qunatum
```

### Interactive — opens in a browser

```bash
.venv/bin/jupyter lab Quantum_Identity_Authentication_Qiskit_v2.ipynb
```

Starts a local server and opens the notebook in your browser; use Run All and read the tables there.
Stop it afterwards with `.venv/bin/jupyter lab stop 8888`. If a server is already running, check
`.venv/bin/jupyter lab list` and open the URL it prints rather than starting a second one. In an IDE,
open the notebook and select the **QEMR auth (qiskit)** kernel instead.

### Headless regression check — no browser

```bash
.venv/bin/jupyter execute --inplace Quantum_Identity_Authentication_Qiskit_v2.ipynb
```

Runs every cell in a background kernel, writes the outputs back into the file and exits, in about
30 seconds. Nothing opens; check the result in the notebook or from the exit code. The notebook asserts
its ground truth — Table 3, the Eq. (20) decoder, the Section 3.3 worked example, the theoretical error
rates — so this exits non-zero if anything regresses. Drop `--inplace` to leave the saved outputs
untouched.

## One-time setup

Requires [uv](https://docs.astral.sh/uv/). Verified on macOS with Python 3.14, Qiskit 2.6.2 and
Aer 0.18.4.

```bash
uv venv .venv --python 3.14
uv pip install --python .venv/bin/python "qiskit>=1.0,<3" "qiskit-aer>=0.14,<1" numpy pandas jupyterlab
.venv/bin/python -m ipykernel install --user --name qemr-auth --display-name "QEMR auth (qiskit)"
```

Don't run the notebook with the system `python3`: it has no Qiskit, and macOS blocks installing into it
(PEP 668). Cell 1 carries a `%pip install` line so the notebook also bootstraps itself on a bare kernel.

## What the notebook checks

| Section | Contents |
|---|---|
| 2–4 | Bell-state preparation, the Eq. (20) key gates, Bell measurement; Table 3 verified exactly for all 8 (state, basis) cases; the encode/decode table derived and confirmed on Aer |
| 5 | Full Algorithm 1: N/2 + p pairs split by random permutation into forward-check, return-check and key-carrying roles, one batched `backend.run` per authentication |
| 6 | Regression test against the paper's Section 3.3 worked example |
| 7 | Entanglement-Measure attack, Eqs. (21)–(24): the Eq. (22) invisibility test, Eq. (23) over a 601-unitary pool, exact return-channel detection, and the Holevo bound on what Eve can learn |
| 8–9 | Wrong key, intercept-measure-resend, depolarising noise, and Monte-Carlo detection statistics against theory |
| 9.1 | What an IMR eavesdropper actually reads off the key |
| 10–12 | Representative circuit diagrams, the mapping to the paper, and the limitations |

Three of the paper's claims do not survive the simulation:

- *"Eve will **definitely** be detected"* (§4.2.2). With p/2 = 5 check samples the forward check misses
  an intercept-resend attacker 24% of the time, since (3/4)^5 = 0.237.
- *"She will not be able to obtain **any** key information"* (§4.2.2). Two Z-basis measurements on the
  round trip give her the first bit of every key symbol with certainty — `I` and `Z` leave a Z eigenstate
  alone, `X` and `iY` flip it. Detection and leakage are separate questions, and only the first holds.
- *The 1/4 probability of Eq. (24)* (§4.2.3). For an Eq. (22)-compliant attack the correct figure is 1,
  and for a full CNOT it is 1/2. The 1/4 is the Haar-random average, i.e. the random-guess level.
