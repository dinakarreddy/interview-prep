# Randomized Practice Problems

Problems below are listed in random order with disguised wordings. The goal: read the prompt, **identify the underlying pattern**, then solve.

- Difficulty marker only — no pattern label inline.
- **Answer Key** at the bottom maps problem # → original name, LeetCode link, and primary pattern.
- Try to commit to a pattern before peeking. That's the actual interview skill.

---

## Problems

**1. (E)** Given a sentence, ignore casing and non-letter characters; determine whether the resulting sequence of characters reads identically when reversed.

**2. (M)** Walk through a string of characters. Find the length of the longest contiguous run where every character is distinct.

**3. (M)** You have a chain of nodes where each points to the next, and each node holds a value. Rearrange the chain so the order alternates between the first and last unvisited node (front, back, front+1, back-1, ...). Mutate the chain in place.

**4. (E)** You're handed a list of integers known to be sorted. Return the position of a query value, or report its absence. Aim for sub-linear time.

**5. (M)** A 2D map shows either land or water at each cell. Two land cells belong to the same region if they share an edge. How many separate regions exist?

**6. (H)** Place N markers on an N-by-N board such that no two markers share a row, column, or diagonal. Enumerate every valid arrangement.

**7. (E)** A worker can move up a staircase by climbing one rung or two rungs at a time. Given the height, count the distinct sequences of moves.

**8. (M)** Given a long list of items each appearing some number of times, return the K items whose occurrence counts are greatest.

**9. (H)** Design a data store that allows insertion, deletion, and retrieval of an arbitrary stored item all in constant time on average — even if duplicate items are allowed.

**10. (E)** As you walk down a number line, accumulate a running total. Return, for each position, the sum of all positions visited so far.

**11. (M)** For each day's high temperature, report how many days you must wait before seeing a strictly warmer day. If no such day exists, report zero.

**12. (H)** On a grid where you can add land tiles over time, after each addition, report the current number of separate landmasses.

**13. (E)** A constellation has a central star connected to many others by single edges. Identify the central star given only the edge list.

**14. (M)** You start at the leftmost square. Each square tells you the maximum number of squares forward you may hop. Determine whether you can reach the last square.

**15. (H)** Given two integers, find the pair of indices in an array whose elements produce the largest possible bitwise mismatch (i.e., XOR).

**16. (E)** Several time ranges may overlap. Combine any that touch or overlap and return the resulting collapsed list of ranges.

**17. (M)** A long DNA strand contains many length-10 substrings; some appear more than once. Return every length-10 substring that recurs.

**18. (H)** Two strings: one literal, one a pattern containing `?` (matches any single char) and `*` (matches any sequence). Decide if the pattern matches the entire literal string.

**19. (E)** Almost-palindrome: given a string of lowercase letters, decide whether deleting at most one character produces a mirror-readable string.

**20. (M)** A dictionary contains short root words. Given a sentence, replace every word that starts with one of these roots by its shortest matching root.

**21. (M)** A row of vertical bars, each with a defined height. Pair two bars so that the rectangle they form (width × shorter height) is as large as possible.

**22. (H)** A sliding view of width K passes over an array left to right. For each window position, report the maximum value visible.

**23. (E)** A chain of nodes may or may not loop back on itself. Decide whether such a loop exists.

**24. (M)** A monkey can swallow `r` items from a pile per hour. Given several piles and a time limit, what's the minimum `r` that empties every pile within the limit?

**25. (H)** You step onto a transit network where each line stops at fixed stations. Given a source and destination, find the minimum number of line transfers (initial boarding counts as one).

**26. (E)** Given a string, return every variant obtained by independently toggling the case of each letter (digits remain).

**27. (M)** A strictly ascending pick from an unordered list — what's the longest such pick you can make while preserving original order? (Items need not be adjacent.)

**28. (H)** Merge K already-sorted chains of nodes into a single sorted chain.

**29. (E)** Decide whether any element of a list appears more than once.

