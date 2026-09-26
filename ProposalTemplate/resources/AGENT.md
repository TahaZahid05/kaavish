# System Context Specification: Taxonomy-Grounded Heterogeneous Multi-Robot Task Allocation via Large Language Models

## 1. Executive Summary & Core Hypothesis

Current applications of Large Language Models (LLMs) in Multi-Robot Systems (MRS) predominantly suffer from two failure modes:

1. **Low-Level Over-Extension:** Querying LLMs for mid-level metric motion planning, numerical coordinate arithmetic, or low-level actuator control, where LLMs struggle with numerical reasoning, spatial precision, and high inference latency.


2. **Homogenized Heuristic Planning:** Utilizing one-size-fits-all heuristic pipelines or ungrounded greedy auction engines across all multi-robot scenarios, failing to exploit the formal mathematical structure of the underlying problem.



This project proposes a **Taxonomy-Grounded Heterogeneous Multi-Robot System Architecture**. The central hypothesis is that **using a centralized LLM strictly as a high-level cognitive engine**—tasked with natural language interpretation, task decomposition into a Directed Acyclic Graph (DAG), and formal classification into the **MRTA Taxonomy (Gerkey & Matarić, 2004; Korsah et al. iTax, 2013)**—allows the system to route each workload to a **specialized, mathematically rigorous combinatorial optimization algorithm**. This decoupled division of responsibilities guarantees algorithmic feasibility, optimizes global utility, and eliminates the vulnerabilities of ungrounded LLM control.

---

## 2. System Architecture & Information Flow

The end-to-end operational pipeline proceeds linearly across five decoupled layers:

```
[User Natural Language Prompt]
             │
             ▼
┌────────────────────────────────────────────────────────┐
│               1. Cognitive Orchestration Layer         │
│  - Centralized LLM + RAG (Taxonomy Definitions)        │
│  - Robot Fleet Competence Library Retrieval            │
│  - Map Representation Integration                      │
│  Outputs:                                              │
│    (a) Semantic DAG (Subtasks, Capabilities, Locations)│
│    (b) Taxonomy Subclass Label (1 of 7)                │
└──────────────────────────┬─────────────────────────────┘
                           │
             ┌─────────────┴─────────────┐
             ▼                           ▼
┌──────────────────────────┐┌───────────────────────────┐
│  2. DAG Parse & Validation││3. Subclass Algorithmic    │
│  - Edge verification     ││   Routing                  │
│  - Capability resolution ││  - Selects formal solver  │
│  - Cost/Utility matrix   ││    tailored to the chosen │
│    generation            ││    mathematical model      │
└────────────┬─────────────┘└─────────────┬─────────────┘
             └─────────────┬─────────────┘
                           │
                           ▼
┌────────────────────────────────────────────────────────┐
│               4. Combinatorial Optimization Layer      │
│  Executes subclass-specific solvers:                   │
│  Exact Solvers (MILP) / Heuristics / Metaheuristics   │
│  Outputs: Optimal schedule & task-to-robot assignments │
└──────────────────────────┬─────────────────────────────┘
                           │
                           ▼
┌────────────────────────────────────────────────────────┐
│               5. Execution & Simulation Layer          │
│  - ROS Middleware Bridge                               │
│  - Open-loop dispatch to Gazebo robot controllers      │
│  - Trajectory tracking and execution                   │
└────────────────────────────────────────────────────────┘

```

1. **User Input:** Natural language instruction specifying a multi-step objective within an environment.
2. **Cognitive Orchestration (LLM + RAG):** The centralized LLM processes the user prompt alongside:
* The **Competence Library** (catalog of robot IDs, mobility profiles, sensor payloads, and functional capabilities).


* **Map Semantic Annotations** (labeled topological nodes, coordinates, drop-offs, zones).


* **RAG Taxonomy Index** (formal operational definitions and dependency criteria from Korsah et al.).




3. **Structured Outputs:** The LLM generates two primary artifacts:
* A structured **Task DAG** specifying subtasks, spatial goals, execution prerequisites, and required capability tokens.


