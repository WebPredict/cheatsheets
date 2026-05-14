**The Pattern-First Guide**

**to FAANG Algorithmic Interviews**

*A decision-tree approach to recognizing which technique to reach for*

JavaScript examples throughout

**How to use this guide**

This guide assumes you can already write code, design systems, and
reason about performance. What it focuses on instead is the specific
skill that algorithmic interviews actually test: pattern recognition
under time pressure.

The structure of every section is the same. First, the underlying
mechanic --- what the pattern is actually doing and what invariant it
maintains. Then, the recognition signals --- what features of a problem
statement should make you reach for this tool. Then, a worked example in
JavaScript with commentary. Finally, the variations you should expect to
see.

Two things to keep in mind:

- **Patterns are not algorithms.** \"Sliding window\" is not an
  algorithm; it is a way of structuring a loop so that work done on one
  iteration is reused on the next. The algorithm changes with the
  problem; the structural idea does not.

- **Recognition beats recall.** If you can map a problem to a pattern in
  the first two minutes, the rest is largely mechanical. If you cannot,
  no amount of code fluency will save you. Most of the value in this
  guide is in the recognition signals, not the code.

The decision tree in Chapter 2 is the single most valuable artifact
here. It captures, in flowchart form, the routing logic that experienced
interviewers and interviewees run in their heads. Internalize it before
anything else.

+-----------------------------------------------------------------------+
| **A note on scope**                                                   |
|                                                                       |
| This guide deliberately omits topics that show up in less than \~5%   |
| of FAANG loops at the SWE level: max flow, suffix arrays, KMP,        |
| advanced number theory, computational geometry. If you encounter one  |
| of these, the round was designed against you and there is little you  |
| can do; spend your preparation time on what actually appears.         |
+-----------------------------------------------------------------------+

**Chapter 1 --- The foundations you can\'t skip**

You asked for this section to be brief, and it will be. The goal is not
to teach data structures from scratch but to lock in the operational
characteristics you need at your fingertips during an interview. If you
cannot recite these from memory, no pattern in the rest of the guide
will help you, because every pattern is built on top of them.

**Big-O at a glance**

Interviewers care less about derivation than about whether you can state
and defend the complexity of your solution. The numbers that come up
repeatedly:

- O(1) --- hash map lookup, array index, stack/queue push/pop

- O(log n) --- binary search, balanced tree operations, heap push/pop

- O(n) --- single loop, single tree/graph traversal

- O(n log n) --- sorting (the default cost of any sort-based approach)

- O(n²) --- nested loops, naive pair generation

- O(2ⁿ) and O(n!) --- subset and permutation enumeration

A rough constraint heuristic for problem inputs:

- n ≤ 10 → O(n!) or O(2ⁿ) is fine; expect backtracking

- n ≤ 20 → bitmask DP territory

- n ≤ 5,000 → O(n²) acceptable

- n ≤ 10⁵ → need O(n log n) or O(n)

- n ≤ 10⁷ → must be O(n) or O(log n)

**Data structures and their operational profiles**

**Arrays**

Random access in O(1), insertion/deletion at the end in amortized O(1),
insertion/deletion in the middle in O(n). The workhorse. In JavaScript,
arrays are not contiguous-memory arrays under the hood --- they behave
more like hash-backed sparse arrays --- but for interview complexity
analysis you treat them as classical arrays.

**Hash maps and sets**

O(1) average lookup, insertion, and deletion. The single most over-used
data structure in interviews, and that is appropriate --- converting an
O(n²) problem to O(n) almost always involves trading space for time via
a hash map. In JS, use Map and Set rather than plain objects: Map
preserves insertion order, allows any key type, and gives you a real
.size.

**Linked lists**

O(1) insertion/deletion given a node reference, O(n) lookup. Almost
never the right choice for solving the underlying problem --- but a
category of interview questions specifically tests linked-list pointer
manipulation (reverse a list, detect a cycle, merge sorted lists). Know
how to traverse with a dummy head node and how to reverse in place with
three pointers (prev, curr, next).

**Stacks and queues**

Stacks: LIFO, O(1) push/pop. Queues: FIFO, O(1) enqueue/dequeue. In JS,
a plain array works as a stack via push/pop. For a queue, array.shift()
is O(n) --- for any non-trivial input size, implement a queue with two
stacks, or use a deque (often a circular buffer). The need for a queue
shows up in BFS; do not lose points by using shift().

**Heaps (priority queues)**

O(log n) push and pop, O(1) peek. Use when you need repeated access to
the min or max of a changing set. JavaScript has no built-in heap, which
is annoying --- be prepared to either implement a small binary heap from
scratch in 20--30 lines, or to verbalize \"in a language with a built-in
priority queue, I would use one here\" and proceed at the algorithm
level. Most FAANG interviewers will accept the latter, but having an
implementation memorized is safer.

**Trees**

Binary trees: each node has up to two children. Binary search trees:
left subtree \< node \< right subtree, which gives O(log n) search on a
balanced tree and O(n) on a degenerate one. Most tree problems in
interviews are on plain binary trees, not BSTs. The three traversal
orders (pre-, in-, post-order) and BFS-by-level are non-negotiable; you
should be able to write any of them iteratively or recursively without
thinking.

**Tries**

A tree where each edge is a character. O(k) lookup where k is the length
of the key, independent of the number of stored strings. Use when the
problem involves prefixes --- autocomplete, longest-common-prefix, word
search in a grid. The implementation is a single class with a children
map and an isEnd flag.

**Graphs**

Adjacency list is the right representation in 95% of interviews: a Map
where each key is a node and each value is an array (or Set) of
neighbors. Adjacency matrix only makes sense for very dense graphs or
when you need O(1) edge lookup. Directed vs undirected, weighted vs
unweighted, cyclic vs acyclic --- these distinctions drive technique
choice and should be the first questions you ask the interviewer.

**Union-Find (Disjoint Set Union)**

Two operations: find(x) returns the representative of x\'s set; union(x,
y) merges the sets containing x and y. With path compression and union
by rank, both are effectively O(1) (technically inverse Ackermann). Use
for connected-components and cycle-detection problems on undirected
graphs, and for problems involving incremental merging of equivalence
classes.

+-----------------------------------------------------------------------+
| **The under-appreciated structures**                                  |
|                                                                       |
| Monotonic stacks and monotonic deques are not separate data           |
| structures --- they are arrays/deques used in a particular way (only  |
| push if the new element maintains a monotonic order, popping          |
| violators first). They show up in next-greater-element problems and   |
| sliding-window-maximum. Treat them as a pattern, not a structure.     |
| Covered in Chapter 6.                                                 |
+-----------------------------------------------------------------------+

**Chapter 2 --- The decision tree**

Read this chapter twice. The decision tree below is the routing logic
experienced interviewers use to map a problem statement to a technique.
Most interview failures are not failures of execution; they are failures
of routing --- the candidate picks the wrong tool, spends 20 minutes on
it, and runs out of time.

The tree is organized as a series of questions you ask about the problem
statement, in priority order. Stop at the first one that fires.

**Question 1: Is the input sorted, or can it be sorted cheaply?**

If yes, two techniques become available that are not otherwise:

- **Binary search** --- if you are looking for a specific element, the
  position where an element belongs, or the boundary between two
  regions.

- **Two pointers** --- if you are looking for a pair, a triplet, or a
  contiguous region with a property that depends on order.

