One of [our](http://www.enrichconsulting.com/) clients has a [job-shop problem](http://en.wikipedia.org/wiki/Job-shop_problem), so I played with some solutions
that I could fit into my [javascript project](https://github.com/enrich/portviz), which
is a client-side-only, one-page thing.

The specific need involves N tasks, each of which requires some amount of M resource types.
For playing-around purposes, I set M to 1, so it's just a constant budget.
Each task has a different value, and a different cost.  For now, I represented both
cost and value as a constant, for the duration of the task.  In general, the cost and value
would not be constant, or coincident in time.

You can play with [the thing](https://github.com/enrich/scheduler) if you want.  Install is
simple:

```
git clone https://github.com/enrich/scheduler
cd scheduler
npm install
grunt
firefox dist/sched.html
```

The display shows two schedules: an ROI-sorted greedy packer, and the same greedy packer applied to an annealed tasklist.

Let's look in detail.

First, the dataset:

```javascript
sched.data = 
[
  {
    "cost_per_period": 1,
    "duration": 5,
    "value_per_period": 10,
    "i": 0
  },
...
```

This is the greedy packer:

```javascript
var populateGreedy = function(sorted) {
  var sched_current_schedule = sched.none();
  _.each(sorted, function(s) {
    var t = 0;
    while (!sched.allowed(s, t, sched_current_schedule)) {
      t++;
    }
    sched_current_schedule[s.i] = t;
  });
  return sched_current_schedule;
};

```

We use this function to decide if a task can be scheduled:

```javascript
sched.allowed = function(project, starttime, theschedule) {
  if (starttime < 0) {return false;}
  var thecost = sched.cost(theschedule);
  return _.every(_.range(starttime, project.duration + starttime), function(i) {
    var c = (thecost[i] || 0);
    if (c + project.cost_per_period <= sched.budget) {
      return true;
    } else {
      return false;
    }
  });
};
```

First, we make the simple schedule, with ROI-sorted data, and render it:

```javascript
var sorted = _.sortBy(sched.projects, function(x){return 0 - (x.value_per_period/x.cost_per_period);});
sched.current_schedule = populateGreedy(sorted);
var obj = sched.objective(sched.current_schedule);
sched.render(4, sched.current_schedule); 
sched.renderseries(sched.cost, 'cost4', sched.current_schedule);
sched.renderseries(sched.value, 'val4', sched.current_schedule);
d3.select('div#viz4').append('h3').text(srt.label + ' obj: ' + obj);
```

We plot the schedule as kinda like a Gantt chart, with bar widths showing duration, bar height showing
cost, and bar shade showing value.

```javascript
sched.render = function(addr, schedule) {

  var width = 960;
  var height = 450;
  var bm = 30;
  var tm = 30;
  var lm = 50;
  var rm = 30;
  var innerheight = height - bm - tm;
  var innerwidth = width - rm - lm;

  var minOpacity = 0.2;
  var tt = sched.totalTime(schedule);

  var xscale = d3.scale.linear()
    .domain([0,tt])
    .range([0,innerwidth]);
  var xaxis = d3.svg.axis().scale(xscale).orient('bottom');
  var cumcost = 0;
  var costs = _.map(_.pluck(sched.projects, 'cost_per_period'),
    function(cpp) {cumcost += cpp; return cumcost - cpp;});
  var yscale = d3.scale.ordinal()
    .domain(_.map(sched.projects, function(p,i){return i;}))
    .range(_.map(costs, function(cc) { return cc * innerheight / cumcost; }));
  var yaxis = d3.svg.axis().scale(yscale).orient('left');


  var rd = _.compact(_.map(sched.projects, function (p, i) {
    if (_.isUndefined(schedule[i]) || schedule[i] < 0) {return null;}
    return {
      x: xscale(schedule[i]),
      y: yscale(i),
      width: xscale(schedule[i] + p.duration) - xscale(schedule[i]),
      height: p.cost_per_period * innerheight / cumcost,
      opacity: (1 - minOpacity) * p.value_per_period / sched.maxValue + minOpacity
    };
  }));

  var svg = d3.select('div#viz' + addr).selectAll('svg').data([rd]);
  svg.enter().append('svg');
  svg.exit().remove();
  svg.attr('width',width)
    .attr('height',height)
    .attr('class','myviz');

  var gg = svg.selectAll('g.container').data(function(d){return [d];});
  gg.enter().append('g');
  gg.exit().remove();
  gg.attr('class','container')
    .attr('transform','translate(' + lm + ',' + tm + ')');

  var xs = gg.selectAll('.x.axis').data(['hi']);
  xs.enter().append('g');
  xs.attr('class','x axis').attr('transform','translate(0,'+innerheight+')');
  xs.call(xaxis);
  var ys = gg.selectAll('.y.axis').data(['hi']);
  ys.enter().append('g');
  ys.attr('class','y axis');
  ys.call(yaxis);

  var rect = gg.selectAll('rect.proj').data(function(d){return d;});
  rect.enter().append('rect')
    .attr('class','proj')
    .attr('x', function(d){return d.x;})
    .attr('y', function(d){return d.y;})
    .attr('width', function(d){return d.width;})
    .attr('height', function(d){return d.height;})
    .attr('opacity', function(d){return d.opacity;});
  rect.exit().remove();
  rect.transition().duration(50)
    .attr('x', function(d){return d.x;})
    .attr('y', function(d){return d.y;})
    .attr('width', function(d){return d.width;})
    .attr('height', function(d){return d.height;})
    .attr('opacity', function(d){return d.opacity;});

};
```

We plot the total cost and total value over time:

```javascript
sched.cost = function(theschedule) {
  return _.reduce(sched.projects, function(memo, project, index) {
    var start = theschedule[index];
    if (start < 0) { return memo; }
    var end = start + project.duration;
    _.each(_.range(start, end), function(i) {
      memo[i] = (memo[i] || 0) + project.cost_per_period;
    });
    return memo;
  }, []);
};

sched.value = function(theschedule) {
  return _.reduce(sched.projects, function(memo, project, index) {
    var start = theschedule[index];
    if (start < 0) { return memo; }
    var end = start + project.duration;
    _.each(_.range(start, end), function(i) {
      memo[i] = (memo[i] || 0) + project.value_per_period;
    });
    return memo;
  }, []);
};
```

This is the rendering code.

```javascript
sched.renderseries = function(fn, addr, schedule) {
  var width = 960;
  var height = 150;
  var bm = 30;
  var tm = 30;
  var lm = 50;
  var rm = 30;
  var innerheight = height - bm - tm;
  var innerwidth = width - rm - lm;

  var c = fn(schedule);
  _.each(_.range(c.length), function(i) {
    if (_.isUndefined(c[i])) {c[i] = 0;}
  });

  var tt = sched.totalTime(schedule);

  var xscale = d3.scale.linear()
    .domain([0,c.length])
    .range([0,innerwidth]);
  var xaxis = d3.svg.axis().scale(xscale).orient('bottom');

  var yscale = d3.scale.linear()
    .domain([0, _.max(c)])
    .range([innerheight, 0]);
  var yaxis = d3.svg.axis().scale(yscale).orient('left');


  var rd = _.map(c, function (ci, i) {
    return {
      x: xscale(i),
      y: yscale(c[i]),
      width: xscale(1) - xscale(0),
      height: yscale(0) - yscale(c[i])
    };
  });

  var svg = d3.select('div#' + addr).selectAll('svg').data([rd]);
  svg.enter().append('svg');
  svg.exit().remove();
  svg.attr('width',width)
    .attr('height',height)
    .attr('class','mycost');

  var gg = svg.selectAll('g.container').data(function(d){return [d];});
  gg.enter().append('g');
  gg.exit().remove();
  gg.attr('class','container')
    .attr('transform','translate(' + lm + ',' + tm + ')');

  var xs = gg.selectAll('.x.axis').data(['hi']);
  xs.enter().append('g');
  xs.attr('class','x axis').attr('transform','translate(0,'+innerheight+')');
  xs.call(xaxis);
  var ys = gg.selectAll('.y.axis').data(['hi']);
  ys.enter().append('g');
  ys.attr('class','y axis');
  ys.call(yaxis);

  var rect = gg.selectAll('rect.proj').data(function(d){return d;});
  rect.enter().append('rect')
    .attr('class','proj')
    .attr('x', function(d){return d.x;})
    .attr('y', function(d){return d.y;})
    .attr('width', function(d){return d.width;})
    .attr('height', function(d){return d.height;});
  rect.exit().remove();
  rect.transition().duration(50)
    .attr('x', function(d){return d.x;})
    .attr('y', function(d){return d.y;})
    .attr('width', function(d){return d.width;})
    .attr('height', function(d){return d.height;});

};
```

Next, we run the greedy packer inside a simulated-annealing loop, and do the same renderings.

This is the objective function.  It just discounts the value.

```javascript
sched.objective = function(theschedule) {
  return _.reduce(sched.projects, function(memo, project, index) {
    var start = theschedule[index];
    if (start < 0) { return memo; }
    var end = start + project.duration;
    return _.reduce(_.range(start, end), function(memo2, i) {
      var d = Math.pow((1 - sched.discount), i);
      return memo2 + project.value_per_period * d;
    }, memo);
  }, 0);
};
```

This is the annealer.

```javascript
var newlist = _.sortBy(sched.projects, function(x){return 0 - (x.value_per_period/x.cost_per_period);});
var popsched = populateGreedy(newlist);
var popobj = sched.objective(popsched);
var temp = 0.5;

// attainable objectives:
//     0 iterations => 638
//   500 iterations => 641
//  5000 iterations => 647
// 20000 iterations => 649
_.each(_.range(500), function(i) {
  // in each iteration, swap two items
  var s1 = _.random(newlist.length - 1);
  var s2 = _.random(newlist.length - 1);
  // make a copy of the schedule
  var maybe = newlist.slice();
  var t = maybe[s1];
  maybe[s1] = maybe[s2];
  maybe[s2] = t;
  var mp = populateGreedy(maybe);
  var mo = sched.objective(mp);
  if (mo > popobj) {
    popobj = mo;
    newlist = maybe;
    popsched = mp;
    console.log('iter ' + i + ' t ' + temp + ' new better ' + popobj);
  } else if (Math.abs(mo - popobj) < 0.0001) {
    // no change
  } else if (Math.exp((mo - popobj) / temp) > Math.random())  {
    popobj = mo;
    newlist = maybe;
    popsched = mp;
    console.log('iter ' + i + ' t ' + temp + ' allow worse ' + popobj);
  } else {
    //console.log('no change');
  }
  temp *= 0.9995;
});
```

That's it!  I thought it was interesting that the ROI packed thing worked as well as it did;
I was also surprised that the annealer performed as poorly as it did.

The realistic case is quite a bit different, so I'll be looking at that next.