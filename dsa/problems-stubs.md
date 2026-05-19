# Problem Stubs (Java + Python)

Minimal stubs for each of the 120 problems in `problems-randomized.md`. Each has:
- A Java stub with a `public class ProblemNNN { main }` and hardcoded input variables.
- A Python stub with hardcoded inputs in an `if __name__ == "__main__"` block.

**Workflow:** read the prompt in `problems-randomized.md`, copy the corresponding stub here into a fresh file (e.g., `practice/NNN.java`), implement the TODO, compile and run.

Linked-list / tree / multi-class-design problems include a brief note where the input construction is non-trivial — bring your own `ListNode` / `TreeNode` definition or open the LeetCode link from the answer key.

---

## Problem 1 (E)

```java
public class Problem001 {
    public static void main(String[] args) {
        String s = "A man, a plan, a canal: Panama";
        // TODO: see problems-randomized.md #1
    }
}
```

```python
if __name__ == "__main__":
    s = "A man, a plan, a canal: Panama"
    # TODO: see problems-randomized.md #1
```

---

## Problem 2 (M)

```java
public class Problem002 {
    public static void main(String[] args) {
        String s = "abcabcbb";
        // TODO: see problems-randomized.md #2
    }
}
```

```python
if __name__ == "__main__":
    s = "abcabcbb"
    # TODO: see problems-randomized.md #2
```

---

## Problem 3 (M)

```java
// Note: requires a ListNode class. Build the chain from int[] {1,2,3,4,5} before calling.
public class Problem003 {
    public static void main(String[] args) {
        int[] values = {1, 2, 3, 4, 5};
        // TODO: see problems-randomized.md #3
    }
}
```

```python
if __name__ == "__main__":
    values = [1, 2, 3, 4, 5]
    # TODO: see problems-randomized.md #3 (build a singly linked list from values)
```

---

## Problem 4 (E)

```java
public class Problem004 {
    public static void main(String[] args) {
        int[] nums = {-1, 0, 3, 5, 9, 12};
        int target = 9;
        // TODO: see problems-randomized.md #4
    }
}
```

```python
if __name__ == "__main__":
    nums = [-1, 0, 3, 5, 9, 12]
    target = 9
    # TODO: see problems-randomized.md #4
```

---

## Problem 5 (M)

```java
public class Problem005 {
    public static void main(String[] args) {
        char[][] grid = {
            {'1','1','0','0','0'},
            {'1','1','0','0','0'},
            {'0','0','1','0','0'},
            {'0','0','0','1','1'}
        };
        // TODO: see problems-randomized.md #5
    }
}
```

```python
if __name__ == "__main__":
    grid = [
        ["1","1","0","0","0"],
        ["1","1","0","0","0"],
        ["0","0","1","0","0"],
        ["0","0","0","1","1"],
    ]
    # TODO: see problems-randomized.md #5
```

---

## Problem 6 (H)

```java
public class Problem006 {
    public static void main(String[] args) {
        int n = 4;
        // TODO: see problems-randomized.md #6
    }
}
```

```python
if __name__ == "__main__":
    n = 4
    # TODO: see problems-randomized.md #6
```

---

## Problem 7 (E)

```java
public class Problem007 {
    public static void main(String[] args) {
        int n = 5;
        // TODO: see problems-randomized.md #7
    }
}
```

```python
if __name__ == "__main__":
    n = 5
    # TODO: see problems-randomized.md #7
```

---

## Problem 8 (M)

```java
public class Problem008 {
    public static void main(String[] args) {
        int[] nums = {1, 1, 1, 2, 2, 3};
        int k = 2;
        // TODO: see problems-randomized.md #8
    }
}
```

```python
if __name__ == "__main__":
    nums = [1, 1, 1, 2, 2, 3]
    k = 2
    # TODO: see problems-randomized.md #8
```

---

## Problem 9 (H)

```java
// Class-design problem: implement a data structure with insert/remove/getRandom.
public class Problem009 {
    public static void main(String[] args) {
        // TODO: see problems-randomized.md #9
        // Exercise: insert(1), insert(1), insert(2), getRandom(), remove(1), getRandom()
    }
}
```

```python
if __name__ == "__main__":
    # TODO: see problems-randomized.md #9
    # Exercise: insert(1), insert(1), insert(2), getRandom(), remove(1), getRandom()
    pass
```

---

## Problem 10 (E)

```java
public class Problem010 {
    public static void main(String[] args) {
        int[] nums = {1, 2, 3, 4};
        // TODO: see problems-randomized.md #10
    }
}
```

```python
if __name__ == "__main__":
    nums = [1, 2, 3, 4]
    # TODO: see problems-randomized.md #10
```

---

## Problem 11 (M)

```java
public class Problem011 {
    public static void main(String[] args) {
        int[] temperatures = {73, 74, 75, 71, 69, 72, 76, 73};
        // TODO: see problems-randomized.md #11
    }
}
```

```python
if __name__ == "__main__":
    temperatures = [73, 74, 75, 71, 69, 72, 76, 73]
    # TODO: see problems-randomized.md #11
```

---

## Problem 12 (H)

```java
public class Problem012 {
    public static void main(String[] args) {
        int m = 3;
        int n = 3;
        int[][] positions = {{0,0},{0,1},{1,2},{2,1}};
        // TODO: see problems-randomized.md #12
    }
}
```

```python
if __name__ == "__main__":
    m, n = 3, 3
    positions = [[0,0],[0,1],[1,2],[2,1]]
    # TODO: see problems-randomized.md #12
```

---

## Problem 13 (E)

```java
public class Problem013 {
    public static void main(String[] args) {
        int[][] edges = {{1, 2}, {2, 3}, {4, 2}};
        // TODO: see problems-randomized.md #13
    }
}
```

```python
if __name__ == "__main__":
    edges = [[1, 2], [2, 3], [4, 2]]
    # TODO: see problems-randomized.md #13
```

---

## Problem 14 (M)

```java
public class Problem014 {
    public static void main(String[] args) {
        int[] nums = {2, 3, 1, 1, 4};
        // TODO: see problems-randomized.md #14
    }
}
```

```python
if __name__ == "__main__":
    nums = [2, 3, 1, 1, 4]
    # TODO: see problems-randomized.md #14
```

---

## Problem 15 (H)