* A **Taxonomy Classification Label** identifying the problem as one of the 7 designated subclasses.




4. **Algorithmic Routing & Optimization:** The execution engine reads the taxonomy classification and instantiates the corresponding mathematical optimization model, populating its cost and constraint matrices using the DAG and robot parameters.


5. **Dispatch & Execution:** The computed optimal task sequences are translated into ROS action goals and executed by the heterogeneous fleet inside the Gazebo simulation environment.



---

## 3. Operational Assumptions & FYP Boundary Conditions

To establish tractability and isolate the algorithmic performance of the task allocation pipeline, the system operates under the following conditions:

* **Fully Mapped Environment:** The metric/topological map, navigation graph, obstacle bounds, and semantic points of interest (POIs) are known *a priori*.


* **Unconstrained Ideal Communication:** Inter-robot and central-controller communication links are lossless, have infinite bandwidth, and introduce negligible propagation delay (no packet drops, range constraints, or network partitioning).


* **Single Centralized LLM:** All natural language interpretation, task decomposition, and taxonomy classification decisions are handled by a single centralized reasoning agent.


* **Open-Loop / Ideal World Execution:** The environment is quasi-static, and low-level robot actions are assumed deterministic and succeed as dispatched; dynamic replanning loops, runtime failure recovery, and real-time obstacle reactivity are factored out of the initial benchmark scope.



---

## 4. The 7 Targeted Problem Subclasses & Mathematical Formulations

The project targets seven distinct subclasses spanning the **No Dependencies (ND)**, **In-Schedule Dependencies (ID)**, and **Cross-Schedule Dependencies (XD)** tiers of Korsah's taxonomy.

### 4.1. Class ND[ST-SR-IA]

* **Taxonomy Profile:** No Dependencies | Single-Task Robots | Single-Robot Tasks | Instantaneous Assignment.


* **Description:** A set of independent tasks must be assigned in a 1-to-1 matching to available single-task robots immediately, without scheduling future steps.


* **Underlying Model:** Linear Sum Assignment Problem (LSAP).


* Mathematical Formulation:



$$\max \sum_{i \in \mathcal{R}} \sum_{j \in \mathcal{T}} U_{ij} x_{ij}$$

$$\text{Subject to: } \sum_{i \in \mathcal{R}} x_{ij} \le 1 \quad \forall j \in \mathcal{T}$$

$$\sum_{j \in \mathcal{T}} x_{ij} \le 1 \quad \forall i \in \mathcal{R}$$

$$x_{ij} \in \{0, 1\} \quad \forall i \in \mathcal{R}, j \in \mathcal{T}$$

where $U_{ij}$ represents the estimated utility of robot $i$ executing task $j$.

* **Solution Paradigms:** Polynomial-time exact methods: Hungarian Algorithm ($O(n^3)$), Jonker-Volgenant, or Min-Cost Flow simplex.



### 4.2. Class ND[ST-SR-TA]

* **Taxonomy Profile:** No Dependencies | Single-Task Robots | Single-Robot Tasks | Time-Extended Assignment.


* **Description:** Robots are assigned multiple tasks over time, but because agent-task utilities are strictly independent, the order of task execution within an agent's route does not alter the cumulative utility or system makespan.


* **Underlying Model:** Clone-Expanded Linear Assignment Problem.


* **Mathematical Formulation:** Solved in polynomial time by instantiating virtual "clones" of each robot up to task capacity $\vert{}\mathcal{T}\vert{}$, populating dummy nodes with negative penalty utilities, and solving as a standard LSAP over the expanded bipartite graph.


* **Solution Paradigms:** Hungarian method on expanded clone-task matrix.



### 4.3. Class ID[ST-SR-TA]

* **Taxonomy Profile:** In-Schedule Dependencies | Single-Task Robots | Single-Robot Tasks | Time-Extended Assignment.


* **Description:** Tasks are spatially distributed and decoupled across robots, but an individual robot's utility/cost for a task depends directly on the sequence in which it visits its assigned locations (e.g., routing distance, fuel consumption, accumulated transit time).


