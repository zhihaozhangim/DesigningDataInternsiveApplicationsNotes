# Data Models and Query Languages

#### Imperative Language vs Declarative Language

- An imperative language tells the computer to perform certain operations in a certain order. You can imagine stepping through the code line by line, evaluating conditions, updating variables, and deciding whether to go around the loop one more time. For example, if you have a list of animal species, you might write something like this to return only the sharks in the list:

  ```javascript
  function getSharks() { 
    var sharks = [];
  	for (var i = 0; i < animals.length; i++) { 
      if (animals[i].family === "Sharks") {
  			sharks.push(animals[i]); }
  		}
  		return sharks; 
  }
  ```

- In a declarative query language, like SQL or relational algebra, you just specify the pattern of the data you want—what conditions the results must meet, and how you want the data to be transformed (e.g., sorted, grouped, and aggregated)—but not *how* to achieve that goal. It is up to the database system’s query optimizer to decide which indexes and which join methods to use, and in which order to execute various parts of the query.

  ```sql
  SELECT * FROM animals WHERE family = 'Sharks';
  ```



#### Summary

Historically, data started out being represented as one big tree (the hierarchical model), but that wasn’t good for representing many-to-many relationships, so the relational model was invented to solve that problem. More recently, developers found that some applications don’t fit well in the relational model either. New nonrelational “NoSQL” datastores have diverged in two main directions:

1. *Document databases* target use cases where data comes in self-contained docu‐ ments and relationships between one document and another are rare.
2. *Graph databases* go in the opposite direction, targeting use cases where anything is potentially related to everything.



