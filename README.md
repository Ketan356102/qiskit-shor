ull implementation of Shor factoring algorithm with Qiskit SDK.

Packages versions are specified in requirements.txt.

Installation:

Open a terminal window
Clone the repository on your local machine
Navigate to the cloned directory
[Optional] Create and activate a virtual environment
Run pip install -r requirements.txt
A pedagogical walkthrough on the implementation of the algortihm is given this blog post.

[Update: 2026-06] Refactor AdderCircuit class and implement Ripple-Carry quantum adder.

Disclaimer
The work presented here is a complete implementation of Shor's algorithm, which, in theory, can factorize large integers and prove the quantum advantage of Shor's algorithm. This would be the case if the quantum hardware was available, namely if one had access to a quantum processor with enough qubits and low enough quantum noise. As of 2025, the IBM quantum platform provides access to quantum processors, which allows to test the present code on a real quantum computer. However, the number of qubits available is limited, with up to a few hundreds of qubits, and the degree of quantum noise is still too high for applications such as Shor's factorization, even for small integers. Therefore, the code presented here is more of an illustration of how Shor's algorithm works and how to implement it in Qiskit. It can be tested on noise-free simulators and it could be used in the future to test Shor's algorithm, as the hardware improves.

Overview of Shor's algorithm
Shor's algorithm is a quantum algorithm which allows to factorize a composite integer 
N
, in polynomial time complexity in 
n
=
log
⁡
N
, with high probability. The algorithm steps are:

If 
N
 is even, 2 is a factor, or, if 
N
 is a power of a prime 
N
=
p
k
, with k >1, then 
p
 is a factor and we are done.
Choose a random integer 
1
<
A
<
N
. If 
g
c
d
(
A
,
N
)
>
1
, then 
g
c
d
(
A
,
N
)
 is a non-trivial factor of 
N
 (lucky case) and we are done. Otherwise 
A
 and 
N
 are coprime (typical case).
Find the order of A in 
Z
N
, i.e. the smallest integer 
1
<
r
<
N
 such that 
A
r
=
1
mod
N
, using the phase estimation quantum algorithm.
If 
r
 is odd, go to step 1 and choose another 
A
. Otherwise, compute 
g
c
d
(
A
r
/
2
−
1
,
N
)
. If it is larger than 1, then it is a non-trivial factor of 
N
. If not, start again in step 1.
This algorithm finds a factor of 
N
 in polynomial time, with high probability.

The order finding step 3 is a quantum search algorithm. It is probabilistic in nature, so one needs to repeat it until it succeeds. Its success rate depends on the 'precision' required, which gets reflected into the size of the quantum circuit.

Introductions to Shor's algorithm are easy to find on the web (see e.g. on Wikipedia).

Original paper: Shor, P. W. (1999). Polynomial-time algorithms for prime factorization and discrete logarithms on a quantum computer. SIAM review, 41(2), 303-332.

Qubit conventions
The implementation uses Qiskit conventions:

Classical integers are represented by capital letters, e.g. 
X
, 
Y
, 
A
.
Quantum integers are represented by small letters, e.g. 
x
, 
y
. A quantum integer 
x
 corresponds to a quantum state 
|
x
⟩
:=
|
x
0
⟩
|
x
1
⟩
.
.
.
|
x
m
⟩
, with 
x
k
∈
0
,
1
, where 
x
=
∑
k
=
0
m
−
1
x
k
2
k
. In binary string notation, it is the integer 
x
m
.
.
.
x
1
x
0
, so 
x
0
 is the least significant bit (LSB), and it is ordered in the qubit register as [qubit_0, qubit_1, ..., qubit_m], where qubit_k is in the state 
|
x
k
⟩
. Often the quantum register in the state representing 
x
 is called "x_reg".
Modular arithmetics
The modular arithmetic operations are implemented in adder.py, via the AdderCircuit class methods. For the elementary quantum additions we provide two independent implementations, in qft_adder.py and in rc_adder.py, following the two prominent approaches in the literature.

QFT approach
The default implementation (in qft_adder.py) performs additions in Fourier space, leveraging the rotation gates, which are part of the basic set of gates in Qiskit, and the "QFT gate" to convert between computational and Fourier space arithmetic representations. This QFT approach and the implementation of modular operations are based on the work of Beauregard: Circuit for Shor's algorithm using 2n+ 3 qubits. arXiv preprint quant-ph/0205095.

To create a circuit, use the QFTAdderCircuit class, which is a subclass of QuantumCircuit.

from qiskit.circuit import ClassicalRegister, QuantumRegister
from qiskit_shor.qft_adder import QFTAdderCircuit

# Create a circuit with 5 qubits
qc1 = QFTAdderCircuit(5)