```java
public class Problem015 {
    public static void main(String[] args) {
        int[] nums = {3, 10, 5, 25, 2, 8};
        // TODO: see problems-randomized.md #15
    }
}
```

```python
if __name__ == "__main__":
    nums = [3, 10, 5, 25, 2, 8]
    # TODO: see problems-randomized.md #15
```

---

## Problem 16 (E)

```java
public class Problem016 {
    public static void main(String[] args) {
        int[][] intervals = {{1, 3}, {2, 6}, {8, 10}, {15, 18}};
        // TODO: see problems-randomized.md #16
    }
}
```

```python
if __name__ == "__main__":
    intervals = [[1, 3], [2, 6], [8, 10], [15, 18]]
    # TODO: see problems-randomized.md #16
```

---

## Problem 17 (M)

```java
public class Problem017 {
    public static void main(String[] args) {
        String s = "AAAAACCCCCAAAAACCCCCCAAAAAGGGTTT";
        // TODO: see problems-randomized.md #17
    }
}
```

```python
if __name__ == "__main__":
    s = "AAAAACCCCCAAAAACCCCCCAAAAAGGGTTT"
    # TODO: see problems-randomized.md #17
```

---

## Problem 18 (H)

```java
public class Problem018 {
    public static void main(String[] args) {
        String s = "adceb";
        String p = "*a*b";
        // TODO: see problems-randomized.md #18
    }
}
```

```python
if __name__ == "__main__":
    s = "adceb"
    p = "*a*b"
    # TODO: see problems-randomized.md #18
```

---

## Problem 19 (E)

```java
public class Problem019 {
    public static void main(String[] args) {
        String s = "abca";
        // TODO: see problems-randomized.md #19
    }
}
```

```python
if __name__ == "__main__":
    s = "abca"
    # TODO: see problems-randomized.md #19
```

---

## Problem 20 (M)

```java
import java.util.*;
public class Problem020 {
    public static void main(String[] args) {
        List<String> dictionary = Arrays.asList("cat", "bat", "rat");
        String sentence = "the cattle was rattled by the battery";
        // TODO: see problems-randomized.md #20
    }
}
```

```python
if __name__ == "__main__":
    dictionary = ["cat", "bat", "rat"]
    sentence = "the cattle was rattled by the battery"
    # TODO: see problems-randomized.md #20
```

---

## Problem 21 (M)

```java
public class Problem021 {
    public static void main(String[] args) {
        int[] height = {1, 8, 6, 2, 5, 4, 8, 3, 7};
        // TODO: see problems-randomized.md #21
    }
}
```

```python
if __name__ == "__main__":
    height = [1, 8, 6, 2, 5, 4, 8, 3, 7]
    # TODO: see problems-randomized.md #21
```

---

## Problem 22 (H)

```java
public class Problem022 {
    public static void main(String[] args) {
        int[] nums = {1, 3, -1, -3, 5, 3, 6, 7};
        int k = 3;
        // TODO: see problems-randomized.md #22
    }
}
```

```python
if __name__ == "__main__":
    nums = [1, 3, -1, -3, 5, 3, 6, 7]
    k = 3
    # TODO: see problems-randomized.md #22
```

---

## Problem 23 (E)

```java
// Note: requires a ListNode class. Build chain from values; the tail's .next set to nodes[pos] forms the cycle.
public class Problem023 {
    public static void main(String[] args) {
        int[] values = {3, 2, 0, -4};
        int cyclePos = 1;
        // TODO: see problems-randomized.md #23
    }
}
```

```python
if __name__ == "__main__":
    values = [3, 2, 0, -4]
    cycle_pos = 1
    # TODO: see problems-randomized.md #23 (build a singly linked list with cycle)
```

---

## Problem 24 (M)

```java
public class Problem024 {
    public static void main(String[] args) {
        int[] piles = {3, 6, 7, 11};
        int h = 8;
        // TODO: see problems-randomized.md #24
    }
}
```

```python
if __name__ == "__main__":
    piles = [3, 6, 7, 11]
    h = 8
    # TODO: see problems-randomized.md #24
```

---

## Problem 25 (H)

```java
public class Problem025 {
    public static void main(String[] args) {
        int[][] routes = {{1, 2, 7}, {3, 6, 7}};
        int source = 1;
        int target = 6;
        // TODO: see problems-randomized.md #25
    }
}
```

```python
if __name__ == "__main__":
    routes = [[1, 2, 7], [3, 6, 7]]
    source = 1
    target = 6
    # TODO: see problems-randomized.md #25
```

---

## Problem 26 (E)

```java
public class Problem026 {
    public static void main(String[] args) {
        String s = "a1b2";
        // TODO: see problems-randomized.md #26
    }
}
```

```python
if __name__ == "__main__":
    s = "a1b2"
    # TODO: see problems-randomized.md #26
```

---

## Problem 27 (M)

```java
public class Problem027 {
    public static void main(String[] args) {
        int[] nums = {10, 9, 2, 5, 3, 7, 101, 18};
        // TODO: see problems-randomized.md #27
    }
}
```

```python
if __name__ == "__main__":
    nums = [10, 9, 2, 5, 3, 7, 101, 18]
    # TODO: see problems-randomized.md #27
```

---

## Problem 28 (H)

```java
// Note: requires a ListNode class. Build each chain from its values array.
public class Problem028 {
    public static void main(String[] args) {
        int[][] lists = {{1, 4, 5}, {1, 3, 4}, {2, 6}};
        // TODO: see problems-randomized.md #28
    }
}
```

```python
if __name__ == "__main__":
    lists = [[1, 4, 5], [1, 3, 4], [2, 6]]
    # TODO: see problems-randomized.md #28 (build k singly linked lists)
```

---

## Problem 29 (E)

```java
public class Problem029 {
    public static void main(String[] args) {
        int[] nums = {1, 2, 3, 1};
        // TODO: see problems-randomized.md #29
    }
}
```

```python
if __name__ == "__main__":
    nums = [1, 2, 3, 1]
    # TODO: see problems-randomized.md #29
```

---

## Problem 30 (M)

```java
public class Problem030 {
    public static void main(String[] args) {
        int[] nums = {1, 1, 1};
        int k = 2;
        // TODO: see problems-randomized.md #30
    }
}
```

```python
if __name__ == "__main__":
    nums = [1, 1, 1]
    k = 2
    # TODO: see problems-randomized.md #30
```

