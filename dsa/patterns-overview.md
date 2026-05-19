# DSA Patterns — Overview & Example Problems

Reference of the major patterns used in coding interviews. Each section includes:
- **Signal phrases** — phrases in the prompt that hint at this pattern
- **Approach** — one-line summary of the technique
- **Example problems** — 2 Easy, 2 Medium, 2 Hard. Click links to open on LeetCode.

---

## 1. Two Pointers

**Signal phrases:** sorted array, pair/triplet with target sum, palindrome check, remove duplicates, "in-place"

**Approach:** Two indices moving toward each other (or in tandem) based on a comparison; eliminates O(N²) brute force on sorted/structured input.

**Problems:**
- (E) [Valid Palindrome](https://leetcode.com/problems/valid-palindrome/)
- (E) [Two Sum II — Input Array Is Sorted](https://leetcode.com/problems/two-sum-ii-input-array-is-sorted/)
- (M) [3Sum](https://leetcode.com/problems/3sum/)
- (M) [Container With Most Water](https://leetcode.com/problems/container-with-most-water/)
- (H) [Trapping Rain Water](https://leetcode.com/problems/trapping-rain-water/)
- (H) [Minimum Window Substring](https://leetcode.com/problems/minimum-window-substring/) (combo: two pointers + sliding window)

---

## 2. Sliding Window

**Signal phrases:** "longest/shortest/min/max" of a contiguous subarray or substring, "at most K," "exactly K," "containing all of..."

**Approach:** Maintain a window `[left, right]`; expand `right`, shrink `left` when invariant breaks. Fixed-size for "of size K"; variable-size for optimization questions.

**Problems:**
- (E) [Maximum Average Subarray I](https://leetcode.com/problems/maximum-average-subarray-i/)
- (E) [Best Time to Buy and Sell Stock](https://leetcode.com/problems/best-time-to-buy-and-sell-stock/)
- (M) [Longest Substring Without Repeating Characters](https://leetcode.com/problems/longest-substring-without-repeating-characters/)
- (M) [Permutation in String](https://leetcode.com/problems/permutation-in-string/)
- (H) [Sliding Window Maximum](https://leetcode.com/problems/sliding-window-maximum/)
- (H) [Substring with Concatenation of All Words](https://leetcode.com/problems/substring-with-concatenation-of-all-words/)

---

## 3. Fast & Slow Pointers (Floyd's Cycle Detection)

**Signal phrases:** linked list cycle, find middle node, nth-from-end, "cycle" in a number sequence

**Approach:** Slow pointer moves 1 step; fast pointer moves 2. If they meet, there's a cycle. Useful for finding cycle starts, middles, and detecting structure.

**Problems:**
- (E) [Linked List Cycle](https://leetcode.com/problems/linked-list-cycle/)
- (E) [Middle of the Linked List](https://leetcode.com/problems/middle-of-the-linked-list/)
- (M) [Linked List Cycle II](https://leetcode.com/problems/linked-list-cycle-ii/)
- (M) [Happy Number](https://leetcode.com/problems/happy-number/)
- (H) [Find the Duplicate Number](https://leetcode.com/problems/find-the-duplicate-number/)
- (H) [Reorder List](https://leetcode.com/problems/reorder-list/)

---

## 4. Binary Search

**Signal phrases:** sorted input, "find min/max value that satisfies X," monotonic predicate, O(log n) hinted, rotated sorted array

**Approach:** Maintain `lo`, `hi`; compute `mid`; based on predicate, collapse to half. "Binary search on the answer" applies the same when the candidate answer space is monotonic.

**Problems:**
- (E) [Binary Search](https://leetcode.com/problems/binary-search/)
- (E) [First Bad Version](https://leetcode.com/problems/first-bad-version/)
- (M) [Search in Rotated Sorted Array](https://leetcode.com/problems/search-in-rotated-sorted-array/)
- (M) [Koko Eating Bananas](https://leetcode.com/problems/koko-eating-bananas/)
- (H) [Median of Two Sorted Arrays](https://leetcode.com/problems/median-of-two-sorted-arrays/)
- (H) [Split Array Largest Sum](https://leetcode.com/problems/split-array-largest-sum/)

---

## 5. BFS / DFS on Graphs & Trees

**Signal phrases:** shortest path (unweighted), level-order, connected components, all paths, reachability, "explore the grid"

**Approach:** BFS with queue for shortest path / level-order. DFS with recursion or stack for connectivity, topological order, all paths.

**Problems:**
- (E) [Binary Tree Level Order Traversal](https://leetcode.com/problems/binary-tree-level-order-traversal/)
- (E) [Flood Fill](https://leetcode.com/problems/flood-fill/)
- (M) [Number of Islands](https://leetcode.com/problems/number-of-islands/)
- (M) [Clone Graph](https://leetcode.com/problems/clone-graph/)
- (H) [Word Ladder](https://leetcode.com/problems/word-ladder/)
- (H) [Bus Routes](https://leetcode.com/problems/bus-routes/)

---

## 6. Backtracking

**Signal phrases:** "all combinations / permutations / subsets," constraint satisfaction (N-Queens, Sudoku), enumeration of valid configurations

**Approach:** DFS over a decision tree. Choose → recurse → unchoose. Prune branches that can't lead to a valid solution.

**Problems:**
- (E) [Letter Case Permutation](https://leetcode.com/problems/letter-case-permutation/)
- (E) [Generate Parentheses](https://leetcode.com/problems/generate-parentheses/)
- (M) [Subsets](https://leetcode.com/problems/subsets/)
- (M) [Combination Sum](https://leetcode.com/problems/combination-sum/)
- (H) [N-Queens](https://leetcode.com/problems/n-queens/)
- (H) [Sudoku Solver](https://leetcode.com/problems/sudoku-solver/)

---

## 7. Dynamic Programming

**Signal phrases:** "max/min/count ways," overlapping subproblems, optimal substructure, "find the best way to..."

**Approach:** Identify state, recurrence, base case, computation order. Common shapes: 1D (prefix-based), 2D (two-string / grid), knapsack, interval DP.

**Problems:**
- (E) [Climbing Stairs](https://leetcode.com/problems/climbing-stairs/)
- (E) [House Robber](https://leetcode.com/problems/house-robber/)
- (M) [Coin Change](https://leetcode.com/problems/coin-change/)
- (M) [Longest Increasing Subsequence](https://leetcode.com/problems/longest-increasing-subsequence/)
- (H) [Edit Distance](https://leetcode.com/problems/edit-distance/)
- (H) [Burst Balloons](https://leetcode.com/problems/burst-balloons/)

---

## 8. Heap / Priority Queue

**Signal phrases:** "top K," "K-th largest/smallest," merge K sorted lists, scheduling, running median

**Approach:** Min-heap of size K for top-K-largest (and vice versa). Two heaps for running median. Heap of (value, source) for K-way merge.

**Problems:**
- (E) [Kth Largest Element in a Stream](https://leetcode.com/problems/kth-largest-element-in-a-stream/)
- (E) [Last Stone Weight](https://leetcode.com/problems/last-stone-weight/)
- (M) [Top K Frequent Elements](https://leetcode.com/problems/top-k-frequent-elements/)
- (M) [K Closest Points to Origin](https://leetcode.com/problems/k-closest-points-to-origin/)
- (H) [Merge k Sorted Lists](https://leetcode.com/problems/merge-k-sorted-lists/)
- (H) [Find Median from Data Stream](https://leetcode.com/problems/find-median-from-data-stream/)

---

## 9. Hash Map / Set

**Signal phrases:** O(1) lookup, count frequencies, detect duplicates, complement pairs, group by key

**Approach:** Trade space for time. Frequency counters, prefix-sum-to-index maps, complement lookups, anagram-key grouping.

**Problems:**
- (E) [Two Sum](https://leetcode.com/problems/two-sum/)
- (E) [Contains Duplicate](https://leetcode.com/problems/contains-duplicate/)
- (M) [Group Anagrams](https://leetcode.com/problems/group-anagrams/)
- (M) [Subarray Sum Equals K](https://leetcode.com/problems/subarray-sum-equals-k/)
- (H) [Longest Consecutive Sequence](https://leetcode.com/problems/longest-consecutive-sequence/)
- (H) [Substring with Concatenation of All Words](https://leetcode.com/problems/substring-with-concatenation-of-all-words/)

---

## 10. Prefix Sum

**Signal phrases:** range sum queries, "subarray sum equals K," running totals, 2D submatrix sum

**Approach:** Precompute `prefix[i] = sum(arr[0..i-1])` so `sum(i..j) = prefix[j+1] - prefix[i]`. Combine with hashmap for counting subarrays.

**Problems:**
- (E) [Running Sum of 1d Array](https://leetcode.com/problems/running-sum-of-1d-array/)
- (E) [Range Sum Query — Immutable](https://leetcode.com/problems/range-sum-query-immutable/)
- (M) [Subarray Sum Equals K](https://leetcode.com/problems/subarray-sum-equals-k/)
- (M) [Continuous Subarray Sum](https://leetcode.com/problems/continuous-subarray-sum/)
- (H) [Maximum Sum of 3 Non-Overlapping Subarrays](https://leetcode.com/problems/maximum-sum-of-3-non-overlapping-subarrays/)
- (H) [Count of Range Sum](https://leetcode.com/problems/count-of-range-sum/)

---

## 11. Monotonic Stack / Deque

**Signal phrases:** "next greater / smaller," "previous greater / smaller," largest rectangle, span/distance to warmer day, sliding window max

**Approach:** Stack maintained in strictly increasing or decreasing order. Pop on order violation — each pop records a relationship between popped and new element. Deque variant for sliding window max/min.

**Problems:**
- (E) [Next Greater Element I](https://leetcode.com/problems/next-greater-element-i/)
- (E) [Baseball Game](https://leetcode.com/problems/baseball-game/) (stack practice)
- (M) [Daily Temperatures](https://leetcode.com/problems/daily-temperatures/)
- (M) [Online Stock Span](https://leetcode.com/problems/online-stock-span/)
- (H) [Largest Rectangle in Histogram](https://leetcode.com/problems/largest-rectangle-in-histogram/)
- (H) [Sum of Subarray Minimums](https://leetcode.com/problems/sum-of-subarray-minimums/)

---

## 12. Union-Find (Disjoint Set Union)

**Signal phrases:** connected components, "is A connected to B?" with dynamic merges, MST (Kruskal's), equivalence groups

**Approach:** `find` with path compression, `union` by rank. O(α(N)) per operation, effectively constant.

**Problems:**
- (E) [Find if Path Exists in Graph](https://leetcode.com/problems/find-if-path-exists-in-graph/)
- (E) [Number of Provinces](https://leetcode.com/problems/number-of-provinces/)
- (M) [Redundant Connection](https://leetcode.com/problems/redundant-connection/)
- (M) [Accounts Merge](https://leetcode.com/problems/accounts-merge/)
- (H) [Number of Islands II](https://leetcode.com/problems/number-of-islands-ii/)
- (H) [Swim in Rising Water](https://leetcode.com/problems/swim-in-rising-water/)

---

## 13. Topological Sort

**Signal phrases:** dependency ordering, DAG, prerequisites, "can finish all courses?" build order, scheduling with constraints

**Approach:** Kahn's algorithm (BFS over zero-in-degree nodes) or DFS post-order. Cycle detection = no valid topological order exists.

**Problems:**
- (E) [Find Center of Star Graph](https://leetcode.com/problems/find-center-of-star-graph/)
- (E) [Find the Town Judge](https://leetcode.com/problems/find-the-town-judge/) (in/out-degree style)
- (M) [Course Schedule](https://leetcode.com/problems/course-schedule/)
- (M) [Course Schedule II](https://leetcode.com/problems/course-schedule-ii/)
- (H) [Alien Dictionary](https://leetcode.com/problems/alien-dictionary/)
- (H) [Parallel Courses III](https://leetcode.com/problems/parallel-courses-iii/)

---

## 14. Greedy

**Signal phrases:** "minimum number of," "is it possible," interval scheduling, local choice → global optimum

**Approach:** Make the locally optimal choice at each step. Prove correctness by exchange argument. Often paired with sorting.

**Problems:**
- (E) [Assign Cookies](https://leetcode.com/problems/assign-cookies/)
- (E) [Lemonade Change](https://leetcode.com/problems/lemonade-change/)
- (M) [Jump Game](https://leetcode.com/problems/jump-game/)
- (M) [Gas Station](https://leetcode.com/problems/gas-station/)
- (H) [Candy](https://leetcode.com/problems/candy/)
- (H) [Task Scheduler](https://leetcode.com/problems/task-scheduler/)

---

## 15. Bit Manipulation

**Signal phrases:** "find the unique," powers of 2, subsets via bitmask, "without using extra space," XOR tricks

**Approach:** `x & (x-1)` clears the lowest set bit; `x ^ x = 0`; XOR for "find single non-duplicate"; bitmask for state in DP.

**Problems:**
- (E) [Single Number](https://leetcode.com/problems/single-number/)
- (E) [Number of 1 Bits](https://leetcode.com/problems/number-of-1-bits/)
- (M) [Single Number III](https://leetcode.com/problems/single-number-iii/)
- (M) [Sum of Two Integers](https://leetcode.com/problems/sum-of-two-integers/)
- (H) [Maximum XOR of Two Numbers in an Array](https://leetcode.com/problems/maximum-xor-of-two-numbers-in-an-array/)
- (H) [Minimum Number of Work Sessions to Finish the Tasks](https://leetcode.com/problems/minimum-number-of-work-sessions-to-finish-the-tasks/) (bitmask DP)

---

## 16. Sweep Line / Interval Overlap

**Signal phrases:** intervals, meeting rooms, calendar bookings, "max concurrent," peak occupancy, capacity check

**Approach:** Treat each interval as two events (+1 start, −1 end); sort by time; sweep with running count. Difference array for bounded-integer times.

**Problems:**
- (E) [Merge Intervals](https://leetcode.com/problems/merge-intervals/)
- (E) [Meeting Rooms](https://leetcode.com/problems/meeting-rooms/)
- (M) [Meeting Rooms II](https://leetcode.com/problems/meeting-rooms-ii/)
- (M) [Car Pooling](https://leetcode.com/problems/car-pooling/)
- (H) [The Skyline Problem](https://leetcode.com/problems/the-skyline-problem/)
- (H) [Employee Free Time](https://leetcode.com/problems/employee-free-time/)

---

## 17. String Matching (KMP / Rabin-Karp / Z)

**Signal phrases:** "find pattern P in text T," "implement strStr / indexOf," duplicate substring, longest repeated substring

**Approach:** KMP with prefix-function for linear match. Rabin-Karp with rolling hash for finding duplicates / many patterns. Z-array as an alternative.

**Problems:**
- (E) [Find the Index of the First Occurrence in a String](https://leetcode.com/problems/find-the-index-of-the-first-occurrence-in-a-string/)
- (E) [Repeated Substring Pattern](https://leetcode.com/problems/repeated-substring-pattern/)
- (M) [Repeated DNA Sequences](https://leetcode.com/problems/repeated-dna-sequences/)
- (M) [Longest Happy Prefix](https://leetcode.com/problems/longest-happy-prefix/)
- (H) [Shortest Palindrome](https://leetcode.com/problems/shortest-palindrome/)
- (H) [Longest Duplicate Substring](https://leetcode.com/problems/longest-duplicate-substring/)

---

## 18. String DP (Edit Distance / LCS / Wildcard / Regex)

**Signal phrases:** convert string A to string B, longest common subsequence, wildcard / regex matching, interleaving

**Approach:** 2D DP indexed by prefix lengths of the two strings. Base cases: empty prefix. Transitions depend on whether last characters match.

**Problems:**
- (E) [Is Subsequence](https://leetcode.com/problems/is-subsequence/)
- (E) [Longest Common Prefix](https://leetcode.com/problems/longest-common-prefix/)
- (M) [Longest Common Subsequence](https://leetcode.com/problems/longest-common-subsequence/)
- (M) [Delete Operation for Two Strings](https://leetcode.com/problems/delete-operation-for-two-strings/)
- (H) [Edit Distance](https://leetcode.com/problems/edit-distance/)
- (H) [Wildcard Matching](https://leetcode.com/problems/wildcard-matching/)

---

## 19. Palindrome Patterns

**Signal phrases:** palindrome, mirror-symmetric, "longest palindromic," "count palindromic substrings"

**Approach:** Two pointers (validate). Expand-around-center (O(N²) longest / count). Manacher's (O(N) longest). DP for partition-style.

**Problems:**
- (E) [Valid Palindrome](https://leetcode.com/problems/valid-palindrome/)
- (E) [Palindrome Number](https://leetcode.com/problems/palindrome-number/)
- (M) [Longest Palindromic Substring](https://leetcode.com/problems/longest-palindromic-substring/)
- (M) [Palindromic Substrings](https://leetcode.com/problems/palindromic-substrings/)
- (H) [Palindrome Partitioning II](https://leetcode.com/problems/palindrome-partitioning-ii/)
- (H) [Shortest Palindrome](https://leetcode.com/problems/shortest-palindrome/)

---

## 20. Trie (Prefix Tree)

**Signal phrases:** dictionary lookup, autocomplete, prefix search, "find words with prefix," "longest word in dictionary"

**Approach:** N-ary tree where each node is a character; root → leaf = full word. Insert/search in O(L) where L = word length.

**Problems:**
- (E) [Longest Common Prefix](https://leetcode.com/problems/longest-common-prefix/)
- (E) [Implement Trie (Prefix Tree)](https://leetcode.com/problems/implement-trie-prefix-tree/)
- (M) [Design Add and Search Words Data Structure](https://leetcode.com/problems/design-add-and-search-words-data-structure/)
- (M) [Replace Words](https://leetcode.com/problems/replace-words/)
- (H) [Word Search II](https://leetcode.com/problems/word-search-ii/)
- (H) [Concatenated Words](https://leetcode.com/problems/concatenated-words/)

---

## How to Use This Document

1. **Refresh before an interview:** scan signal phrases and approaches for each pattern. 30 seconds per section.
2. **Practice deliberately:** pick a pattern, do all six problems in order (E → H) before moving on.
3. **Cross-pattern fluency:** open `problems-randomized.md` and try to identify the pattern from a disguised prompt — that's the actual interview skill.
4. **Difficulty mix:** start with Easy to internalize the template, then climb. If a Hard feels impenetrable, drop back to Medium and try variations first.