**30. (M)** Given an array of integers and a target sum, count the number of contiguous segments whose elements add up to that target.

**31. (H)** Given a row of vertical columns of varying heights, find the largest axis-aligned rectangle that fits entirely below the column tops.

**32. (E)** A friend graph: people are linked by friendship edges. Decide whether person A can reach person B by walking the friendship links.

**33. (M)** A pile of prerequisite pairs describes which courses must precede which. Decide whether you can finish every course.

**34. (H)** Children stand in a row, each with a rating. Hand out sweets so every child gets at least one, and any child with a higher rating than an adjacent neighbor receives strictly more sweets. Return the minimum total sweets.

**35. (E)** Every number in an array appears twice — except for one. Find the loner. Constant extra space.

**36. (M)** Several meetings are scheduled with start and end times. What is the minimum number of conference rooms you must reserve to host them all?

**37. (H)** Among all contiguous substrings of a given string, find the longest one that appears at least twice (overlaps allowed). Return any one such substring.

**38. (E)** Given two strings `s` and `t`, decide whether `s` can be obtained from `t` by deleting some (possibly zero) characters while preserving order.

**39. (M)** Find the longest contiguous run of characters in a given string that reads identically when reversed.

**40. (H)** A grid of letters and a list of target words. Return every target word that can be traced by starting on some cell and stepping to adjacent (4-directional) cells without revisiting any cell during a single trace.

**41. (H)** A skyline of vertical columns with varying heights stands in the rain. After the rain stops, water settles on top. Compute the total volume trapped between columns.

**42. (E)** A streaming reading of a number per time tick over a window of length K. Report the largest possible mean of any window.

**43. (M)** An array of n+1 integers contains values from 1 to n, with exactly one value appearing more than once. Locate the repeated value without modifying the array and in O(1) extra space.

**44. (H)** Two sorted arrays of sizes m and n. Find the value at the middle index of their merged sort in O(log(min(m, n))).

**45. (E)** Given a grid of pixel colors, you click on one pixel. Repaint every pixel connected to it (4-directionally) and matching its original color with a new color.

**46. (M)** Given a set of distinct values, list every subset (including empty and the full set).

**47. (H)** Two strings. Find the minimum number of single-character edits (insert, delete, replace) to transform one into the other.

**48. (E)** Implement a class that, after each insertion to a stream of numbers, returns the K-th largest number seen so far.

**49. (M)** Bucket a list of strings such that strings within the same bucket are rearrangements of each other.

**50. (H)** Given an array of integers and a window size K, return the three non-overlapping windows of size K whose summed values are jointly maximized. (Lexicographically smallest tuple of starts on tie.)

**51. (E)** Given an array of distinct integers and a subset of them, for each subset element report the first element to its right in the original array that's strictly larger; report -1 if none.

**52. (M)** Edges connect pairs of nodes to form what was originally a tree, except one extra edge was added creating a single cycle. Return that extra edge (if multiple answers exist, return the one appearing last in the input).

**53. (H)** Words in an unknown language are sorted by an unknown lexicographic order on a known alphabet. Given the sorted list, deduce a valid ordering of letters (or report inconsistency).

**54. (E)** A cashier serves a queue. Each customer pays with a $5, $10, or $20 bill for a $5 item, expecting correct change. Starting with no money, can every customer receive correct change in turn?

**55. (M)** Two integers are given. Return a list containing the two integers in the input that each appear exactly once (all others appear exactly twice).

**56. (H)** Given an array of buildings each defined by left x, right x, and height, compute the silhouette they collectively make against the sky as a sequence of key points (x, y).

**57. (E)** Implement substring lookup: given a haystack and a needle, return the smallest start index where the needle appears in the haystack, or -1.

**58. (M)** Two strings. Find the length of the longest series of characters that appears in both strings while preserving relative order (not necessarily contiguous in either).

**59. (H)** Given a string, split it into pieces such that every piece is a mirror-readable string. Find the minimum number of cuts required.

**60. (E)** Build a tree-based data structure that supports inserting words and asking whether a queried word has been previously inserted.

