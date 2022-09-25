---
jupyter:
  jupytext:
    text_representation:
      extension: .md
      format_name: markdown
      format_version: '1.3'
      jupytext_version: 1.13.8
  kernelspec:
    display_name: Python 3 (ipykernel)
    language: python
    name: python3
---

# Examples for OptPulseProcessor


Author: Boxi Li (etamin1201@gmail.com)

```python
from numpy import pi
from qutip import basis, fidelity, identity, sigmax, sigmaz, tensor
from qutip_qip.circuit import QubitCircuit
from qutip_qip.device import OptPulseProcessor
from qutip_qip.operations import expand_operator, toffoli
from qutip.ipynbtools import version_table
import qutip_qip
```

The `qutip.OptPulseProcessor` is a noisy quantum device simulator integrated with the optimal pulse algorithm from the `qutip.control` module. It is a subclass of `qutip.Processor` and is equipped with a method to find the optimal pulse sequence (hence the name `OptPulseProcessor`) for a `qutip.QubitCircuit` or a list of `qutip.Qobj`. For the user guide of `qutip.Processor`, please refer to [the introductory guide](https://qutip.org/docs/latest/guide/qip/qip-processor.html).

## Single-qubit gate
Like in the parent class `Processor`, we need to first define the available Hamiltonians in the system. The `OptPulseProcessor` has one more parameter, the drift Hamiltonian, which has no time-dependent coefficients and thus won't be optimized.

```python
num_qubits = 1
# Drift Hamiltonian
H_d = sigmaz()
# The (single) control Hamiltonian
H_c = sigmax()
processor = OptPulseProcessor(num_qubits, drift=H_d)
processor.add_control(H_c, 0)
```

The method `load_circuit` calls `qutip.control.optimize_pulse_unitary` and returns the pulse coefficients.

```python
qc = QubitCircuit(num_qubits)
qc.add_gate("SNOT", 0)

# This method calls optimize_pulse_unitary
tlist, coeffs = processor.load_circuit(
    qc, min_grad=1e-20, init_pulse_type="RND", num_tslots=6,
    evo_time=1, verbose=True
)
processor.plot_pulses(
    title="Control pulse for the Hadamard gate", use_control_latex=False
);
```

Like the `Processor`, the simulation is calculated with a QuTiP solver. The method `run_state` calls `mesolve` and returns the result. One can also add noise to observe the change in the fidelity, e.g. the t1 decoherence time.

```python
rho0 = basis(2, 1)
plus = (basis(2, 0) + basis(2, 1)).unit()
minus = (basis(2, 0) - basis(2, 1)).unit()
result = processor.run_state(init_state=rho0)
print("Fidelity:", fidelity(result.states[-1], minus))

# add noise
processor.t1 = 40.0
result = processor.run_state(init_state=rho0)
print("Fidelity with qubit relaxation:", fidelity(result.states[-1], minus))
```

## Multi-qubit gate


In the following example, we use `OptPulseProcessor` to find the optimal control pulse of a multi-qubit circuit. For simplicity, the circuit contains only one Toffoli gate.

```python
toffoli()
```

We have single-qubit control $\sigma_x$ and $\sigma_z$, with the argument `cyclic_permutation=True`, it creates 3 operators each targeted on one qubit.

```python
N = 3
H_d = tensor([identity(2)] * 3)
test_processor = OptPulseProcessor(N, H_d)
test_processor.add_control(sigmaz(), cyclic_permutation=True)
test_processor.add_control(sigmax(), cyclic_permutation=True)
```

The interaction is generated by $\sigma_x\sigma_x$ between the qubit 0 & 1 and qubit 1 & 2. `expand_operator` can be used to expand the operator to a larger dimension with given target qubits.

```python
sxsx = tensor([sigmax(), sigmax()])
sxsx01 = expand_operator(sxsx, 3, targets=[0, 1])
sxsx12 = expand_operator(sxsx, 3, targets=[1, 2])
test_processor.add_control(sxsx01)
test_processor.add_control(sxsx12)
```

Use the above defined control Hamiltonians, we now find the optimal pulse for the Toffoli gate with 6 time slots. Instead of a `QubitCircuit`, a list of operators can also be given as an input.

```python
def get_control_latex():
    """
    Get the labels for each Hamiltonian.
    It is used in the method``plot_pulses``.
    It is a 2-d nested list, in the plot,
    a different color will be used for each sublist.
    """
    return [
        [r"$\sigma_z^%d$" % n for n in range(test_processor.num_qubits)],
        [r"$\sigma_x^%d$" % n for n in range(test_processor.num_qubits)],
        [r"$g_01$", r"$g_12$"],
    ]


test_processor.model.get_control_latex = get_control_latex
```

```python
test_processor.dims = [2, 2, 2]
```

```python
test_processor.load_circuit([toffoli()], num_tslots=6,
                            evo_time=1, verbose=True)

test_processor.plot_pulses(title="Contorl pulse for toffoli gate");
```

## Merging a quantum circuit
If there are multiple gates in the circuit, we can choose if we want to first merge them and then find the pulse for the merged unitary.

```python
qc = QubitCircuit(3)
qc.add_gate("CNOT", controls=0, targets=2)
qc.add_gate("RX", targets=2, arg_value=pi / 4)
qc.add_gate("RY", targets=1, arg_value=pi / 8)
```

```python
setting_args = {
    "CNOT": {"num_tslots": 20, "evo_time": 3},
    "RX": {"num_tslots": 2, "evo_time": 1},
    "RY": {"num_tslots": 2, "evo_time": 1},
}

test_processor.load_circuit(
    qc, merge_gates=False, setting_args=setting_args, verbose=True
)
fig, axes = test_processor.plot_pulses(
    title="Control pulse for a each gate in the circuit", show_axis=True
)
axes[-1].set_xlabel("time");
```

In the above figure, the pulses from $t=0$ to $t=3$ are for the CNOT gate while the rest for are the two single qubits gates. The difference in the frequency of change is merely a result of our choice of `evo_time`. Here we can see that the three gates are carried out in sequence.

```python
qc = QubitCircuit(3)
qc.add_gate("CNOT", controls=0, targets=2)
qc.add_gate("RX", targets=2, arg_value=pi / 4)
qc.add_gate("RY", targets=1, arg_value=pi / 8)
test_processor.load_circuit(
    qc, merge_gates=True, verbose=True, num_tslots=20, evo_time=5
)
test_processor.plot_pulses(title="Control pulse for a \
                           merged unitary evolution");
```

In this figure there are no different stages, the three gates are first merged and then the algorithm finds the optimal pulse for the resulting unitary evolution.

```python
print("qutip-qip version:", qutip_qip.version.version)
version_table()
```