* **Underlying Model:** Multiple Traveling Salesperson Problem (m-TSP) / Multi-Depot Capacitated Vehicle Routing Problem (MD-CVRP).


* Mathematical Formulation:



$$\min \sum_{k \in \mathcal{R}} \sum_{i \in \mathcal{V}} \sum_{j \in \mathcal{V}} c_{ij} x_{ijk}$$

$$\text{Subject to: } \sum_{k \in \mathcal{R}} \sum_{j \in \mathcal{V}, j \neq i} x_{ijk} = 1 \quad \forall i \in \mathcal{T}$$

$$\sum_{j \in \mathcal{T}} x_{0jk} = 1, \quad \sum_{i \in \mathcal{T}} x_{i(n+1)k} = 1 \quad \forall k \in \mathcal{R}$$

$$\text{Subtour Elimination Constraints (e.g., MTZ formulation) } \forall k \in \mathcal{R}$$

$$x_{ijk} \in \{0, 1\}$$

* **Solution Paradigms:** Mixed-Integer Linear Programming (MILP via branch-and-cut), Genetic Algorithms (GA) with edge recombination operators, Large Neighborhood Search (LNS), or Lin-Kernighan heuristics.



### 4.4. Class XD[ST-SR-IA]

* **Taxonomy Profile:** Cross-Schedule Dependencies | Single-Task Robots | Single-Robot Tasks | Instantaneous Assignment.


* **Description:** Tasks are executed instantaneously by single robots, but side constraints span across different robot assignments (e.g., shared global energy budget, mutual exclusion zones, joint capability limits).


* **Underlying Model:** Assignment Problem with Side Constraints (APSC).


* Mathematical Formulation:



$$\max \sum_{i \in \mathcal{R}} \sum_{j \in \mathcal{T}} U_{ij} x_{ij}$$

$$\text{Subject to standard assignment constraints, plus:}$$

$$\sum_{i \in \mathcal{R}} \sum_{j \in \mathcal{T}} A_{k,ij} x_{ij} \le B_k \quad \forall k \in \mathcal{K}_{\text{joint}}$$

$$x_{ij} \in \{0, 1\}$$

* **Solution Paradigms:** Branch-and-bound linear solvers, Lagrangian relaxation heuristics.



### 4.5. Class XD[ST-SR-TA]

* **Taxonomy Profile:** Cross-Schedule Dependencies | Single-Task Robots | Single-Robot Tasks | Time-Extended Assignment.


* **Description:** Robots construct multi-task schedules, but individual tasks assigned to different robots possess inter-task constraints, including precedence relations (Task $A$ must precede Task $B$), strict time-window synchronization, or delay penalties.


* **Underlying Model:** Vehicle Routing Problem with Time Windows and Precedence/Synchronization Constraints (VRPTW-S) / Machine Scheduling with Precedence ($R \vert{} prec \vert{} \sum w_j C_j$).


* **Mathematical Formulation:** Standard vehicle routing formulation augmented with temporal variables $S_j$ (start time of task $j$):



$$S_u + D_u + T_{uv} \le S_v + M(1 - x_{uvk}) \quad \forall (u, v) \in \mathcal{E}_{\text{path}}$$

$$S_a + D_a \le S_b \quad \forall (a, b) \in \mathcal{E}_{\text{precedence}}$$

$$\vert{}S_p - S_q\vert{} \le \epsilon \quad \forall (p, q) \in \mathcal{E}_{\text{sync}}$$

* **Solution Paradigms:** Mixed-Integer Programming (MIP), Genetic Algorithms with repair operators for temporal feasibility, or Constraint Programming (CP) engines.



### 4.6. Class XD[ST-MR-IA]

* **Taxonomy Profile:** Cross-Schedule Dependencies | Single-Task Robots | Multi-Robot Tasks | Instantaneous Assignment.


* **Description:** One or more tasks require the simultaneous cooperative effort of a group (coalition) of robots satisfying combined capability vectors (e.g., heavy object transport), where each robot belongs to at most one coalition.


