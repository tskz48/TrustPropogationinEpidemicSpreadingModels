# Trust Propagation Through the Bitcoin OTC Network

A Python project investigating trust propagation through the Bitcoin OTC network using the Susceptible–Infected–Recovered (SIR) epidemic spreading model. Users are represented as nodes and recorded trust ratings as directed connections. The model explores how a trust-related influence can spread from an initial group of users through these connections.

The Bitcoin trust network is compared with a directed random network using the same global propagation and cessation rates. This comparison examines how the arrangement of trust relationships affects the speed, reach, and duration of simulated trust propagation.

## Overview

The implementation:

- Loads a filtered Bitcoin OTC trust dataset from CSV.
- Builds a directed graph with trust ratings stored as edge weights.
- Plots the total-degree distribution and the network structure.
- Normalizes trust ratings and uses their averages to define global SIR rates.
- Simulates trust propagation on the Bitcoin network and a random graph.
- Compares users who have not adopted the modeled trust influence, users actively propagating it, and users who have stopped propagating it.

The SIR framework provides an abstraction of trust propagation. Its infection terminology describes the adoption and spread of a modeled trust influence. The simulation does not calculate an individual user’s trustworthiness or predict changes in pairwise trust ratings; its states and rate mapping are modeling assumptions.

## Dataset

The code expects `filtered_soc-sign-bitcoinotc.csv` with these columns:

| Column | Purpose |
| --- | --- |
| `SOURCE` | Node issuing a trust rating; source of a directed edge |
| `TARGET` | Node receiving the rating; target of the directed edge |
| `RATING` | Trust rating, renamed to `weight` during preprocessing |

The dataset is not included in this README. Use the same filtered dataset to reproduce the intended comparison; the filtering procedure is not specified in the supplied code.

The input path currently targets Google Colab:

```python
df = pd.read_csv('/content/drive/MyDrive/filtered_soc-sign-bitcoinotc.csv')
```

For local use, replace it with the path to your CSV:

```python
df = pd.read_csv('filtered_soc-sign-bitcoinotc.csv')
```

## Network Construction

```python
G = nx.from_pandas_edgelist(
    df,
    source='SOURCE',
    target='TARGET',
    edge_attr='weight',
    create_using=nx.DiGraph()
)
```

Nodes represent users, and directed edges represent recorded trust relationships. The model allows the trust influence to propagate from `SOURCE` to `TARGET`. A rating records that the source evaluates the target; it does not by itself establish that social influence travels in that direction. The chosen propagation direction is therefore an assumption of this investigation.

The degree histogram uses `G.degree()`, which measures total degree: incoming plus outgoing edges. A `DiGraph` stores one edge per ordered node pair, so repeated ratings for the same pair do not produce separate edges.

The comparison network is generated with:

```python
H = nx.gnm_random_graph(2187, 9903, directed=True)
```

This is a directed G(n, m) random graph with exactly 2,187 nodes and 9,903 edges. Those counts are hard-coded and should be checked against the printed counts for `G`. To match the loaded graph dynamically, use:

```python
H = nx.gnm_random_graph(
    G.number_of_nodes(), G.number_of_edges(), directed=True, seed=42
)
```

## Modeling Trust Propagation with SIR

| SIR state | Interpretation in this trust-propagation model |
| --- | --- |
| Susceptible (`S`) | A user who has not yet adopted the modeled trust influence but can receive it through an incoming connection from an active propagator |
| Infected (`I`) | A user who has adopted the trust influence and is actively propagating it to connected users |
| Recovered (`R`) | A user who has stopped propagating the trust influence and does not resume propagation during the simulation |

The transitions are `S → I → R`: a user adopts the trust influence and later stops actively spreading it. Entering `R` does not necessarily mean the user has lost trust; it indicates the end of active propagation. SIR assumes that users do not return to the susceptible or active state.

Here, “trust influence” is an abstract spreading process, rather than a claim about a particular person or transaction. The supplied code does not identify a specific object of trust or a mechanism for updating ratings.

### Research questions

- How quickly does the modeled trust influence spread through the Bitcoin network?
- How many users actively propagate it at the peak?
- How many users does it reach over the simulation?
- How do these patterns differ from a random network with comparable node and edge counts?

### Mapping trust ratings to propagation rates

Each rating is transformed using min–max normalization:

```python
transmission_rate = (weight - min_rating) / (max_rating - min_rating)
recovery_rate = 1 - transmission_rate
```

The implementation clips normalized values to `[0, 1]` and computes:

```python
average_tau = df['transmission_rate'].mean()
average_gamma = df['recovery_rate'].mean()
```

In the trust interpretation, `tau` controls how quickly active users pass the trust influence along their outgoing edges. `gamma` controls how quickly users stop actively propagating it. These retain the SIR meanings of transmission rate per edge and recovery rate per node, expressed per unit of simulation time. They are rates, rather than per-step probabilities. Higher normalized ratings contribute to a higher global propagation rate and a lower global cessation rate. Their numerical sum is 1 under this chosen mapping. This assumes that stronger average trust supports faster and more sustained propagation.

**The simulation uses global average rates.** Although ratings are stored on graph edges, the calls do not supply `transmission_weight` or `recovery_weight`. Consequently, individual edge ratings do not directly control transmission during the simulation. The computed rates are averaged over CSV rows, which can differ from averaging over unique graph edges if ratings are repeated.