# Create a circuit with 3 qubits and 3 classical bits 
q_reg = QuantumRegister(5)
c_reg = ClassicalRegister(2)
qc2 = QFTAdderCircuit(q_reg, c_reg)
To perform an operation, use the methods of the QFTAdderCircuit class. Since we are dealing with finite size quantum registers, all operations, even the non-modular ones, are enforced modulo 
2
k
, where 
k
 is the size of the target quantum register, conventionally named 
y
 in this package. One typically ensures that the size of the target register is large enough to contain the result of the operation.

from qiskit.circuit import QuantumRegister
from qiskit_shor.qft_adder import QFTAdderCircuit

x_reg = QuantumRegister(4)
y_reg = QuantumRegister(4)
ancilla_reg = QuantumRegister(1)
overflow_reg = QuantumRegister(1)

qc = QFTAdderCircuit(x_reg, y_reg, ancilla_reg, overflow_reg)
# x -> x + 3
qc.add_classical(3, x_reg)
# y -> y + 6
qc.add_classical(6, y_reg)
# y -> y + x
qc.add_quantum(x_reg, y_reg)
# y -> y + 10*x
qc.add_quantum(x_reg, y_reg, A=10)
# x -> (x + 7) mod 9
qc.add_classical_modulo(7, x_reg, ancilla_reg[0], overflow_reg[0], N=9)
# y -> (y + 4*x) mod 9
qc.add_quantum_modulo(x_reg, y_reg, ancilla_reg[0], overflow_reg[0], N=9, A=4)

z_reg = QuantumRegister(6)
qc = QFTAdderCircuit(x_reg, y_reg, z_reg)
# x -> 4*x mod 9
qc.multiply_modulo(
    A=4,
    x_reg=x_reg,
    y_reg=z_reg[:4],
    overflow_bit=z_reg[4],
    ancilla_bit=z_reg[5],
    N=9,
)
# (x, y) -> (x, y*(4^x) mod 9)
qc.exponentiate_modulo(A=4, x_reg=x_reg, y_reg=y_reg, ancilla_reg=z_reg, N=9)
Controlled operations are also supported.

control_reg = QuantumRegister(1)
x_reg = QuantumRegister(4)
y_reg = QuantumRegister(4)
ancilla_reg = QuantumRegister(1)
overflow_reg = QuantumRegister(1)

qc = QFTAdderCircuit(control_reg, x_reg, y_reg, ancilla_reg, overflow_reg)
# Flip control bit
qc.x(control_reg[0])
# Controlled operation x -> x + 3
qc.c_add_classical(control_reg, 3, x_reg)
# Controlled operation x -> (x + 6) mod 9
qc.c_add_classical_modulo(control_reg, 6, x_reg, ancilla_reg[0], overflow_reg[0], N=9)
# Controlled operation y -> y + x
qc.c_add_quantum(control_reg, x_reg, y_reg)
# Controlled operation y -> (y + 10*x) mod 9
qc.c_add_quantum_modulo(control_reg, x_reg, y_reg, ancilla_reg[0], overflow_reg[0], N=9, A=10)

z_reg = QuantumRegister(6)
qc = QFTAdderCircuit(control_reg, x_reg, z_reg)
# Controlled operation x -> 4*x mod 9
qc.c_multiply_modulo(control_reg, 4, x_reg, z_reg[:4], z_reg[4], z_reg[5], N=9)
The control register input can be a register of several qubits, to implement a multi-qubit controlled operation. It can also be a single Qubit.

The available operations, their input qubits requirements and input state assumptions are described in adder.py and qft_adder.py (see method descriptions).

An option to use approximate QFT gates is available, dropping phase gates with angle smaller than 
π
/
2
d
, with 
d
=
⌈
log
2
⁡
(
n
)
⌉
+
2
, in all addition operations, where 
n
 is the number of qubits in the target register.

# Use approximate QFT gates
q_reg = QuantumRegister(6)
qc = QFTAdderCircuit(q_reg, approx_QFT=True)
Ripple-Carry approach
The second implementation of quantum additions (in rc_adder.py) uses the Ripple-Carry algorithm of Cucarro et al: A new quantum ripple-carry addition circuit. arXiv preprint quant-ph/0410184. It uses fewer gates (asymptotically) than the QFT approach, but it requires extra ancilla qubits (see section Algorithm complexity). Importantly, all operations use only X, CX and CCX gates (i.e. NOT, CNOT and Toffoli gates), which makes this approach more suitable for quantum error correction attempts. The precise operations and the qubit requirements are given in the method descriptions in rc_adder.py.

Creating quantum adder circuits and performing operations is similar to the QFT approach, except that we need the additional ancilla qubits, conventionally given in a QuantumRegister a_reg in the code. While adding two quantum numbers requires only one extra ancilla qubit, adding a classical integer to a quantum integer requires n+1 ancilla qubits, where n qubits are used to encode the classical integer into a quantum integer. The operations return the ancilla qubits in their initial |0> state.

from qiskit.circuit import QuantumRegister
from qiskit_shor.rc_adder import RCAdderCircuit

