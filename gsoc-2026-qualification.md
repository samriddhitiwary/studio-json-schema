# GSoC 2026 Qualification Task: JSON Schema Studio

This document contains the deliverables for the initial qualification tasks for the JSON Schema Studio GSoC 2026 project.

## Part 1: `dependentSchemas` Keyword Handler

### Implementation Logic
The handler for the `dependentSchemas` keyword has been implemented in `processAST.ts`. 

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

### Why I Chose This Approach
1. **Handling Hyperjump's Compiled AST Structure**: Internally, Hyperjump compiles object schemas into iterable structures. The `dependentSchemas` object arrives as an array of tuples (`[propertyName, schemaUri][]`). I explicitly cast `keywordValue` to this type to seamlessly iterate over the dependencies without additional parsing overhead.
2. **Handle Collision Prevention (`childId`)**: By prefixing the `childId` as `dependentSchemas-${propertyName}`, we prevent handle and edge collisions in ReactFlow. A JSON schema might have a `properties` definition and a `dependentSchemas` definition sharing the exact same property name. Giving it a unique ID ensures the visual graph separates the regular property relationship from the dependency relationship.
3. **Delegating to `processAST`**: Rather than manually crafting nodes and edges for the dependent schemas, I leveraged the recursive nature of `processAST`. Passing the specific `schemaUri` and the new `childId` ensures the sub-schema nodes are identically processed, properly formatted, and automatically linked to the parent node.
4. **Returning Metadata for Handles**: Returning the collected `dependentPropertyNames` array communicates back to the parent node mapping that it needs to render custom outward handles for these dependencies.

---

## Part 2: Topologically Sorted Rendering of `$defs`

### Approach Overview
To implement topologically sorted rendering of `$defs`, the logic should be centralized directly inside the `$defs` keyword handler in `processAST.ts`. 

Currently, the `$defs` handler simply loops through the array of definitions sequentially as they appear:

```typescript
"https://json-schema.org/keyword/$defs": (ast, keywordValue, nodes, edges, parentId, nodeDepth, renderedNodes) => {
    const value = keywordValue as string[];
    // CURRENT: Sequential processing based on JSON order
    for (const [index, item] of value.entries()) {
        processAST({ ast, schemaUri: item, ... });
    }
    // ...
}
```

My approach is to dynamically sort the `value` array of definition URIs based on `$ref` dependencies *before* executing the `processAST` loop.

1. **Pre-scan the AST for `$ref`s**: Before calling `processAST`, loop through each URI in the `value` array and inspect its parsed AST node (`ast[schemaUri]`). We look specifically for the `"https://json-schema.org/keyword/ref"` keyword to determine if it references another definition within the same `$defs` group.
2. **Dependency Resolution**: Based on these references, we reorder the `value` array. For example, if Definition A contains a `$ref` pointing to Definition B, Definition B is pushed to be evaluated before Definition A in the array. Any definitions without references (leaf nodes) take priority.
3. **Sequential `processAST` Call**: Finally, loop over this newly ordered array. The rest of the `processAST` logic remains completely unchanged.

### Why I Chose This Approach

1. **Works with existing architecture**: The AST already contains all resolved references, so we can analyze dependencies without additional schema parsing.
2. **Minimal disruption**: Reordering happens in the `$defs` handler before the rendering loop - no changes needed to core rendering logic.
3. **Leveraging the `renderedNodes` Map**: In `processAST`, as nodes are parsed, they are stored in the `renderedNodes` Map. If Definition A depends on Definition B, processing Definition B *first* guarantees that it already exists in `renderedNodes`. When Definition A is finally processed and tries to reference it, `processAST` instantly finds the cached node, correctly calculating handles and edges without duplicating rendering work.
4. **Cleaner Graph Layouting**: Auto-layout algorithms evaluate dimensions sequentially. By ensuring fundamental dependencies are generated and calculated in `renderedNodes` first, the layout engine can position elements into structured tiers (roots before leaves). This prevents overlapping edges and erratic layout dimensions which happen when parent nodes try to connect to child nodes that haven't been fully formulated yet.
