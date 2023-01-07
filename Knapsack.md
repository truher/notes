For my [javascript project,](https://github.com/enrich/portviz) I needed a solution for the [0-1 knapsack problem,](http://en.wikipedia.org/wiki/Knapsack_problem#0-1_knapsack_problem) to find optimal portfolios.  Not finding one handy, I wrote one.  You can find it, and a unit test, [at this gist.](https://gist.github.com/truher/4715551)  If you're curious about the details, read on.

I liked the Go implementation [here,](http://rosettacode.org/mw/index.php?title=Knapsack_problem/0-1) so I started with something similar.  A few points to note:

### Usage

There are useful examples in the tests, e.g.:

```javascript
  var combiner =
    portviz.knapsack.combiner(allwants,
      function(x){return x.weight;},
      function(x){return x.value;});
  var oneport = combiner.one(400);
  ok(near(oneport.totalValue, 1030, 0.01), "correct total value");
  ok(near(oneport.totalValue, 1030, 0.01), "correct total value");
  equal(oneport.totalWeight, 396, "correct total weight");
```

### Accessor functions

To glue the knapsack to the rest of the app, I use simple accessor functions, rather than requiring the knapsack items to have "value" and "weight" methods, we can pluck whatever field or function we want:

```javascript
    portviz.knapsack.combiner(projects,
      function(x){return x.cost;},
      function(x){return x.benefit;});
```

### Memoization

The recursive method needs memoization to perform efficiently.

A very simple object handles memo storage:

```javascript
    var _memo = (function(){
      var _mem = {};
      var _key = function(i, w) { return i + '::' + w;};
      return {
        get: function(i, w) {return _mem[_key(i,w)];},
        put: function(i, w, r) { _mem[_key(i,w)]=r; return r;}
      };
    })();
```

### Approximation

For our large solution space, performance is an issue, so I implemented the [polynomial time approximation](http://en.wikipedia.org/wiki/Fully_polynomial_time_approximation_scheme) for this problem.

```javascript
    var _epsilon = 0.01;
    var _p = _.max(_.map(items,valuefn));
    var _k = _epsilon * _p / items.length;
```

The value is scaled by K within the calculation:

```javascript
    totalValue: included.totalValue + Math.floor(valuefn(item)/_k)
```

and rescaled on output:

```javascript
return {items: scaled.items,
        totalWeight: scaled.totalWeight,
        totalValue: scaled.totalValue * _k};
```

### Efficient frontier

I added a simple function to calculate the entire efficient frontier.

```javascript
      ef: function(maxweight, step) {
        return _.map(_.range(0, maxweight+1, step), function(weight) {
          var scaled = _m(items.length - 1, weight);
          return {
            items: scaled.items,
            totalWeight: scaled.totalWeight,
            totalValue: scaled.totalValue * _k
          };
        });
      }
```

### Future work

One of the next things to do is to model set-dependent value, i.e. the value of an item depends on the rest of the contents of the knapsack.  I think this is just a matter of changing the value function, but I haven't tried that yet.
