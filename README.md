## Overview

This is the core of our routing engine. This is the application that calculates the shortest path from A->B. We also accept queries with in-between nodes, for example A->B->C. This case is easy because we calculate 2 shortest paths, one from A to B and one from B to C and then we combine them.

We receive queries (sequence of coordinates) from the user, we then compute the shortest path and send the response (the shortest path as a sequence of coordinates) back to the user.

## Query and Response messages format

The technology we use to send messages (communicate) between the server and the client is **Websockets**, since that technology is built-in Javascript. However, for our C++ application, that technology is not built-in. At least not in an easy cross-platform way for Windows and Linux. This is the reason why we use a (headers-only) library called [Websocketpp](https://github.com/zaphoyd/websocketpp). This is actually the only 3rd-party-library we use for the server-side.

When the user performs certain actions in the website, for example adding/moving/deleting nodes, a query is sent to the `server-side`. The **query** format we follow right now is really simple, for example:

Format: `Latitude_A Longitude_A Latitude_B Longitude_B`<br>
Example query: `38.0006436 23.7786841 38.0022161 23.7793064`

The message we send to the client as a **response** from the c++ application after the shortest path algorithm is computed, is the **nodes sequence of the shortest path**. The response message has the same format as the query, only this time we have a lot more nodes. Each 2 adjacent nodes basically represent an edge in the shortest path. For example, here is a **response message** from the server:

`38.0006897 23.7785669 38.0007352 23.7785848 38.0009246 23.7786363 38.0012873 23.7786724 38.0013635 23.7793300 38.0017295 23.7791770 38.0019797 23.7790311 38.0022526 23.7792636`

# Computing the shortest path

## Finding nearest edges from user's given coordinates

The first thing we need to do before we can calculate the shortest path from A->B is to find the nearest point of our graph for each of the nodes A and B. Intuitively this means that we want to **snap the user's given coordinates to actual edges on our graph**. By that I mean that the user might select coordinates that are not on edges and might even be kilometers far from any edge of our graph. Since a car can obviously only move on our edges (aka road segments), we want to find the nearest edge, as well as the projection of the user's given coordinate to the nearest edge (the projection needs to always be on the edge segment, not outside of the segment). We do this for each coordinates A and B.
 
To find the nearest edge for each of the user's given coordinates we make use of our **KD-Tree**. The **KD-Tree** is a data structure that can help us search for nodes (not edges yet) within a radius. In our application, we start searching for nodes within a 250-meter radius. If no nearby node is found, then we increase that range by some amount until we find one or more nodes. The starting value of 250 meters is not abstract. It is the longest edge length in our entire graph. You might want to take a look [here](https://github.com/outerpixels/routing-engine-graph-extractor/blob/master/README.md) for the graph parsing. We split the edges that are longer than 250 meters by adding more nodes and edges to the graph. Overall, we end up adding only a tiny amount (1-2%) compared to the size of the graph. We don't have to split AB edges, where A or B are not `startpoints`.

We keep performing `radius-search` in our **KD-Tree** in wider and wider circles (radius) from the user's given coordinate, until we find a collection of one or more nodes in this manner. For each node `u` of that collection, we look in our graph's edges (more info about the graph in-memory storage [below](#graph-representation)) and find every edge that starts or ends (our graph representation allows us to look into the graph backwards as well) in `u`. This will result in getting a `set` of edges. Because we search in a radius of 250 or more, and the longest edge of the entire graph is 250, it is guaranteed that the nearest edge, is included in that `set` on edges. Otherwise, if we searched in a radius less than 250, we would ran into [this problem](http://stackoverflow.com/questions/19892564/find-nearest-edge-in-graph). For each edge in that set, we then calculate the distance from point (user's given coordinate) the the edge. The nearest distance will give us the nearest edge. 

This whole process takes only a couple of milliseconds on less than average hardware. So it is very effecient, and also, memory usage is the absolute bare minimum (1 array of integers for the KD-Tree).

## Query graph

The projection of the user's given coordinate to the nearest edge we just calculated is a point on the nearest edge. That point will be added to what we call `query graph` as a `virtual node`. For example:

![2](https://i.gyazo.com/f804092e638dd6884ef84ef926161993.png)

In the example above, we added 2 `virtual nodes` (black circles) and 8 `virtual directed edges` (red arrows). This way, our shortest path algorithm will actually consider both ways of the road, since it so happened in this example that both of those ways were double-ways. The query graph, is the graph on which the shortest path is actually performed on. It is essentially our original graph, plus few more `virtual nodes` and `virtual directed edges`.

## Graph representation

All graph data are stored in-memory. We use Adjacency-Lists. Which means that we have a big array that is indexed with `my_ids` which are integers from 0 to N. The `i-element` of that array simply contains a list (adjacency list) with all the edges that **start or end** on the node `i`. If the edge starts with `i`, then we have a `forward edge`. If it ended with `i` we have a `backward edge`. We also have a third "flag" which represents `double edges`. This is important because marking `double edges` with a special flag, instead of adding 2 different edges only costs us half the memory. Remember that we want to be able to access both the forward and the backward edges of any given node. This is required in shortest path algorithms such as Bidirectional-Dijkstra. The flag is stored on the element as a `char` (1 byte) that takes values `F`, `B` or `D`.

![3](https://i.gyazo.com/515ec7ebbda21eb34e71fe6197fb8d8f.png)

## KD-Tree implementation

The KD-Tree is a binary tree. In our application we only save the `node_id` on our KD-Tree, because we already have an array of `coordinates` that is indexed with `node_ids` and can give us the node's `LatLng` information. Also, because KD-Tree is a binary tree, we can store it in memory as an array and thus saving memory.

![4](https://upload.wikimedia.org/wikipedia/commons/thumb/8/86/Binary_tree_in_array.svg/450px-Binary_tree_in_array.svg.png)

When creating the KD-Tree, we use the efficient KD-Tree creation method, which has **O(kn log n)** time complexity, and create a **balanced** binary tree. This is possible because our data are static and also we never add or remove nodes on run-time. [This](http://jcgt.org/published/0004/01/03/paper.pdf) is the paper that I followed to create the KD-Tree efficiently.

## Latitude Longitude coordinates

Another trick to reduce memory is to store the world coordinates as integers. The `.osm` data provide [accuracy to 7 decimal places](http://wiki.openstreetmap.org/wiki/Node), which translates to cm-accurancy. Latitude takes values from -90 to +90, and Longitude takes values from -180 to +180. That means that this value can be stored as an integer. In the worst case, we would have to "compress" the Longitude value 180, which would be equal to 180 * 10.000.000 = 1.800.000.000 and it fits on a 4-byte integer perfectly. 

If we opted for float (also 4-byte) we would lose accuracy because floats are accurate to about [5 decimal places](https://en.wikipedia.org/wiki/Single-precision_floating-point_format).

If we opted for double (8-byte) we would spend twice the memory and we would gain only a tiny bit faster operations between `LatLngs` because of the fact that we wouldn't have to "convert" the latitude and longitude values from integers to doubles (by multiplying or dividing by 10.000.000). I should also note that some operations, for example `comparison`, don't even require this "convertion".

## Connected components labeling 

A quick way to determine if a shortest path does not exist before we even start the shortest path algorithm is [connected components labeling](https://en.wikipedia.org/wiki/Connected-component_labeling). When the graph data are loaded in memory, we can perform an algorithm that will label each node with an id, so that if node `x` and node `y` have the same label, then they belong in the same `connected component`. Now, that does not necessarily mean that the path exists, but if 2 nodes belong in 2 different connected components, then the shortest path does not exist. This method is important, because without it, if we performed a shortest path algorithm from nodes that belong in 2 different isolated graphs, then the shortest path algorithm would end up searching many nodes before it is able to detect that the shortest path does not exist. 

When implementing the connected components labeling algorithm, it is important to use a `stack` that allocates memory in the heap and not in the stack. That means that we don't want to use recursive algorithms, as that would result in **stack-overflow**. This occurs because our connected components are either really huge, or really tiny (parking spots, private/blocked roads, etc.).

If the number of connected components is less than `65535` then it is better to use an `unsigned short` to reduce the memory consumpion to half (as opposed to `unsigned integer`).

## Future work

 - For the shortest path algorithm we are currently using Bidirectional Dijkstra. This can be improved immensely with the method of [Contraction Hierarchies](https://en.wikipedia.org/wiki/Contraction_hierarchies). [Here](http://algo2.iti.kit.edu/schultes/hwy/contract.pdf) is a great paper and [here](https://algo2.iti.kit.edu/download/presentation.pdf) is a presentation that explains the method in detail. Fortunately, we have already implemented Bidirectional-Dijkstra. All we have to do to add this method in our application is to modify our `AdjElement` class so that it contains another property, the `middle_node`, which will help us unravel the shortcut-edges into the actual edges that we will show on the map. Then, we need to add a method that will add the shortcut edges in our graph, and finally, slightly modify our Bidirectional Dijkstra algorithm so that it relaxes edges only higher priority nodes in the forward search and lower priority nodes in the backward search.

 - Right now, we use `connected components labeling` to determine if 2 given coordinates from the user are connected and if they are not, then we respond with a `path-does-not-exist` message. This occurs if the user gives coordinates between 2 different isolated graphs. However, a more elegant solution is to keep searching (info [here](#finding-nearest-edges-from-users-given-coordinates) to explain how we do this search) until we find 2 nodes that belong to the same connected components. This way we would be able to find a shortest path, even though it would not include the nearest road from the user's given coordinate. This is not trivial to implement, and it requires the use of a few tuning parameters. We have to remember that our routing engine supports in-between nodes as well. The user could give a A->B->C query where the 3 coordinates are close to 3 different edges that belong to 3 different connected components. In this case we would have to determine which **alternative edges** we have to select, so that the query does not result in `path-does-not-exist`. To do this we would need to take into account the **size** of each of the 3 connected components (we prefer edges that belong to larger components, instead of tiny ones like parking or a blocked road) as well as the **distance** between the alternative edge and the user's given coordinate (compared to the nearest edge). We would also need to consider the fact that searching for an alternative edge could be costly. If the user gave a query from America to Africa, then it would be a waste of time to search for alternative edges to make this query happen, because, if anything, those alternative edges would be far off from the user's given coordinate. That means that our search for alternative edges has to have a few proper stopping conditions. Those stopping conditions would prevent spending time to try and solve illogical queries, for example from America to Africa, but at the same time, be forgiving enough to find an alternative edge if the user mistakenly given a coordinate a little closer to a blocked road, compared to a free road. A less sophisticated but common approach to solve this problem, is to remove tiny connected components from the graph completely. [Graphopper](https://github.com/graphhopper/graphhopper) does make use of this approach.