* **Underlying Model:** Set Partitioning Problem (SPP) / Coalition Formation.


* Mathematical Formulation:



$$\max \sum_{c \in \mathcal{C}} U(c) z_c$$

$$\text{Subject to: } \sum_{c \in \mathcal{C} : r \in c} z_c \le 1 \quad \forall r \in \mathcal{R}$$

$$\sum_{c \in \mathcal{C} : t(c) = j} z_c \le 1 \quad \forall j \in \mathcal{T}_{\text{MR}}$$

$$z_c \in \{0, 1\} \quad \forall c \in \mathcal{C}$$

where $\mathcal{C}$ is the set of feasible coalition-task candidate pairings.

* **Solution Paradigms:** Greedy coalition pruning heuristics, Set-Partitioning Branch-and-Price, or distributed anytime coalition formation.



### 4.7. Class XD[ST-MR-TA]

* **Taxonomy Profile:** Cross-Schedule Dependencies | Single-Task Robots | Multi-Robot Tasks | Time-Extended Assignment.


* **Description:** The most challenging class in the FYP scope. Coalitions of heterogeneous robots must be dynamically formed and coordinated across time-extended schedules, subject to routing costs, precedence relations, and coalition synchronization.


* **Underlying Model:** Coalition Formation with Spatial and Temporal Constraints (CFSTP) / Multi-Mode Multi-Processor Task Scheduling.


* **Solution Paradigms:** Hybrid Decomposition: Metaheuristics (GA / Tabu Search) optimizing outer coalition assignments paired with lower-level scheduling/routing solvers, or unified MILP models.



---

## 5. Algorithmic Routing Matrix

The system utilizes a structured dispatch table to map LLM classifications to corresponding combinatorial optimization engines:

| Subclass ID | Taxonomy Subclass

 | Abstract Mathematical Model

 | Primary Exact Solver Paradigm

 | Alternative Heuristic / Metaheuristic

 |
| --- | --- | --- | --- | --- |
| **SC-1** | `ND[ST-SR-IA]` | Linear Sum Assignment Problem (LSAP) | Hungarian Algorithm ($O(n^3)$) | Greedy Eligibility Auction (BLE-style) |
| **SC-2** | `ND[ST-SR-TA]` | Expanded Linear Assignment Problem | Clone-expanded Hungarian Method | Greedy Multi-Item Queue Dispatch |
| **SC-3** | `ID[ST-SR-TA]` | Multiple TSP / Multi-Depot CVRP | MILP with MTZ subtour elimination | Multi-Chromosome Genetic Algorithm |
| **SC-4** | `XD[ST-SR-IA]` | Assignment Problem w/ Side Constraints | Branch-and-Cut Integer Linear Program | Lagrangian Relaxation + Local Search |
| **SC-5** | `XD[ST-SR-TA]` | VRPTW with Precedence & Synchronization | Temporal MILP formulation | GA with Topological-Sort Chromosomes |
| **SC-6** | `XD[ST-MR-IA]` | Set Partitioning Problem (Coalition Formation) | Binary Set-Partitioning ILP | Greedy Vector-Matching Coalition Search |
| **SC-7** | `XD[ST-MR-TA]` | Coalition Formation with Spatio-Temporal Constraints | Multi-mode Resource-Constrained MILP | Two-Stage GA (Coalition Selector + Scheduler) |

---

## 6. DAG Formal Definition & LLM Output Specification

The centralized LLM parses the user prompt and emits a single, strictly formatted JSON payload adhering to the schema below.

### 6.1. Formal Definition

A mission DAG is defined as $\mathcal{D} = (\mathcal{V}, \mathcal{E})$, where:

* $\mathcal{V} = \{v_1, v_2, \dots, v_m\}$ denotes the atomic subtasks. Each subtask $v_k$ carries spatial coordinates $\mathbf{x}_k = (x, y, z)$, an estimated execution duration $d_k$, and a required capability vector $\mathbf{r}_k = [c_1, c_2, \dots, c_p]$.
* $\mathcal{E} \subset \mathcal{V} \times \mathcal{V}$ denotes directed dependency edges. An edge $(v_a, v_b) \in \mathcal{E}$ indicates that task $v_b$ cannot commence until task $v_a$ has successfully completed.

