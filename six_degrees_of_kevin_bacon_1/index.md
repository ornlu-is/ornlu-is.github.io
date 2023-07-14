# Six Degrees of Kevin Bacon: Introduction


Today I found out that there is parlor game named "Six Degrees of Kevin Bacon" and its rules are very simple: a player picks an actor and other players have to connect that actor to another actor via a film both starred in. This process is then repeated until Kevin Bacon is reached. The name of this game is a reference to the concept of "six degrees of separation" where, in the world's social network, every person is, at most, six acquaintance links apart from any other person in the world.

## Networks

So what is a network and how can we visualise it? In this context, a network is a set of actors, people, or things that are connected to each other via a relation. For example, in a social network, you are connected to your parents via a parental relation and connected to your friends via a friendship relation. Similarly, two computers can also form a network where they are connected via the Internet. Networks are a wonderful concept and they are absolutely everywhere.

Networks are represented as graphs, which are a set of nodes and edges. We can visualise these in the following way:

{{< figure src="/images/kevin_bacon_example_graph.png" title="Kevin Bacon and his parents as a graph" >}}

Hopefully, the above picture made it abundantly clear what a graph, nodes and edges are. In our graph we have three nodes, $|N|=3$, Kevin Bacon and his parents, and two edges, $|E|=2$, which connect Kevin Bacon to his parents via a parental relation. Mathematically, we characterise a graph, $G$, by its set of nodes, $N$, and edges, $E$, as follows:

$$G(N, E),$$

where

$$N = \\{ \text{Edmund Bacon}, \text{Kevin Bacon}, \text{Ruth Hilda Bacon} \\},$$

$$E = \\{ \\{ \text{Edmund Bacon} , \text{Kevin Bacon} \\}, \\{ \text{Kevin Bacon}, \text{Ruth Hilda Bacon} \\} \\}.$$

Technically, the above graph should be a *directed* graph, in the sense that edges have a given direction: Edmund Bacon is Kevin Bacon's father, but Kevin Bacon is not Edmund Bacon's father, such shenanigans are left to time travel movies. However, for simplicity's sake, I decided to represent the graph as an *undirected* graph, where edges represent bidirectional relations. Additionally, it is also an *unweighted* graph, meaning that all edges have the same weight, because I am assuming Kevin Bacon loves both his parents equally. 

## Degree and Distance in Networks

In a network, any node, $i$, is associated with a quantity known as the degree, $k_i$, which simply represents the number of edges a node has. If we associate Kevin Bacon with $i=1$, and is mother and father with, respectively, $i=2$ and $i=3$, we have that:
* $k_1 = 2$;
* $k_2 = 1$;
* $k_3 = 1$.

From a network's degrees, we can calculate its average degree, a quantity that plays a central role in network analysis. This is given by the following formula:

$$ \langle k \rangle = \frac{1}{|N|} \sum_{i=1}^{|N|} k_i = \frac{2|E|}{|N|}. $$

While the concept of a node's degree is very intuitive, the concept of distance in a network is a tougher concept to handle, simply because it doesn't necessarily translate nicely into reality. In networks, the distance between two nodes depends on the path that connects them. This path is simply the set of edges that we must transverse to go from one node to another, and, the number of edges transversed is the path length. Effectively, we equate the path length to the distance between two nodes. 

However, in most graphs, there isn't a single path that connects two nodes together. As such, it is more natural to refer to the shortest path, *i.e.*, the path that consists of the minimal set of edges. Derived from this, another interesting quantity in networks is the average path length, $\langle d \rangle$, which is simply the average of the shortest paths between all pairs of nodes.

## Small World Networks

In 1967, an American social psychologist named Stanley Milgram, designed an experiment to measure distances in social networks. He began by selecting a stock broker from Boston and a divinity student from Sharon (Massachussetts) as target nodes. Simultaneously, he also randomly selected residents of Wichita and Omaha, which would act as starting nodes. These residents were sent a letter by Milgram explaining the study's objective, a photograph, the name, address and some more information about their target person. These letters also included instructions for these people to forward the letter to the person they knew and thought would most likely know the target person.

While many of the letters did not make it back, those that did led allowed Milgram to calculate the median number of people required to reach the target. Surprisingly, the median number of intermediate acquaintances was 5, which is where the expression "six degrees of separation" and the concept of small world phenomenon stems from. 

In network terms, this just means that the average distance between two nodes in a network is short. More precisely, it is short enough that the approximation:

$$\langle d \rangle \approx \frac{\ln N}{ \ln \langle k \rangle },$$

is reasonably close. 

## The Goal of this Project

I have a very straightforward goal: to verify if, and if so, how many, actors are more than six degrees apart from Kevin Bacon. This project is deceptively hard due to the data required to achieve it. The first step is to scrape movie data off the Internet (without DDoS'ing anyone) and store it efficiently. Then, this data has to be cleaned to avoid having any malformed data, *e.g.*, duplicated entries, skewing the results. Finally, the entire graph has to be computed and the distance to Kevin Bacon must be calculated for every single node. Additionally, I also want to verify if the small world property holds true for the movie actors network and, in case it does not, when, in time, does it break down.

## Trivia

Stanley Milgram is mostly known for authoring another *very* controversial experiment, which is known as the Stanley Milgram Shock Experiment.

## References

Barabasi's book
* Albert-László Barabási, *Network Science*, 2018
* Kevin Bacon's Wikipedia page: https://en.wikipedia.org/wiki/Kevin_Bacon

