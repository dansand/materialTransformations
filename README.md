## Material transitions

In geodynamic simulations, it is common to impose compositional / material changes during the model evolution. An example is a creation (and destruction) of weak crust, which is often used to help the thermal boundary layers decouple (subduct) in a realistic manner. The transition from one material type to another could be a function of depth, pressure, temperature, age, accumulated strain etc. 

Functions, along with `fn.branching.conditional` provide an easy way to track and update materials. 

## Functions

Here's a simple way to set the value of particles on a swarm:

```
Conditions = [(depthFn > 0.5, 2),
                (True, materialVariable)]

materialVariable.data[:] = fn.branching.conditional(Conditions).evaluate(swarm)
```



Often, a material transition depends not only on the current condition of th particle (depth, pressure, temperature, age, accumulated strain etc. ), but also on the _current_ material type of the partcle. We may allow 'mantle' material to become 'crust', but not allow harzburgite to become crust. 

The following expression looks reasonable:

```
combinedConditions = (depthFn > 0.5) and (materialVariable == 1)
```

However, in uw2 functions, this type of equality comparison is not supported (because of the underlying c++ data types). We can instead use greater than / less than comparison:

```
dta = 1e-6 #small number
combinedConditions = (depthFn > 0.5) and \
                     (materialVariable > (1 - dta) ) and \
                     (materialVariable < (1 + dta) )
```

We can make also use functional forms of the operators, provided by the __operator__ package:

```
dta = 1e-6 #small number
indexCheck = operator.and_((materialVariable > (1 - dta) ), 
                           (materialVariable < (1 + dta) ))
             
combinedConditions = operator.and_((depthFn > 0.5) , indexCheck)
```



## Material graphs

A more structured way to represent a series of material transitions is through a directed graph. In a graph, nodes represent different material types, edges (directed) represent all possible transitions that could occur. We then need to attach conditions to those edges, and then be able to query the graph for any particle, to determine if a material transition should take place.

The main advantage of this approach is simply that it provides a uniform way to store and access our material model. It is also potentially provided a clearer way of writing the material transition conditions, particularly for many materials. 

The code for material graph Class can be found in the example notebook here:

Here is the how we would build the graph object, with nodes representing each material's index:

```
#Materials
matname1 = 1
matname2 = 2
matname3 = 3
matname4 = 4

#list of all material indexes
material_list = [1,2,3,4]

#Setup the graph object
DG = MatGraph()

#add all the material types to the graph (i.e add nodes)
DG.add_nodes_from(material_list)
```

We can also visualise the graph...

```
import networkx as nx
nx.draw_circular(DG, with_labels=True)
plt.show()
```



![alt text](graph.png )



Now the graph is set up, here is how we could could add a condition. For instance, given an underworld function that returns the depth, we could specify that _matname1_ transforms to _matname2_ when the depth is less than 0.5:

```
DG.add_transition((matname1,matname2), depthFn, operator.lt, 0.5)
```

The docstring for the add_transition method is provided below:

```
Parameters
----------
nodes : Tuple
    (a,b) represents the possibility of a transition fram material a to material b 
function: underworld.function._function.Function
    (could also be a constant)
nOperator: operator
    operators must be provided in function form through the operator package, eg. operator.gt(1., 2.)
value: float
    the value will be compared to the providided function, given the provided operator
combineby: string
    'and' or 'or', defaults to 'and'. If multiple rules are provided for a single edge in the graph (representing the material transition)
    then they be applied in the sense of any ('or'), or all ('and')
```



## Multiple conditions

Multiple conditions can be attached to a graph edge. For instance, here we can set both depth and temperature constraints on the transition:

```
DG.add_transition((matname1,matname2), depthFn, operator.lt, 0.5)
DG.add_transition((matname1,matname2), temperatureField, operator.gt, 0.5)
```

If multiple rules are provided for a single edge in the graph (representing the material transition)  then they be applied in the sense of any ('or'), or all ('and'). The default is 'and'.

