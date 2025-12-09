# Neo4j

- A graph database stores data in a graph structure.
- Data is represented as -
    - Nodes (entities/objects)
    - Relationships (connections between nodes)
- Useful when connections between data are as important as the data itself.
- Neo4j uses the Labeled Property Graph Model.

### Main Components

- **Nodes**
    - Represent entities or objects (e.g., `Person`, `Company`, `Location`).
    - Visualized as circles in a graph.
    - Each node represents a single entity instance.

- **Relationships**
    - Represent how two nodes are connected.
    - Consists of -
        - Type (e.g., `WORKS_AT`, `LIVES_IN`)
        - Direction (e.g., `Michael` → `WORKS_AT` → `Neo4j`)
    - Describe different kinds of connections -
        - Personal - `KNOWS`, `MARRIED_TO`
        - Facts - `OWNS`, `LIVES_IN`
        - Hierarchies - `PARENT_OF`, `DEPENDS_ON`
        - Generic - `CONNECTED_TO`
    - Bi-directional relationships need two explicit relationships (e.g., `Michael` `LOVES` `Sarah` and `Sarah` `LOVES` `Michael`).

- **Labels** 
    - Categorize nodes into groups.
    - Examples - `Person`, `Company`, `Location`.
    - Nodes can have multiple labels (e.g., `Person` + `Employee`). 
    - Labels should be singular nouns (`Person`, `Product`, `Event`).
    - Useful for filtering and organizing nodes.

- **Properties**
    - Information stored as key-value pairs on nodes and relationships.
    - Examples - `firstName`, `age`, `rating`, `position`
    - Can store any type - integer, string, boolean, list, etc.
    - Neo4j is _schemaless_ - same type of node doesn’t need the same properties.
    - Used for identifiers (unique keys).

### Graph Structure

| Component        | Meaning                  | Example                      |
| ---------------- | ------------------------ | ---------------------------- |
| **Node**         | Entity/object            | Person: Michael              |
| **Label**        | Category of node         | Person, Company              |
| **Relationship** | Connection between nodes | Michael WORKS_AT Neo4j       |
| **Property**     | Attributes               | name: “Michael”, since: 2020 |

## The `O(n)` Problem

- In Relational Databases -
    - Data stored in tables with indexes.
    - Queries join tables by scanning indexes.
    - As data grows - index grows - join cost increases.

- Result -
    - Query performance degrades linearly (`O(n)`) with data size.
    - Especially problematic for relationship-heavy queries.

## Why Graphs Are Fast

- Pointer-based navigation -
    - When two nodes are connected, Neo4j stores a direct pointer to the relationship.
    - Querying follows these pointers in memory instead of scanning an index.

- Result -
    - Traversal time stays constant (O(1)) regardless of total data size.
    - Only the number of relationships expanded affects performance.
    - Ideal for deep, complex, or dynamic relationship queries.