---

## Problem 31 (H)

```java
public class Problem031 {
    public static void main(String[] args) {
        int[] heights = {2, 1, 5, 6, 2, 3};
        // TODO: see problems-randomized.md #31
    }
}
```

```python
if __name__ == "__main__":
    heights = [2, 1, 5, 6, 2, 3]
    # TODO: see problems-randomized.md #31
```

---

## Problem 32 (E)

```java
public class Problem032 {
    public static void main(String[] args) {
        int n = 6;
        int[][] edges = {{0, 1}, {0, 2}, {3, 5}, {5, 4}, {4, 3}};
        int source = 0;
        int destination = 5;
        // TODO: see problems-randomized.md #32
    }
}
```

```python
if __name__ == "__main__":
    n = 6
    edges = [[0, 1], [0, 2], [3, 5], [5, 4], [4, 3]]
    source = 0
    destination = 5
    # TODO: see problems-randomized.md #32
```

---

## Problem 33 (M)

```java
public class Problem033 {
    public static void main(String[] args) {
        int numCourses = 2;
        int[][] prerequisites = {{1, 0}};
        // TODO: see problems-randomized.md #33
    }
}
```

```python
if __name__ == "__main__":
    num_courses = 2
    prerequisites = [[1, 0]]
    # TODO: see problems-randomized.md #33
```

---

## Problem 34 (H)

```java
public class Problem034 {
    public static void main(String[] args) {
        int[] ratings = {1, 0, 2};
        // TODO: see problems-randomized.md #34
    }
}
```

```python
if __name__ == "__main__":
    ratings = [1, 0, 2]
    # TODO: see problems-randomized.md #34
```

---

## Problem 35 (E)

```java
public class Problem035 {
    public static void main(String[] args) {
        int[] nums = {2, 2, 1};
        // TODO: see problems-randomized.md #35
    }
}
```

```python
if __name__ == "__main__":
    nums = [2, 2, 1]
    # TODO: see problems-randomized.md #35
```

---

## Problem 36 (M)

```java
public class Problem036 {
    public static void main(String[] args) {
        int[][] intervals = {{0, 30}, {5, 10}, {15, 20}};
        // TODO: see problems-randomized.md #36
    }
}
```

```python
if __name__ == "__main__":
    intervals = [[0, 30], [5, 10], [15, 20]]
    # TODO: see problems-randomized.md #36
```

---

## Problem 37 (H)

```java
public class Problem037 {
    public static void main(String[] args) {
        String s = "banana";
        // TODO: see problems-randomized.md #37
    }
}
```

```python
if __name__ == "__main__":
    s = "banana"
    # TODO: see problems-randomized.md #37
```

---

## Problem 38 (E)

```java
public class Problem038 {
    public static void main(String[] args) {
        String s = "abc";
        String t = "ahbgdc";
        // TODO: see problems-randomized.md #38
    }
}
```

```python
if __name__ == "__main__":
    s = "abc"
    t = "ahbgdc"
    # TODO: see problems-randomized.md #38
```

---

## Problem 39 (M)

```java
public class Problem039 {
    public static void main(String[] args) {
        String s = "babad";
        // TODO: see problems-randomized.md #39
    }
}
```

```python
if __name__ == "__main__":
    s = "babad"
    # TODO: see problems-randomized.md #39
```

---

## Problem 40 (H)

```java
public class Problem040 {
    public static void main(String[] args) {
        char[][] board = {
            {'o','a','a','n'},
            {'e','t','a','e'},
            {'i','h','k','r'},
            {'i','f','l','v'}
        };
        String[] words = {"oath", "pea", "eat", "rain"};
        // TODO: see problems-randomized.md #40
    }
}
```

```python
if __name__ == "__main__":
    board = [
        ["o","a","a","n"],
        ["e","t","a","e"],
        ["i","h","k","r"],
        ["i","f","l","v"],
    ]
    words = ["oath", "pea", "eat", "rain"]
    # TODO: see problems-randomized.md #40
```

---

## Problem 41 (H)

```java
public class Problem041 {
    public static void main(String[] args) {
        int[] height = {0, 1, 0, 2, 1, 0, 1, 3, 2, 1, 2, 1};
        // TODO: see problems-randomized.md #41
    }
}
```

```python
if __name__ == "__main__":
    height = [0, 1, 0, 2, 1, 0, 1, 3, 2, 1, 2, 1]
    # TODO: see problems-randomized.md #41
```

---

## Problem 42 (E)

```java
public class Problem042 {
    public static void main(String[] args) {
        int[] nums = {1, 12, -5, -6, 50, 3};
        int k = 4;
        // TODO: see problems-randomized.md #42
    }
}
```

```python
if __name__ == "__main__":
    nums = [1, 12, -5, -6, 50, 3]
    k = 4
    # TODO: see problems-randomized.md #42
```

---

## Problem 43 (M)

```java
public class Problem043 {
    public static void main(String[] args) {
        int[] nums = {1, 3, 4, 2, 2};
        // TODO: see problems-randomized.md #43
    }
}
```

```python
if __name__ == "__main__":
    nums = [1, 3, 4, 2, 2]
    # TODO: see problems-randomized.md #43
```

---

## Problem 44 (H)

```java
public class Problem044 {
    public static void main(String[] args) {
        int[] nums1 = {1, 3};
        int[] nums2 = {2};
        // TODO: see problems-randomized.md #44
    }
}
```

```python
if __name__ == "__main__":
    nums1 = [1, 3]
    nums2 = [2]
    # TODO: see problems-randomized.md #44
```

---

## Problem 45 (E)

```java
public class Problem045 {
    public static void main(String[] args) {
        int[][] image = {{1, 1, 1}, {1, 1, 0}, {1, 0, 1}};
        int sr = 1, sc = 1, color = 2;
        // TODO: see problems-randomized.md #45
    }
}
```

```python
if __name__ == "__main__":
    image = [[1, 1, 1], [1, 1, 0], [1, 0, 1]]
    sr, sc, color = 1, 1, 2
    # TODO: see problems-randomized.md #45
```

---

## Problem 46 (M)

```java
public class Problem046 {
    public static void main(String[] args) {
        int[] nums = {1, 2, 3};
        // TODO: see problems-randomized.md #46
    }
}
```

