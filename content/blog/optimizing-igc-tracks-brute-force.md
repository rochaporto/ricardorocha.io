+++
title = "Optimizing IGC Flight Tracks, Part 1: Brute Force"
description = "Using brute force, montecarlo and genetic algorithms to calculate gliding distance"
tags = ["IGC", "Gliding", "Brute Force", "Optimization"]
series = [ "Optimizing IGC Flight Tracks" ]
draft = true
date = "2018-11-12T01:00:11+02:00"
author = "Ricardo Rocha"
+++

The [IGC (International Gliding Comission) format](http://igc.com) is widely used by gliding and paragliding flight recorders, analysis and visualization software. It captures tracks as a sequence of timed GPS coordinates, along with extra flight metadata. Here's an example flight track overlayed on a map:
{{% figure src="../../images/post/flight-track-map.png"
      class="figcaption"
      caption="Track of gliding flight over the French and Swiss Alps"
%}}

Calculating the total flight distance is a trivial task, we just sum the great circle distance between every two consecutive points. But such a number is of limited interest as it doesn't take into account the area covered by a flight, which is where the challenge lies. For this reason pilots, whether comparing performance of their leisure flights in online contests or during official competitions, prefer to calculate flight distance considering a number of turnpoints, with some types having specific names.

* **Out and Return**: 1 turnpoint
* **Triangle**: 2 turnpoints
* **FAI Triangle**: 2 turnpoints, with the shortest side being at least 28% of the total
* **3, 4 or more turnpoints**: no specific names

{{% figure src="../../images/story/vaga-wavecamp.jpg"
      class="figcaption"
      caption="French glider in Norway, on the way up to 21000 feet ( -40 °C )"
%}}

This calculation is an optimization problem where we try to find the maximum for a certain score function considering the restrictions above.

A possible implementation is to evaluate every possible combination, which guarantees an optimal result but is O(n^m) - with *n* being the number of recorded GPS points and *m* the number of desired turnpoints. Long flights often record several thousand GPS coordinates, so this naive approach quickly becomes prohibitive for even a low number of turnpoints.

In the first part of the series we look at at this approach and how we can use the excellent Go tools to try to optimize our code. The next entries will look how domain knowledge can help simplify the task, and cover faster and more efficient approximative solutions such as shortest path, simulated annealing and genetic algorithms, but where a global optimum is not always guaranteed.

Depending on available time to experiment with other algorithms, this series risks to be a trilogy extended to many more parts.

### Brute Force

Here's what a loop over all possible combinations for 1 turnpoint could look like:
{{< highlight go "hl_lines=2 3" >}}
var optimalDistance float64
var distance float64
var task Task
var optimalTask Task

for i := 0; i < len(track.Points)-2; i++ {
	for j := i + 1; j < len(track.Points)-1; j++ {
		for z := j + 1; z < len(track.Points); z++ {
			task = Task{
				Start:      track.Points[i],
				Turnpoints: []Point{track.Points[j]},
				Finish:     track.Points[z],
			}
			distance = task.Distance()
			if distance > optimalDistance {
				optimalDistance = distance
				optimalTask = Task(task)
			}
		}
	}
}
return optimalTask, nil
{{< /highlight >}}

The results are disappointing, given we're calculating a 1 turnpoint maximum for a flight with around 1000 points:
{{< highlight go >}}
go test -bench=BenchmarkBruteForceOptimize -run=x -cpuprofile=cpu.pprof
PASS
ok  	github.com/ezgliding/goigc	37.609s
{{< /highlight >}}

You've probably already noticed a couple obvious issues with the code above, but let's look at the pprof output for hints:
{{< highlight go >}}
go tool pprof -cum -text cpu.pprof
37.04s of 39.30s total (94.25%)
Dropped 181 nodes (cum <= 0.20s)
      flat  flat%   sum%        cum   cum%
         0     0%     0%     37.85s 96.31%  runtime.goexit
         0     0%     0%     31.09s 79.11%  testing.tRunner
         0     0%     0%     30.54s 77.71%  github.com/ezgliding/goigc.(*bruteForceOptimizer).Optimize
     0.37s  0.94%  0.94%     30.54s 77.71%  github.com/ezgliding/goigc.(*bruteForceOptimizer).optimize1
         0     0%  0.94%     30.54s 77.71%  github.com/ezgliding/goigc.TestBruteForceOptimize.func1
     0.74s  1.88%  2.82%     26.70s 67.94%  github.com/ezgliding/goigc.(*Task).Distance
     2.17s  5.52%  8.35%     14.47s 36.82%  runtime.growslice
     2.53s  6.44% 14.78%     13.94s 35.47%  runtime.mallocgc
     0.46s  1.17% 15.95%     10.31s 26.23%  github.com/ezgliding/goigc.(*Point).Distance
     2.28s  5.80% 21.76%      9.85s 25.06%  github.com/ezgliding/goigc/vendor/github.com/golang/geo/s2.LatLng.Distance
{{< /highlight >}}

We would expect this process to be dominated by math calculations for the distances, but this doesn't seem to be the case. Between the Task and Point distance computation we see ~70% of the time spent in memory allocation and slices. A quick look at the Task.Distance() function shows the issue:
{{< highlight go "hl_lines=2 3" >}}
func (task *Task) Distance() float64 {
	d := 0.0
	p := []Point{task.Start}
	p = append(p, task.Turnpoints...)
	p = append(p, task.Finish)
	for i := 0; i < len(p)-1; i++ {
		d += p[i].Distance(p[i+1])
	}
	return d
}
{{< /highlight >}}

We could hardly do any worse than allocating a single item array and extending it for each item, in a function that is called on every loop. This makes it a bit better:
{{< highlight go "hl_lines=2 3" >}}
func (task *Task) Distance() float64 {
	d := 0.0
	d += task.Start.Distance(task.Turnpoints[0])
	for i := 0; i < len(task.Turnpoints)-1; i++ {
		d += task.Turnpoints[i].Distance(task.Turnpoints[i+1])
	}
	d += task.Turnpoints[len(task.Turnpoints)-1].Distance(task.Finish)
	return d
}
{{< /highlight >}}

{{< highlight bash >}}
go test -bench=BenchmarkBruteForceOptimize-run=x -cpuprofile=cpu.pprof
PASS
ok  	github.com/ezgliding/goigc	19.902s
{{< /highlight >}}

Almost 2 times faster, but another look and it seems we still have memory allocation issues in the main loop (though the math functions start showing up on top, which is good):
{{< highlight go >}}
go tool pprof -cum -text cpu.pprof 
15.11s of 16.14s total (93.62%)
Dropped 158 nodes (cum <= 0.08s)
      flat  flat%   sum%        cum   cum%
         0     0%     0%     15.96s 98.88%  runtime.goexit
         0     0%     0%     14.62s 90.58%  testing.tRunner
         0     0%     0%     14.08s 87.24%  github.com/ezgliding/goigc.(*bruteForceOptimizer).Optimize
     0.38s  2.35%  2.35%     14.08s 87.24%  github.com/ezgliding/goigc.(*bruteForceOptimizer).optimize1
         0     0%  2.35%     14.08s 87.24%  github.com/ezgliding/goigc.TestBruteForceOptimize.func1
     0.30s  1.86%  4.21%     10.67s 66.11%  github.com/ezgliding/goigc.(*Task).Distance
     0.38s  2.35%  6.57%     10.15s 62.89%  github.com/ezgliding/goigc.(*Point).Distance
     1.82s 11.28% 17.84%      9.77s 60.53%  github.com/ezgliding/goigc/vendor/github.com/golang/geo/s2.LatLng.Distance
     0.70s  4.34% 22.18%      2.98s 18.46%  math.atan2
     0.31s  1.92% 24.10%      2.65s 16.42%  runtime.newobject
     2.53s 15.68% 39.78%      2.53s 15.68%  math.sin
     0.82s  5.08% 44.86%      2.39s 14.81%  runtime.mallocgc
     0.41s  2.54% 47.40%      2.28s 14.13%  math.atan
        2s 12.39% 59.79%         2s 12.39%  math.cos
     1.87s 11.59% 71.38%      1.87s 11.59%  math.satan
{{< /highlight >}}

Again, you probably noticed we're still creating a new Task on every loop. Avoiding useless allocations inside a loop always gives better results, we're now down to (showing the inner loop only):
{{< highlight go "hl_lines=2 3" >}}
		for z := j + 1; z < len(track.Points); z++ {
			task.Start = track.Points[i]
			task.Turnpoints[0] = track.Points[j]
			task.Finish = track.Points[z]
			distance = task.Distance()
			if distance > optimalDistance {
				optimalDistance = distance
				optimalTask = Task(task)
			}
		}
{{< /highlight >}}

{{< highlight go "hl_lines=2 3" >}}
go tool pprof -cum -text cpu.pprof 
10.87s of 11.43s total (95.10%)
Dropped 93 nodes (cum <= 0.06s)
      flat  flat%   sum%        cum   cum%
         0     0%     0%     11.42s 99.91%  runtime.goexit
         0     0%     0%     11.38s 99.56%  testing.tRunner
         0     0%     0%     10.84s 94.84%  github.com/ezgliding/goigc.(*bruteForceOptimizer).Optimize
     0.33s  2.89%  2.89%     10.84s 94.84%  github.com/ezgliding/goigc.(*bruteForceOptimizer).optimize1
         0     0%  2.89%     10.84s 94.84%  github.com/ezgliding/goigc.TestBruteForceOptimize.func1
     0.34s  2.97%  5.86%     10.16s 88.89%  github.com/ezgliding/goigc.(*Task).Distance
     0.35s  3.06%  8.92%      9.63s 84.25%  github.com/ezgliding/goigc.(*Point).Distance
     1.93s 16.89% 25.81%      9.28s 81.19%  github.com/ezgliding/goigc/vendor/github.com/golang/geo/s2.LatLng.Distance
     0.81s  7.09% 32.90%      3.27s 28.61%  math.atan2
     0.41s  3.59% 36.48%      2.46s 21.52%  math.atan
     2.05s 17.94% 54.42%      2.05s 17.94%  math.satan
     1.89s 16.54% 70.95%      1.89s 16.54%  math.cos
     1.82s 15.92% 86.88%      1.82s 15.92%  math.sin
{{< /highlight >}}

Most of the time is now spent on math operations.

#### Concurrency

If you've used Go you've read about concurrency by now. The language has great
primitives to simplify the task, and after the first simple code changes we
can try to execute the different steps concurrently and hope for a speed up.

Here's the sample code. We create a new go routine for each loop element, and
feed the results into a channel. These are later collected in a separate loop,
and the optimal result is calculated.

{{< highlight go "hl_lines=2 3" >}}
	d := make(chan float64, len(track.Points)-2)
	t := make(chan Task, len(track.Points)-2)

	for i := 0; i < len(track.Points)-2; i++ {
		go func(i int) {
			var optimalDistance float64
			var distance float64
			var task Task
			var optimalTask Task

			task.Turnpoints = []Point{Point{}}
			optimalTask.Turnpoints = []Point{Point{}}
			for j := i + 1; j < len(track.Points)-1; j++ {
				for z := j + 1; z < len(track.Points); z++ {
					task.Start = track.Points[i]
					task.Turnpoints[0] = track.Points[j]
					task.Finish = track.Points[z]
					distance = task.Distance()
					if distance > optimalDistance {
						optimalDistance = distance
						optimalTask.Start = track.Points[i]
						optimalTask.Turnpoints[0] = track.Points[j]
						optimalTask.Finish = track.Points[z]
					}
				}
			}
			d <- optimalDistance
			t <- optimalTask
		}(i)
	}

	for i := 0; i < len(track.Points)-2; i++ {
		distance = <-d
		task = <-t
		if distance > optimalDistance {
			optimalDistance = distance
			optimalTask = task
		}
	}
{{< /highlight >}}

How does it look?
{{< highlight go "hl_lines=2 3" >}}
aaaa
{{< /highlight >}}

The result is terrible, and it can hang the box pretty quickly. The reason for
this is the number of go routines we're launching - too many results in too
much context switching and little work done. But how many go routines should we be
launching? We can look at what sounds reasonable, but to debug this sort of
cases Go provides a tool called tracer, and we can instruct our tests to
generate a trace:
{{< highlight go "hl_lines=2 3" >}}

{{< /highlight >}}

{{% figure src="../../images/post/bruteforce_optimize_o3.png"
      class="figcaption"
%}}

This visualization is clearly telling us our cores are often idle, again
a sign we might be doing too much context switching. We can reduce the number
of go routines by moving the routine launch up in the nested loops:
{{< highlight go "hl_lines=2 3" >}}

{{< /highlight >}}

Here's the new trace visualization, and the matching benchmark results:
{{% figure src="../../images/post/bruteforce_optimize_o4.png"
      class="figcaption"
%}}

This looks much better.

{{< highlight go "hl_lines=2 3" >}}
go test -bench=BenchmarkBruteForceOptimizeO* -run=x .
goos: linux
goarch: amd64
pkg: github.com/ezgliding/goigc
BenchmarkBruteForceOptimizeO0/optimize-short-flight-1/1-4         	       1	30179925082 ns/op
BenchmarkBruteForceOptimizeO1/optimize-short-flight-1/1-4         	       1	13795767469 ns/op
BenchmarkBruteForceOptimizeO2/optimize-short-flight-1/1-4         	       1	10700455677 ns/op
BenchmarkBruteForceOptimizeO3/optimize-short-flight-1/1-4         	       1	4684287373 ns/op
BenchmarkBruteForceOptimizeO4/optimize-short-flight-1/1-4         	       1	3767170974 ns/op
{{< /highlight >}}
```

Further optimizations can be done by reducing the number of calculations, and there are a few options available:

* Cache distances between points, as the loop often recomputes values for the same two points
* *Clean* the flight at the start, removing any redundant points (overlapping or too close to each other)
* Stop a loop once the points left cannot give a better solution than the current max (more complex, we'll cover it below)

#### Caching

We've seen that most time in now spent on math calculations, which are quite
expensive. The next step is then to reduce their number, and caching is one way
of doing it. Here's the code:

{{< highlight go "hl_lines=2 3" >}}
func (b *bruteForceOptimizer) optimize1(track Track, score Score) (Task, error) {

	var optimalDistance float64
	var distance float64
	var optimalTask Task

	cache := make([][]float64, len(track.Points))
	for i := range track.Points {
		cache[i] = make([]float64, len(track.Points))
	}
	var d1, d2 float64
	for i := 0; i < len(track.Points)-2; i++ {
		for j := i + 1; j < len(track.Points)-1; j++ {
			for z := j + 1; z < len(track.Points); z++ {
				if cache[i][j] != 0 {
					d1 = cache[i][j]
				} else {
					d1 = track.Points[i].Distance(track.Points[j])
					cache[i][j] = d1
				}
				if cache[j][z] != 0 {
					d2 = cache[j][z]
				} else {
					d2 = track.Points[j].Distance(track.Points[z])
					cache[j][z] = d2
				}
				distance = d1 + d2
				if distance > optimalDistance {
					optimalDistance = distance
					optimalTask = Task{Start: track.Points[i],
						Turnpoints: []Point{track.Points[j]}, Finish: track.Points[z]}
				}
			}
		}
	}
	return optimalTask, nil
}
{{< /highlight >}}


{{< highlight go "hl_lines=2 3" >}}
go test -bench=BenchmarkBruteForceOptimize-run=x -cpuprofile=cpu.pprof
PASS
ok  	github.com/ezgliding/goigc	5.743s
{{< /highlight >}}

We're now at a 7x improvement compared to our first attempt.

* + Always shows the global optimum
* - Very slow for long flights and high number of turnpoints
* - Potential optimizations make the code harder to read

Later we'll dive into optimization algorithms (montecarlo, genetic, and funny
nature inspired ones). But next we look at how we can further improve by using
domain knowledge to simplify the problem, also considering [the fastest code is
the code that never
runs](http://www.ilikebigbits.com/2015_12_06_gauntlet.html).

If you're interested in the implementation details, want to contribute to improving the optimization algorithms, or simply use the software, check the [goigc project](https://github.com/ezgliding/goigc) homepage.
