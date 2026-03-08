# GSoC 2026 Qualification Task: JSON Schema Studio

This document contains the deliverables for the initial qualification tasks for the JSON Schema Studio GSoC 2026 project.

## Part 1: `dependentSchemas` Keyword Handler

### Implementation Logic
The handler for the `dependentSchemas` keyword has been implemented in `processAST.ts`. Let's break down the logic of the implementation to explain how it integrates with the Hyperjump JSON Schema internal compilation and ReactFlow node/edge generation.

```typescript
"https://json-schema.org/keyword/dependentSchemas": (ast, keywordValue, nodes, edges, parentId, nodeDepth, renderedNodes) => {
    const value = keywordValue as [string, string][];
    const dependentPropertyNames = [];
    
    for (const item of value) {
        const propertyName = item[0];
        const schemaUri = item[1];
        const childId = `dependentSchemas-${propertyName}`;
        
        dependentPropertyNames.push(childId);
        
        processAST({ 
            ast, 
            schemaUri: schemaUri, 
            nodes, 
            edges, 
            parentId, 
            renderedNodes, 
            childId: childId, 
            nodeTitle: `dependentSchemas["${propertyName}"]`, 
            nodeDepth 
        });
    }
    
    return { key: "dependentSchemas", data: { value: dependentPropertyNames } }
}
```

#### Detailed Breakdown:
1. **Type Assertion of Compiled AST (`keywordValue as [string, string][]`)**: 
   When Hyperjump compiles a JSON schema, it parses object properties into iterable mappings. For `dependentSchemas`, instead of remaining a raw `Record` or Object mapping property names to schema objects, it arrives in the handler compiled as an array of tuples (e.g., `[[propertyName, schemaURI], ...]`). Correctly understanding this data structure assertion is the first vital step.
   
2. **Iteration over Dependencies:** 
   The handler loops through each item in this tuple array, extracting the `propertyName` (which is the property that triggers the dependent schema requirement) and the `schemaUri` (the reference pointer to the compiled definition of that dependent schema).
   
3. **Node and Handle Relationship Generation (`childId`)**: 
   To ensure ReactFlow can connect the corresponding source node handle to the correct target node without duplication or cross-linking errors, we generate a unique `childId` string (`dependentSchemas-${propertyName}`). This maps the graph dependency back to the explicit property triggering it.
   
4. **Recursive AST Processing (`processAST`)**: 
   For each dependent schema, we recursively invoke `processAST`. This function is responsible for the actual graph data mutation:
   - Generating graph **nodes** for the dependent schema definitions based on the `schemaUri`.
   - Creating the connecting **edges**, correctly tying the `parentId` to the newly referenced schema node.
   - Assigning a clear human-readable UI label via `nodeTitle: dependentSchemas["${propertyName}"]`.
   
5. **Return Metadata for Graph Reactivity:** 
   Finally, we return an object containing the keyword identifier (`"dependentSchemas"`) and the data slice holding the registered `childId` values (`dependentPropertyNames`). This returned data guarantees that the parent ReactFlow node can render the correct corresponding outward handles matching the listed dependencies.

---

## Part 2: Topologically Sorted Rendering of `$defs`

### Approach
In complex schemas, `$defs` heavily reference each other, creating tangled dependency chains. When generating graph data purely based on the sequential order keys appear in the JSON document, node processing occurs haphazardly. My approach to implementing topologically sorted rendering involves calculating the correct traversal flow prior to mapping nodes.

#### 1. Dependency Graph Extraction
Before mapping the properties to graph nodes, performing a pre-traversal of all `$defs` is necessary. We should scan the schema properties for references (`$ref` and `$dynamicRef`) pointing to other local `$defs` URIs. This results in building an abstract directed graph reflecting these relationships (e.g., `Definition A -> depends on -> Definition B`).

#### 2. Topological Sorting (Kahn's Algorithm / DFS)
Apply a topological sorting algorithm (like Kahn's Algorithm or depth-first search (DFS)) over the mapped dependency graph. 
- Schema nodes with an in-degree of 0 (they don't reference other custom definitions, only native types) are ranked to be processed first.
- As these are removed from the abstract graph, we update the degrees of remaining references until we've parsed an ordered, dependency-safe array.

#### 3. Resolving Circular Dependencies
JSON Schema strictly allows circular definitions (e.g., recursive data structures like trees or linked lists). A standard topological sort throws errors during cycles. We need to implement cycle detection tracking the visited recursion stack. When a back-edge creating an endless cycle is discovered, we temporarily isolate it from the dependency graph logic. We mark these edges as "recursive instances", which the graph engine can subsequently style uniquely (e.g., utilizing an alternate dotted-line style to denote recursion, preventing visual clutter).

#### 4. Ordered Rendering Pass
Instead of indiscriminately looping through the `ast` keys, the engine iterates through our pre-computed topologically sorted array of URIs, reliably triggering `processAST` to generate nodes precisely based on dependency-weight.

### WHY this approach?
I opted for this specific model due to its vast benefits for both the rendering engine and graph aesthetic readability:

- **Guaranteeing Layout Accuracy & Pre-Calculation**: Visualization auto-layout algorithms (e.g., Dagre.js standardly paired with ReactFlow) behave erratically if node dependencies are populated out-of-order. Computing dependency flow before creating ReactFlow states guarantees that when a node renders, constraints for the children it references have already been accurately measured and established.
- **Edge Crossing Reduction**: By ensuring roots are evaluated prior to leaves, auto-layout ranks nodes into precise tiered layers, drastically mitigating overlapping edges or confusing crossed spaghetti-wires in major schemas (a highly requested element in the GSoC outcomes).
- **Graceful Handling of Recursive Code**: Explicitly extracting connection maps before rendering ensures infinite recursive loops crash neither the compiler nor the DOM, all while letting the user visibly explore cyclical nature via distinct styling patterns.