Sorting costs O(n log n). If the problem requires better than that and
the input is unsorted, sorting is not an option --- you must use a hash
map or a different structure.

**Question 2: Is the problem about contiguous subarrays or substrings?**

If yes, sliding window is almost always the answer. The keyword is
contiguous --- \"longest substring with at most k distinct characters,\"
\"minimum window containing all letters,\" \"maximum sum subarray of
size k.\" If the problem is about subsequences (non-contiguous), it is
not sliding window; it is probably DP.

**Question 3: Are you traversing a tree or graph?**

BFS vs DFS is the next routing question:

- **BFS** --- shortest path in an unweighted graph, level-order
  traversal, problems where you need to process nodes in order of
  distance from the source.

- **DFS** --- any-path problems, cycle detection, topological sort,
  problems where you need to explore deeply before backtracking,
  problems on trees where you need bottom-up information.

- **Dijkstra** --- shortest path in a weighted graph with non-negative
  weights. BFS with a priority queue instead of a queue.

- **Topological sort** --- anything involving dependencies,
  prerequisites, or ordering with constraints (\"can you finish all
  courses?\", \"build order\").

- **Union-Find** --- connected components, cycle detection in an
  undirected graph, problems where you incrementally merge groups.

**Question 4: Is the problem asking for the k-th something, or for a
running min/max from a stream?**

Heap. K-th largest, k-th smallest, top-k frequent, median of a stream
(two heaps), merge k sorted lists. Whenever you would otherwise sort but
only need partial information, ask whether a heap would do.

**Question 5: Does the problem involve choices, with overlap between
sub-problems?**

This is the DP question. The signals are:

- Optimization (min, max, count, longest, shortest)

- Sequential or grid-based input with a notion of \"so far\"

- Naive recursion would explore the same subproblem multiple times

If the problem asks you to enumerate all valid configurations (rather
than just count or optimize), it is backtracking, not DP. The line is:
backtracking yields the configurations themselves; DP yields a number
(or a single optimal configuration reconstructed from a table).

**Question 6: Are you generating combinations, permutations, or
subsets?**

Backtracking. Specifically: build a partial solution, recurse, undo the
choice, try the next option. The complexity is exponential by
definition, and that is fine --- these problems have small input sizes
(n ≤ 15 typically).

**Question 7: Are you looking for the \"next greater\" or \"previous
smaller\" element?**

Monotonic stack. The pattern: maintain a stack where elements are kept
in monotonic order (strictly increasing or decreasing); pop elements
that violate the order before pushing a new one. This pattern is narrow
but very recognizable, and when it fits, it turns an O(n²) brute force
into O(n).

**Question 8: Is there a bitwise structure to the problem?**

Bit manipulation. Signals: \"single number\" (XOR), \"count set bits,\"
subset enumeration with n ≤ 20 (bitmask DP), problems explicitly stated
in terms of binary representation.

**If none of the above fire**

Re-read the problem. The vast majority of FAANG questions route cleanly
through one of the above eight questions. If you have read it three
times and nothing fires, the most likely culprits are:

- You missed a sortedness signal in Question 1

- The problem is a thin disguise over DP and you didn\'t see the
  overlapping subproblems

- The problem uses a niche technique (segment tree, suffix array) ---
  rare, and you can usually punt with a brute force and ask the
  interviewer for hints

**The decision tree as a quick reference**

The table below condenses Chapter 2 into a glanceable lookup. The left
column is the cue you should be listening for; the right column is the
technique to reach for.

  -----------------------------------------------------------------------
  **Cue in the problem**              **Technique**
  ----------------------------------- -----------------------------------
  Sorted array, find target /         **Binary search**
  boundary                            

  Sorted array, find pair / triplet   **Two pointers**

  Contiguous subarray / substring     **Sliding window**

  Shortest path, unweighted graph     **BFS**

  Shortest path, weighted graph       **Dijkstra**
  (non-negative)                      

  All paths, any path, cycle          **DFS**
  detection                           

  Dependencies / prerequisites /      **Topological sort**
  ordering                            

  Connected components, dynamic       **Union-Find**
  merging                             

  K-th largest/smallest, top-k,       **Heap**
  median of stream                    

  Optimization with overlapping       **Dynamic programming**
  subproblems                         

  Enumerate combinations /            **Backtracking**
  permutations / subsets              

  Next/previous greater/smaller       **Monotonic stack**
  element                             

  Sliding window maximum / minimum    **Monotonic deque**

  Prefix-based string lookup          **Trie**

  Range sums, frequency counting over **Prefix sums**
  ranges                              

  Single number, XOR tricks, subset   **Bit manipulation**
  masks                               
  -----------------------------------------------------------------------

**Chapter 3 --- Two pointers and sliding window**

These two patterns are grouped because they share a mechanic: maintain a
region of the array defined by two indices, and advance the indices
according to some rule. The difference is how the indices move and what
they bound.

**Two pointers --- the mechanic**

You have an array (almost always sorted, or sortable cheaply). You
maintain two indices, typically left at 0 and right at n-1, or both
starting at 0. On each iteration, you examine the elements at the two
indices and decide which pointer to advance, based on the relationship
between the two values and your target.

The invariant: every pair you skip by advancing a pointer is provably
not a valid answer. This is what makes the technique work --- you
process O(n) pairs, not O(n²).

**Recognition signals**

- \"Find a pair / triplet with sum equal to target\" in a sorted array

- \"Remove duplicates\" or \"partition\" an array in place

- \"Reverse\" or \"palindrome\" --- pointers from both ends

- \"Merge two sorted arrays\" --- pointers, one per array

**Canonical example: two-sum on sorted input**

Given a sorted array and a target, find two indices whose values sum to
the target.

> **function** twoSumSorted(nums, target) {
>
> **let** left = 0, right = nums.length - 1;
>
> **while** (left \< right) {
>
> **const** sum = nums\[left\] + nums\[right\];
>
> **if** (sum === target) **return** \[left, right\];
>
> **if** (sum \< target) left++; *// sum too small, increase left value*
>
> **else** right\--; *// sum too large, decrease right value*
>
> }
>
> **return** \[-1, -1\];
>
> }

The reason this is correct: if nums\[left\] + nums\[right\] is too
small, then nums\[left\] paired with anything smaller than nums\[right\]
is also too small --- so we can eliminate all pairs (left, j) for j \<
right without examining them. Same logic in reverse for the too-large
case.

**Extending to three-sum**

Three-sum is two-sum wrapped in an outer loop. Fix the first element
with an outer loop, then run two-sum on the remainder. Total complexity
O(n²).

> **function** threeSum(nums) {
>
> nums.sort((a, b) =\> a - b);
>
> **const** result = \[\];
>
> **for** (**let** i = 0; i \< nums.length - 2; i++) {
>
> **if** (i \> 0 && nums\[i\] === nums\[i - 1\]) **continue**; *// skip
> duplicates*
>
> **let** left = i + 1, right = nums.length - 1;
>
> **while** (left \< right) {
>
> **const** sum = nums\[i\] + nums\[left\] + nums\[right\];
>
> **if** (sum === 0) {
>
> result.push(\[nums\[i\], nums\[left\], nums\[right\]\]);
>
> **while** (left \< right && nums\[left\] === nums\[left + 1\]) left++;
>
> **while** (left \< right && nums\[right\] === nums\[right - 1\])
> right\--;
>
> left++; right\--;
>
> } **else** **if** (sum \< 0) left++;
>
> **else** right\--;
>
> }
>
> }
>
> **return** result;
>
> }

Note the duplicate-skipping logic. This is a common source of bugs ---
the problem typically asks for unique triplets, and without the skip you
will emit duplicates.

**Sliding window --- the mechanic**

A specialization of two pointers where left and right both move only to
the right, never backward. The window \[left, right\] represents a
contiguous region of the array, and you maintain some aggregate (a sum,
a count, a frequency map) as the window expands and contracts.

Two flavors:

- **Fixed-size window.** Window size is given in the problem (\"maximum
  sum subarray of size k\"). Expand right until you hit size k, then
  slide both pointers in lockstep.

- **Variable-size window.** Window size depends on a constraint
  (\"longest substring with at most k distinct characters\"). Expand
  right while the constraint holds; when it breaks, contract left until
  it holds again.

**Recognition signals**

- \"Contiguous\" subarray or substring

- \"Longest\" / \"shortest\" / \"maximum\" / \"minimum\" with a
  constraint on the contents

- \"Find all anagrams\" or \"permutation in string\" --- fixed-size
  window with a frequency map

**Canonical example: longest substring without repeating characters**

> **function** lengthOfLongestSubstring(s) {
>
> **const** seen = **new** Map();
>
> **let** left = 0, best = 0;
>
> **for** (**let** right = 0; right \< s.length; right++) {
>
> **const** ch = s\[right\];
>
> *// If we\'ve seen this char inside the current window, jump left past
> it*
>
> **if** (seen.has(ch) && seen.get(ch) \>= left) {
>
> left = seen.get(ch) + 1;
>
> }
>
> seen.set(ch, right);
>
> best = Math.max(best, right - left + 1);
>
> }
>
> **return** best;
>
> }

The window invariant: every character inside \[left, right\] is unique.
When we encounter a repeat, we jump left forward to one past the
previous occurrence, restoring the invariant. Each character is visited
at most twice (once by right, once by left), so the algorithm is O(n).

**Canonical example: minimum window substring**

Given strings s and t, find the smallest substring of s that contains
all characters of t (with multiplicity). The variable-window template:

> **function** minWindow(s, t) {
>
> **const** need = **new** Map();
>
> **for** (**const** c **of** t) need.set(c, (need.get(c) \|\| 0) + 1);
>
> **let** required = need.size; *// distinct chars needed*
>
> **let** formed = 0; *// distinct chars currently satisfied*
>
> **const** have = **new** Map();
>
> **let** left = 0, bestLen = Infinity, bestLeft = 0;
>
> **for** (**let** right = 0; right \< s.length; right++) {
>
> **const** c = s\[right\];
>
> have.set(c, (have.get(c) \|\| 0) + 1);
>
> **if** (need.has(c) && have.get(c) === need.get(c)) formed++;
>
> *// Contract from the left while window is still valid*
>
> **while** (formed === required) {
>
> **if** (right - left + 1 \< bestLen) {
>
> bestLen = right - left + 1;
>
> bestLeft = left;
>
> }
>
> **const** lc = s\[left\];
>
> have.set(lc, have.get(lc) - 1);
>
> **if** (need.has(lc) && have.get(lc) \< need.get(lc)) formed\--;
>
> left++;
>
> }
>
> }
>
> **return** bestLen === Infinity ? \"\" : s.substring(bestLeft,
> bestLeft + bestLen);
>
> }

This is the prototypical variable-window problem. Expand right to
acquire what you need; when the window is valid, contract left to find
the minimum valid window; record the best. Master this template --- at
least half a dozen common problems are minor variations of it.

+-----------------------------------------------------------------------+
| **Common bug**                                                        |
|                                                                       |
| When contracting the window, the order of operations matters:         |
| decrement the count first, then check whether the constraint is       |
| violated. Doing it the other way leads to off-by-one errors in the    |
| \'formed\' counter.                                                   |
+-----------------------------------------------------------------------+

**Chapter 4 --- Binary search**

Binary search has a reputation as an easy topic that everyone gets
wrong. Both halves of that reputation are deserved. The basic idea is
trivial; the implementation details --- half-open vs closed intervals,
when to use \<= vs \<, when to return mid vs left --- are where mistakes
happen. This chapter gives you one template and asks you to use it
everywhere.

**The template**

There is exactly one binary search template worth memorizing. Use it for
every problem and you will eliminate an entire class of off-by-one
errors.

> **function** binarySearch(arr, target) {
>
> **let** left = 0, right = arr.length; *// half-open: \[left, right)*
>
> **while** (left \< right) {
>
> **const** mid = left + Math.floor((right - left) / 2);
>
> **if** (arr\[mid\] \< target) left = mid + 1;
>
> **else** right = mid;
>
> }
>
> **return** left; *// insertion point; check arr\[left\] === target
> separately*
>
> }

Read this carefully. Three things to notice:

- The interval \[left, right) is half-open. right is one past the last
  valid index. This is why right starts at arr.length, not arr.length -
  1.

- The loop condition is left \< right (not \<=). When left === right,
  the interval is empty and we stop.

- The function returns the insertion point --- the leftmost index where
  target could be inserted while keeping the array sorted. To find an
  actual match, check arr\[left\] === target after the loop.

This formulation generalizes. \"Find the first true\" in a boolean
predicate is the same loop with arr\[mid\] \< target replaced by
!predicate(mid).

**Binary search on the answer**

The most powerful variant --- and the one most likely to come up in a
hard interview round --- is binary search on the answer space, not on an
input array. The setup:

- You are trying to find the minimum (or maximum) value of some quantity
  that satisfies a constraint

- Given a candidate value, you can check the constraint in O(n) or so

- The constraint is monotonic in the candidate --- once it holds, it
  keeps holding for larger (or smaller) values

When all three are true, you binary search over the candidate values and
the loop runs in O(n log V) where V is the answer range.

**Canonical example: capacity to ship packages within D days**

You have packages with weights\[i\] and need to ship them in D days.
Each day, you load packages onto the ship in order until the weight
limit is reached. Find the minimum ship capacity.

> **function** shipWithinDays(weights, D) {
>
> *// Answer space: \[max(weights), sum(weights)\]*
>
> *// Lower bound: must fit the heaviest single package*
>
> *// Upper bound: ship everything in one day*
>
> **let** left = Math.max(\...weights);
>
> **let** right = weights.reduce((a, b) =\> a + b, 0);
>
> **const** canShip = (capacity) =\> {
>
> **let** days = 1, load = 0;
>
> **for** (**const** w **of** weights) {
>
> **if** (load + w \> capacity) { days++; load = 0; }
>
> load += w;
>
> }
>
> **return** days \<= D;
>
> };
>
> **while** (left \< right) {
>
> **const** mid = left + Math.floor((right - left) / 2);
>
> **if** (canShip(mid)) right = mid; *// mid works, try smaller*
>
> **else** left = mid + 1; *// mid too small, try larger*
>
> }
>
> **return** left;
>
> }

The insight that converts this from a hard problem to a routine one:
\"minimum capacity such that we can finish in D days\" is monotonic ---
if capacity C works, every capacity \> C also works. So the answer space
partitions cleanly into a no-no-no-yes-yes-yes pattern, which is exactly
what binary search exploits.

**Other shapes binary search takes**

- Finding the rotation point in a rotated sorted array --- predicate:
  \"is arr\[mid\] in the left-rotated half?\"

- Finding a peak element --- predicate: \"is arr\[mid\] \<
  arr\[mid+1\]?\" (peak is to the right)

- Square root, kth root --- answer space is \[0, n\], predicate is mid²
  ≤ n

- Median of two sorted arrays --- binary search on the partition point
  of the smaller array

+-----------------------------------------------------------------------+
| **Recognition trick**                                                 |
|                                                                       |
| If a problem asks for the minimum / maximum of something and you can  |
| write a function isFeasible(x) that returns true or false, and        |
| isFeasible is monotonic in x, you have a binary-search-on-the-answer  |
| problem. This shape is far more common than people realize, and       |
| recognizing it earns easy points.                                     |
+-----------------------------------------------------------------------+

**Chapter 5 --- BFS, DFS, and the graph patterns**

Every tree problem is a graph problem with extra structure. Every graph
problem either traverses the graph once (BFS or DFS), computes shortest
paths (BFS for unweighted, Dijkstra for weighted), orders the nodes
(topological sort), or partitions them into groups (Union-Find). Routing
between these five sub-techniques is the entire game.

**BFS --- the mechanic**

Process nodes in order of distance from the source. The data structure
is a queue: dequeue a node, process it, enqueue its unvisited neighbors,
mark them as visited. The invariant: when a node is first dequeued, the
path that led to it is the shortest path from the source (in number of
edges).

Use a visited set populated at enqueue time, not at dequeue time.
Otherwise you will enqueue the same node multiple times before
processing it, blowing up both time and memory.

**Canonical example: shortest path in an unweighted graph**

> **function** shortestPath(graph, start, end) {
>
> **if** (start === end) **return** 0;
>
> **const** visited = **new** Set(\[start\]);
>
> **const** queue = \[\[start, 0\]\]; *// \[node, distance\]*
>
> **while** (queue.length) {
>
> **const** \[node, dist\] = queue.shift(); *// see note below*
>
> **for** (**const** neighbor **of** graph.get(node) \|\| \[\]) {
>
> **if** (neighbor === end) **return** dist + 1;
>
> **if** (!visited.has(neighbor)) {
>
> visited.add(neighbor);
>
> queue.push(\[neighbor, dist + 1\]);
>
> }
>
> }
>
> }
>
> **return** -1;
>
> }

+-----------------------------------------------------------------------+
| **JavaScript queue warning**                                          |
|                                                                       |
| queue.shift() is O(n), which silently turns your O(V+E) BFS into      |
| O(V² + VE). For small inputs this is invisible; for n ≥ 10⁴ it        |
| dominates. Either implement a queue with a head pointer (don\'t       |
| actually shift, just advance an index), or use a deque. In an         |
| interview, mention this trade-off out loud --- interviewers           |
| appreciate it.                                                        |
+-----------------------------------------------------------------------+

**Level-order BFS**

When you need to process the graph in distinct levels (\"return the
rightmost node at each level of a binary tree\"), the structure is the
same but you process the queue one level at a time:

> **function** levelOrder(root) {
>
> **if** (!root) **return** \[\];
>
> **const** result = \[\];
>
> **let** queue = \[root\];
>
> **while** (queue.length) {
>
> **const** levelSize = queue.length; *// snapshot size before
> iterating*
>
> **const** level = \[\];
>
> **for** (**let** i = 0; i \< levelSize; i++) {
>
> **const** node = queue\[i\];
>
> level.push(node.val);
>
> **if** (node.left) queue.push(node.left);
>
> **if** (node.right) queue.push(node.right);
>
> }
>
> queue = queue.slice(levelSize); *// pop the level we just processed*
>
> result.push(level);
>
> }
>
> **return** result;
>
> }

**DFS --- the mechanic**

Process nodes by exploring as deeply as possible before backtracking.
The data structure is a stack --- explicit, or implicit via recursion.
The invariant is different from BFS: when DFS returns from a node, it
has visited every node reachable from it (through unvisited paths). This
makes DFS ideal for problems that need bottom-up information from the
recursion.

**Recursive DFS template**

> **function** dfs(node, visited = **new** Set()) {
>
> **if** (!node \|\| visited.has(node)) **return**;
>
> visited.add(node);
>
> *// pre-order work here*
>
> **for** (**const** neighbor **of** graph.get(node) \|\| \[\]) {
>
> dfs(neighbor, visited);
>
> }
>
> *// post-order work here*
>
> }

The choice between doing work before vs after the recursive call is
significant. Pre-order processing (working before the recursion) is
appropriate when each node can be processed independently of its
descendants. Post-order processing (working after) is appropriate when a
node\'s result depends on its descendants --- counting subtree sizes,
computing tree heights, deciding whether a subtree is balanced.

**Tree DP via post-order DFS**

A whole class of \"tree DP\" problems are post-order DFS in disguise.
Example: given a binary tree, find the diameter (longest path between
any two nodes, in number of edges).

> **function** diameterOfBinaryTree(root) {
>
> **let** diameter = 0;
>
> **function** depth(node) {
>
> **if** (!node) **return** 0;
>
> **const** left = depth(node.left);
>
> **const** right = depth(node.right);
>
> *// longest path through this node = left + right*
>
> diameter = Math.max(diameter, left + right);
>
> **return** 1 + Math.max(left, right); *// depth of subtree rooted
> here*
>
> }
>
> depth(root);
>
> **return** diameter;
>
> }

Notice the structure: the function returns one thing (depth) but updates
a closure variable with another (diameter). This \"return one thing,
update another\" pattern is ubiquitous in tree problems and worth
recognizing.

**Topological sort**

Linear ordering of nodes in a directed acyclic graph such that for every
edge u → v, u comes before v. Two implementations: Kahn\'s algorithm
(BFS-based, repeatedly pop nodes with in-degree zero) and DFS-based
(post-order traversal, reverse the result).

Kahn\'s is easier to write under pressure and detects cycles for free
(if the output has fewer nodes than the graph, there is a cycle).

> **function** topologicalSort(numNodes, edges) {
>
> **const** adj = **new** Map();
>
> **const** inDegree = **new** Array(numNodes).fill(0);
>
> **for** (**let** i = 0; i \< numNodes; i++) adj.set(i, \[\]);
>
> **for** (**const** \[u, v\] **of** edges) {
>
> adj.get(u).push(v);
>
> inDegree\[v\]++;
>
> }
>
> **const** queue = \[\];
>
> **for** (**let** i = 0; i \< numNodes; i++) {
>
> **if** (inDegree\[i\] === 0) queue.push(i);
>
> }
>
> **const** order = \[\];
>
> **while** (queue.length) {
>
> **const** node = queue.shift();
>
> order.push(node);
>
> **for** (**const** next **of** adj.get(node)) {
>
> **if** (\--inDegree\[next\] === 0) queue.push(next);
>
> }
>
> }
>
> **return** order.length === numNodes ? order : \[\]; *// empty =
> cycle*
>
> }

Recognition signals: \"can you finish all courses given prerequisites?\"
(Course Schedule), \"alien dictionary\" (derive a character ordering
from sorted words), \"build order,\" \"task scheduling with
dependencies.\"

**Dijkstra**

Shortest path in a weighted graph with non-negative edge weights.
Structurally identical to BFS, except the queue is replaced by a
min-heap keyed on distance from the source. The invariant: when a node
is first popped from the heap, its recorded distance is the true
shortest distance.

> **function** dijkstra(graph, start) {
>
> **const** dist = **new** Map();
>
> dist.set(start, 0);
>
> **const** heap = \[\[0, start\]\]; *// \[distance, node\] --- pretend
> this is a min-heap*
>
> **while** (heap.length) {
>
> **const** \[d, node\] = heapPop(heap);
>
> **if** (d \> (dist.get(node) ?? Infinity)) **continue**; *// stale
> entry*
>
> **for** (**const** \[neighbor, weight\] **of** graph.get(node) \|\|
> \[\]) {
>
> **const** newDist = d + weight;
>
> **if** (newDist \< (dist.get(neighbor) ?? Infinity)) {
>
> dist.set(neighbor, newDist);
>
> heapPush(heap, \[newDist, neighbor\]);
>
> }
>
> }
>
> }
>
> **return** dist;
>
> }

The \"stale entry\" check is important: because we never decrease-key
the heap (decrease-key is expensive without a custom heap), we instead
push new entries and discard old ones when we pop them. This makes the
heap size up to O(E) instead of O(V), but the asymptotic complexity
remains O((V+E) log V).

**Union-Find**

Two operations on a partition of n elements into disjoint sets:

> **class** UnionFind {
>
> constructor(n) {
>
> **this**.parent = Array.from({ length: n }, (\_, i) =\> i);
>
> **this**.rank = **new** Array(n).fill(0);
>
> **this**.components = n;
>
> }
>
> find(x) {
>
> **if** (**this**.parent\[x\] !== x) {
>
> **this**.parent\[x\] = **this**.find(**this**.parent\[x\]); *// path
> compression*
>
> }
>
> **return** **this**.parent\[x\];
>
> }
>
> union(x, y) {
>
> **const** px = **this**.find(x), py = **this**.find(y);
>
> **if** (px === py) **return** **false**; *// already connected*
>
> *// union by rank: attach smaller tree under larger*
>
> **if** (**this**.rank\[px\] \< **this**.rank\[py\])
> **this**.parent\[px\] = py;
>
> **else** **if** (**this**.rank\[px\] \> **this**.rank\[py\])
> **this**.parent\[py\] = px;
>
> **else** { **this**.parent\[py\] = px; **this**.rank\[px\]++; }
>
> **this**.components\--;
>
> **return** **true**;
>
> }
>
> }

Memorize this. It comes up surprisingly often: number of connected
components, Kruskal\'s MST, accounts merge, redundant connection,
regions cut by slashes. Whenever the problem involves \"merge groups one
at a time and ask questions about connectivity,\" reach for Union-Find
rather than re-running BFS.

**Chapter 6 --- Dynamic programming**

DP is where most interview prep grinds to a halt. The reason is that DP
is not really one technique; it is a family of techniques unified by a
single idea --- recompute nothing --- applied to a handful of recurring
problem shapes. Once you learn the shapes, DP stops feeling like dark
magic.

**The mechanic**

Three ingredients are required for DP to apply:

- **Overlapping subproblems.** Naive recursion would solve the same
  subproblem multiple times. If subproblems don\'t overlap, you have
  divide-and-conquer, not DP.

- **Optimal substructure.** The optimal solution to the full problem can
  be expressed in terms of optimal solutions to subproblems.

- **A state that captures everything you need.** The hardest part. State
  design is the entirety of DP problem-solving --- once the state is
  right, the recurrence usually writes itself.

Two flavors of implementation:

- **Top-down (memoized recursion).** Easier to derive. Write the natural
  recursion, then add a cache.

- **Bottom-up (tabulation).** Faster (no recursion overhead) and often
  more space-efficient. Harder to derive directly --- many people write
  top-down first and convert.

**The DP shapes**

Almost every DP problem in interviews fits one of these shapes.
Recognize the shape and you have 80% of the solution.

**Shape 1: 1D sequence DP**

State: dp\[i\] = optimal answer considering the first i elements.
Transition: dp\[i\] depends on dp\[i-1\], dp\[i-2\], or some bounded
window of previous values. Examples: house robber, climbing stairs,
longest increasing subsequence (O(n²) version), maximum subarray
(Kadane\'s).

> *// House robber: max sum of non-adjacent elements*
>
> **function** rob(nums) {
>
> **let** prev2 = 0, prev1 = 0; *// dp\[i-2\], dp\[i-1\]*
>
> **for** (**const** n **of** nums) {
>
> **const** curr = Math.max(prev1, prev2 + n);
>
> prev2 = prev1;
>
> prev1 = curr;
>
> }
>
> **return** prev1;
>
> }

Notice the O(1) space optimization: when the recurrence only depends on
a bounded number of previous values, you don\'t need the full dp array.

**Shape 2: 2D grid DP**

State: dp\[i\]\[j\] = optimal answer considering position (i, j).
Transition: dp\[i\]\[j\] depends on dp\[i-1\]\[j\], dp\[i\]\[j-1\],
and/or dp\[i-1\]\[j-1\]. Examples: unique paths, minimum path sum,
longest common subsequence, edit distance.

> *// Edit distance (Levenshtein): min ops to transform word1 → word2*
>
> **function** editDistance(w1, w2) {
>
> **const** m = w1.length, n = w2.length;
>
> **const** dp = Array.from({ length: m + 1 }, () =\> **new** Array(n +
> 1).fill(0));
>
> **for** (**let** i = 0; i \<= m; i++) dp\[i\]\[0\] = i; *// delete i
> chars*
>
> **for** (**let** j = 0; j \<= n; j++) dp\[0\]\[j\] = j; *// insert j
> chars*
>
> **for** (**let** i = 1; i \<= m; i++) {
>
> **for** (**let** j = 1; j \<= n; j++) {
>
> **if** (w1\[i-1\] === w2\[j-1\]) {
>
> dp\[i\]\[j\] = dp\[i-1\]\[j-1\];
>
> } **else** {
>
> dp\[i\]\[j\] = 1 + Math.min(
>
> dp\[i-1\]\[j\], *// delete from w1*
>
> dp\[i\]\[j-1\], *// insert into w1*
>
> dp\[i-1\]\[j-1\] *// replace*
>
> );
>
> }
>
> }
>
> }
>
> **return** dp\[m\]\[n\];
>
> }

Edit distance is worth memorizing. It\'s a common direct question and
the template generalizes to longest common subsequence, regex matching,
and wildcard matching.

**Shape 3: Knapsack DP**

State: dp\[i\]\[w\] = best value using items from the first i, with
weight budget w. Two main variants: 0/1 knapsack (each item used at most
once) and unbounded knapsack (each item used any number of times).

> *// Coin change: fewest coins to make \'amount\' (unbounded knapsack)*
>
> **function** coinChange(coins, amount) {
>
> **const** dp = **new** Array(amount + 1).fill(Infinity);
>
> dp\[0\] = 0;
>
> **for** (**let** a = 1; a \<= amount; a++) {
>
> **for** (**const** c **of** coins) {
>
> **if** (c \<= a) dp\[a\] = Math.min(dp\[a\], dp\[a - c\] + 1);
>
> }
>
> }
>
> **return** dp\[amount\] === Infinity ? -1 : dp\[amount\];
>
> }

The loop ordering matters and is the source of constant bugs. For 0/1
knapsack, iterate items in the outer loop and weight backwards in the
inner loop. For unbounded, iterate weight forwards. If you get this
wrong, you\'ll count items multiple times (or zero times) when you
shouldn\'t.

**Shape 4: Interval DP**

State: dp\[i\]\[j\] = optimal answer for the subarray (or substring)
from i to j. The transition typically considers a split point k between
i and j: dp\[i\]\[j\] = min/max over k of f(dp\[i\]\[k\],
dp\[k+1\]\[j\], cost(i, k, j)). Examples: matrix chain multiplication,
burst balloons, palindromic substrings, stone games.

The iteration order is unusual: by length, not by left or right
endpoint. You compute all intervals of length 1, then length 2, etc.,
because dp\[i\]\[j\] depends on shorter intervals contained within.

> *// Longest palindromic substring length*
>
> **function** longestPalindromeLen(s) {
>
> **const** n = s.length;
>
> **const** dp = Array.from({ length: n }, () =\> **new**
> Array(n).fill(**false**));
>
> **let** best = 1;
>
> **for** (**let** i = 0; i \< n; i++) dp\[i\]\[i\] = **true**; *//
> length 1*
>
> **for** (**let** len = 2; len \<= n; len++) {
>
> **for** (**let** i = 0; i + len - 1 \< n; i++) {
>
> **const** j = i + len - 1;
>
> **if** (s\[i\] === s\[j\] && (len === 2 \|\| dp\[i+1\]\[j-1\])) {
>
> dp\[i\]\[j\] = **true**;
>
> best = Math.max(best, len);
>
> }
>
> }
>
> }
>
> **return** best;
>
> }

**Shape 5: DP on trees**

State is implicit in the recursion --- for each node, return the optimal
answer for the subtree rooted at that node, possibly returning multiple
values (the optimum with the node included, and without). Mechanically
this is post-order DFS, but the framing is DP. Examples: house robber
III, diameter of a binary tree, longest univalue path.

**Shape 6: Bitmask DP**

State: dp\[mask\] = optimal answer when the set of completed items is
the bits set in mask. Used when n is small (≤ 20) and you need to
consider subsets. Examples: traveling salesman, partition to k equal
subsets, shortest superstring.

The complexity is typically O(2ⁿ · n), which is fine for n ≤ 20 but
explodes beyond. The recognition signal is a small n combined with a
problem that naturally involves subsets.

**State design --- the actual hard part**

Most DP failures are state design failures, not coding failures. Two
questions to ask:

- **What do I need to know to decide what to do next?** Whatever your
  answer is, that\'s your state. If you find yourself thinking \"and I
  also need to remember whether I bought a stock recently,\" that\'s
  another state dimension.

- **Is the state independent of the path taken to reach it?** DP
  requires the Markov property: the optimal continuation depends only on
  the current state, not on how you got there. If it doesn\'t, your
  state is incomplete.

+-----------------------------------------------------------------------+
| **How to derive a DP solution under pressure**                        |
|                                                                       |
| Start with a recursive brute-force. Identify the parameters that      |
| change between calls --- those are your state. Write the recurrence   |
| in terms of recursive calls. Add memoization. Once it works,          |
| optionally convert to bottom-up for space optimization. This sequence |
| will not fail you, even when you cannot see the elegant solution      |
| directly.                                                             |
+-----------------------------------------------------------------------+

**Chapter 7 --- Backtracking**

Backtracking is the technique you reach for when you need to enumerate
all valid configurations, not just count or optimize them. The mechanic
is simple: build a partial solution incrementally, recurse, and undo
your choice before trying the next option. The complexity is exponential
by definition, which is acceptable because the problems that use it have
small inputs.

**The template**

Every backtracking problem has the same shape. Internalize this template
and you can write any of them in two minutes.

> **function** backtrack(state, choices, result) {
>
> **if** (isComplete(state)) {
>
> result.push(\[\...state\]); *// CRITICAL: clone, don\'t push
> reference*
>
> **return**;
>
> }
>
> **for** (**const** choice **of** validChoices(state, choices)) {
>
> state.push(choice); *// make choice*
>
> backtrack(state, choices, result);
>
> state.pop(); *// undo choice*
>
> }
>
> }

Three pieces to fill in for any specific problem: the completion check,
the choice generation, and the validity filter (sometimes the filter is
inside choice generation; sometimes it\'s a separate check before
recursing).

**Worked example: permutations**

> **function** permutations(nums) {
>
> **const** result = \[\];
>
> **const** used = **new** Array(nums.length).fill(**false**);
>
> **function** backtrack(current) {
>
> **if** (current.length === nums.length) {
>
> result.push(\[\...current\]);
>
> **return**;
>
> }
>
> **for** (**let** i = 0; i \< nums.length; i++) {
>
> **if** (used\[i\]) **continue**;
>
> used\[i\] = **true**;
>
> current.push(nums\[i\]);
>
> backtrack(current);
>
> current.pop();
>
> used\[i\] = **false**;
>
> }
>
> }
>
> backtrack(\[\]);
>
> **return** result;
>
> }

**Worked example: subsets**

Subsets is structurally different from permutations: order does not
matter, and you can choose to include or skip each element. The choice
index advances each recursion rather than restarting from 0.

> **function** subsets(nums) {
>
> **const** result = \[\];
>
> **function** backtrack(start, current) {
>
> result.push(\[\...current\]); *// every state is a valid subset*
>
> **for** (**let** i = start; i \< nums.length; i++) {
>
> current.push(nums\[i\]);
>
> backtrack(i + 1, current); *// i + 1, not start + 1*
>
> current.pop();
>
> }
>
> }
>
> backtrack(0, \[\]);
>
> **return** result;
>
> }

**Worked example: N-Queens**

The harder version: validity is non-trivial (no two queens attack each
other), and pruning matters enormously for performance.

> **function** solveNQueens(n) {
>
> **const** result = \[\];
>
> **const** cols = **new** Set(), diag1 = **new** Set(), diag2 = **new**
> Set();
>
> **const** board = \[\];
>
> **function** backtrack(row) {
>
> **if** (row === n) {
>
> result.push(board.map(c =\>
>
> \'.\'.repeat(c) + \'Q\' + \'.\'.repeat(n - c - 1)));
>
> **return**;
>
> }
>
> **for** (**let** c = 0; c \< n; c++) {
>
> **if** (cols.has(c) \|\| diag1.has(row - c) \|\| diag2.has(row + c))
> **continue**;
>
> cols.add(c); diag1.add(row - c); diag2.add(row + c);
>
> board.push(c);
>
> backtrack(row + 1);
>
> board.pop();
>
> cols.delete(c); diag1.delete(row - c); diag2.delete(row + c);
>
> }
>
> }
>
> backtrack(0);
>
> **return** result;
>
> }

The trick is recognizing that two queens on the same diagonal have
either row - col equal or row + col equal --- so tracking three sets
(columns, two diagonal types) gives O(1) validity checking instead of
O(n).

**Pruning is everything**

Pure backtracking without pruning is just enumeration. The interesting
case is when you can detect early that a partial solution cannot lead to
a valid full solution, and prune the entire subtree. Sudoku solvers,
word search, combination sum --- all benefit massively from aggressive
pruning. The instinct to ask \"can I prune this branch right now?\"
before recursing is what separates a slow solution from a fast one.

+-----------------------------------------------------------------------+
| **Backtracking vs DP**                                                |
|                                                                       |
| If the problem asks for the configurations themselves, it\'s          |
| backtracking. If it asks for the count or the optimum, it\'s DP. The  |
| two techniques have similar recursive structure, but DP collapses     |
| identical subproblems while backtracking enumerates all paths through |
| them. Confusing the two is a common interview mistake --- make sure   |
| you\'ve read the question carefully before committing.                |
+-----------------------------------------------------------------------+

**Chapter 8 --- Heaps, monotonic structures, and the rest**

**Heaps**

A heap is a tree-shaped data structure that maintains a partial order:
in a min-heap, every parent is ≤ its children. This gives you O(1)
access to the min, O(log n) insertion, and O(log n) deletion of the min.
JavaScript has no built-in heap; for interview purposes, either
implement one or describe one and hand-wave.

**When to reach for a heap**

- \"K-th largest / smallest\" --- maintain a heap of size k

- \"Top k frequent\" --- count with a hash map, then heap of size k

- \"Median of a data stream\" --- two heaps (max-heap of lower half,
  min-heap of upper half)

- \"Merge k sorted lists / arrays\" --- heap keyed by current element
  from each list

- Dijkstra\'s algorithm --- already covered

**K-th largest as a template**

> *// Maintain a min-heap of size k. The top is the k-th largest.*
>
> **function** findKthLargest(nums, k) {
>
> **const** heap = \[\]; *// pretend this is a min-heap with O(log n)
> push/pop*
>
> **for** (**const** n **of** nums) {
>
> heapPush(heap, n);
>
> **if** (heap.length \> k) heapPop(heap);
>
> }
>
> **return** heap\[0\]; *// top of min-heap = smallest of k largest =
> k-th largest*
>
> }

The trick: by maintaining a heap of size k and popping the smallest
whenever it overflows, the heap always contains the k largest elements
seen so far. Its minimum is therefore the k-th largest. O(n log k)
instead of O(n log n) for a full sort.

**Monotonic stack**

A stack maintained in monotonic order (strictly increasing or
decreasing). When you push a new element, you first pop everything that
would violate the order. The pattern shows up specifically in \"find the
next/previous greater/smaller element\" problems and is the single most
efficient technique for them: O(n) where the brute force is O(n²).

> *// Next greater element: result\[i\] is the first j \> i with
> nums\[j\] \> nums\[i\],*
>
> *// or -1 if no such j exists.*
>
> **function** nextGreaterElement(nums) {
>
> **const** result = **new** Array(nums.length).fill(-1);
>
> **const** stack = \[\]; *// indices, monotonically decreasing by
> value*
>
> **for** (**let** i = 0; i \< nums.length; i++) {
>
> **while** (stack.length && nums\[stack\[stack.length - 1\]\] \<
> nums\[i\]) {
>
> result\[stack.pop()\] = nums\[i\];
>
> }
>
> stack.push(i);
>
> }
>
> **return** result;
>
> }

Read this carefully. Each index is pushed and popped at most once across
the entire loop, which is why this is O(n) despite the inner while loop.
The amortized analysis is the key insight --- without it, you might
mistake this for O(n²) and miss the point.

Other problems in this family: largest rectangle in histogram, daily
temperatures, trapping rain water, sum of subarray minimums, remove k
digits.

**Monotonic deque**

A deque (double-ended queue) maintained in monotonic order. Used for
sliding-window-maximum-type problems where you need O(1) access to the
max of the current window, with O(1) updates as the window slides.

> *// Sliding window maximum of size k*
>
> **function** maxSlidingWindow(nums, k) {
>
> **const** dq = \[\]; *// indices, values monotonically decreasing*
>
> **const** result = \[\];
>
> **for** (**let** i = 0; i \< nums.length; i++) {
>
> *// Remove indices outside the window*
>
> **while** (dq.length && dq\[0\] \<= i - k) dq.shift();
>
> *// Maintain decreasing order*
>
> **while** (dq.length && nums\[dq\[dq.length - 1\]\] \< nums\[i\])
> dq.pop();
>
> dq.push(i);
>
> **if** (i \>= k - 1) result.push(nums\[dq\[0\]\]);
>
> }
>
> **return** result;
>
> }

**Tries**

A tree of characters used for prefix-based string operations. O(k)
insertion and lookup where k is the key length, independent of how many
strings are stored. Use when the problem involves prefixes,
autocomplete, or word search.

> **class** Trie {
>
> constructor() { **this**.root = {}; }
>
> insert(word) {
>
> **let** node = **this**.root;
>
> **for** (**const** c **of** word) {
>
> **if** (!node\[c\]) node\[c\] = {};
>
> node = node\[c\];
>
> }
>
> node.isEnd = **true**;
>
> }
>
> search(word) {
>
> **let** node = **this**.root;
>
> **for** (**const** c **of** word) {
>
> **if** (!node\[c\]) **return** **false**;
>
> node = node\[c\];
>
> }
>
> **return** node.isEnd === **true**;
>
> }
>
> startsWith(prefix) {
>
> **let** node = **this**.root;
>
> **for** (**const** c **of** prefix) {
>
> **if** (!node\[c\]) **return** **false**;
>
> node = node\[c\];
>
> }
>
> **return** **true**;
>
> }
>
> }

**Prefix sums**

Precompute an array P where P\[i\] = sum of arr\[0..i-1\]. Then sum of
arr\[i..j\] is P\[j+1\] - P\[i\], in O(1). Trivial idea, surprisingly
powerful when combined with hash maps.

> *// Subarray sum equals k: count subarrays whose sum is exactly k*
>
> **function** subarraySum(nums, k) {
>
> **const** count = **new** Map(\[\[0, 1\]\]); *// prefix sum 0 occurs
> once (empty)*
>
> **let** sum = 0, result = 0;
>
> **for** (**const** n **of** nums) {
>
> sum += n;
>
> **if** (count.has(sum - k)) result += count.get(sum - k);
>
> count.set(sum, (count.get(sum) \|\| 0) + 1);
>
> }
>
> **return** result;
>
> }

The insight: a subarray ending at index i with sum k exists iff there\'s
an earlier prefix sum equal to (current prefix sum - k). Hash map counts
the occurrences. This O(n) trick converts what looks like an O(n²)
problem.

**Bit manipulation**

A small bag of tricks that come up occasionally. Worth being fluent in,
even if the problems are rarer.

- x & (x - 1) clears the lowest set bit --- useful for counting set bits
  in O(popcount) instead of O(32).

- x \^ x === 0 and x \^ 0 === x --- the basis of \"single number\"
  problems where every value appears twice except one.

- 1 \<\< i creates a mask with bit i set. mask & (1 \<\< i) tests bit i.
  mask \| (1 \<\< i) sets it. mask & \~(1 \<\< i) clears it.

- Iterating over all subsets of a set: for (let mask = 0; mask \< (1
  \<\< n); mask++). Iterating over all subsets of a specific mask m: for
  (let s = m; s \> 0; s = (s - 1) & m) --- niche, but it appears in
  bitmask DP.

**Chapter 9 --- Interview mechanics**

Knowing the patterns is necessary but not sufficient. The 45-minute
interview format places its own demands on top of the algorithmic
content, and candidates who treat the conversation as a code-only
exercise consistently underperform candidates who treat it as a
structured collaboration. This chapter is about the structure.

**The five phases of a typical interview**

**Phase 1: Clarification (2--5 minutes)**

Before writing anything, restate the problem and ask questions. The
interviewer almost always under-specifies on purpose. Things to clarify:

- Input constraints: size, range, types, nulls, duplicates, sorted vs
  unsorted

- Output format: single value, all values, in-place, return new

- Edge cases: empty input, single element, negatives, overflow

- Whether the input can be modified

Skipping this phase costs more time than it saves. Two minutes of
clarification will save you ten minutes of rewriting after you discover
halfway through that the input can have negative numbers.

**Phase 2: Brute force (3--5 minutes)**

State the brute force out loud, including its complexity, even if you
can immediately see the optimal. This serves three purposes: it confirms
you and the interviewer agree on the problem, it gives you a fallback if
you can\'t find the optimal in time, and it makes the optimization
visible as progress when you do find it.

If you can\'t see the optimal, code the brute force. Partial credit is
real. A working brute force is worth more than a half-finished optimal
solution.

**Phase 3: Optimization (5--10 minutes)**

This is where the decision tree from Chapter 2 earns its keep. Walk
through the recognition signals out loud. \"The input is sorted, so
binary search or two pointers are available. We\'re looking for a pair,
so two pointers fits the shape.\" Even if you\'re wrong, the interviewer
can redirect you, which is far better than them watching you flail
silently.

**Phase 4: Code (15--25 minutes)**

Write the code with running commentary. Not every line --- that\'s
annoying --- but the structural decisions: \"I\'m going to use a hash
map here to track frequencies. The key is the character, the value is
the count.\" When you make a non-obvious choice, explain it. When
you\'re tracking an invariant, state it.

Use meaningful variable names. left and right are fine for binary
search; nums\[i\] and j are fine for inner loops; but anything more
complex deserves a real name. Interviewers grade readability, often
informally.

**Phase 5: Test and debug (5--10 minutes)**

Walk through your code with a small input. Trace the variables on the
whiteboard or in comments. Then try edge cases --- empty, single
element, the value that triggers the boundary condition. If you find a
bug, fix it. Finding and fixing your own bug is worth more than not
having one, because it demonstrates the testing instinct the interviewer
is looking for.

**How to recover when stuck**

Everyone gets stuck. The signal isn\'t whether you get stuck; it\'s how
you recover.

- **Re-read the problem.** Often the path forward is in a constraint you
  missed.

- **Try a small example by hand.** Trace through n=3 or n=4 on paper.
  The pattern often reveals itself.

- **Ask for a hint.** Not the answer --- a hint. \"I\'m thinking about
  this as a DP problem but I\'m not sure what to make the state. Am I in
  the right neighborhood?\" Interviewers will almost always engage with
  this and the conversation moves forward.

- **Switch to brute force.** If you cannot see the optimal, code the
  brute force fast. A working brute force at minute 25 leaves you 20
  minutes to optimize, which is more than enough.

**What signal interviewers are actually looking for**

Mostly, three things:

- **Problem-solving process.** Can you decompose, hypothesize, verify?
  Do you reach for known patterns? Can you assess your own approach for
  correctness and complexity?

- **Communication.** Can a reasonable interviewer follow your thinking
  in real time? Would your future teammates be able to work with you on
  a hard problem?

- **Code quality.** Is the code readable? Are edge cases handled? Would
  it pass review without major rewrites?

Notice what isn\'t on the list: producing the optimal solution. Many
candidates who don\'t reach optimal still pass, on the strength of
process and communication. Many candidates who do reach optimal still
fail, because they did so silently with messy code. The grading is
holistic and the algorithm is one input among several.

+-----------------------------------------------------------------------+
| **The single most underrated skill**                                  |
|                                                                       |
| Thinking out loud is hard. Most engineers spend their working hours   |
| thinking silently and writing code privately. The interview format is |
| the opposite of that, and the gap is real. Practice it: solve         |
| LeetCode problems while narrating to an empty room, or record         |
| yourself. The first few sessions will feel ridiculous; by the tenth,  |
| it will feel natural. By the twentieth, the narration will actually   |
| help you think.                                                       |
+-----------------------------------------------------------------------+

**Chapter 10 --- A practical study plan**

Reading this guide is the start. Getting to interview-ready takes
deliberate practice --- and surprisingly little of it, if it\'s
targeted. The plan below assumes you have 6--8 weeks of evening time
available. Compress or expand as needed.

**Weeks 1--2: Foundations and recognition**

Re-read Chapters 1 and 2. Implement from scratch in JavaScript, without
looking anything up: a binary heap, a trie, and a union-find. Then work
through 2--3 problems per pattern, focusing on recognition: read the
problem, articulate which pattern fits and why, then code it. The goal
at this stage is fluency in routing, not speed.

**Weeks 3--4: Pattern depth**

Pick two patterns per week and work 8--10 problems in each. The aim is
to internalize the variations --- by the time you\'ve seen a dozen
sliding-window problems, the eleventh feels boring. Boring is good;
boring means you\'ve absorbed the pattern. Suggested pairs: weeks 3 →
two pointers + sliding window, BFS + DFS. Weeks 4 → DP (one full week,
it deserves it), backtracking + heaps.

**Weeks 5--6: Mock interviews**

This is where most people skip and most people regret it. Find a partner
(interviewing.io, Pramp, a friend) and do 2--3 mock interviews per week,
alternating sides. Interviewing someone else teaches you what
interviewers are actually grading. The first three mocks will be
painful; persist through them. The signal-to-noise of mocks is higher
than the signal-to-noise of solo LeetCode.

**Weeks 7--8: Hard problems and weak spots**

By now you have a sense of which patterns are weakest. Spend the final
two weeks on hard problems in those areas, plus a steady diet of easier
problems in your strong areas to maintain speed. Do at least one full
45-minute mock per week.

**Problem volume --- how much is enough?**

Less than you think. Most successful FAANG candidates have done 150--300
problems total. The first 50 teach you the patterns; the next 100 build
fluency; problems beyond 300 are diminishing returns. If you have done
500 and still don\'t feel ready, the issue is not problem volume ---
it\'s quality of practice. Slow down and study the solutions to problems
you couldn\'t solve, in depth, instead of grinding for breadth.

**Resources worth knowing about**

- NeetCode 150 (free YouTube + neetcode.io) --- curated list, video
  walkthroughs, very example-heavy

- LeetCode Patterns by Sean Prashad (free) --- problems organized by
  pattern, similar shape to this guide

- Cracking the Coding Interview by Gayle McDowell --- long but
  comprehensive, includes the soft-skills material

- Elements of Programming Interviews --- denser than CTCI, more advanced
  problems, language-specific versions

- interviewing.io --- anonymous mock interviews with engineers from the
  target companies, free tier available

**Closing**

The hardest thing about FAANG interview prep is not the material; it\'s
the discipline to engage with it as pattern recognition rather than as
memorization. Memorize the templates in this guide --- every one of them
is short --- but spend your real effort on the routing layer. The
pattern catalog is finite. The problems built on top of it are infinite,
but they are reducible. Recognize, route, execute, communicate. That is
the entire game.

Good luck.