**61. (E)** A sorted list of numbers and a target sum. Return the 1-indexed positions of the two numbers (left then right) that sum to the target.

**62. (M)** Given strings `p` and `s`, decide whether `s` contains any rearrangement of `p` as a contiguous window.

**63. (M)** A chain of nodes loops back on itself somewhere. Identify the node where the loop begins.

**64. (E)** A monotonic boolean function is true on a contiguous suffix of {1, 2, ..., n}. Find the smallest index where it becomes true, minimizing calls to the function.

**65. (M)** A graph node and a reference to it are given. Produce an isomorphic deep copy of the entire connected component.

**66. (H)** A 9x9 board partially filled with digits 1–9. Complete the board such that every row, column, and each of the nine 3x3 boxes contains every digit exactly once.

**67. (E)** A row of houses, each with a stash of cash, sits on a street. You can't take from two adjacent houses on the same night. Maximize total cash taken.

**68. (M)** Given a list of 2D coordinates and an integer K, return the K points nearest to the origin (Euclidean distance).

**69. (H)** Build a data store with insert/access operations such that the least-frequently-used item is the one evicted when capacity is exceeded; ties broken by least recently used. All operations average O(1).

**70. (E)** Given an array, build a data structure that supports many subsequent queries of "what is the sum of values between index i and j inclusive?"

**71. (M)** A market gives you a real-time stream of daily prices for a stock. For each new price, return the number of consecutive prior days (including today) where today's price is at least as high as that day's price.

**72. (H)** A 2D elevation grid models a region. A swimmer starts at the top-left and wants to reach the bottom-right. The swimmer can only enter a cell once the water level reaches at least its elevation. Find the lowest water level that allows the swimmer to make the trip.

**73. (E)** In a directed graph, exactly one node has incoming edges from every other node, and no outgoing edges. Find it.

**74. (M)** A row of gas stations is arranged in a circle. Each station offers some fuel; driving from station i to i+1 costs some fuel. Find a starting station from which you can complete the loop, or report impossibility.

**75. (H)** A list of integer tasks must be scheduled. Each session can handle at most H units of total work. A task may be split across sessions, but no session can have unused capacity sit idle past its end. Using bitmask state, find the minimum number of sessions to finish every task.

**76. (E)** Several meetings have start and end times. Decide whether one person could possibly attend every meeting.

**77. (M)** Given a string and a target pattern of the same length, find all start indices in the string where a permutation of the pattern occurs.

**78. (H)** Given two strings, count the number of ways to obtain one as a subsequence of the other (with order preserved, characters may be skipped). Result may be very large.

**79. (E)** A non-negative integer is read forward, then reversed. Decide whether the original and the reverse are identical (without converting to a string).

**80. (M)** A dictionary supports insertion of words and queries with `.` as a wildcard matching any single character. Implement both operations efficiently.

**81. (M)** An unsorted integer list. Return every distinct triple of values whose sum is zero, with no triple repeated.

**82. (H)** Given a string and a list of equal-length words, find every starting position in the string where a window of length (number of words × word length) is exactly the concatenation of every given word, each used exactly once and in any order.

**83. (E)** A single-linked chain of nodes. Return the middle node; if even length, return the second of the two middles.

**84. (M)** A function `f(n)` reaches its maximum somewhere on an array `arr` (defined on integers) where `arr[i] != arr[i+1]`. The array's neighbors of any local maximum are strictly smaller. Find any index of such a local maximum in O(log n).

**85. (H)** A 2x3 puzzle with five numbered tiles and a blank. Find the minimum number of slides to reach the goal configuration `[[1,2,3],[4,5,0]]` from an arbitrary start, or report unsolvable.

**86. (E)** Given a root of a binary tree, return every root-to-leaf path as a string of node values joined by `->`.

**87. (M)** Given a string and a dictionary of words, decide whether the string can be segmented into a sequence of dictionary words (words may repeat).