```python
if __name__ == "__main__":
    nums = [1, 2, 3]
    # TODO: see problems-randomized.md #46
```

---

## Problem 47 (H)

```java
public class Problem047 {
    public static void main(String[] args) {
        String word1 = "horse";
        String word2 = "ros";
        // TODO: see problems-randomized.md #47
    }
}
```

```python
if __name__ == "__main__":
    word1 = "horse"
    word2 = "ros"
    # TODO: see problems-randomized.md #47
```

---

## Problem 48 (E)

```java
// Class-design problem: KthLargest with constructor(int k, int[] nums) and add(int val).
public class Problem048 {
    public static void main(String[] args) {
        int k = 3;
        int[] nums = {4, 5, 8, 2};
        // Exercise: build with (k, nums); then add(3), add(5), add(10), add(9), add(4)
        // TODO: see problems-randomized.md #48
    }
}
```

```python
if __name__ == "__main__":
    k = 3
    nums = [4, 5, 8, 2]
    # Exercise: build with (k, nums); then add(3), add(5), add(10), add(9), add(4)
    # TODO: see problems-randomized.md #48
```

---

## Problem 49 (M)

```java
public class Problem049 {
    public static void main(String[] args) {
        String[] strs = {"eat", "tea", "tan", "ate", "nat", "bat"};
        // TODO: see problems-randomized.md #49
    }
}
```

```python
if __name__ == "__main__":
    strs = ["eat", "tea", "tan", "ate", "nat", "bat"]
    # TODO: see problems-randomized.md #49
```

---

## Problem 50 (H)

```java
public class Problem050 {
    public static void main(String[] args) {
        int[] nums = {1, 2, 1, 2, 6, 7, 5, 1};
        int k = 2;
        // TODO: see problems-randomized.md #50
    }
}
```

```python
if __name__ == "__main__":
    nums = [1, 2, 1, 2, 6, 7, 5, 1]
    k = 2
    # TODO: see problems-randomized.md #50
```

---

## Problem 51 (E)

```java
public class Problem051 {
    public static void main(String[] args) {
        int[] nums1 = {4, 1, 2};
        int[] nums2 = {1, 3, 4, 2};
        // TODO: see problems-randomized.md #51
    }
}
```

```python
if __name__ == "__main__":
    nums1 = [4, 1, 2]
    nums2 = [1, 3, 4, 2]
    # TODO: see problems-randomized.md #51
```

---

## Problem 52 (M)

```java
public class Problem052 {
    public static void main(String[] args) {
        int[][] edges = {{1, 2}, {1, 3}, {2, 3}};
        // TODO: see problems-randomized.md #52
    }
}
```

```python
if __name__ == "__main__":
    edges = [[1, 2], [1, 3], [2, 3]]
    # TODO: see problems-randomized.md #52
```

---

## Problem 53 (H)

```java
public class Problem053 {
    public static void main(String[] args) {
        String[] words = {"wrt", "wrf", "er", "ett", "rftt"};
        // TODO: see problems-randomized.md #53
    }
}
```

```python
if __name__ == "__main__":
    words = ["wrt", "wrf", "er", "ett", "rftt"]
    # TODO: see problems-randomized.md #53
```

---

## Problem 54 (E)

```java
public class Problem054 {
    public static void main(String[] args) {
        int[] bills = {5, 5, 5, 10, 20};
        // TODO: see problems-randomized.md #54
    }
}
```

```python
if __name__ == "__main__":
    bills = [5, 5, 5, 10, 20]
    # TODO: see problems-randomized.md #54
```

---

## Problem 55 (M)

```java
public class Problem055 {
    public static void main(String[] args) {
        int[] nums = {1, 2, 1, 3, 2, 5};
        // TODO: see problems-randomized.md #55
    }
}
```

```python
if __name__ == "__main__":
    nums = [1, 2, 1, 3, 2, 5]
    # TODO: see problems-randomized.md #55
```

---

## Problem 56 (H)

```java
public class Problem056 {
    public static void main(String[] args) {
        int[][] buildings = {{2, 9, 10}, {3, 7, 15}, {5, 12, 12}, {15, 20, 10}, {19, 24, 8}};
        // TODO: see problems-randomized.md #56
    }
}
```

```python
if __name__ == "__main__":
    buildings = [[2, 9, 10], [3, 7, 15], [5, 12, 12], [15, 20, 10], [19, 24, 8]]
    # TODO: see problems-randomized.md #56
```

---

## Problem 57 (E)

```java
public class Problem057 {
    public static void main(String[] args) {
        String haystack = "sadbutsad";
        String needle = "sad";
        // TODO: see problems-randomized.md #57
    }
}
```

```python
if __name__ == "__main__":
    haystack = "sadbutsad"
    needle = "sad"
    # TODO: see problems-randomized.md #57
```

---

## Problem 58 (M)

```java
public class Problem058 {
    public static void main(String[] args) {
        String text1 = "abcde";
        String text2 = "ace";
        // TODO: see problems-randomized.md #58
    }
}
```

```python
if __name__ == "__main__":
    text1 = "abcde"
    text2 = "ace"
    # TODO: see problems-randomized.md #58
```

---

## Problem 59 (H)

```java
public class Problem059 {
    public static void main(String[] args) {
        String s = "aab";
        // TODO: see problems-randomized.md #59
    }
}
```

```python
if __name__ == "__main__":
    s = "aab"
    # TODO: see problems-randomized.md #59
```

---

## Problem 60 (E)

```java
// Class-design problem: Trie with insert(word), search(word), startsWith(prefix).
public class Problem060 {
    public static void main(String[] args) {
        // Exercise: insert("apple"); search("apple") -> true; search("app") -> false; startsWith("app") -> true
        // TODO: see problems-randomized.md #60
    }
}
```

```python
if __name__ == "__main__":
    # Exercise: insert("apple"); search("apple") -> True; search("app") -> False; startsWith("app") -> True
    # TODO: see problems-randomized.md #60
    pass
```

---

## Problem 61 (E)

```java
public class Problem061 {
    public static void main(String[] args) {
        int[] numbers = {2, 7, 11, 15};
        int target = 9;
        // TODO: see problems-randomized.md #61
    }
}
```

```python
if __name__ == "__main__":
    numbers = [2, 7, 11, 15]
    target = 9
    # TODO: see problems-randomized.md #61
```

