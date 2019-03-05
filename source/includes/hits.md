# HITS App Development Experiences

## Iterative algorithms
Some graph ranking algorithms, such as Pagerank, HITS, and SALSA expect that multiple iterations of the algorithm will be run so that vertex ranking scores will converge to their true value over time. By default, the Hello app does not include support for multiple iterations. 

A Stop_Condition function must be defined that tells the IterationLoop struct's Core() function (where the user's app-specific algorithm is written) to exit. By default, the iteration loop's Stop_Condition function uses the stop function from the base IterationLoop class in enactor_iteration.cuh, which in turn calls the All_Done function in enactor_helper.cuh. All_Done always returns true if only one GPU is in use, as is the case for Hello, HITS, and many other apps.

The Stop_Condition function must be redefined in the IterationLoop struct in your app's enactor file. An example is given below, which instructs the loop to exit after a user-set number of iterations

```
bool Stop_Condition(int gpu_num = 0) {
    auto &data_slice = this->enactor->problem->data_slices[this->gpu_num][0];
    auto &enactor_slices = this->enactor->enactor_slices;
    auto iter = enactor_slices[0].enactor_stats.iteration;
    auto max_iter = data_slice.max_iter;
    
    // user defined stop condition
    if (iter == max_iter) return true;
    return false;
}
```

## App Building Blocks

### Operators
One of the most difficult parts of developing and optimizing this app was simply being aware of the different tools available. For future development, it would be useful to know what operators are available, with descriptions of their use and sample code to reference.

Currently, the only ways to determine what Gunrock has available is to:
1. Read the current documentation, which primarily focuses on advance and filter. This is a useful starting point to understand the Gunrock model, but this has significant holes. For example, NeighborReduce is not mentioned.
2. Dig through the gunrock/gunrock/oprtr folders to try to determine the use case of each operator. While this is reasonable to expect of a member of the Gunrock development team, average end-users would benefit from a well-documented list of operators they can use in their apps.
3. Examine other current apps to determine what strategies other developers have used. This can be especially useful if your app has some similarities to other currently-implemented apps (HITS is similar in many ways to PageRank for example). However, one problem with this approach are that the app may or may not have been updated for the new API, therefore requiring the user to dig through the source code of the operator to determine the usage pattern (not a terrible option, but this adds more friction),

### Frontier Initialization
Many of the default apps (Hello, Pagerank, BFS) begin with a single source node. However, some applications, such as HITS, require that all vertices be initialized into the frontior. This can be accomplished by using the following procedure:

```
util::Array1D<SizeT, VertexT> *null_frontier = NULL;
        frontier.queue_length = graph.nodes;
        frontier.queue_reset = true;
```

An advance operator (as an example) can then be called using null_frontier to perform advance on all nodes in the graph.
```
// Perform advance operation
GUARD_CU(oprtr::Advance<oprtr::OprtrType_V2V>(
    graph.csr(), null_frontier, null_frontier,
    oprtr_parameters, advance_op));
```

While an example of this procedure can be found in other apps such as Graph Coloring, the developer must know where to look. For future ease of development, frontier initialization options (single node, multiple nodes, all nodes) should be better documented.

# Unused Example Code
Large sections of the example code are commented out and are unused. hello_app.cu contains nearly 150 commented lines of old test code. hello_problem.cuh contains unused and broken multi-GPU code. While there is no significant downside to keeping this code in-place, it should either be fixed so that the app developer can use it if needed, or removed.

# 