Normalization requires at least two distinct rating values. If all ratings are equal, `max_rating - min_rating` is zero and the rate calculation needs an explicit fallback. Missing or nonnumeric ratings should also be handled before simulation.

### Simulation settings

Both networks use:

```python
t, S, I, R = EoN.fast_SIR(
    G,
    tau=average_tau,
    gamma=average_gamma,
    rho=10 / 2187,
    tmax=50
)
```

| Parameter | Setting | Interpretation |
| --- | --- | --- |
| `tau` | Mean normalized rating | Global trust-propagation rate per edge |
| `gamma` | Mean complementary rating | Global rate of stopping active propagation per node |
| `rho` | `10 / 2187` | Initial fraction of active trust propagators |
| `tmax` | `50` | Maximum continuous simulation time |

For a graph with 2,187 nodes, `rho` corresponds to 10 initial active trust propagators (the SIR infected nodes), selected randomly. If the loaded graph has a different size, the initial infected count can differ. The two simulations select their initial nodes independently.

`EoN.fast_SIR` is a stochastic continuous-time simulation with exponentially distributed transmission and recovery waiting times. Returned times are event times, rather than equally spaced timesteps. The simulation may end before `tmax` when no infected nodes remain.

## Installation

Install the imported third-party packages in your Python environment:

```bash
python -m pip install networkx numpy matplotlib EoN pandas
```

`random` belongs to the Python standard library and does not need installation. Package versions are not pinned in the supplied implementation.

## Running the Project

1. Obtain the filtered CSV and check that it contains the required columns.
2. Save the supplied Python code as a script, for example `epidemic_spreading.py`, or paste it into a notebook.
3. Update the CSV path. In Colab, mount Google Drive before reading from `/content/drive`:

   ```python
   from google.colab import drive
   drive.mount('/content/drive')
   ```

4. Run the notebook cells, or execute the saved script:

   ```bash
   python epidemic_spreading.py
   ```

The script filename above is a suggested name; this README does not include the source code or dataset.

## Outputs

The implementation prints:

- The first five dataset rows.
- The trust graph's node and edge counts.
- The average transmission and recovery rates.
- The initial susceptible counts for both simulations.

It displays three figures:

1. **Degree distribution:** frequency of total node degrees in the trust network.
2. **Network visualization:** a spring-layout view of the directed trust graph.
3. **Trust-propagation comparison:** user counts over time, with solid curves for the Bitcoin network and dashed curves for the random network. Green represents users who have not adopted the influence (`S`), red active trust propagators (`I`), and blue users who have stopped propagating it (`R`). The original plot legends retain the SIR state names.

Plots are displayed with `plt.show()`; the supplied code does not save figures to disk.

The curves show the dynamics of modeled trust propagation: a falling `S` curve indicates adoption, the `I` peak shows the maximum number of simultaneous active propagators, and rising `R` indicates users leaving the active propagation stage. The number reached by time `t` is `I(t) + R(t)` (including the initial propagators), or equivalently `N - S(t)`, since no users start in `R`.

Useful comparison measures include peak active propagators, time to peak, and the fraction reached. If active propagators remain at `tmax`, the final reach is not yet known. Differences between single runs can also reflect random initial users and simulation events.

## Limitations and Reproducibility

- **Single stochastic run:** initial trust propagators, random graph structure, and simulation events vary between runs. Repeat simulations and report variability before drawing conclusions.
- **Uniform simulation rates:** individual trust weights are reduced to global averages, so the comparison primarily concerns topology under shared rates.
- **Heuristic cessation mapping:** defining the rate of stopping propagation as the complement of normalized trust is a modeling choice, rather than an observed user behavior.
- **One-way adoption process:** SIR does not represent repeated trust adoption, renewed propagation, distrust spreading, or evolving trust ratings. Normalization also maps the lowest observed rating to zero; it does not model negative trust as a separate process.
- **Hard-coded graph size:** the random graph and initial infected fraction assume 2,187 nodes; verify this against the input data.
- **Limited structural matching:** matching node and edge counts does not preserve degree distribution, clustering, communities, or reciprocal edges.
- **Layout seed is not used in the drawing call:** the code computes `pos = nx.spring_layout(G, seed=42)` but calls `nx.draw` without `pos=pos`. Pass the computed positions to use the seeded layout.
- **No complete seeding:** importing `random` does not seed it. Seed graph generation and configure the simulation random generator using the API supported by the installed EoN version; record dependency versions.

## Possible Extensions

- Run multiple realizations and summarize trust-propagation metrics with uncertainty intervals.
- Compare incoming and outgoing degree distributions separately.
- Use nonnegative normalized edge attributes with EoN's `transmission_weight` option to model heterogeneous transmission. Recovery weighting, if used, requires node attributes.
- Investigate alternative rate mappings and strategies for choosing initial trust propagators.
- Compare against random networks that preserve more of the observed network structure.
- Save figures and simulation arrays for reproducible analysis.

## Documentation

- [EoN fast_SIR API](https://epidemicsonnetworks.readthedocs.io/en/latest/functions/EoN.fast_SIR.html)
- [NetworkX documentation](https://networkx.org/documentation/stable/)
- [pandas documentation](https://pandas.pydata.org/docs/)
- [Matplotlib documentation](https://matplotlib.org/stable/)