x_reg = QuantumRegister(4)
y_reg = QuantumRegister(4)
a_reg = QuantumRegister(6)
ancilla_reg = QuantumRegister(1)
overflow_reg = QuantumRegister(1)

qc = RCAdderCircuit(x_reg, y_reg, a_reg, ancilla_reg, overflow_reg)
# x -> x + 3
qc.add_classical(3, x_reg, a_reg)
# y -> y + 6
qc.add_classical(6, y_reg, a_reg)
# y -> y + x
qc.add_quantum(x_reg, y_reg, a_reg[0])
# x -> (x + 7) mod 9
qc.add_classical_modulo(7, x_reg, ancilla_reg[0], overflow_reg[0], N=9, a_reg=a_reg)
# y -> (y + 4*x) mod 9
qc.add_quantum_modulo(x_reg, y_reg, ancilla_reg[0], overflow_reg[0], N=9, A=4, a_reg=a_reg)
As for the QFT approach, controlled operations are also supported.

Shor factorization
The order finding circuit and Shor factorization algorithm are implemented in shor.py, leveraging the modular arithmetic operations discussed above, using the QFT approach by default.

The main API functions are find_order and find_factor, which build the order finding circuit and run it on the provided quantum backend or simulator.

from qiskit_shor.shor import find_order, find_factor

# Define your sampler and pass_manager
sampler = ...
pass_manager = ...

N = 15
A = 7
# Compute the order of A in Z_N, running the circuit 100 times. Return the order and the distribution of measurement outcomes.
order, distribution = find_order(
    A, N, sampler, pass_manager, num_shots=100, adder="qft_adder", one_control_circuit=True,
)
# Compute a factor of N using Shor algortihm, trying 3 random values for A and running the circuit 100 times for each try.
factor = find_factor(
    N, sampler, pass_manager, num_tries=3, num_shots_per_trial=100, adder="qft_adder", one_control_circuit=True,
)
The order finding circuit is implemented in two variants: the basic circuit using 
4
n
+
2
 qubits with measurements at the end of the circuit, and the "one-control" circuit using 
2
n
+
3
 qubits and control flow operations on one qubit. These two variants are described in Beauregard's paper. They are toggled using the argument one_control_circuit.

The Ripple-Carry approach to modular operations is also supported, via the argument adder="rc_adder". E.g.

order, distribution = find_order(
    A, N, sampler, pass_manager, num_shots=100, adder="rc_adder", one_control_circuit=True,
)
Some examples of the code usage on simulators and real devices can be found in the example notebook.

Algorithm complexity
The implementation chosen in this repository is not meant to include state-of-the-art optimizations and one may find more efficient implementations in terms of number of qubits required or number of gates, in the literature. But it is arguably the simplest and easiest to understand implementation of the modular operations needed to create the order-finding quantum circuit, in the Fourier Transform paradigm or Ripple-Carry paradigm.

Fourier Transform approach:

With 
n
:=
⌈
log
2
⁡
N
⌉
, the basic order finding circuit requires 
4
n
+
2
 qubits, while the circuit using a single control qubit requires 
2
n
+
3
 qubits in total. The number of gates is 
O
(
n
4
)
 (or 
O
(
n
3
log
⁡
n
)
 with approximate QFT) and the depth is 
O
(
n
3
)
 (or 
O
(
n
2
log
⁡
n
)
 with approximate QFT).

Depth-optimized controlled quantum additions: A method c_add_quantum_optmized_depth is available to use controlled quantum addition operations with reduced circuit depth (
O
(
n
)
 instead of $O(n^2)$), following Pavlidis and Gizopoulos, but the it does not carry to modular controlled additions, so it is not used in the order finding circuit.

Pavlidis and Gizopoulos: Fast Quantum Modular Exponentiation Architecture for Shor's Factorization Algorithm. arXiv preprint arXiv:1207.0511.

Ripple-Carry approach:

The Ripple-Carry approach requires 
5
n
+
4
 qubits, or 
3
n
+
5
 qubits with the one-control circuit, the number of gates is 
O
(
n
3
)
 and the depth is 
O
(
n
3
)
.

Note: This kind of gate and depth counting is somewhat ambiguous (which is why we don't give precise numbers). For gates we count single-qubit and two-qubit gates of the Qiskit API and for the depth we consider the depth of the non-transpiled circuit. This would be practically relevant if all physical qubits were pairwise-connected in the underlying QPU (i.e. physical two-qubit gates exist for all pairs of qubits). In pratice physical qubits are only sparsely connected and chains of two-qubit gates are required to implement two-qubit gates between distant qubits. The actual number of gates and depth of the circuits depend on the underlying QPU.

Testing
Unit tests can be run with pytest.

python -m pytest <TEST_FILE>.py
Resources
IBM Quantum Learning, Shor factorization course.

IBM Quantum Platform