---

## Problem 62 (M)

```java
public class Problem062 {
    public static void main(String[] args) {
        String s1 = "ab";
        String s2 = "eidbaooo";
        // TODO: see problems-randomized.md #62
    }
}
```

```python
if __name__ == "__main__":
    s1 = "ab"
    s2 = "eidbaooo"
    # TODO: see problems-randomized.md #62
```

---

## Problem 63 (M)

```java
// Note: requires a ListNode class with cycle injection.
public class Problem063 {
    public static void main(String[] args) {
        int[] values = {3, 2, 0, -4};
        int cyclePos = 1;
        // TODO: see problems-randomized.md #63
    }
}
```

```python
if __name__ == "__main__":
    values = [3, 2, 0, -4]
    cycle_pos = 1
    # TODO: see problems-randomized.md #63 (build a singly linked list with cycle)
```

---

## Problem 64 (E)

```java
public class Problem064 {
    public static void main(String[] args) {
        int n = 10;
        int firstBad = 4;
        // TODO: see problems-randomized.md #64
        // Note: simulate the isBadVersion API as a helper: boolean isBadVersion(int v) { return v >= firstBad; }
    }
}
```

```python
if __name__ == "__main__":
    n = 10
    first_bad = 4
    # TODO: see problems-randomized.md #64
    # Note: simulate is_bad_version(v) as: lambda v: v >= first_bad
```

---

## Problem 65 (M)

```java
// Note: requires a Node class with int val and List<Node> neighbors.
public class Problem065 {
    public static void main(String[] args) {
        int[][] adjacency = {{2, 4}, {1, 3}, {2, 4}, {1, 3}};
        // TODO: see problems-randomized.md #65
    }
}
```

```python
if __name__ == "__main__":
    adjacency = [[2, 4], [1, 3], [2, 4], [1, 3]]
    # TODO: see problems-randomized.md #65 (build a graph from the adjacency list)
```

---

## Problem 66 (H)

```java
public class Problem066 {
    public static void main(String[] args) {
        char[][] board = {
            {'5','3','.','.','7','.','.','.','.'},
            {'6','.','.','1','9','5','.','.','.'},
            {'.','9','8','.','.','.','.','6','.'},
            {'8','.','.','.','6','.','.','.','3'},
            {'4','.','.','8','.','3','.','.','1'},
            {'7','.','.','.','2','.','.','.','6'},
            {'.','6','.','.','.','.','2','8','.'},
            {'.','.','.','4','1','9','.','.','5'},
            {'.','.','.','.','8','.','.','7','9'}
        };
        // TODO: see problems-randomized.md #66
    }
}
```

```python
if __name__ == "__main__":
    board = [
        ["5","3",".",".","7",".",".",".","."],
        ["6",".",".","1","9","5",".",".","."],
        [".","9","8",".",".",".",".","6","."],
        ["8",".",".",".","6",".",".",".","3"],
        ["4",".",".","8",".","3",".",".","1"],
        ["7",".",".",".","2",".",".",".","6"],
        [".","6",".",".",".",".","2","8","."],
        [".",".",".","4","1","9",".",".","5"],
        [".",".",".",".","8",".",".","7","9"],
    ]
    # TODO: see problems-randomized.md #66
```

---

## Problem 67 (E)

```java
public class Problem067 {
    public static void main(String[] args) {
        int[] nums = {2, 7, 9, 3, 1};
        // TODO: see problems-randomized.md #67
    }
}
```

```python
if __name__ == "__main__":
    nums = [2, 7, 9, 3, 1]
    # TODO: see problems-randomized.md #67
```

---

## Problem 68 (M)

```java
public class Problem068 {
    public static void main(String[] args) {
        int[][] points = {{1, 3}, {-2, 2}};
        int k = 1;
        // TODO: see problems-randomized.md #68
    }
}
```

```python
if __name__ == "__main__":
    points = [[1, 3], [-2, 2]]
    k = 1
    # TODO: see problems-randomized.md #68
```

---

## Problem 69 (H)

```java
// Class-design problem: LFUCache(int capacity) with get(key) and put(key, value).
public class Problem069 {
    public static void main(String[] args) {
        // Exercise: put(1,1); put(2,2); get(1); put(3,3); get(2); get(3); put(4,4); get(1); get(3); get(4)
        // TODO: see problems-randomized.md #69
    }
}
```

```python
if __name__ == "__main__":
    # Exercise: put(1,1); put(2,2); get(1); put(3,3); get(2); get(3); put(4,4); get(1); get(3); get(4)
    # TODO: see problems-randomized.md #69
    pass
```

---

## Problem 70 (E)

```java
// Class-design problem: NumArray(int[] nums) with sumRange(left, right).
public class Problem070 {
    public static void main(String[] args) {
        int[] nums = {-2, 0, 3, -5, 2, -1};
        // Exercise: sumRange(0,2) -> 1; sumRange(2,5) -> -1; sumRange(0,5) -> -3
        // TODO: see problems-randomized.md #70
    }
}
```

```python
if __name__ == "__main__":
    nums = [-2, 0, 3, -5, 2, -1]
    # Exercise: sumRange(0,2) -> 1; sumRange(2,5) -> -1; sumRange(0,5) -> -3
    # TODO: see problems-randomized.md #70
```

---

## Problem 71 (M)

```java
// Class-design problem: StockSpanner() with next(int price).
public class Problem071 {
    public static void main(String[] args) {
        int[] prices = {100, 80, 60, 70, 60, 75, 85};
        // Expected spans: 1, 1, 1, 2, 1, 4, 6
        // TODO: see problems-randomized.md #71
    }
}
```

```python
if __name__ == "__main__":
    prices = [100, 80, 60, 70, 60, 75, 85]
    # Expected spans: 1, 1, 1, 2, 1, 4, 6
    # TODO: see problems-randomized.md #71
```

---

## Problem 72 (H)

```java
public class Problem072 {
    public static void main(String[] args) {
        int[][] grid = {{0, 2}, {1, 3}};
        // TODO: see problems-randomized.md #72
    }
}
```

```python
if __name__ == "__main__":
    grid = [[0, 2], [1, 3]]
    # TODO: see problems-randomized.md #72
```

---

## Problem 73 (E)

