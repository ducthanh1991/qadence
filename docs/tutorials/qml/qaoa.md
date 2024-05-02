This tutorial shows how to solve the maximum cut (MaxCut) combinatorial
optimization problem on a graph using the Quantum Approximate Optimization
Algorithm (QAOA), first introduced by Farhi *et al.* in 2014 [^1].

Given an arbitrary graph, the MaxCut problem consists in finding a
graph [*cut*](https://en.wikipedia.org/wiki/Cut_(graph_theory)) which partitions
the nodes into two disjoint sets, such that the number of edges in the
cut is maximized. This is a very common combinatorial optimization problem known to be computationally hard (NP-hard).

The graph used for this tutorial is an unweighted graph randomly generated using the `networkx` library with
a certain probability $p$ of having an edge between two arbitrary nodes (known as [Erdős–Rényi](https://en.wikipedia.org/wiki/Erd%C5%91s%E2%80%93R%C3%A9nyi_model) graph).

```python exec="on" source="material-block" html="1" session="qaoa"
import numpy as np
import networkx as nx
import matplotlib.pyplot as plt
import random

# ensure reproducibility
seed = 42
np.random.seed(seed)
random.seed(seed)

# Create random graph
n_nodes = 4
edge_prob = 0.8
graph = nx.gnp_random_graph(n_nodes, edge_prob)

plt.clf() # markdown-exec: hide
nx.draw(graph)
from docs import docsutils # markdown-exec: hide
print(docsutils.fig_to_html(plt.gcf())) # markdown-exec: hide
```

The goal of the MaxCut algorithm is to maximize the following cost function:

$$\mathcal{C}(p) = \sum_{\alpha}^m \mathcal{C}_{\alpha}(p)$$

where $p$ is a given cut of the graph, $\alpha$ is an index over the edges and $\mathcal{C}_{\alpha}(p)$ is written
such that if the nodes connected by the $\alpha$ edge are in the same set, it returns $0$, otherwise it returns $1$.
We will represent a cut $p$ as a bitstring of length $N$, where $N$ is the number of nodes, and where the bit in position $i$ shows to which partition node $i$ belongs. We assign value 0 to one of the partitions defined by the cut and 1 to the other. Since this choice is arbitrary, every cut is represented by two bitstrings, e.g. "0011" and "1100" are equivalent.

Since in this tutorial we are only dealing with small graphs, we can find the maximum cut by brute force to make sure QAOA works as intended.
```python exec="on" source="material-block" result="json" session="qaoa"
# Function to calculate the cost associated with a cut
def calculate_cost(cut: str, graph: nx.graph) -> float:
    """Returns the cost of a given cut (represented by a bitstring)"""
    cost = 0
    for edge in graph.edges():
        (i, j) = edge
        if cut[i] != cut[j]:
            cost += 1
    return cost


# Function to get a binary representation of an int
get_binary = lambda x, n: format(x, "b").zfill(n)

# List of all possible cuts
all_possible_cuts = [bin(k)[2:].rjust(n_nodes, "0") for k in range(2**n_nodes)]

# List with the costs associated to each cut
all_costs = [calculate_cost(cut, graph) for cut in all_possible_cuts]

# Get the maximum cost
maxcost = max(all_costs)

# Get all cuts that correspond to the maximum cost
maxcuts = [get_binary(i, n_nodes) for i, j in enumerate(all_costs) if j == maxcost]
print(f"The maximum cut is represented by the bitstrings {maxcuts}, with a cost of {maxcost}")
```

## The QAOA quantum circuit

The Max-Cut problem can be solved by using the QAOA algorithm. QAOA belongs to the class of Variational Quantum Algorithms (VQAs), which means that its quantum circuit contains a certain number of parametrized quantum gates that need to be optimized with a classical optimizer.
The QAOA circuit is composed of two operators:

* The **cost operator $U_c$**: a circuit generated by the cost Hamiltonian which
encodes the cost function described above into a quantum circuit. The solution to the optimization problem is encoded in the ground state of the cost Hamiltonian $H_c$.
The cost operator  is simply the evolution of the cost Hamiltonian parametrized by a variational parameter $\gamma$ so that $U_c = e^{i\gamma H_c}.$
* The **mixing operator $U_b$**: a simple set of single-qubit rotations with adjustable
  angles which are tuned during the classical optimization loop to minimize the cost

The cost Hamiltonian of the MaxCut problem can be written as:

$$H_c = \frac12 \sum_{\langle i,j\rangle} (\mathbb{1} - Z_iZ_j)$$

where $\langle i,j\rangle$ represents the edge between nodes $i$ and $j$. The solution of the MaxCut problem is encoded in the ground state of the above Hamiltonian.

The QAOA quantum circuit consists of a number of layers, each layer containing a cost and a mixing operator.
Below, the QAOA quantum circuit is defined using
`qadence` operations.
First, a layer of Hadamard gates is applied to all qubits to prepare the initial state $|+\rangle ^{\otimes n}$.
The cost operator of each layer can be built "manually", implementing the $e^{iZZ\gamma}$ terms with CNOTs and a $\rm{RZ}(2\gamma)$ rotation, or it can also be automatically decomposed
into digital single and two-qubits operations via the `.digital_decomposition()` method.
The decomposition is exact since the Hamiltonian generator is diagonal.

```python exec="on" source="material-block" html="1" session="qaoa"
from qadence import tag, kron, chain, RX, RZ, Z, H, CNOT, I, add
from qadence import HamEvo, QuantumCircuit, Parameter

n_qubits = graph.number_of_nodes()
n_edges = graph.number_of_edges()
n_layers = 6

# Generate the cost Hamiltonian
zz_ops = add(Z(edge[0]) @ Z(edge[1]) for edge in graph.edges)
cost_ham = 0.5 * (n_edges * kron(I(i) for i in range(n_qubits)) - zz_ops)


# QAOA circuit
def build_qaoa_circuit(n_qubits, n_layers, graph):
    layers = []
    # Layer of Hadamards
    initial_layer = kron(H(i) for i in range(n_qubits))
    layers.append(initial_layer)
    for layer in range(n_layers):

        # cost layer with digital decomposition
        # cost_layer = HamEvo(cost_ham, f"g{layer}").digital_decomposition(approximation="basic")
        cost_layer = []
        for edge in graph.edges():
            (q0, q1) = edge
            zz_term = chain(
                CNOT(q0, q1),
                RZ(q1, Parameter(f"g{layer}")),
                CNOT(q0, q1),
            )
            cost_layer.append(zz_term)
        cost_layer = chain(*cost_layer)
        cost_layer = tag(cost_layer, "cost")

        # mixing layer with single qubit rotations
        mixing_layer = kron(RX(i, f"b{layer}") for i in range(n_qubits))
        mixing_layer = tag(mixing_layer, "mixing")

        # putting all together in a single ChainBlock
        layers.append(chain(cost_layer, mixing_layer))

    final_b = chain(*layers)
    return QuantumCircuit(n_qubits, final_b)


circuit = build_qaoa_circuit(n_qubits, n_layers, graph)

from qadence.draw import html_string # markdown-exec: hide
# Print a single layer of the circuit
print(html_string(build_qaoa_circuit(4,1,graph))) # markdown-exec: hide
```

## Train the QAOA circuit to solve MaxCut

Given the QAOA circuit above, one can construct the associated Qadence `QuantumModel`
and train it using standard gradient based optimization.

The loss function to be minimized reads:

$$\mathcal{L} =-\langle \psi | H_c| \psi \rangle= -\frac12 \sum_{\langle i,j\rangle}  \left(1 - \langle \psi | Z_i Z_j | \psi \rangle \right)$$

where $|\psi\rangle(\beta, \gamma)$ is the wavefunction obtained by running the QAQA
quantum circuit and the sum runs over the edges of the graph $\langle i,j\rangle$.

```python exec="on" source="material-block" result="json" session="qaoa"
import torch
from qadence import QuantumModel

torch.manual_seed(seed)


def loss_function(model: QuantumModel):
    # The loss corresponds to the expectation
    # value of the cost Hamiltonian
    return -1.0 * model.expectation().squeeze()


# initialize the parameters to random values
model = QuantumModel(circuit, observable=cost_ham)
model.reset_vparams(torch.rand(model.num_vparams))
initial_loss = loss_function(model)
print(f"Initial loss: {initial_loss}")

# train the model
n_epochs = 100
lr = 0.1

optimizer = torch.optim.Adam(model.parameters(), lr=lr)

for i in range(n_epochs):
    optimizer.zero_grad()
    loss = loss_function(model)
    loss.backward()
    optimizer.step()
    if (i + 1) % (n_epochs // 10) == 0:
        print(f"MaxCut cost at iteration {i+1}: {-loss.item()}")
```

Qadence offers some convenience functions to implement this training loop with advanced
logging and metrics track features. You can refer to [this tutorial](ml_tools.md) for more details.


## Results

Given the trained quantum model, one needs to sample the resulting quantum state to
recover the bitstring with the highest probability which corresponds to the maximum
cut of the graph.

```python exec="on" source="material-block" html="1" session="qaoa"
samples = model.sample(n_shots=100)[0]
most_frequent = max(samples, key=samples.get)

print(f"Most frequently sampled bitstring corresponding to the maximum cut: {most_frequent}")

# let's now draw the cut obtained with the QAOA procedure
colors = []
labels = {}
for node, b in zip(graph.nodes(), most_frequent):
    colors.append("green") if int(b) == 0 else colors.append("red")
    labels[node] = "A" if int(b) == 0 else "B"

plt.clf() # markdown-exec: hide
nx.draw_networkx(graph, node_color=colors, with_labels=True, labels=labels)
from docs import docsutils # markdown-exec: hide
print(docsutils.fig_to_html(plt.gcf())) # markdown-exec: hide
```

## References

[^1]: [Farhi et al.](https://arxiv.org/abs/1411.4028) - A Quantum Approximate Optimization Algorithm