**88. (H)** A continuous stream of numbers arrives. At any point, return the median of the latest window of size K (or all numbers seen, depending on variant — here use sliding window of K).

**89. (E)** A string of lowercase letters. Return the index of the first character that does not repeat anywhere in the string, or -1.

**90. (M)** An array of non-negative integers and an integer K. Decide whether any contiguous segment has length at least 2 and a sum that's a multiple of K.

**91. (H)** Given an array of integers, return the sum (over all contiguous segments) of the minimum value in each segment. Modulo a large prime.

**92. (E)** A nation has provinces, each made up of cities. A connectivity matrix tells you which cities are directly linked. Two cities sharing a chain of links are in the same province. Count the provinces.

**93. (M)** Given a list of (course, prerequisite) pairs, output any course completion order satisfying every prerequisite.

**94. (H)** Tasks labeled A through Z, each appearing many times. After running any task, you must wait at least N intervals before re-running that same task (idle intervals are allowed). Find the minimum total time to finish every task.

**95. (E)** Count the number of 1-bits in the binary representation of an unsigned integer.

**96. (M)** A bus has finite seating. Trips are scheduled, each declaring how many passengers board at location A and leave at location B. Decide whether every trip can be honored without ever exceeding capacity.

**97. (E)** A string `s` may or may not be formable by concatenating copies of a non-empty proper substring of itself. Decide which.

**98. (M)** Two strings. Compute the minimum number of single-character deletions across either string to make them equal.

**99. (M)** Count the total number of mirror-readable contiguous substrings in a given string. Each occurrence counts separately, even if equal substrings overlap.

**100. (H)** Given a list of distinct words, return every word that's the concatenation of at least two other words in the list.

**101. (H)** Given a long string `s` and a short string `t`, find the smallest window of `s` (by length) containing every character of `t` (including multiplicities). Empty string if none.

**102. (E)** Given a series of daily prices, find the maximum profit from buying once and selling later on a single day.

**103. (E)** Repeatedly replace a positive integer `n` by the sum of squares of its decimal digits. Decide whether this process eventually reaches 1.

**104. (H)** Given an array of non-negative integers and an integer K, partition the array into K non-empty contiguous segments such that the largest segment sum is as small as possible.

**105. (E)** Given a binary tree, return the values level-by-level from top to bottom.

**106. (M)** Given a candidate pool of positive integers (no duplicates) and a target sum, enumerate every distinct multiset of candidates (any candidate may be picked any number of times) whose sum is exactly the target.

**107. (H)** A row of balloons each has a point value. Bursting balloon `i` earns you `left * value[i] * right`, where `left` and `right` are the values of the balloons currently adjacent to `i`. Bursting consumes the balloon. Maximize total earnings.

**108. (E)** Given an array of positive integers representing stone weights, repeatedly smash the two heaviest stones; if equal, both are destroyed; if not, the difference is returned to the pile. Return the last stone's weight, or 0.

**109. (M)** Given an unsorted integer array, find the length of the longest run of consecutive integers (e.g., 4,2,3,1 → 4). Must run in O(n).

**110. (H)** Given an integer array and bounds (lower, upper), count the number of contiguous segments whose sum falls inclusively within [lower, upper].

**111. (E)** Implement a stack that processes a series of operations: integers push the integer; `+` pushes the sum of the top two; `D` pushes double the top; `C` pops the top. Return the final sum on the stack.

**112. (M)** A list of accounts each contains a name and emails. Accounts sharing any email belong to the same person (even transitively). Output merged accounts, each with deduplicated, sorted emails.

**113. (H)** Each course requires a duration to complete and has prerequisites. You can run any number of courses in parallel as long as prerequisites are done. Compute the minimum time to finish every course.

**114. (E)** A child plucks cookies from a tray. Each child has a minimum cookie size they'll accept; each cookie has a size. Maximize the number of satisfied children.

**115. (M)** Add two integers without using `+` or `-`.

**116. (H)** Several employees each have a list of non-overlapping work intervals (sorted). Return the list of common free intervals shared by every employee, sorted.