```java
public class Problem073 {
    public static void main(String[] args) {
        int n = 3;
        int[][] trust = {{1, 3}, {2, 3}};
        // TODO: see problems-randomized.md #73
    }
}
```

```python
if __name__ == "__main__":
    n = 3
    trust = [[1, 3], [2, 3]]
    # TODO: see problems-randomized.md #73
```

---

## Problem 74 (M)

```java
public class Problem074 {
    public static void main(String[] args) {
        int[] gas = {1, 2, 3, 4, 5};
        int[] cost = {3, 4, 5, 1, 2};
        // TODO: see problems-randomized.md #74
    }
}
```

```python
if __name__ == "__main__":
    gas = [1, 2, 3, 4, 5]
    cost = [3, 4, 5, 1, 2]
    # TODO: see problems-randomized.md #74
```

---

## Problem 75 (H)

```java
public class Problem075 {
    public static void main(String[] args) {
        int[] tasks = {1, 2, 3};
        int sessionTime = 3;
        // TODO: see problems-randomized.md #75
    }
}
```

```python
if __name__ == "__main__":
    tasks = [1, 2, 3]
    session_time = 3
    # TODO: see problems-randomized.md #75
```

---

## Problem 76 (E)

```java
public class Problem076 {
    public static void main(String[] args) {
        int[][] intervals = {{0, 30}, {5, 10}, {15, 20}};
        // TODO: see problems-randomized.md #76
    }
}
```

```python
if __name__ == "__main__":
    intervals = [[0, 30], [5, 10], [15, 20]]
    # TODO: see problems-randomized.md #76
```

---

## Problem 77 (M)

```java
public class Problem077 {
    public static void main(String[] args) {
        String s = "cbaebabacd";
        String p = "abc";
        // TODO: see problems-randomized.md #77
    }
}
```

```python
if __name__ == "__main__":
    s = "cbaebabacd"
    p = "abc"
    # TODO: see problems-randomized.md #77
```

---

## Problem 78 (H)

```java
public class Problem078 {
    public static void main(String[] args) {
        String s = "rabbbit";
        String t = "rabbit";
        // TODO: see problems-randomized.md #78
    }
}
```

```python
if __name__ == "__main__":
    s = "rabbbit"
    t = "rabbit"
    # TODO: see problems-randomized.md #78
```

---

## Problem 79 (E)

```java
public class Problem079 {
    public static void main(String[] args) {
        int x = 121;
        // TODO: see problems-randomized.md #79
    }
}
```

```python
if __name__ == "__main__":
    x = 121
    # TODO: see problems-randomized.md #79
```

---

## Problem 80 (M)

```java
// Class-design problem: WordDictionary with addWord(word) and search(word) ('.' = any char).
public class Problem080 {
    public static void main(String[] args) {
        // Exercise: add("bad"); add("dad"); add("mad"); search("pad")->F; search("bad")->T; search(".ad")->T; search("b..")->T
        // TODO: see problems-randomized.md #80
    }
}
```

```python
if __name__ == "__main__":
    # Exercise: add("bad"); add("dad"); add("mad"); search("pad")->F; search("bad")->T; search(".ad")->T; search("b..")->T
    # TODO: see problems-randomized.md #80
    pass
```

---

## Problem 81 (M)

```java
public class Problem081 {
    public static void main(String[] args) {
        int[] nums = {-1, 0, 1, 2, -1, -4};
        // TODO: see problems-randomized.md #81
    }
}
```

```python
if __name__ == "__main__":
    nums = [-1, 0, 1, 2, -1, -4]
    # TODO: see problems-randomized.md #81
```

---

## Problem 82 (H)

```java
public class Problem082 {
    public static void main(String[] args) {
        String s = "barfoothefoobarman";
        String[] words = {"foo", "bar"};
        // TODO: see problems-randomized.md #82
    }
}
```

```python
if __name__ == "__main__":
    s = "barfoothefoobarman"
    words = ["foo", "bar"]
    # TODO: see problems-randomized.md #82
```

---

## Problem 83 (E)

```java
// Note: requires a ListNode class.
public class Problem083 {
    public static void main(String[] args) {
        int[] values = {1, 2, 3, 4, 5};
        // TODO: see problems-randomized.md #83
    }
}
```

```python
if __name__ == "__main__":
    values = [1, 2, 3, 4, 5]
    # TODO: see problems-randomized.md #83 (build a singly linked list)
```

---

## Problem 84 (M)

```java
public class Problem084 {
    public static void main(String[] args) {
        int[] nums = {1, 2, 3, 1};
        // TODO: see problems-randomized.md #84
    }
}
```

```python
if __name__ == "__main__":
    nums = [1, 2, 3, 1]
    # TODO: see problems-randomized.md #84
```

---

## Problem 85 (H)

```java
public class Problem085 {
    public static void main(String[] args) {
        int[][] board = {{4, 1, 2}, {5, 0, 3}};
        // TODO: see problems-randomized.md #85
    }
}
```

```python
if __name__ == "__main__":
    board = [[4, 1, 2], [5, 0, 3]]
    # TODO: see problems-randomized.md #85
```

---

## Problem 86 (E)

```java
// Note: requires a TreeNode class.
public class Problem086 {
    public static void main(String[] args) {
        Integer[] tree = {1, 2, 3, null, 5};
        // TODO: see problems-randomized.md #86 (build the tree from level-order array)
    }
}
```

```python
if __name__ == "__main__":
    tree = [1, 2, 3, None, 5]
    # TODO: see problems-randomized.md #86 (build the tree from level-order list)
```

---

## Problem 87 (M)

```java
import java.util.*;
public class Problem087 {
    public static void main(String[] args) {
        String s = "leetcode";
        List<String> wordDict = Arrays.asList("leet", "code");
        // TODO: see problems-randomized.md #87
    }
}
```

```python
if __name__ == "__main__":
    s = "leetcode"
    word_dict = ["leet", "code"]
    # TODO: see problems-randomized.md #87
```

---

## Problem 88 (H)

```java
public class Problem088 {
    public static void main(String[] args) {
        int[] nums = {1, 3, -1, -3, 5, 3, 6, 7};
        int k = 3;
        // TODO: see problems-randomized.md #88
    }
}
```

```python
if __name__ == "__main__":
    nums = [1, 3, -1, -3, 5, 3, 6, 7]
    k = 3
    # TODO: see problems-randomized.md #88
```

---

