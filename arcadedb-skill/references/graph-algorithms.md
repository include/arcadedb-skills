# ArcadeDB Graph Algorithms Reference

## Table of Contents
1. [Algorithm Index](#algorithm-index)
2. [Path Finding Algorithms](#path-finding-algorithms)
3. [Centrality Algorithms](#centrality-algorithms)
4. [Community Detection Algorithms](#community-detection-algorithms)
5. [Structural Analysis](#structural-analysis)
6. [Similarity and Link Prediction](#similarity-and-link-prediction)
7. [Network Flow](#network-flow)
8. [Node Embedding](#node-embedding)

---

## Algorithm Index

### Complete List of Available Graph Algorithms

This section provides a comprehensive index of all graph algorithms available in ArcadeDB, organized by category and use case.

#### Path Finding & Shortest Path
- **Dijkstra's Algorithm**: Single-source shortest path algorithm for weighted graphs
- **A* (A-Star)**: Informed search algorithm using heuristics for optimal pathfinding
- **Breadth-First Search (BFS)**: Unweighted shortest path and graph traversal
- **Depth-First Search (DFS)**: Graph traversal and cycle detection
- **Bellman-Ford**: Shortest path with negative edge weight support

#### Centrality & Importance Measures
- **PageRank**: Probabilistic measure of node importance in networks
- **Betweenness Centrality**: Identifies nodes acting as bridges in the network
- **Closeness Centrality**: Measures average distance from one node to all others
- **Degree Centrality**: Simple count of direct connections per node
- **Eigenvector Centrality**: Importance based on connections to important nodes
- **Harmonic Centrality**: Reciprocal-sum variant of closeness centrality

#### Community Detection
- **Louvain Algorithm**: Multi-level modularity optimization for community detection
- **Label Propagation**: Fast community detection through label spreading
- **Triangle Counting**: Identifies clusters through triangular relationships
- **Connected Components**: Partitions graph into connected subgraphs

#### Graph Structure Analysis
- **Clique Detection**: Identifies fully connected subgraphs
- **Motif Detection**: Finds recurring structural patterns
- **Transitivity**: Clustering coefficient and triangle patterns
- **Diameter**: Maximum shortest path in the graph
- **Density**: Ratio of actual to possible edges

#### Similarity & Link Prediction
- **Common Neighbors**: Nodes sharing direct connections
- **Jaccard Similarity**: Set-based similarity coefficient
- **Cosine Similarity**: Vector-based similarity metric
- **Adamic-Adar**: Weighted common neighbors based on degree
- **Preferential Attachment**: Probability-based link prediction

#### Network Flow & Connectivity
- **Maximum Flow**: Ford-Fulkerson and related algorithms
- **Minimum Cut**: Graph partitioning based on edge capacity
- **All Pairs Shortest Path**: Floyd-Warshall algorithm variants

#### Node Embeddings & Representations
- **DeepWalk**: Random walk-based node embedding
- **Node2Vec**: Biased random walk embedding method
- **Graph Neural Network (GNN) embeddings**: Neural network-based representations

---

## Path Finding Algorithms

### Overview
Path finding algorithms discover optimal or efficient routes through graphs, essential for navigation, routing, and network optimization tasks.

### Dijkstra's Algorithm
**Purpose**: Find the shortest path from a source node to all other nodes in a weighted graph with non-negative edge weights.

**Characteristics**:
- Time Complexity: O((V + E) log V) with binary heap implementation
- Space Complexity: O(V)
- Supports: Weighted graphs, non-negative weights only
- Returns: Shortest distance and path to each reachable node

**Use Cases**:
- GPS navigation and route planning
- Network routing protocols
- Social network path finding
- Game AI pathfinding

**Parameters in ArcadeDB**:
- Source vertex: Starting point for pathfinding
- Target vertex (optional): Destination for early termination
- Edge weight property: Attribute name containing edge weights
- Maximum depth/distance: Limit search scope

### A* (A-Star) Algorithm
**Purpose**: Find optimal path using heuristic guidance for faster convergence compared to Dijkstra's algorithm.

**Characteristics**:
- Time Complexity: O((V + E) log V) with good heuristics
- Space Complexity: O(V)
- Requires: Heuristic function (typically distance-based)
- Returns: Optimal path from source to target

**Use Cases**:
- Game pathfinding with obstacles
- Robot motion planning
- Map-based routing with coordinates
- AI agent navigation

**Heuristic Requirements**:
- Distance-based heuristics (Euclidean, Manhattan)
- Node coordinates or position metadata
- Admissible heuristics (never overestimate true distance)

### Breadth-First Search (BFS)
**Purpose**: Traverse or search unweighted graphs level by level, finding shortest paths in unweighted graphs.

**Characteristics**:
- Time Complexity: O(V + E)
- Space Complexity: O(V)
- Queue-based implementation
- Guarantees shortest path in unweighted graphs

**Use Cases**:
- Level-based graph traversal
- Finding shortest unweighted path
- Network reachability analysis
- Social network degree analysis

### Depth-First Search (DFS)
**Purpose**: Traverse graph using depth-first exploration, useful for cycle detection and topological sorting.

**Characteristics**:
- Time Complexity: O(V + E)
- Space Complexity: O(V) for recursion stack
- Stack-based (recursive or iterative)
- Detects cycles and back edges

**Use Cases**:
- Cycle detection in directed graphs
- Topological sorting
- Connected component identification
- Strongly connected component analysis

### Bellman-Ford Algorithm
**Purpose**: Find shortest paths from source to all vertices, supporting negative edge weights (but not negative cycles).

**Characteristics**:
- Time Complexity: O(V × E)
- Space Complexity: O(V)
- Handles negative weights
- Detects negative-weight cycles

**Use Cases**:
- Currency exchange rate arbitrage detection
- Networks with negative costs or weights
- General shortest path with constraints

---

## Centrality Algorithms

### Overview
Centrality algorithms measure the importance, influence, or prominence of nodes within a network. Different centrality measures capture different aspects of importance.

### PageRank Algorithm
**Purpose**: Measure node importance based on probabilistic random walk interpretation, originally developed by Google for web page ranking.

**Algorithm Principle**:
- Importance of a node depends on importance of nodes linking to it
- Random walk probability of reaching a node
- Iterative convergence process

**Characteristics**:
- Time Complexity: O(V + E) per iteration, typically 20-50 iterations
- Space Complexity: O(V)
- Returns: Normalized importance scores [0, 1]
- Handles: Directed graphs, dangling nodes

**Parameters**:
- Damping Factor: Usually 0.85 (probability of following links vs. random jump)
- Convergence Threshold: Iteration stop condition
- Maximum Iterations: Safety limit
- Normalization: Score normalization across nodes

**Use Cases**:
- Web page ranking and search relevance
- Influence measurement in social networks
- Recommendation systems
- Citation analysis in academic networks
- Network hub identification

**Interpretation**:
- Higher scores: More central, more "important" nodes
- Scores reflect popularity and connectivity of neighbors
- Useful for identifying key network nodes

### Betweenness Centrality
**Purpose**: Measure how often a node appears on shortest paths between other nodes, indicating bridge-like importance.

**Characteristics**:
- Time Complexity: O(V³) for exact computation
- Space Complexity: O(V²)
- High value: Node is crucial for network connectivity
- Useful for: Identifying critical network points

**Interpretation**:
- High betweenness: Node acts as bridge or connector
- Low betweenness: Node is peripheral or in a cluster
- Useful for network bottleneck identification

**Use Cases**:
- Network vulnerability assessment
- Critical infrastructure identification
- Social influence in information spread
- Organizational hierarchy analysis
- Supply chain bottleneck detection

**Variants**:
- Weighted betweenness: Considers edge weights
- Normalized betweenness: Scores [0, 1]
- Random walk betweenness: Uses random walks instead of shortest paths

### Closeness Centrality
**Purpose**: Measure how close a node is to all other nodes, indicating central position in network.

**Characteristics**:
- Formula: Inverse of sum of distances to all other nodes
- Time Complexity: O(V × (V + E))
- Space Complexity: O(V)
- High value: Node is centrally located
- Useful for: Finding central meeting points

**Interpretation**:
- High closeness: Can reach other nodes quickly
- Low closeness: Peripheral position in network
- Average distance matters

**Use Cases**:
- Identifying central location for infrastructure
- Communication efficiency analysis
- Supply chain optimization
- Epidemic spread modeling
- Network redundancy planning

**Variants**:
- Harmonic centrality: Reciprocal sum (handles disconnected graphs)
- Normalized closeness: Scores [0, 1]

### Degree Centrality
**Purpose**: Measure node importance by counting direct connections, simplest centrality measure.

**Characteristics**:
- Time Complexity: O(V + E)
- Space Complexity: O(V)
- Interpretation: Number of direct neighbors
- Two variants for directed graphs: In-degree, Out-degree

**Use Cases**:
- Quick importance assessment
- Hub identification in networks
- Network resilience analysis
- Popularity/connectivity overview

**Limitations**:
- Only considers direct connections
- Ignores network structure beyond immediate neighborhood
- May not capture actual influence

### Eigenvector Centrality
**Purpose**: Measure node importance based on connections to other important nodes, recursive importance concept.

**Characteristics**:
- Time Complexity: O(V + E) per iteration
- Iterative convergence required
- Considers neighbor importance recursively
- Useful for: Identifying influential hubs

**Use Cases**:
- Identifying influential people in social networks
- Finding important research papers (similar to PageRank)
- Network cascade/influence analysis

### Harmonic Centrality
**Purpose**: Reciprocal-sum variant of closeness, handles disconnected graphs and isolated nodes.

**Characteristics**:
- More robust than closeness centrality
- Handles disconnected components
- Assigns meaningful scores to all nodes
- Time Complexity: O(V × (V + E))

---

## Community Detection Algorithms

### Overview
Community detection algorithms identify groups of densely connected nodes that form communities, clusters, or modules within networks.

### Louvain Algorithm
**Purpose**: Multi-level modularity optimization algorithm for detecting communities in large networks.

**Algorithm Approach**:
1. **Phase 1 (Local Optimization)**: Each node evaluated for moving to neighboring communities
2. **Phase 2 (Aggregation)**: Identified communities are grouped into new super-nodes
3. **Iteration**: Phases repeat on condensed graph until modularity stabilizes

**Characteristics**:
- Time Complexity: O((V + E) log V) typically
- Space Complexity: O(V + E)
- Handles: Large networks efficiently
- Returns: Hierarchical community structure
- Greedy optimization of modularity metric

**Parameters**:
- Resolution parameter: Controls granularity of communities (0.5-2.0 typical range)
- Randomization seed: For reproducibility
- Iteration limit: Convergence safety limit

**Use Cases**:
- Social network community discovery
- Biological network module detection
- Internet topology clustering
- Recommendation system user grouping
- Organization/team structure analysis

**Output**:
- Community assignments for each node
- Hierarchical structure (dendogram)
- Modularity score indicating quality

**Interpretation**:
- Higher modularity: Better-defined communities
- Multi-level output: Hierarchical clustering from fine to coarse

### Label Propagation Algorithm
**Purpose**: Fast, simple community detection through iterative label spreading from seed nodes.

**Algorithm Approach**:
1. Initialize each node with unique label
2. Iteratively update labels based on neighborhood majority
3. Continue until convergence or fixed iterations

**Characteristics**:
- Time Complexity: O((V + E) × iterations)
- Space Complexity: O(V)
- Very fast compared to optimization-based methods
- Semi-supervised variant possible with seed labels

**Parameters**:
- Maximum iterations: Convergence limit
- Randomization: For tie-breaking in label selection
- Weighted/unweighted: Handle edge weights

**Use Cases**:
- Quick community structure discovery
- Semi-supervised clustering with partial labels
- Streaming/online graph community detection
- Large-scale network analysis

**Advantages**:
- Computational efficiency
- No parameters to tune (except iterations)
- Naturally handles overlapping assignments (probabilistic variant)

**Limitations**:
- Depends on node ordering/randomization
- May produce imbalanced community sizes
- Less optimal modularity than Louvain

### Triangle Counting
**Purpose**: Count triangles (3-cliques) in graphs, measuring local clustering and community structure.

**Characteristics**:
- Time Complexity: O(V × E) or O(E^1.5) depending on implementation
- Space Complexity: O(V + E)
- Counts closed triplets
- Useful for clustering coefficient calculation

**Metrics Derived**:
- Clustering coefficient: Ratio of triangles to possible triangles for node
- Global clustering: Network-wide clustering coefficient
- Transitivity: Overall network triangle density

**Use Cases**:
- Social network cohesion analysis
- Fraud detection (triangle patterns)
- Network robustness assessment
- Biological network structure analysis

### Connected Components
**Purpose**: Identify connected subgraphs where every node can reach every other node.

**Characteristics**:
- Time Complexity: O(V + E)
- Space Complexity: O(V)
- Directed variant: Strongly Connected Components (SCC)
- Returns: Component membership for each node

**Use Cases**:
- Network fragmentation analysis
- Communication reachability assessment
- Isolated group identification
- Network redundancy evaluation

---

## Structural Analysis

### Overview
Structural analysis algorithms examine overall network properties and topology, identifying patterns and characteristics.

### Clique Detection
**Purpose**: Identify fully connected subgraphs (cliques) where every pair of nodes is connected.

**Characteristics**:
- Time Complexity: Exponential (NP-complete problem)
- Space Complexity: O(V)
- Finds maximal cliques or k-cliques
- Useful for: Identifying tightly-knit groups

**Use Cases**:
- Close-knit group identification in social networks
- Biological complex detection in protein interaction networks
- Organizational team analysis
- Fraud ring detection

### Motif Detection
**Purpose**: Find recurring structural patterns (graphlets) that appear frequently in networks.

**Characteristics**:
- Analyzes small subgraph patterns (typically 3-5 nodes)
- Time Complexity: Depends on motif size and network size
- Compares observed vs. expected frequencies
- Indicates functional building blocks

**Biological Applications**:
- Feed-forward loops in gene regulatory networks
- Network motifs as fundamental circuits
- Predicting network function from motifs

**Use Cases**:
- Biological network function prediction
- Network fingerprinting and classification
- Design pattern identification in technological networks

### Transitivity & Clustering
**Purpose**: Measure tendency of triangles and local clustering in networks.

**Clustering Coefficient**:
- **Local**: Fraction of possible triangles around a node (0 to 1)
- **Global**: Average of local clustering coefficients
- **Transitivity**: Ratio of triangles to connected triples

**Interpretation**:
- High clustering: Nodes tend to form tight groups
- Low clustering: Network is more hierarchical/random
- Indicates small-world properties

**Use Cases**:
- Small-world network detection
- Community structure assessment
- Network robustness evaluation
- Random graph comparison

### Network Diameter & Distances
**Purpose**: Measure overall network size and path lengths.

**Metrics**:
- **Diameter**: Maximum shortest path length
- **Radius**: Minimum eccentricity
- **Average Path Length**: Mean shortest path across all pairs
- **Eccentricity**: Maximum distance from node to farthest node

**Use Cases**:
- Network scale assessment
- Efficiency evaluation
- Information diffusion time estimation
- Network design evaluation

### Network Density
**Purpose**: Measure how many edges exist relative to maximum possible edges.

**Formula**: Actual Edges / Possible Edges

**Interpretation**:
- High density: Highly connected, clique-like
- Low density: Sparse, hierarchical structure
- Range: [0, 1]

**Use Cases**:
- Network connectivity assessment
- Comparison across networks
- Efficiency vs. robustness trade-off analysis

---

## Similarity and Link Prediction

### Overview
Similarity and link prediction algorithms measure node resemblance and predict missing or future connections based on network structure.

### Common Neighbors
**Purpose**: Predict links based on shared connections; foundational similarity measure.

**Formula**: Nodes sharing direct neighbors tend to form links

**Characteristics**:
- Time Complexity: O(min(deg(u), deg(v)))
- Simple and interpretable
- Often effective despite simplicity
- Naturally weighted by number of common neighbors

**Use Cases**:
- Recommendation systems (friend suggestions)
- Collaboration prediction
- Gene interaction prediction
- Citation prediction

### Jaccard Similarity
**Purpose**: Set-based similarity coefficient measuring overlap in neighborhoods.

**Formula**: Jaccard(u,v) = |N(u) ∩ N(v)| / |N(u) ∪ N(v)|

**Characteristics**:
- Normalized to [0, 1]
- Symmetric measure
- Accounts for neighborhood size differences
- Independent of network scale

**Use Cases**:
- Link prediction in social networks
- Content-based recommendation
- Document/page similarity
- Biological sequence comparison

**Advantages**:
- Normalized scoring
- Fair comparison across different node degrees
- Geometric interpretation as set overlap

### Cosine Similarity
**Purpose**: Vector-based similarity treating node neighborhoods as vectors.

**Characteristics**:
- Based on vector angle between neighborhoods
- Range: [-1, 1], typically [0, 1] for graphs
- Considers neighborhood structure as vectors
- Computationally efficient

**Use Cases**:
- Network-based recommendation
- Content similarity in bipartite graphs
- User similarity in social networks
- Dimensionality-independent comparison

### Adamic-Adar Index
**Purpose**: Weighted common neighbors emphasizing rare shared connections.

**Formula**: Σ(1/log(deg(z))) for z in common neighbors

**Characteristics**:
- Gives more weight to connections through low-degree nodes
- Often more predictive than simple common neighbors
- Symmetric measure
- Handles degree bias

**Use Cases**:
- Improved link prediction in social networks
- Academic collaboration prediction
- Citation network prediction
- Biological interaction prediction

**Advantage**: Uncommon neighbors carry more predictive weight

### Preferential Attachment
**Purpose**: Model link formation probability based on node degree (rich-get-richer phenomenon).

**Formula**: P(link) ∝ deg(u) × deg(v)

**Characteristics**:
- Captures "rich-get-richer" dynamic
- Simple degree-based probability
- Explains power-law degree distributions
- Predictive in growing networks

**Use Cases**:
- Network evolution prediction
- Growth modeling
- Influence prediction in expanding networks
- Generative model for synthetic networks

---

## Network Flow

### Overview
Network flow algorithms optimize movement of resources through networks with capacity constraints.

### Maximum Flow (Ford-Fulkerson)
**Purpose**: Find maximum amount of flow from source to sink respecting edge capacities.

**Algorithm Variants**:
- **Ford-Fulkerson**: Basic augmenting path method
- **Edmonds-Karp**: Uses BFS for O(VE²) complexity
- **Dinic's Algorithm**: Level graph approach, O(V²E)
- **Push-Relabel**: O(V³) or O(V²√E) implementations

**Characteristics**:
- Time Complexity: Varies by implementation (O(VE²) to O(V³))
- Space Complexity: O(V + E)
- Returns: Maximum flow value and flow decomposition
- Handles: Directed graphs with capacities

**Parameters**:
- Source vertex: Flow origin
- Sink vertex: Flow destination
- Capacity attribute: Edge weight representing capacity

**Use Cases**:
- Network capacity planning
- Airline scheduling and routing
- Supply chain optimization
- Telecommunications bandwidth optimization
- Bipartite matching problems

### Minimum Cut
**Purpose**: Find minimum-weight edge set that disconnects source from sink (dual to maximum flow).

**Relationship**: By max-flow min-cut theorem, minimum cut equals maximum flow value

**Use Cases**:
- Network partitioning
- Community structure discovery
- Graph separation with minimal disruption
- Robustness assessment

### All Pairs Shortest Path (Floyd-Warshall)
**Purpose**: Compute shortest paths between all pairs of nodes.

**Characteristics**:
- Time Complexity: O(V³)
- Space Complexity: O(V²)
- Handles: Negative weights (but not negative cycles)
- Returns: Complete distance matrix

**Use Cases**:
- Network-wide distance analysis
- Routing table computation
- Network diameter calculation
- Distance-dependent algorithms

---

## Node Embedding

### Overview
Node embedding algorithms create vector representations of nodes that preserve network structure and enable machine learning on graphs.

### DeepWalk
**Purpose**: Generate node embeddings using random walk trajectories and word2vec-like approach.

**Algorithm**:
1. Generate random walks from each node
2. Treat walks as sentences
3. Apply skip-gram word embedding model
4. Learn vector representations

**Characteristics**:
- Time Complexity: O(walks × length × embedding_dim)
- Space Complexity: O(V × embedding_dim)
- Unsupervised learning approach
- Preserves local network structure

**Parameters**:
- Walk length: Steps per random walk
- Number of walks: Walks per node
- Embedding dimension: Vector size (typically 64-256)
- Window size: Skip-gram context window

**Use Cases**:
- Node classification in semi-supervised settings
- Link prediction using embedded vectors
- Visualization of network structure
- Similarity search in embedding space

**Advantages**:
- Scalable to large networks
- Captures structural equivalence
- Simple to implement and understand

### Node2Vec
**Purpose**: Biased random walk node embedding with tunable exploration-exploitation trade-off.

**Algorithm Enhancement**:
- Biased random walks using parameters p and q
- p: Return probability (tendency to return to previous node)
- q: Breadth-first vs. depth-first trade-off
- Otherwise similar to DeepWalk with skip-gram

**Characteristics**:
- More flexible than DeepWalk
- Better control over network exploration
- Captures both homophily and structural roles
- Time Complexity: O(walks × length × embedding_dim)

**Parameters**:
- p: Return parameter (0.5-2.0 typical range)
- q: In-out parameter (0.5-2.0 typical range)
- p < 1: More likely to return (DFS-like)
- p > 1: Less likely to return (BFS-like)

**Use Cases**:
- Improved link prediction
- Node classification with structural similarity
- Network visualization
- Downstream machine learning tasks

**Advantages**:
- Captures role equivalence better than DeepWalk
- Flexible exploration strategies
- Better performance on many benchmarks

### Graph Neural Network (GNN) Embeddings
**Purpose**: Learn node embeddings through neural network layers that aggregate neighborhood information.

**Architecture Types**:
- **Graph Convolutional Networks (GCN)**: Spectral convolution approach
- **GraphSAGE**: Inductive learning by sampling and aggregating
- **Graph Attention Networks (GAT)**: Attention mechanism for neighbor weighting
- **Message Passing Neural Networks**: General framework for GNN variants

**Characteristics**:
- Time Complexity: O(layers × (V + E) × hidden_dim)
- Space Complexity: O(V × embedding_dim)
- Supervised or semi-supervised learning
- End-to-end differentiable

**Learning Process**:
1. Aggregate neighbor features
2. Apply non-linear transformations
3. Update node representations
4. Backpropagate error gradients

**Use Cases**:
- Node classification (semi-supervised)
- Graph classification (entire graph embedding)
- Link prediction (edge prediction)
- Graph-level predictions
- Inductive learning on new unseen nodes

**Advantages**:
- Leverages both structure and features
- End-to-end learning
- Handles transductive and inductive settings
- State-of-the-art performance on many tasks

**Disadvantages**:
- Requires node features (not pure structure)
- More complex than random walk methods
- Requires labeled training data for supervised learning
- Computationally expensive for very large graphs

---

## Implementation Considerations

### Algorithm Selection Guide

**For Shortest Paths**:
- Unweighted: Use BFS
- Non-negative weights: Use Dijkstra
- Negative weights: Use Bellman-Ford
- Multiple targets: Use A* with good heuristic

**For Node Importance**:
- General importance: PageRank
- Bridge nodes: Betweenness Centrality
- Central nodes: Closeness Centrality
- Direct influence: Degree Centrality
- Recursive importance: Eigenvector Centrality

**For Community Detection**:
- High quality communities: Louvain
- Very large networks: Label Propagation
- Hierarchical structure: Louvain with multi-level
- Fast results: Connected Components (if already known)

**For Link Prediction**:
- Simple baseline: Common Neighbors
- Normalized scores: Jaccard or Cosine
- Better accuracy: Adamic-Adar
- Growth networks: Preferential Attachment

**For Node Embeddings**:
- Pure structure: DeepWalk or Node2Vec
- Structure + features: Graph Neural Networks
- Interpretability: DeepWalk
- Flexibility: Node2Vec

### Performance Optimization

**Memory Efficiency**:
- Use sparse representations for large networks
- Stream process large graphs
- Cache intermediate results strategically

**Computational Speed**:
- Approximate algorithms for very large networks
- Parallelize independent computations
- Use efficient data structures (adjacency lists)
- Sample nodes/edges for approximation

**Convergence Tuning**:
- Set appropriate iteration limits
- Use early stopping criteria
- Balance precision vs. computation time

---

## Conclusion

This comprehensive reference covers the major categories of graph algorithms available in ArcadeDB. Selection of appropriate algorithms depends on:

- **Network characteristics**: Size, density, directionality, weights
- **Problem objective**: What question you're asking about the network
- **Performance constraints**: Available time and memory
- **Result interpretation**: Understanding what each algorithm measures

For production use, consider testing multiple algorithms and comparing results to find the best fit for your specific use case and network.

---

**References**: ArcadeDB Manual Pages 636-731
**Format**: Markdown Reference Guide
**Scope**: Graph algorithms for network analysis and optimization