**117. (H)** Given a string, return the longest proper prefix that is also a suffix (a prefix that does not span the entire string).

**118. (H)** Given a literal string and a pattern with `.` (any single char) and `*` (zero or more of the previous char), decide whether the pattern fully matches the literal string.

**119. (H)** A list of distinct words. Return every pair (i, j) such that words[i] + words[j] reads identically forwards and backwards.

**120. (E)** A dictionary of words. Find the longest word that can be assembled one letter at a time from shorter words in the same dictionary. Ties broken lexicographically.

---

## Answer Key

| # | Original Problem | Pattern | LeetCode |
|---|---|---|---|
| 1 | Valid Palindrome | Two Pointers | https://leetcode.com/problems/valid-palindrome/ |
| 2 | Longest Substring Without Repeating Characters | Sliding Window | https://leetcode.com/problems/longest-substring-without-repeating-characters/ |
| 3 | Reorder List | Fast & Slow Pointers | https://leetcode.com/problems/reorder-list/ |
| 4 | Binary Search | Binary Search | https://leetcode.com/problems/binary-search/ |
| 5 | Number of Islands | BFS / DFS | https://leetcode.com/problems/number-of-islands/ |
| 6 | N-Queens | Backtracking | https://leetcode.com/problems/n-queens/ |
| 7 | Climbing Stairs | Dynamic Programming | https://leetcode.com/problems/climbing-stairs/ |
| 8 | Top K Frequent Elements | Heap | https://leetcode.com/problems/top-k-frequent-elements/ |
| 9 | Insert Delete GetRandom O(1) — Duplicates allowed | Hash Map | https://leetcode.com/problems/insert-delete-getrandom-o1-duplicates-allowed/ |
| 10 | Running Sum of 1d Array | Prefix Sum | https://leetcode.com/problems/running-sum-of-1d-array/ |
| 11 | Daily Temperatures | Monotonic Stack | https://leetcode.com/problems/daily-temperatures/ |
| 12 | Number of Islands II | Union-Find | https://leetcode.com/problems/number-of-islands-ii/ |
| 13 | Find Center of Star Graph | Topological Sort / Graph | https://leetcode.com/problems/find-center-of-star-graph/ |
| 14 | Jump Game | Greedy | https://leetcode.com/problems/jump-game/ |
| 15 | Maximum XOR of Two Numbers in an Array | Bit Manipulation (Trie) | https://leetcode.com/problems/maximum-xor-of-two-numbers-in-an-array/ |
| 16 | Merge Intervals | Sweep Line / Intervals | https://leetcode.com/problems/merge-intervals/ |
| 17 | Repeated DNA Sequences | String Matching (Rolling Hash) | https://leetcode.com/problems/repeated-dna-sequences/ |
| 18 | Wildcard Matching | String DP | https://leetcode.com/problems/wildcard-matching/ |
| 19 | Valid Palindrome II | Palindrome | https://leetcode.com/problems/valid-palindrome-ii/ |
| 20 | Replace Words | Trie | https://leetcode.com/problems/replace-words/ |
| 21 | Container With Most Water | Two Pointers | https://leetcode.com/problems/container-with-most-water/ |
| 22 | Sliding Window Maximum | Sliding Window (Deque) | https://leetcode.com/problems/sliding-window-maximum/ |
| 23 | Linked List Cycle | Fast & Slow Pointers | https://leetcode.com/problems/linked-list-cycle/ |
| 24 | Koko Eating Bananas | Binary Search on Answer | https://leetcode.com/problems/koko-eating-bananas/ |
| 25 | Bus Routes | BFS | https://leetcode.com/problems/bus-routes/ |
| 26 | Letter Case Permutation | Backtracking | https://leetcode.com/problems/letter-case-permutation/ |
| 27 | Longest Increasing Subsequence | Dynamic Programming | https://leetcode.com/problems/longest-increasing-subsequence/ |
| 28 | Merge k Sorted Lists | Heap | https://leetcode.com/problems/merge-k-sorted-lists/ |
| 29 | Contains Duplicate | Hash Set | https://leetcode.com/problems/contains-duplicate/ |
| 30 | Subarray Sum Equals K | Prefix Sum + Hash Map | https://leetcode.com/problems/subarray-sum-equals-k/ |
| 31 | Largest Rectangle in Histogram | Monotonic Stack | https://leetcode.com/problems/largest-rectangle-in-histogram/ |
| 32 | Find if Path Exists in Graph | Union-Find / BFS | https://leetcode.com/problems/find-if-path-exists-in-graph/ |
| 33 | Course Schedule | Topological Sort | https://leetcode.com/problems/course-schedule/ |
| 34 | Candy | Greedy | https://leetcode.com/problems/candy/ |
| 35 | Single Number | Bit Manipulation (XOR) | https://leetcode.com/problems/single-number/ |
| 36 | Meeting Rooms II | Sweep Line / Heap | https://leetcode.com/problems/meeting-rooms-ii/ |
| 37 | Longest Duplicate Substring | String Matching (Rolling Hash + Binary Search) | https://leetcode.com/problems/longest-duplicate-substring/ |
| 38 | Is Subsequence | Two Pointers / String DP | https://leetcode.com/problems/is-subsequence/ |
| 39 | Longest Palindromic Substring | Palindrome (Expand Around Center) | https://leetcode.com/problems/longest-palindromic-substring/ |
| 40 | Word Search II | Trie + Backtracking | https://leetcode.com/problems/word-search-ii/ |
| 41 | Trapping Rain Water | Two Pointers / Monotonic Stack | https://leetcode.com/problems/trapping-rain-water/ |
| 42 | Maximum Average Subarray I | Sliding Window | https://leetcode.com/problems/maximum-average-subarray-i/ |
| 43 | Find the Duplicate Number | Fast & Slow Pointers | https://leetcode.com/problems/find-the-duplicate-number/ |
| 44 | Median of Two Sorted Arrays | Binary Search | https://leetcode.com/problems/median-of-two-sorted-arrays/ |
| 45 | Flood Fill | BFS / DFS | https://leetcode.com/problems/flood-fill/ |
| 46 | Subsets | Backtracking | https://leetcode.com/problems/subsets/ |
| 47 | Edit Distance | String DP | https://leetcode.com/problems/edit-distance/ |
| 48 | Kth Largest Element in a Stream | Heap | https://leetcode.com/problems/kth-largest-element-in-a-stream/ |
| 49 | Group Anagrams | Hash Map | https://leetcode.com/problems/group-anagrams/ |
| 50 | Maximum Sum of 3 Non-Overlapping Subarrays | Prefix Sum + DP | https://leetcode.com/problems/maximum-sum-of-3-non-overlapping-subarrays/ |
| 51 | Next Greater Element I | Monotonic Stack | https://leetcode.com/problems/next-greater-element-i/ |
| 52 | Redundant Connection | Union-Find | https://leetcode.com/problems/redundant-connection/ |
| 53 | Alien Dictionary | Topological Sort | https://leetcode.com/problems/alien-dictionary/ |
| 54 | Lemonade Change | Greedy | https://leetcode.com/problems/lemonade-change/ |
| 55 | Single Number III | Bit Manipulation (XOR) | https://leetcode.com/problems/single-number-iii/ |
| 56 | The Skyline Problem | Sweep Line + Heap | https://leetcode.com/problems/the-skyline-problem/ |
| 57 | Find the Index of the First Occurrence in a String | String Matching | https://leetcode.com/problems/find-the-index-of-the-first-occurrence-in-a-string/ |
| 58 | Longest Common Subsequence | String DP | https://leetcode.com/problems/longest-common-subsequence/ |
| 59 | Palindrome Partitioning II | Palindrome + DP | https://leetcode.com/problems/palindrome-partitioning-ii/ |
| 60 | Implement Trie (Prefix Tree) | Trie | https://leetcode.com/problems/implement-trie-prefix-tree/ |
| 61 | Two Sum II — Input Array Is Sorted | Two Pointers | https://leetcode.com/problems/two-sum-ii-input-array-is-sorted/ |
| 62 | Permutation in String | Sliding Window | https://leetcode.com/problems/permutation-in-string/ |
| 63 | Linked List Cycle II | Fast & Slow Pointers | https://leetcode.com/problems/linked-list-cycle-ii/ |
| 64 | First Bad Version | Binary Search | https://leetcode.com/problems/first-bad-version/ |
| 65 | Clone Graph | BFS / DFS | https://leetcode.com/problems/clone-graph/ |
| 66 | Sudoku Solver | Backtracking | https://leetcode.com/problems/sudoku-solver/ |
| 67 | House Robber | Dynamic Programming | https://leetcode.com/problems/house-robber/ |
| 68 | K Closest Points to Origin | Heap | https://leetcode.com/problems/k-closest-points-to-origin/ |
| 69 | LFU Cache | Hash Map + Doubly Linked List | https://leetcode.com/problems/lfu-cache/ |
| 70 | Range Sum Query — Immutable | Prefix Sum | https://leetcode.com/problems/range-sum-query-immutable/ |
| 71 | Online Stock Span | Monotonic Stack | https://leetcode.com/problems/online-stock-span/ |
| 72 | Swim in Rising Water | Union-Find / Binary Search | https://leetcode.com/problems/swim-in-rising-water/ |
| 73 | Find the Town Judge | Graph / In-Out Degree | https://leetcode.com/problems/find-the-town-judge/ |
| 74 | Gas Station | Greedy | https://leetcode.com/problems/gas-station/ |
| 75 | Minimum Number of Work Sessions to Finish the Tasks | Bitmask DP | https://leetcode.com/problems/minimum-number-of-work-sessions-to-finish-the-tasks/ |
| 76 | Meeting Rooms | Sweep Line / Sorting | https://leetcode.com/problems/meeting-rooms/ |
| 77 | Find All Anagrams in a String | Sliding Window / String Matching | https://leetcode.com/problems/find-all-anagrams-in-a-string/ |
| 78 | Distinct Subsequences | String DP | https://leetcode.com/problems/distinct-subsequences/ |
| 79 | Palindrome Number | Palindrome | https://leetcode.com/problems/palindrome-number/ |
| 80 | Design Add and Search Words Data Structure | Trie | https://leetcode.com/problems/design-add-and-search-words-data-structure/ |
| 81 | 3Sum | Two Pointers | https://leetcode.com/problems/3sum/ |
| 82 | Substring with Concatenation of All Words | Sliding Window + Hash Map | https://leetcode.com/problems/substring-with-concatenation-of-all-words/ |
| 83 | Middle of the Linked List | Fast & Slow Pointers | https://leetcode.com/problems/middle-of-the-linked-list/ |
| 84 | Find Peak Element | Binary Search | https://leetcode.com/problems/find-peak-element/ |
| 85 | Sliding Puzzle | BFS | https://leetcode.com/problems/sliding-puzzle/ |
| 86 | Binary Tree Paths | Backtracking / DFS | https://leetcode.com/problems/binary-tree-paths/ |
| 87 | Word Break | Dynamic Programming + Trie | https://leetcode.com/problems/word-break/ |
| 88 | Sliding Window Median | Heap (Two Heaps) | https://leetcode.com/problems/sliding-window-median/ |
| 89 | First Unique Character in a String | Hash Map | https://leetcode.com/problems/first-unique-character-in-a-string/ |
| 90 | Continuous Subarray Sum | Prefix Sum + Hash Map | https://leetcode.com/problems/continuous-subarray-sum/ |
| 91 | Sum of Subarray Minimums | Monotonic Stack | https://leetcode.com/problems/sum-of-subarray-minimums/ |
| 92 | Number of Provinces | Union-Find / DFS | https://leetcode.com/problems/number-of-provinces/ |
| 93 | Course Schedule II | Topological Sort | https://leetcode.com/problems/course-schedule-ii/ |
| 94 | Task Scheduler | Greedy / Heap | https://leetcode.com/problems/task-scheduler/ |
| 95 | Number of 1 Bits | Bit Manipulation | https://leetcode.com/problems/number-of-1-bits/ |
| 96 | Car Pooling | Sweep Line / Difference Array | https://leetcode.com/problems/car-pooling/ |
| 97 | Repeated Substring Pattern | String Matching (KMP) | https://leetcode.com/problems/repeated-substring-pattern/ |
| 98 | Delete Operation for Two Strings | String DP (LCS) | https://leetcode.com/problems/delete-operation-for-two-strings/ |
| 99 | Palindromic Substrings | Palindrome (Expand Around Center) | https://leetcode.com/problems/palindromic-substrings/ |
| 100 | Concatenated Words | Trie / DP | https://leetcode.com/problems/concatenated-words/ |
| 101 | Minimum Window Substring | Sliding Window + Hash Map | https://leetcode.com/problems/minimum-window-substring/ |
| 102 | Best Time to Buy and Sell Stock | Sliding Window / Greedy | https://leetcode.com/problems/best-time-to-buy-and-sell-stock/ |
| 103 | Happy Number | Fast & Slow Pointers | https://leetcode.com/problems/happy-number/ |
| 104 | Split Array Largest Sum | Binary Search on Answer | https://leetcode.com/problems/split-array-largest-sum/ |
| 105 | Binary Tree Level Order Traversal | BFS | https://leetcode.com/problems/binary-tree-level-order-traversal/ |
| 106 | Combination Sum | Backtracking | https://leetcode.com/problems/combination-sum/ |
| 107 | Burst Balloons | Interval DP | https://leetcode.com/problems/burst-balloons/ |
| 108 | Last Stone Weight | Heap | https://leetcode.com/problems/last-stone-weight/ |
| 109 | Longest Consecutive Sequence | Hash Set | https://leetcode.com/problems/longest-consecutive-sequence/ |
| 110 | Count of Range Sum | Prefix Sum + Merge Sort / BIT | https://leetcode.com/problems/count-of-range-sum/ |
| 111 | Baseball Game | Stack | https://leetcode.com/problems/baseball-game/ |
| 112 | Accounts Merge | Union-Find / DFS | https://leetcode.com/problems/accounts-merge/ |
| 113 | Parallel Courses III | Topological Sort + DP | https://leetcode.com/problems/parallel-courses-iii/ |
| 114 | Assign Cookies | Greedy | https://leetcode.com/problems/assign-cookies/ |
| 115 | Sum of Two Integers | Bit Manipulation | https://leetcode.com/problems/sum-of-two-integers/ |
| 116 | Employee Free Time | Sweep Line / Heap | https://leetcode.com/problems/employee-free-time/ |
| 117 | Longest Happy Prefix | String Matching (KMP) | https://leetcode.com/problems/longest-happy-prefix/ |
| 118 | Regular Expression Matching | String DP | https://leetcode.com/problems/regular-expression-matching/ |
| 119 | Palindrome Pairs | Trie / Palindrome | https://leetcode.com/problems/palindrome-pairs/ |
| 120 | Longest Word in Dictionary | Trie / Hash Set | https://leetcode.com/problems/longest-word-in-dictionary/ |

---

## How to Use This File

1. **Pick a random problem number.** Read the prompt — do NOT scroll to the answer key first.
2. **Spend 1–2 minutes naming the pattern out loud** (or in writing). Match it against the patterns in `patterns-overview.md`.
3. **Sketch the approach** — what's the state, what's the transition, what data structure?
4. **Open the LeetCode link** from the answer key and code it up. Only check the original name to confirm you didn't confuse two similar-looking problems.
5. **If you mis-identified the pattern**, note it down — those are the failure modes worth drilling.

Goal: across 30–50 problems, **pattern-recognition accuracy should climb to ~80–90%** before you peek.
