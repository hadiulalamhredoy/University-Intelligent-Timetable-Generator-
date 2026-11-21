# University-Intelligent-Timetable-Generator
# University-Intelligent-Timetable-Generator

[![Python](https://img.shields.io/badge/Python-3.9%2B-blue.svg)](https://www.python.org/)
[![Algorithm](https://img.shields.io/badge/Algorithm-A*%20Search%20%26%20Conflict%20Graph-brightgreen.svg)](#algorithmic-architecture)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](#license)

An automated academic scheduling system designed to solve the **University Course Timetabling Problem (UCTP)** by combining **Graph Theory (Conflict Graph Modeling & DSATUR Graph Coloring)** with **A\* Informed State-Space Search**.

---

## Table of Contents

- [Overview](#overview)
- [Key Features](#key-features)
- [Algorithmic Architecture](#algorithmic-architecture)
  - [1. Conflict Graph Modeling](#1-conflict-graph-modeling)
  - [2. A\* State-Space Search](#2-a-state-space-search)
  - [3. Forward Checking & Domain Pruning](#3-forward-checking--domain-pruning)
- [Constraint Specification](#constraint-specification)
- [Repository Structure](#repository-structure)
- [Getting Started](#getting-started)
- [Sample Output](#sample-output)
- [Programmatic Usage & Customization](#programmatic-usage--customization)
- [Scalability & Future Roadmap](#scalability--future-roadmap)
- [License](#license)

---

## Overview

The **University Course Timetabling Problem (UCTP)** is a classic $\mathcal{NP}$-hard combinatorial optimization problem. Constructing a valid semester schedule requires assigning every lecture and laboratory session to a `(timeslot, room)` pair such that no faculty member, student cohort, or classroom is double-booked, while simultaneously optimizing schedule quality (such as minimizing student idle gaps and honoring instructor availability preferences).

**University-Intelligent-Timetable-Generator** formulates course dependencies as an undirected **Conflict Graph** and explores valid partial schedules using **A\* Search** guided by **DSATUR (Degree of Saturation)** vertex ordering and an **admissible soft-constraint heuristic**.

---

## Key Features

- **Conflict Graph Formulation:** Automatically detects structural resource clashes (shared instructors and shared student cohorts) and builds an adjacency graph $G = (V, E)$.
- **DSATUR + MRV Dynamic Variable Ordering:** Prioritizes the most constrained course sessions first using graph saturation degree, tie-broken by Minimum Remaining Values (MRV) and static vertex degree.
- **Optimal A\* Search:** Evaluates partial schedules via $f(n) = g(n) + h(n)$ using a strictly admissible lower-bound heuristic to guarantee optimal soft-constraint minimization.
- **Forward Checking:** Immediately prunes infeasible `(timeslot, room)` pairs from neighboring vertices upon each assignment, eliminating dead-end branches early.
- **Configurable Penalty Weights:** Easily tune weights for student idle gaps, faculty idle gaps, non-preferred teaching slots, undesirable hours, and room capacity utilization.
- **Zero External Dependencies:** Implemented cleanly using only the Python 3.9+ standard library (`heapq`, `dataclasses`, `collections`, `typing`).

---

## Algorithmic Architecture

### 1. Conflict Graph Modeling

The scheduling domain is compiled into an undirected **Conflict Graph** $G = (V, E)$:

- **Vertices ($V$):** Each vertex $v_i \in V$ represents an individual course session (e.g., `CSE-201 Lecture`, `CSE-202 Lab`) carrying metadata for enrollment count, required room type (`lecture` or `lab`), assigned instructor, and student batch.
- **Conflict Edges ($E$):** An undirected edge $(v_i, v_j) \in E$ connects two sessions whenever they **cannot** be scheduled in the same timeslot:
  1. **Instructor Clash:** $\text{Instructor}(v_i) = \text{Instructor}(v_j)$
  2. **Student Cohort Clash:** $\text{Batch}(v_i) = \text{Batch}(v_j)$

Scheduling timeslots is equivalent to **graph coloring**, where adjacent vertices in $G$ must receive distinct timeslot colors. At each search step, the next unscheduled vertex is chosen dynamically using the **DSATUR** criterion—selecting the vertex whose scheduled neighbors already occupy the highest number of distinct timeslots:

$$
\text{sat}(v) = \left|\{ \text{slot}(u) \mid u \in N(v) \cap V_{\text{assigned}} \}\right|
$$

Ties are broken by the **Minimum Remaining Values (MRV)** heuristic (smallest valid `(timeslot, room)` domain) and highest static vertex degree $\deg(v)$.

### 2. A\* State-Space Search

Each node $n$ in the A\* priority queue represents a partial schedule mapping $V_{\text{assigned}} \subseteq V$ to `(timeslot, room)` pairs, ranked by:

$$
f(n) = g(n) + h(n)
$$

- **Exact Path Cost $g(n)$:** The accumulated soft-constraint penalty of all currently scheduled sessions, including non-preferred faculty timeslots, undesirable hours, seat capacity waste, and finalized daily idle gaps.
- **Admissible Heuristic $h(n)$:** A lower bound on the remaining soft-constraint penalty, calculated by summing the minimum possible incremental penalty across every unscheduled session $u \in V \setminus V_{\text{assigned}}$ over its currently valid domain $D(u)$:

$$
h(n) = \sum_{u \in V \setminus V_{\text{assigned}}} \min_{(t, r) \in D(u)} \Delta \text{cost}(u, t, r)
$$

Because $h(n)$ never overestimates the true remaining cost ($h(n) \le h^*(n)$), A\* Search is guaranteed to find a globally optimal timetable with respect to the configured penalty weights.

### 3. Forward Checking & Domain Pruning

Whenever a vertex $v$ is assigned a pair $(t, r)$:
1. All adjacent vertices $u \in N(v)$ lose timeslot $t$ from their valid domains.
2. All unscheduled vertices lose room $r$ at timeslot $t$ to preserve room exclusivity.
3. If any unscheduled vertex $u \in V \setminus V_{\text{assigned}}$ ends up with an empty valid domain ($|D(u)| = 0$), the search state is pruned immediately before entering the open set.

---

## Constraint Specification

| Category | Type | Rule Description |
| :--- | :--- | :--- |
| **Hard** | Conflict Graph Exclusivity | Adjacent vertices $(v_i, v_j) \in E$ cannot be assigned the same timeslot. |
| **Hard** | Room Exclusivity | At most one course session may occupy a given room $r$ during any timeslot $t$. |
| **Hard** | Room Capacity & Type | Room capacity must satisfy $\text{Capacity}(r) \ge \text{Enrollment}(v)$ and match `lecture`/`lab` type. |
| **Hard** | Faculty Availability | Sessions cannot be placed in an instructor's unavailable timeslots. |
| **Soft** | Student Compactness | Penalizes idle free hours between classes on the same day for a student cohort. |
| **Soft** | Faculty Compactness | Penalizes idle free hours between classes on the same day for an instructor. |
| **Soft** | Faculty Preferences | Penalizes assignments outside an instructor's preferred teaching timeslots. |
| **Soft** | Slot Desirability | Penalizes undesirable early-morning or late-evening timeslots. |
| **Soft** | Room Utilization | Penalizes assigning small classes to disproportionately large lecture halls. |

---

## Repository Structure

```text
University-Intelligent-Timetable-Generator/
├── timetable_solver.py    # ConflictGraph, A* Search solver, DSATUR ordering & sample dataset
└── README.md              # Project documentation, mathematical model & usage guide
```

---

## Getting Started

### Prerequisites

- **Python 3.9 or higher** (No third-party packages required)

### Installation & Execution

1. Clone the repository:
   ```bash
   git clone https://github.com/your-username/University-Intelligent-Timetable-Generator.git
   cd University-Intelligent-Timetable-Generator
   ```

2. Run the timetable generator on the built-in university dataset:
   ```bash
   python timetable_solver.py
   ```

---

## Sample Output

```text
Conflict Graph G=(V, E): |V|=10 sessions, |E|=24 conflict edges, Max Degree=6, Graph Density=0.53
Running A* Search with DSATUR Graph Ordering & Forward Checking...

Optimal Schedule Found in 3.45 ms | Nodes Expanded: 11 | States Pruned: 4 | Soft Penalty Score: 1.54

==============================================================================
 TIMETABLE FOR STUDENT COHORT: CSE-2025
==============================================================================
Timeslot       | Course     | Title                    | Room       | Instructor
------------------------------------------------------------------------------
Mon 09:00      | CSE-305    | Artificial Intelligence  | LH-102     | Dr. N. Jahan
Mon 10:00      | CSE-309    | Computer Networks        | LH-102     | Prof. S. Karim
Tue 09:00      | CSE-301    | Operating Systems        | LH-102     | Dr. A. Rahman
Tue 10:00      | CSE-310    | Networks Lab             | CS-Lab-A   | Prof. S. Karim
Tue 11:00      | CSE-302    | OS Kernel Lab            | CS-Lab-A   | Mr. T. Hasan

==============================================================================
 TIMETABLE FOR STUDENT COHORT: CSE-2026
==============================================================================
Timeslot       | Course     | Title                    | Room       | Instructor
------------------------------------------------------------------------------
Mon 09:00      | CSE-201    | Data Structures          | LH-101     | Dr. A. Rahman
Mon 10:00      | CSE-202    | Data Structures Lab      | CS-Lab-A   | Dr. A. Rahman
Mon 11:00      | MAT-205    | Discrete Mathematics     | LH-101     | Prof. S. Karim
Wed 09:00      | CSE-207    | Digital Logic Design     | LH-101     | Dr. N. Jahan
Wed 11:00      | CSE-208    | Digital Logic Lab        | CS-Lab-A   | Mr. T. Hasan
```

---

## Programmatic Usage & Customization

You can import `AStarTimetableSolver` and the domain models directly into your own scheduling pipeline and customize the soft-constraint weights:

```python
from timetable_solver import (
    AStarTimetableSolver,
    CourseSession,
    Instructor,
    Room,
    Timeslot,
)

# Define timeslots, rooms, instructors, and course sessions
timeslots = [
    Timeslot(id=1, day="Mon", hour=9, label="Mon 09:00"),
    Timeslot(id=2, day="Mon", hour=10, label="Mon 10:00"),
]

rooms = [
    Room(id="R101", name="LH-101", capacity=60, room_type="lecture"),
]

instructors = [
    Instructor(
        id="I1",
        name="Dr. A. Rahman",
        unavailable_slots=frozenset(),
        preferred_slots=frozenset({1, 2}),
    ),
]

sessions = [
    CourseSession(
        id="S1",
        course_code="CSE-201",
        title="Data Structures",
        instructor_id="I1",
        batch_id="CSE-2026",
        enrollment=55,
        room_type="lecture",
    ),
]

# Configure custom penalty weights
custom_weights = {
    "non_preferred_slot": 2.0,
    "undesirable_slot": 3.0,
    "room_waste_ratio": 1.0,
    "student_idle_gap": 5.0,
    "faculty_idle_gap": 2.5,
}

solver = AStarTimetableSolver(
    sessions=sessions,
    rooms=rooms,
    timeslots=timeslots,
    instructors=instructors,
    weights=custom_weights,
)

result = solver.solve(max_expansions=50_000)
if result:
    schedule, penalty_score = result
    print(f"Optimal schedule generated with penalty score: {penalty_score:.2f}")
```

---

## Scalability & Future Roadmap

- **Weighted $A^*$ ($WA^*$) Mode:** Introduce an inflation factor $\epsilon > 1$ in $f(n) = g(n) + \epsilon \cdot h(n)$ to compute bounded-suboptimal schedules rapidly for large university-wide datasets ($500+$ course sessions).
- **Multi-Hour Contiguous Lab Blocks:** Extend the domain generator to schedule 2-hour and 3-hour consecutive laboratory sessions atomically.
- **CSV / JSON Data Loaders:** Add CLI flags to import department courses, faculty availability, and room inventories from external files and export generated timetables to CSV, PDF, and `.ics` calendar formats.
- **Web Interface & REST API:** Wrap `AStarTimetableSolver` in a FastAPI backend paired with an interactive weekly timetable grid.

---

## License

This project is licensed under the **MIT License**. See the `LICENSE` file for details.