## Problem 89 (E)

```java
public class Problem089 {
    public static void main(String[] args) {
        String s = "leetcode";
        // TODO: see problems-randomized.md #89
    }
}
```

```python
if __name__ == "__main__":
    s = "leetcode"
    # TODO: see problems-randomized.md #89
```

---

## Problem 90 (M)

```java
public class Problem090 {
    public static void main(String[] args) {
        int[] nums = {23, 2, 4, 6, 7};
        int k = 6;
        // TODO: see problems-randomized.md #90
    }
}
```

```python
if __name__ == "__main__":
    nums = [23, 2, 4, 6, 7]
    k = 6
    # TODO: see problems-randomized.md #90
```

---

## Problem 91 (H)

```java
public class Problem091 {
    public static void main(String[] args) {
        int[] arr = {3, 1, 2, 4};
        // TODO: see problems-randomized.md #91
    }
}
```

```python
if __name__ == "__main__":
    arr = [3, 1, 2, 4]
    # TODO: see problems-randomized.md #91
```

---

## Problem 92 (E)

```java
public class Problem092 {
    public static void main(String[] args) {
        int[][] isConnected = {{1, 1, 0}, {1, 1, 0}, {0, 0, 1}};
        // TODO: see problems-randomized.md #92
    }
}
```

```python
if __name__ == "__main__":
    is_connected = [[1, 1, 0], [1, 1, 0], [0, 0, 1]]
    # TODO: see problems-randomized.md #92
```

---

## Problem 93 (M)

```java
public class Problem093 {
    public static void main(String[] args) {
        int numCourses = 4;
        int[][] prerequisites = {{1, 0}, {2, 0}, {3, 1}, {3, 2}};
        // TODO: see problems-randomized.md #93
    }
}
```

```python
if __name__ == "__main__":
    num_courses = 4
    prerequisites = [[1, 0], [2, 0], [3, 1], [3, 2]]
    # TODO: see problems-randomized.md #93
```

---

## Problem 94 (H)

```java
public class Problem094 {
    public static void main(String[] args) {
        char[] tasks = {'A', 'A', 'A', 'B', 'B', 'B'};
        int n = 2;
        // TODO: see problems-randomized.md #94
    }
}
```

```python
if __name__ == "__main__":
    tasks = ['A', 'A', 'A', 'B', 'B', 'B']
    n = 2
    # TODO: see problems-randomized.md #94
```

---

## Problem 95 (E)

```java
public class Problem095 {
    public static void main(String[] args) {
        int n = 11;
        // TODO: see problems-randomized.md #95
    }
}
```

```python
if __name__ == "__main__":
    n = 11
    # TODO: see problems-randomized.md #95
```

---

## Problem 96 (M)

```java
public class Problem096 {
    public static void main(String[] args) {
        int[][] trips = {{2, 1, 5}, {3, 3, 7}};
        int capacity = 4;
        // TODO: see problems-randomized.md #96
    }
}
```

```python
if __name__ == "__main__":
    trips = [[2, 1, 5], [3, 3, 7]]
    capacity = 4
    # TODO: see problems-randomized.md #96
```

---

## Problem 97 (E)

```java
public class Problem097 {
    public static void main(String[] args) {
        String s = "abab";
        // TODO: see problems-randomized.md #97
    }
}
```

```python
if __name__ == "__main__":
    s = "abab"
    # TODO: see problems-randomized.md #97
```

---

## Problem 98 (M)

```java
public class Problem098 {
    public static void main(String[] args) {
        String word1 = "sea";
        String word2 = "eat";
        // TODO: see problems-randomized.md #98
    }
}
```

```python
if __name__ == "__main__":
    word1 = "sea"
    word2 = "eat"
    # TODO: see problems-randomized.md #98
```

---

## Problem 99 (M)

```java
public class Problem099 {
    public static void main(String[] args) {
        String s = "abc";
        // TODO: see problems-randomized.md #99
    }
}
```

```python
if __name__ == "__main__":
    s = "abc"
    # TODO: see problems-randomized.md #99
```

---

## Problem 100 (H)

```java
public class Problem100 {
    public static void main(String[] args) {
        String[] words = {"cat", "cats", "catsdogcats", "dog", "dogcatsdog", "hippopotamuses", "rat", "ratcatdogcat"};
        // TODO: see problems-randomized.md #100
    }
}
```

```python
if __name__ == "__main__":
    words = ["cat", "cats", "catsdogcats", "dog", "dogcatsdog", "hippopotamuses", "rat", "ratcatdogcat"]
    # TODO: see problems-randomized.md #100
```

---

## Problem 101 (H)

```java
public class Problem101 {
    public static void main(String[] args) {
        String s = "ADOBECODEBANC";
        String t = "ABC";
        // TODO: see problems-randomized.md #101
    }
}
```

```python
if __name__ == "__main__":
    s = "ADOBECODEBANC"
    t = "ABC"
    # TODO: see problems-randomized.md #101
```

---

## Problem 102 (E)

```java
public class Problem102 {
    public static void main(String[] args) {
        int[] prices = {7, 1, 5, 3, 6, 4};
        // TODO: see problems-randomized.md #102
    }
}
```

```python
if __name__ == "__main__":
    prices = [7, 1, 5, 3, 6, 4]
    # TODO: see problems-randomized.md #102
```

---

## Problem 103 (E)

```java
public class Problem103 {
    public static void main(String[] args) {
        int n = 19;
        // TODO: see problems-randomized.md #103
    }
}
```

```python
if __name__ == "__main__":
    n = 19
    # TODO: see problems-randomized.md #103
```

---

## Problem 104 (H)

```java
public class Problem104 {
    public static void main(String[] args) {
        int[] nums = {7, 2, 5, 10, 8};
        int k = 2;
        // TODO: see problems-randomized.md #104
    }
}
```

```python
if __name__ == "__main__":
    nums = [7, 2, 5, 10, 8]
    k = 2
    # TODO: see problems-randomized.md #104
```

---

## Problem 105 (E)

```java
// Note: requires a TreeNode class.
public class Problem105 {
    public static void main(String[] args) {
        Integer[] tree = {3, 9, 20, null, null, 15, 7};
        // TODO: see problems-randomized.md #105 (build the tree from level-order array)
    }
}
```

