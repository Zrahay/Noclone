# Noclone

A Python prototype for **SIH26141: Quantum-Inspired Cyber Threat Detection for Digital Signature Security**, Smart India Hackathon, Egreen Quanta LLP.

The project combines a teleportation-based Quantum Digital Signature (QDS) simulation with a statistical detection engine. It generates signature elements, verifies them through measurement outcomes, and checks for inconsistent signatures and reused transcripts. Detection uses thresholds and hypothesis tests, with **no AI or machine learning**.

## What the project includes

- Bell-pair preparation, quantum teleportation, Pauli corrections, and channel noise using Qiskit and Aer.
- BB84-style signature elements and state-elimination verification, inspired by the P1 protocol described in `protocol/README.md`.
- Acceptance thresholds derived from an independently measured noise floor and a configured probability parameter using Hoeffding's inequality.
- A chi-square goodness-of-fit statistic for match/mismatch counts, when the expected counts support the test.
- Ten attack scenarios in a Streamlit dashboard, with verdicts, event logs, mismatch history, threshold curves, and an empirical noise sweep.
- Tests for quantum primitives, protocol behavior, adversaries, and detection, run through GitHub Actions.

This is a simulation and research prototype. It requires no quantum hardware or training dataset.

## Run Locally

Use **Python 3.12**. Run these commands from the repository root:

```bash
git clone https://github.com/Zrahay/Noclone.git
cd Noclone
python3.12 -m venv .venv
source .venv/bin/activate
python -m pip install -r requirements.txt
python -m pip install -e .
python -m streamlit run app/main.py
```

On Windows, create the environment with `py -3.12 -m venv .venv` and activate it in PowerShell with `.venv\Scripts\Activate.ps1`.

Open [http://localhost:8501](http://localhost:8501). The editable install makes the project modules available to Streamlit and scripts.

### Try the Demo

1. Choose a signer ID, the number of signature elements per message bit (`L`), and the message length in the sidebar. Click **Generate Keys**.
2. Enter a binary message with the selected length and click **Sign**.
3. Run an attack and inspect its verdict, mismatch rate, and explanation in the event log.
4. Adjust channel noise and compare the mismatch history, threshold curve, and noise sweep.

The dashboard starts with `L = 256` and a four-bit message. Initial calibration and noise sweeps can take time because they execute real Aer circuits; results are cached within the Streamlit session.

### Backend Behavior

The dashboard uses `MockQuantumCore` for fast live signing and verification. This is an analytic reference model: it reproduces the ideal measurement behavior of the states used here, but its noise is modeled as independent bit flips.

Noise-floor calibration and the noise sweep use `M1QuantumCore`, which runs Qiskit/Aer circuits with a depolarizing channel. These two noise models differ, so a noisy live dashboard run is a **hybrid demonstration**, not a consistent end-to-end Aer experiment. For experiments with a single backend, pass `M1QuantumCore` explicitly to the protocol API. See [the protocol documentation](protocol/README.md) for its interfaces and assumptions.

## How Detection Works

```text
keygen -> sign -> verify -> MeasurementRecord[] -> evaluate -> DetectionResult
                  ^                              |
             attack scenario               noise calibration
```

Verification retains conclusive measurement elements and compares their observed bits with the predicted bits. For `n` conclusive records, detection computes the mismatch rate `r` and the following thresholds:

```text
s_a = min(noise_floor + sqrt(-ln(p) / (2*n)), 1)
s_v = min(noise_floor + sqrt(-ln(p/2) / (2*n)), 1)
```

Here, `p` is the configured `target_forgery_prob` parameter, defaulting to `1e-6`. The floor comes from a separate legitimate calibration run, rather than the signature being classified.

| Condition | Verdict |
| --- | --- |
| Previously seen nonce | `REJECT`, reported as `REPLAY` |
| No conclusive measurements | `REJECT`, insufficient data |
| `r < s_a` | `ACCEPT` |
| `s_a <= r < s_v` | `ACCEPT_NO_TRANSFER` |
| `r >= s_v` | `REJECT` |

The chi-square result is supporting evidence; it does not independently determine the verdict. The test needs an expected count of at least five in each cell. When this condition is unmet, the current API returns the placeholder `(0.0, 1.0)`, which should be interpreted as **test unavailable**, rather than evidence that the signature is safe.

Replay detection uses a classical nonce history. Other statistical rejections are reported together as `FORGERY`: measurement statistics alone do not reliably distinguish forgery, impersonation, and channel tampering in this implementation.

## Attack Scenarios

The dashboard exposes:

| Required scenarios | Additional demonstration models |
| --- | --- |
| Forgery | Photon-number splitting (PNS) |
| Impersonation | Collective attack |
| Replay | Faked-state attack |
| Channel tampering | Time-shift attack |
| | Trojan-horse attack |
| | Classical man-in-the-middle attack |

The additional scenarios use simplified signature-level mutations. They do not reproduce physical optical equipment or establish security against general coherent quantum attacks. The attack package also includes partial-key forgery and batch/strength sweep helpers for experiments.

## Security Assumptions and Limits

- Hoeffding's inequality assumes independent measurement outcomes. Calibration is a finite-sample estimate, not an exact channel property.
- The threshold formulas do **not**, on their own, prove a forgery acceptance probability of `1e-6`. The displayed `forgery_prob_bound` round-trips the configured parameter through the Hoeffding formula; it is not a measured attack success probability.
- Small sample sizes produce wide acceptance margins. Some forged signatures can pass, and choosing a suitable `L` requires an attack-specific analysis. Larger `L` does not automatically prove security.
- An observed forgery mismatch rate around one quarter is a measurement statistic, not a forgery success probability or a universal bound on attackers.
- This prototype does not establish the full security guarantees of the source protocol or provide production-ready digital signatures. Keys, event logs, and replay history are held in session memory.

## Repository Layout

| Path | Responsibility |
| --- | --- |
| `core/` | Bell pairs, teleportation, measurement, corrections, and Aer execution |
| `protocol/` | Configuration, quantum adapters, key generation, signing, and verification |
| `attacks/` | Adversaries and experiment helpers |
| `detection/` | Mismatch statistics, thresholds, and verdicts |
| `app/main.py` | Streamlit dashboard |
| `contracts.py` | Shared types exchanged between modules |
| `tests/` | Automated checks |
| `.github/workflows/ci.yml` | Install and test workflow for pushes and pull requests |

## Tests

With the environment activated:

```bash
python -m pytest -q
```

For the quantum execution checks specifically:

```bash
python -m pytest -q tests/test_runtime.py tests/test_teleportation.py
```

## Docker

```bash
docker build -t noclone .
docker run --rm -p 8501:8501 noclone
```

Open [http://localhost:8501](http://localhost:8501). The image runs the same Streamlit dashboard and requires sufficient CPU time for Aer calibration.

## Team

| Track | Owner |
| --- | --- |
| Quantum core | Ashab |
| QDS protocol | Shubhang |
| Attack simulations | Nikita |
| Detection engine | Hemang |
| Dashboard | Yuvraj, with implementation contributions from Nikita |
| Documentation and benchmarks | Anurag |

Shared contracts and dependencies are coordinated across tracks; keep changes scoped and include a runnable check for non-trivial logic.