### 6.2. LLM JSON Output Schema

```json
{
  "mission_id": "mission_alpha_01",
  "taxonomy_classification": {
    "subclass": "XD[ST-MR-TA]",
    "justification": "Task requires multi-robot coalition for transport (MR), time-extended multi-step routes (TA), and precedence constraints between inspection and transport (XD)."
  },
  "subtasks": [
    {
      "task_id": "task_inspect_valve",
      "target_location": {"x": 4.5, "y": 12.2, "z": 0.0},
      "action_type": "optical_inspection",
      "required_capabilities": ["camera_rgb", "precision_hover"],
      "estimated_duration_sec": 30.0,
      "minimum_robots": 1
    },
    {
      "task_id": "task_lift_crate",
      "target_location": {"x": 2.0, "y": 8.0, "z": 0.0},
      "action_type": "heavy_payload_transport",
      "required_capabilities": ["heavy_gripper", "high_payload"],
      "estimated_duration_sec": 120.0,
      "minimum_robots": 2
    }
  ],
  "dependencies": [
    {
      "predecessor": "task_inspect_valve",
      "successor": "task_lift_crate",
      "dependency_type": "PRECEDENCE"
    }
  ]
}

```

---

## 7. Gazebo Simulation & Heterogeneous Robot Testbed

The pipeline is validated within a unified Gazebo indoor/outdoor simulation world containing multiple functional task stations (e.g., loading docks, inspection consoles, debris zones, navigation corridors).

### Fleet Composition & Competence Library

The Gazebo fleet encompasses diverse morphologies to validate heterogeneous capability resolution:

* **Wheeled Mobile Bases (e.g., TurtleBot3 / Clearpath Jackal):** High spatial navigation speed, low payload capacity, equipped with 2D LiDAR and wheel encoders. Capabilities: `["ground_navigation", "patrol", "light_sensor"]`.
* **Mobile Manipulators (e.g., Fetch / Tiago / TurtleBot with Arm):** Moderate navigation speed, object manipulation capability. Capabilities: `["ground_navigation", "object_grasping", "button_press", "tabletop_pick_place"]`.
* **Aerial Drones (e.g., Quadrotor / Iris in RotorS):** Unconstrained 3D mobility, high speed, limited flight endurance, lightweight payload. Capabilities: `["aerial_survey", "high_altitude_camera", "aerial_reconnaissance"]`.

---

## 8. Implementation Roadmap & Milestones

Following the task ordering derived from project planning:

* **Milestone 1: Environment & Task Definition (Step 1)**
* Finalize the comprehensive Gazebo world containing stations and tasks capable of exercising all 7 subclasses.


* Formulate standardized Prompt-to-Task test cases for each subclass.




* **Milestone 2: Parallel Core Development (Step 2)**
* **Track A (Cognitive Engine):** Implement LLM prompt engineering, RAG indexing of iTax definitions, Competence Library matching, and JSON schema validation.


* **Track B (Optimization Engine):** Construct standalone Python solvers for the 7 subclasses (integrating exact solvers like PuLP/Gurobi and custom metaheuristics).




* **Milestone 3: Pipeline Integration (Step 3)**
* Couple the LLM parser directly to the solver routing switchboard. Verify that LLM-emitted JSON objects trigger the corresponding optimization algorithms with valid input matrices.




* **Milestone 4: ROS & Gazebo Bridge (Step 4)**
* Implement ROS nodes that translate solver schedules into sequential action client commands dispatched to individual robot navigation/manipulation stacks.




* **Milestone 5: Benchmarking & Empirical Evaluation (Step 5)**
* Evaluate the taxonomy-grounded pipeline against standard baselines (e.g., generic greedy auction allocations, direct ungrounded LLM task planning).


* Record and report makespan, computation latency, allocation optimality gap, and fleet travel distances.