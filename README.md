# Patent Citation Analysis

Explore patent citation lineages and compare graph search algorithms on the SNAP US Patent Citation Network with NBER patent metadata.

The analysis is implemented in [`patent_citation_analysis.ipynb`](patent_citation_analysis.ipynb). It combines a metadata-backed citation visualization with a benchmark of breadth-first search (BFS), iterative deepening search (IDS), and two A* heuristic variants.

> **Scope:** This README describes the notebook's code and saved outputs. The full analysis was not rerun while preparing this documentation; the source datasets and metadata cache were not supplied for independent validation. Saved timings and counts are observations from the embedded execution, not guaranteed results.

## What the notebook does

1. Loads a directed edge list into a Python adjacency list.
2. Reads selected NBER metadata in chunks and caches it locally.
3. Finds a five-hop citation chain whose nodes all have metadata records, adds any citation edges between those nodes, and plots the subgraph by grant year.
4. Samples reachable source–target pairs at shortest-path distances from 2 through 10 hops.
5. Runs four search configurations on the same pairs and records search effort, a frontier/depth proxy, and elapsed time.
6. Exports aggregate metrics to CSV and produces four benchmark figures.

There is no model training, prediction pipeline, web application, or automatic dataset downloader in the notebook.

## Data and expected inputs