```python
if __name__ == "__main__":
    tree = [3, 9, 20, None, None, 15, 7]
    # TODO: see problems-randomized.md #105 (build the tree from level-order list)
```

---

## Problem 106 (M)

```java
public class Problem106 {
    public static void main(String[] args) {
        int[] candidates = {2, 3, 6, 7};
        int target = 7;
        // TODO: see problems-randomized.md #106
    }
}
```

```python
if __name__ == "__main__":
    candidates = [2, 3, 6, 7]
    target = 7
    # TODO: see problems-randomized.md #106
```

---

## Problem 107 (H)

```java
public class Problem107 {
    public static void main(String[] args) {
        int[] nums = {3, 1, 5, 8};
        // TODO: see problems-randomized.md #107
    }
}
```

```python
if __name__ == "__main__":
    nums = [3, 1, 5, 8]
    # TODO: see problems-randomized.md #107
```

---

## Problem 108 (E)

```java
public class Problem108 {
    public static void main(String[] args) {
        int[] stones = {2, 7, 4, 1, 8, 1};
        // TODO: see problems-randomized.md #108
    }
}
```

```python
if __name__ == "__main__":
    stones = [2, 7, 4, 1, 8, 1]
    # TODO: see problems-randomized.md #108
```

---

## Problem 109 (M)

```java
public class Problem109 {
    public static void main(String[] args) {
        int[] nums = {100, 4, 200, 1, 3, 2};
        // TODO: see problems-randomized.md #109
    }
}
```

```python
if __name__ == "__main__":
    nums = [100, 4, 200, 1, 3, 2]
    # TODO: see problems-randomized.md #109
```

---

## Problem 110 (H)

```java
public class Problem110 {
    public static void main(String[] args) {
        int[] nums = {-2, 5, -1};
        int lower = -2;
        int upper = 2;
        // TODO: see problems-randomized.md #110
    }
}
```

```python
if __name__ == "__main__":
    nums = [-2, 5, -1]
    lower = -2
    upper = 2
    # TODO: see problems-randomized.md #110
```

---

## Problem 111 (E)

```java
public class Problem111 {
    public static void main(String[] args) {
        String[] ops = {"5", "2", "C", "D", "+"};
        // TODO: see problems-randomized.md #111
    }
}
```

```python
if __name__ == "__main__":
    ops = ["5", "2", "C", "D", "+"]
    # TODO: see problems-randomized.md #111
```

---

## Problem 112 (M)

```java
import java.util.*;
public class Problem112 {
    public static void main(String[] args) {
        List<List<String>> accounts = Arrays.asList(
            Arrays.asList("John", "john@mail.com", "john_work@mail.com"),
            Arrays.asList("John", "john@mail.com", "john_home@mail.com"),
            Arrays.asList("Mary", "mary@mail.com")
        );
        // TODO: see problems-randomized.md #112
    }
}
```

```python
if __name__ == "__main__":
    accounts = [
        ["John", "john@mail.com", "john_work@mail.com"],
        ["John", "john@mail.com", "john_home@mail.com"],
        ["Mary", "mary@mail.com"],
    ]
    # TODO: see problems-randomized.md #112
```

---

## Problem 113 (H)

```java
public class Problem113 {
    public static void main(String[] args) {
        int n = 5;
        int[][] relations = {{1, 5}, {2, 5}, {3, 5}, {3, 4}, {4, 5}};
        int[] time = {1, 2, 3, 4, 5};
        // TODO: see problems-randomized.md #113
    }
}
```

```python
if __name__ == "__main__":
    n = 5
    relations = [[1, 5], [2, 5], [3, 5], [3, 4], [4, 5]]
    time = [1, 2, 3, 4, 5]
    # TODO: see problems-randomized.md #113
```

---

## Problem 114 (E)

```java
public class Problem114 {
    public static void main(String[] args) {
        int[] g = {1, 2, 3};
        int[] s = {1, 1};
        // TODO: see problems-randomized.md #114
    }
}
```

```python
if __name__ == "__main__":
    g = [1, 2, 3]
    s = [1, 1]
    # TODO: see problems-randomized.md #114
```

---

## Problem 115 (M)

```java
public class Problem115 {
    public static void main(String[] args) {
        int a = 1;
        int b = 2;
        // TODO: see problems-randomized.md #115
    }
}
```

```python
if __name__ == "__main__":
    a = 1
    b = 2
    # TODO: see problems-randomized.md #115
```

---

## Problem 116 (H)

```java
// Note: requires an Interval class. Each employee has a sorted list of non-overlapping intervals.
public class Problem116 {
    public static void main(String[] args) {
        int[][][] schedule = {
            {{1, 2}, {5, 6}},
            {{1, 3}},
            {{4, 10}}
        };
        // TODO: see problems-randomized.md #116
    }
}
```

```python
if __name__ == "__main__":
    schedule = [
        [[1, 2], [5, 6]],
        [[1, 3]],
        [[4, 10]],
    ]
    # TODO: see problems-randomized.md #116
```

---

## Problem 117 (H)

```java
public class Problem117 {
    public static void main(String[] args) {
        String s = "level";
        // TODO: see problems-randomized.md #117
    }
}
```

```python
if __name__ == "__main__":
    s = "level"
    # TODO: see problems-randomized.md #117
```

---

## Problem 118 (H)

```java
public class Problem118 {
    public static void main(String[] args) {
        String s = "aab";
        String p = "c*a*b";
        // TODO: see problems-randomized.md #118
    }
}
```

```python
if __name__ == "__main__":
    s = "aab"
    p = "c*a*b"
    # TODO: see problems-randomized.md #118
```

---

## Problem 119 (H)

```java
public class Problem119 {
    public static void main(String[] args) {
        String[] words = {"abcd", "dcba", "lls", "s", "sssll"};
        // TODO: see problems-randomized.md #119
    }
}
```

```python
if __name__ == "__main__":
    words = ["abcd", "dcba", "lls", "s", "sssll"]
    # TODO: see problems-randomized.md #119
```

---

## Problem 120 (E)

```java
public class Problem120 {
    public static void main(String[] args) {
        String[] words = {"w", "wo", "wor", "worl", "world"};
        // TODO: see problems-randomized.md #120
    }
}
```

```python
if __name__ == "__main__":
    words = ["w", "wo", "wor", "worl", "world"]
    # TODO: see problems-randomized.md #120
```