Obtain the citation edge list from the [SNAP patent citation network page](https://snap.stanford.edu/data/cit-Patents.html) and extract `cit-Patents.txt.gz`. Obtain the SAS transport patent metadata from the [NBER US patents data page](https://www.nber.org/research/data/us-patents-1975-1999), using the patent-data SAS `.tpt` archive rather than the ASCII CSV archive. Extract it as `pat63_99.tpt`.

Place these files in the notebook's working directory:

```text
patent-citation-analysis/
├── README.md
├── patent_citation_analysis.ipynb
├── cit-Patents.txt
└── pat63_99.tpt
```

| Input | Expected content |
| --- | --- |
| `cit-Patents.txt` | Two whitespace-separated integer patent IDs per nonempty, noncomment line. Lines starting with `#` are skipped. The code treats an edge `u v` as patent `u` citing patent `v`. |
| `pat63_99.tpt` | SAS XPORT data containing `patent`, `gdate`, `gyear`, `cat`, and `subcat`; column names are normalized to lowercase. |
| `meta_cache_v2.pkl` | Optional cache created by the notebook. If present, it is loaded instead of parsing the `.tpt` file. |

Metadata is stored as `patent_id -> (gdate, gyear, cat, subcat)`. Missing dates, years, and classification values become zero. Rows without a patent ID are dropped; rows that fail integer conversion with `ValueError` or `OverflowError` are skipped. Metadata is read in chunks of 250,000 rows.

A valid cache permits execution without the `.tpt` file. Use only a cache you trust: Python pickle loading can execute code. Remove or rename a stale cache to rebuild it after changing the metadata source; the notebook does not validate cache provenance or freshness.

## Installation and execution

Use a Python 3 environment with pandas, Matplotlib, NetworkX, and a Jupyter frontend. The notebook does not pin package versions or declare a minimum supported Python version; compatibility must be checked in your environment.

From the directory containing the notebook and data:

```bash
python -m venv .venv
```

Activate the environment:

```powershell
# Windows PowerShell
.\.venv\Scripts\Activate.ps1
```

```bash
# macOS / Linux
source .venv/bin/activate
```

Install the required packages and launch Jupyter:

```bash
python -m pip install pandas matplotlib networkx jupyterlab
python -m jupyterlab patent_citation_analysis.ipynb
```

Select the environment's Python kernel and run all cells from top to bottom. The benchmark section depends on imports and helper definitions in the first section, even though it can reload `graph` and `meta` if those variables are absent.

The graph and metadata dictionaries remain in memory. Plan for substantial RAM use with the full dataset; the notebook does not establish a minimum memory requirement. Repeated pair sampling also builds and shuffles a list of metadata-backed source nodes.

### Dependencies

| Package | Purpose |
| --- | --- |
| `pandas` | Read SAS XPORT metadata, assemble the benchmark table, and export CSV. |
| `matplotlib` | Display and save figures. |
| `networkx` | Build and draw the small lineage subgraph; full-network searches use the adjacency list directly. |
| `jupyterlab` | Suggested interactive runner; another compatible Jupyter frontend can also be used. |

Other imports (`os`, `sys`, `pickle`, `time`, `collections`, `heapq`, `random`, and `math`) belong to Python's standard library. NumPy is not directly imported by the notebook.

## Implemented methods

### Citation lineage

`find_verified_lineage_subgraph` iterates over adjacency-list source nodes in input order and uses BFS through nodes with metadata. It returns the first path at `target_hops=5`, together with all original edges among the selected path nodes, including cross/skip edges if any exist.

Here, **verified** means that edges exist in the loaded adjacency list and each selected node has a metadata record. It does not establish chronological consistency, correct edge orientation, or independent patent-record validation. The function does not rank start nodes by degree, despite a code comment referring to high connectivity.

### Search configurations

All searches traverse outgoing directed edges with unit hop cost.

| Configuration | Implementation |
| --- | --- |
| BFS | FIFO queue, parent dictionary, and path reconstruction; returns when the target is discovered as a neighbor. |
| IDS | Repeated recursive depth-limited searches, with a per-iteration dictionary recording the shallowest visited depth. Benchmark calls set `max_depth=depth + 4`; the function default is 15. |
| A* (Coarse) | Heap priority `g + h1`, with `h1 = (node_year - target_year) / 25`. Negative differences return infinity. |
| A* (Fine) | Heap priority `g + h2`, with `h2 = (node_gdate - target_gdate) / 9131`, plus classification penalties. Negative date differences return infinity. |

The fine heuristic adds `0.005` if subcategories differ or the node subcategory is zero, and `0.004` if categories differ or the node category is zero. These are additive heuristic terms, not a separate heap tie-breaking key. Both heuristics return zero if either metadata record is absent. Stored zero fields are otherwise used directly.

A* skips enqueueing neighbors with infinite heuristic values, closes nodes when popped, and does not reopen closed nodes. The notebook does not prove admissibility or consistency of either heuristic, or enforce the assumed maximum citation lag. Consequently, A* shortest-path optimality and completeness on the original graph are not established, especially when temporal pruning excludes valid routes.

### Pair sampling and aggregation

`find_k_multi_hop_pairs` shuffles source nodes that have metadata, then runs BFS from each candidate until it finds a metadata-backed target at the requested shortest-path distance. Intermediate nodes in this sampler need not have metadata. It selects the first eligible target encountered per source, so the sample is not uniform over all reachable patent pairs.

Defaults:

| Setting | Value |
| --- | --- |
| Lineage length | 5 hops |
| Evaluation depths | 2–10 inclusive |
| Detailed comparison | 1 pair per depth, seed `100 + depth` |
| Aggregate comparison | Up to 10 pairs per depth (`K_SAMPLES = 10`), seed `2000 + depth` |
| Algorithms | BFS, IDS, A* (Coarse), A* (Fine) |

With all requested pairs found, the aggregate experiment uses 90 pairs and 360 search calls. Each algorithm is timed once per pair using `time.time()`. Loading, sampling, and plotting are outside search timing. Means and population standard deviations (divisor `N`) are computed across pairs; they are not confidence intervals or repeated-run timing estimates.

Seeds improve repeatability for the same inputs and ordering. Dataset ordering, cache content, software versions, and hardware can affect selected paths or measured timings.

## Outputs

Files are written to the current working directory and overwritten on subsequent runs.

| File | Content |
| --- | --- |
| `meta_cache_v2.pkl` | Parsed metadata dictionary, created when no cache exists. |
| `real_citation_lineage.png` | Grant-year layout of the selected lineage; produced only if a chain is found. |
| `search_benchmark_results.csv` | Aggregate metrics by depth and algorithm. |
| `fig1_nodes_expanded.png` | Mean expansion counts on a logarithmic axis with standard-deviation shading. |
| `fig2_runtime_latency.png` | Mean search time in milliseconds. |
| `fig3_heuristic_dominance.png` | Coarse A* mean expansions minus fine A* mean expansions; an empirical difference, not a proof of heuristic dominance. |
| `fig4_throughput_efficiency.png` | Mean expansions divided by mean time in milliseconds. |

Figures are also displayed inline. The lineage figure and benchmark PNGs are saved at 300 DPI. The single-pair comparison is printed in notebook output rather than exported as a separate table.

The CSV columns are:

```text
Depth,Algorithm,Mean_Expanded,Std_Expanded,Mean_Peak_Frontier,Std_Peak_Frontier,Mean_Time_ms,Std_Time_ms
```

The default complete aggregate table contains 36 rows: 9 depths × 4 algorithms. Individual pair records and paths are not exported by the aggregate benchmark.

## Reading the metrics correctly

- **Expansion counts use different stopping rules.** BFS counts dequeued nodes and returns on target discovery, without dequeuing the target. A* counts newly closed nodes, including the target. IDS counts recursive calls across all depth limits, including cutoff and target calls.
- **The memory column is a proxy.** BFS reports peak queue length; A* reports peak heap length, which can include stale entries. IDS reports maximum recursion depth in edges. The IDS value excludes its visited-depth dictionary and other state. These values are not directly comparable total-memory measurements or bytes of RAM.
- **Aggregate rows do not establish success rates or optimality.** The aggregate code discards returned paths and records metrics even if a search returns `None`. The detailed single-pair table does print returned hop counts.
- **Very short timings are noisy.** No warmup, repeated timing per pair, or hardware specification is provided. Throughput is a ratio of aggregate means, not a mean of per-pair throughput.

## Saved execution: observations and caveats

The embedded output reports 16,518,948 loaded edges and 2,923,922 cached metadata records. The latter is the size of the cache used in that execution, not an independently verified count of the complete NBER source.

The saved five-hop lineage is:

```text
3866689 (1975) -> 4692121 (1987) -> 4579404 (1986)
              -> 4269466 (1981) -> 3859787 (1975) -> 3075352 (1963)
```

The 1975-to-1987 step contradicts the visualization's description of strictly backward chronological citations. The notebook does not resolve whether this reflects input orientation, metadata alignment, or another data issue. Validate these before interpreting the figure as historical prior-art lineage.

For illustration, the saved aggregate output at **depth 10** reports:

| Algorithm | Mean expansions ± population SD | Mean time ± population SD (ms) |
| --- | ---: | ---: |
| BFS | 5,254.8 ± 3,755.9 | 2.51 ± 1.92 |
| IDS | 27,798.4 ± 18,405.0 | 11.70 ± 8.68 |
| A* (Coarse) | 5,149.0 ± 3,797.2 | 9.79 ± 7.50 |
| A* (Fine) | 5,038.1 ± 3,786.3 | 9.98 ± 7.76 |

These saved measurements show why fewer expansions should not automatically be interpreted as faster execution. They support observations about this execution and sample, rather than a universal ranking of the algorithms.

## Configuration and troubleshooting

Edit `edge_file`, `tpt_file`, and `cache_file` in the data-loading cell to change input paths. Also check the fallback loading paths in Section 2.6 if running that section separately. Edit `target_hops`, `evaluation_hops`, `K_SAMPLES`, seed expressions, or the heuristic default parameters to change the experiment.

| Issue | Action |
| --- | --- |
| Missing input file | Extract the archives and check the kernel's working directory and configured paths. The loader does not read `.gz` or `.zip` archives directly. |
| Missing metadata columns | Confirm that the file is the expected patent SAS XPORT dataset containing the five required fields. |
| Unexpected metadata or chronology | Rebuild the cache from the intended `.tpt` source and inspect edge direction and patent-ID alignment. |
| `NameError` in Part 2 | Restart the kernel and run from the top so shared imports and helpers are defined. |
| High memory use or slow execution | Use a machine with more available RAM; reducing sample count or evaluation depths can reduce benchmark work but does not shrink the loaded graph. |
| No pairs at a requested depth | Inspect reachability and metadata coverage, or reduce the requested depth. The aggregate code warns about too few pairs but divides by zero if none are found. |

## Data attribution and repository use

Consult the [SNAP dataset page](https://snap.stanford.edu/data/cit-Patents.html) and [NBER source page](https://www.nber.org/research/data/us-patents-1975-1999) for dataset attribution, documentation, and applicable terms. The notebook does not declare a project license; this README does not assign one.

For a GitHub repository, upload the notebook and this README. Generated figures and the CSV may also be included after checking them. Keep large raw datasets and local pickle caches outside version control unless redistribution and repository storage requirements have been checked. The directory layout above describes required local inputs, not files supplied with this README.
