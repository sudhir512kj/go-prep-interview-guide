# Algorithms in Golang

## Introduction

Algorithms are step-by-step procedures for solving problems. Understanding algorithms and their complexity is essential for writing efficient code.

### Algorithm Analysis

**Time Complexity**: How runtime grows with input size
**Space Complexity**: How memory usage grows with input size

**Big O Notation:**
- O(1) < O(log n) < O(n) < O(n log n) < O(n²) < O(2ⁿ) < O(n!)

**Best, Average, Worst Case:**
- Best: Minimum time for any input
- Average: Expected time for random input
- Worst: Maximum time for any input

### Choosing the Right Algorithm

1. **Small datasets**: Simple algorithms (bubble sort) may be fine
2. **Large datasets**: Efficient algorithms (merge sort, quick sort) essential
3. **Nearly sorted data**: Insertion sort performs well
4. **Memory constraints**: In-place algorithms preferred
5. **Stability required**: Merge sort, not quick sort

## Sorting Algorithms

### Understanding Sorting

Sorting arranges elements in a specific order (ascending/descending). It's fundamental to many algorithms and applications.

**Stability**: A stable sort maintains relative order of equal elements
**In-place**: Sorts without requiring extra space proportional to input size

**Comparison of Sorting Algorithms:**

| Algorithm      | Best       | Average    | Worst      | Space | Stable | In-place |
|---------------|------------|------------|------------|-------|--------|----------|
| Bubble Sort   | O(n)       | O(n²)      | O(n²)      | O(1)  | Yes    | Yes      |
| Selection Sort| O(n²)      | O(n²)      | O(n²)      | O(1)  | No     | Yes      |
| Insertion Sort| O(n)       | O(n²)      | O(n²)      | O(1)  | Yes    | Yes      |
| Merge Sort    | O(n log n) | O(n log n) | O(n log n) | O(n)  | Yes    | No       |
| Quick Sort    | O(n log n) | O(n log n) | O(n²)      | O(log n)| No   | Yes      |
| Heap Sort     | O(n log n) | O(n log n) | O(n log n) | O(1)  | No     | Yes      |

### Bubble Sort

**Concept**: Repeatedly swap adjacent elements if they're in wrong order. Largest element "bubbles" to the end.

**When to use**: Educational purposes, nearly sorted data, small datasets

```go
func bubbleSort(arr []int) {
    n := len(arr)
    for i := 0; i < n-1; i++ {
        swapped := false
        for j := 0; j < n-i-1; j++ {
            if arr[j] > arr[j+1] {
                arr[j], arr[j+1] = arr[j+1], arr[j]
                swapped = true
            }
        }
        // Optimization: if no swaps, array is sorted
        if !swapped {
            break
        }
    }
}
// Time: O(n²), Space: O(1)
// Best case O(n) when array is already sorted
```

### Selection Sort

**Concept**: Find minimum element and place it at beginning. Repeat for remaining array.

**When to use**: Small datasets, when memory writes are expensive (fewer swaps than bubble sort)

```go
func selectionSort(arr []int) {
    n := len(arr)
    for i := 0; i < n-1; i++ {
        minIdx := i
        for j := i + 1; j < n; j++ {
            if arr[j] < arr[minIdx] {
                minIdx = j
            }
        }
        arr[i], arr[minIdx] = arr[minIdx], arr[i]
    }
}
// Time: O(n²), Space: O(1)
// Always O(n²) - doesn't adapt to input
```

### Insertion Sort

**Concept**: Build sorted array one element at a time by inserting each element into its correct position.

**When to use**: Small datasets, nearly sorted data, online sorting (elements arrive one at a time)

```go
func insertionSort(arr []int) {
    for i := 1; i < len(arr); i++ {
        key := arr[i]
        j := i - 1
        // Shift elements greater than key to the right
        for j >= 0 && arr[j] > key {
            arr[j+1] = arr[j]
            j--
        }
        arr[j+1] = key
    }
}
// Time: O(n²) worst/average, O(n) best, Space: O(1)
// Efficient for small or nearly sorted arrays
```

### Merge Sort

**Concept**: Divide array into halves, recursively sort them, then merge sorted halves.

**When to use**: Large datasets, need stable sort, guaranteed O(n log n), linked lists

**Advantages:**
- Guaranteed O(n log n) time
- Stable sort
- Good for external sorting (data doesn't fit in memory)

**Disadvantages:**
- Requires O(n) extra space
- Slower than quick sort in practice for arrays

```go
func mergeSort(arr []int) []int {
    if len(arr) <= 1 {
        return arr
    }
    mid := len(arr) / 2
    left := mergeSort(arr[:mid])
    right := mergeSort(arr[mid:])
    return merge(left, right)
}

func merge(left, right []int) []int {
    result := make([]int, 0, len(left)+len(right))
    i, j := 0, 0
    
    for i < len(left) && j < len(right) {
        if left[i] <= right[j] {
            result = append(result, left[i])
            i++
        } else {
            result = append(result, right[j])
            j++
        }
    }
    result = append(result, left[i:]...)
    result = append(result, right[j:]...)
    return result
}
// Time: O(n log n), Space: O(n)
// Stable, not in-place
// Recurrence: T(n) = 2T(n/2) + O(n)
```

### Quick Sort

**Concept**: Pick pivot, partition array so elements < pivot are left, > pivot are right. Recursively sort partitions.

**When to use**: General purpose sorting, average case performance matters, in-place sorting needed

**Pivot Selection Strategies:**
1. First/Last element: Simple but O(n²) on sorted data
2. Random: Good average case
3. Median-of-three: Better for partially sorted data

**Advantages:**
- Fast in practice (good cache locality)
- In-place (O(log n) stack space)
- Average O(n log n)

**Disadvantages:**
- Worst case O(n²)
- Not stable
- Recursive (stack overflow risk for large n)

```go
func quickSort(arr []int, low, high int) {
    if low < high {
        pi := partition(arr, low, high)
        quickSort(arr, low, pi-1)
        quickSort(arr, pi+1, high)
    }
}

func partition(arr []int, low, high int) int {
    pivot := arr[high]
    i := low - 1
    
    for j := low; j < high; j++ {
        if arr[j] < pivot {
            i++
            arr[i], arr[j] = arr[j], arr[i]
        }
    }
    arr[i+1], arr[high] = arr[high], arr[i+1]
    return i + 1
}
// Time: O(n log n) average, O(n²) worst, Space: O(log n)
// Not stable, in-place
// Worst case when pivot is always smallest/largest
```

### Heap Sort

**Concept**: Build max heap, repeatedly extract maximum and place at end.

**When to use**: Guaranteed O(n log n), in-place sorting, memory constrained

**Advantages:**
- Guaranteed O(n log n)
- In-place
- No worst case like quick sort

**Disadvantages:**
- Not stable
- Slower than quick sort in practice
- Poor cache locality

```go
func heapSort(arr []int) {
    n := len(arr)
    
    // Build max heap
    for i := n/2 - 1; i >= 0; i-- {
        heapify(arr, n, i)
    }
    
    // Extract elements from heap
    for i := n - 1; i > 0; i-- {
        arr[0], arr[i] = arr[i], arr[0]
        heapify(arr, i, 0)
    }
}

func heapify(arr []int, n, i int) {
    largest := i
    left := 2*i + 1
    right := 2*i + 2
    
    if left < n && arr[left] > arr[largest] {
        largest = left
    }
    if right < n && arr[right] > arr[largest] {
        largest = right
    }
    if largest != i {
        arr[i], arr[largest] = arr[largest], arr[i]
        heapify(arr, n, largest)
    }
}
// Time: O(n log n), Space: O(1)
// Not stable, in-place
// Build heap: O(n), Extract max n times: O(n log n)
```

## Searching Algorithms

### Linear Search

**Concept**: Check each element sequentially until found or end reached.

**When to use**: Unsorted data, small datasets, searching once

```go
func linearSearch(arr []int, target int) int {
    for i, v := range arr {
        if v == target {
            return i
        }
    }
    return -1
}
// Time: O(n), Space: O(1)
// Works on unsorted data
// Best case: O(1) when element is first
```

### Binary Search

**Concept**: Repeatedly divide sorted array in half, eliminating half of remaining elements.

**Prerequisites**: Array must be sorted

**When to use**: Sorted data, frequent searches, large datasets

**Applications:**
- Finding element in sorted array
- Finding insertion position
- Finding first/last occurrence
- Finding peak element
- Square root calculation

```go
func binarySearch(arr []int, target int) int {
    left, right := 0, len(arr)-1
    
    for left <= right {
        mid := left + (right-left)/2
        if arr[mid] == target {
            return mid
        }
        if arr[mid] < target {
            left = mid + 1
        } else {
            right = mid - 1
        }
    }
    return -1
}
// Time: O(log n), Space: O(1)
// Requires sorted array
// Eliminates half of search space each iteration

// Binary search variants:

// Find first occurrence
func binarySearchFirst(arr []int, target int) int {
    left, right := 0, len(arr)-1
    result := -1
    
    for left <= right {
        mid := left + (right-left)/2
        if arr[mid] == target {
            result = mid
            right = mid - 1 // Continue searching left
        } else if arr[mid] < target {
            left = mid + 1
        } else {
            right = mid - 1
        }
    }
    return result
}

// Find last occurrence
func binarySearchLast(arr []int, target int) int {
    left, right := 0, len(arr)-1
    result := -1
    
    for left <= right {
        mid := left + (right-left)/2
        if arr[mid] == target {
            result = mid
            left = mid + 1 // Continue searching right
        } else if arr[mid] < target {
            left = mid + 1
        } else {
            right = mid - 1
        }
    }
    return result
}
```

### Binary Search (Recursive)

```go
func binarySearchRecursive(arr []int, target, left, right int) int {
    if left > right {
        return -1
    }
    mid := left + (right-left)/2
    if arr[mid] == target {
        return mid
    }
    if arr[mid] < target {
        return binarySearchRecursive(arr, target, mid+1, right)
    }
    return binarySearchRecursive(arr, target, left, mid-1)
}
```

## Recursion

### Understanding Recursion

Recursion is when a function calls itself. Every recursive function needs:
1. **Base case**: Condition to stop recursion
2. **Recursive case**: Function calls itself with modified input
3. **Progress**: Each call moves toward base case

**When to use recursion:**
- Problem has recursive structure (tree traversal, divide-and-conquer)
- Cleaner than iterative solution
- Stack space not a concern

**Recursion vs Iteration:**
- Recursion: More elegant, uses call stack, risk of stack overflow
- Iteration: More efficient, explicit stack/loop, harder to write for some problems

### Fibonacci

**Naive Recursion**: Exponential time due to repeated calculations

```go
func fibonacci(n int) int {
    if n <= 1 {
        return n
    }
    return fibonacci(n-1) + fibonacci(n-2)
}
// Time: O(2ⁿ), Space: O(n) for call stack
// Recalculates same values many times

// Optimized with memoization (top-down DP)
func fibonacciMemo(n int, memo map[int]int) int {
    if n <= 1 {
        return n
    }
    if val, exists := memo[n]; exists {
        return val
    }
    memo[n] = fibonacciMemo(n-1, memo) + fibonacciMemo(n-2, memo)
    return memo[n]
}
// Time: O(n), Space: O(n)
// Each value calculated once
```

### Factorial

```go
func factorial(n int) int {
    if n <= 1 {
        return 1
    }
    return n * factorial(n-1)
}
```

### Power

**Concept**: Calculate base^exp efficiently using divide-and-conquer

```go
func power(base, exp int) int {
    if exp == 0 {
        return 1
    }
    if exp%2 == 0 {
        half := power(base, exp/2)
        return half * half // Square the half
    }
    return base * power(base, exp-1)
}
// Time: O(log n), Space: O(log n)
// Much better than O(n) iterative multiplication
```

## Dynamic Programming

### Understanding DP

Dynamic Programming solves problems by breaking them into overlapping subproblems and storing results to avoid recomputation.

**Key Characteristics:**
1. **Optimal Substructure**: Optimal solution contains optimal solutions to subproblems
2. **Overlapping Subproblems**: Same subproblems solved multiple times

**Approaches:**
1. **Top-Down (Memoization)**: Recursive with caching
2. **Bottom-Up (Tabulation)**: Iterative, fill table

**Steps to Solve DP:**
1. Define subproblems
2. Find recurrence relation
3. Identify base cases
4. Determine computation order
5. Optimize space if possible

### Fibonacci (DP)

**Bottom-up approach**: Build solution iteratively

```go
func fibonacciDP(n int) int {
    if n <= 1 {
        return n
    }
    dp := make([]int, n+1)
    dp[0], dp[1] = 0, 1
    for i := 2; i <= n; i++ {
        dp[i] = dp[i-1] + dp[i-2]
    }
    return dp[n]
}
// Time: O(n), Space: O(n)

// Space optimized: only need last two values
func fibonacciOptimized(n int) int {
    if n <= 1 {
        return n
    }
    prev, curr := 0, 1
    for i := 2; i <= n; i++ {
        prev, curr = curr, prev+curr
    }
    return curr
}
// Time: O(n), Space: O(1)
```

### Longest Common Subsequence

**Problem**: Find longest subsequence common to two sequences.

**Recurrence**:
- If chars match: LCS[i][j] = 1 + LCS[i-1][j-1]
- Else: LCS[i][j] = max(LCS[i-1][j], LCS[i][j-1])

```go
func longestCommonSubsequence(text1, text2 string) int {
    m, n := len(text1), len(text2)
    dp := make([][]int, m+1)
    for i := range dp {
        dp[i] = make([]int, n+1)
    }
    
    for i := 1; i <= m; i++ {
        for j := 1; j <= n; j++ {
            if text1[i-1] == text2[j-1] {
                dp[i][j] = dp[i-1][j-1] + 1
            } else {
                dp[i][j] = max(dp[i-1][j], dp[i][j-1])
            }
        }
    }
    return dp[m][n]
}

func max(a, b int) int {
    if a > b {
        return a
    }
    return b
}
// Time: O(m*n), Space: O(m*n)
// Can be optimized to O(min(m,n)) space
```

### Knapsack Problem (0/1)

**Problem**: Given weights and values of items, maximize value in knapsack of capacity W.

**Recurrence**:
- If weight[i] <= w: dp[i][w] = max(value[i] + dp[i-1][w-weight[i]], dp[i-1][w])
- Else: dp[i][w] = dp[i-1][w]

```go
func knapsack(weights, values []int, capacity int) int {
    n := len(weights)
    dp := make([][]int, n+1)
    for i := range dp {
        dp[i] = make([]int, capacity+1)
    }
    
    for i := 1; i <= n; i++ {
        for w := 1; w <= capacity; w++ {
            if weights[i-1] <= w {
                dp[i][w] = max(
                    values[i-1]+dp[i-1][w-weights[i-1]],
                    dp[i-1][w],
                )
            } else {
                dp[i][w] = dp[i-1][w]
            }
        }
    }
    return dp[n][capacity]
}
// Time: O(n*W), Space: O(n*W)
// Pseudo-polynomial time (depends on W value)
```

### Coin Change

**Problem**: Find minimum coins needed to make amount.

**Recurrence**: dp[i] = min(dp[i], dp[i-coin] + 1) for each coin

```go
func coinChange(coins []int, amount int) int {
    dp := make([]int, amount+1)
    for i := 1; i <= amount; i++ {
        dp[i] = amount + 1
    }
    dp[0] = 0
    
    for i := 1; i <= amount; i++ {
        for _, coin := range coins {
            if coin <= i {
                dp[i] = min(dp[i], dp[i-coin]+1)
            }
        }
    }
    
    if dp[amount] > amount {
        return -1
    }
    return dp[amount]
}

func min(a, b int) int {
    if a < b {
        return a
    }
    return b
}
// Time: O(amount * len(coins)), Space: O(amount)
// Classic DP problem demonstrating optimal substructure
```

### Longest Increasing Subsequence

**Problem**: Find length of longest strictly increasing subsequence.

**Approach 1**: DP O(n²)
**Approach 2**: Binary search O(n log n)

```go
func lengthOfLIS(nums []int) int {
    if len(nums) == 0 {
        return 0
    }
    dp := make([]int, len(nums))
    for i := range dp {
        dp[i] = 1
    }
    
    maxLen := 1
    for i := 1; i < len(nums); i++ {
        for j := 0; j < i; j++ {
            if nums[i] > nums[j] {
                dp[i] = max(dp[i], dp[j]+1)
            }
        }
        maxLen = max(maxLen, dp[i])
    }
    return maxLen
}
// Time: O(n²), Space: O(n)
// Can be optimized to O(n log n) using binary search
```

## Greedy Algorithms

### Understanding Greedy

Greedy algorithms make locally optimal choice at each step, hoping to find global optimum.

**When Greedy Works:**
- Problem has greedy choice property
- Problem has optimal substructure
- Local optimum leads to global optimum

**Greedy vs DP:**
- Greedy: Makes choice without looking ahead, can't backtrack
- DP: Considers all possibilities, can backtrack

**Common Greedy Problems:**
- Activity selection
- Huffman coding
- Fractional knapsack
- Minimum spanning tree (Kruskal, Prim)
- Dijkstra's shortest path

### Activity Selection

```go
type Activity struct {
    start, finish int
}

func activitySelection(activities []Activity) []Activity {
    // Sort by finish time
    sort.Slice(activities, func(i, j int) bool {
        return activities[i].finish < activities[j].finish
    })
    
    selected := []Activity{activities[0]}
    lastFinish := activities[0].finish
    
    for i := 1; i < len(activities); i++ {
        if activities[i].start >= lastFinish {
            selected = append(selected, activities[i])
            lastFinish = activities[i].finish
        }
    }
    return selected
}
```

### Fractional Knapsack

```go
type Item struct {
    value, weight int
}

func fractionalKnapsack(items []Item, capacity int) float64 {
    // Sort by value/weight ratio
    sort.Slice(items, func(i, j int) bool {
        ratioI := float64(items[i].value) / float64(items[i].weight)
        ratioJ := float64(items[j].value) / float64(items[j].weight)
        return ratioI > ratioJ
    })
    
    totalValue := 0.0
    for _, item := range items {
        if capacity >= item.weight {
            totalValue += float64(item.value)
            capacity -= item.weight
        } else {
            totalValue += float64(item.value) * float64(capacity) / float64(item.weight)
            break
        }
    }
    return totalValue
}
```

## Two Pointers

### Understanding Two Pointers

Two pointers technique uses two pointers to iterate through data structure, often reducing time complexity from O(n²) to O(n).

**Common Patterns:**
1. **Opposite ends**: Start from both ends, move toward center
2. **Same direction**: Both move left to right at different speeds
3. **Sliding window**: Maintain window of elements

**When to Use:**
- Sorted array problems
- Finding pairs/triplets
- Removing duplicates
- Palindrome checking

### Two Sum (Sorted Array)

```go
func twoSum(numbers []int, target int) []int {
    left, right := 0, len(numbers)-1
    
    for left < right {
        sum := numbers[left] + numbers[right]
        if sum == target {
            return []int{left, right}
        } else if sum < target {
            left++
        } else {
            right--
        }
    }
    return []int{}
}
```

### Remove Duplicates

```go
func removeDuplicates(nums []int) int {
    if len(nums) == 0 {
        return 0
    }
    slow := 0
    for fast := 1; fast < len(nums); fast++ {
        if nums[fast] != nums[slow] {
            slow++
            nums[slow] = nums[fast]
        }
    }
    return slow + 1
}
```

## Sliding Window

### Understanding Sliding Window

Sliding window maintains a subset of elements (window) and slides it across array/string.

**Types:**
1. **Fixed size**: Window size constant
2. **Variable size**: Window size changes based on condition

**When to Use:**
- Subarray/substring problems
- Finding maximum/minimum in subarrays
- Problems with contiguous elements

**Pattern:**
1. Expand window by moving right pointer
2. Process current window
3. Shrink window by moving left pointer when condition met

### Maximum Sum Subarray of Size K

```go
func maxSumSubarray(arr []int, k int) int {
    if len(arr) < k {
        return 0
    }
    
    windowSum := 0
    for i := 0; i < k; i++ {
        windowSum += arr[i]
    }
    
    maxSum := windowSum
    for i := k; i < len(arr); i++ {
        windowSum = windowSum - arr[i-k] + arr[i]
        maxSum = max(maxSum, windowSum)
    }
    return maxSum
}
// Time: O(n), Space: O(1)
// Much better than O(n*k) naive approach
```

### Longest Substring Without Repeating Characters

```go
func lengthOfLongestSubstring(s string) int {
    charMap := make(map[byte]int)
    left, maxLen := 0, 0
    
    for right := 0; right < len(s); right++ {
        if idx, exists := charMap[s[right]]; exists && idx >= left {
            left = idx + 1
        }
        charMap[s[right]] = right
        maxLen = max(maxLen, right-left+1)
    }
    return maxLen
}
// Time: O(n), Space: O(min(n, charset_size))
// Sliding window with hash map for character tracking
```

## Backtracking

### Understanding Backtracking

Backtracking is a systematic way to try all possibilities. It builds solution incrementally and abandons (backtracks) when it determines current path won't lead to solution.

**Template:**
```go
func backtrack(state, choices) {
    if isGoal(state) {
        recordSolution(state)
        return
    }
    
    for choice in choices {
        if isValid(choice) {
            makeChoice(choice)
            backtrack(newState, newChoices)
            undoChoice(choice) // Backtrack
        }
    }
}
```

**Common Problems:**
- N-Queens
- Sudoku solver
- Permutations/Combinations
- Subset sum
- Graph coloring

### Generate Permutations

```go
func permute(nums []int) [][]int {
    result := [][]int{}
    backtrack(nums, []int{}, &result)
    return result
}

func backtrack(nums []int, current []int, result *[][]int) {
    if len(current) == len(nums) {
        temp := make([]int, len(current))
        copy(temp, current)
        *result = append(*result, temp)
        return
    }
    
    for _, num := range nums {
        if contains(current, num) {
            continue
        }
        current = append(current, num)
        backtrack(nums, current, result)
        current = current[:len(current)-1] // Backtrack
    }
}

func contains(arr []int, val int) bool {
    for _, v := range arr {
        if v == val {
            return true
        }
    }
    return false
}
// Time: O(n!), Space: O(n)
```

### N-Queens Problem

```go
func solveNQueens(n int) [][]string {
    result := [][]string{}
    board := make([][]byte, n)
    for i := range board {
        board[i] = make([]byte, n)
        for j := range board[i] {
            board[i][j] = '.'
        }
    }
    backtrackQueens(board, 0, &result)
    return result
}

func backtrackQueens(board [][]byte, row int, result *[][]string) {
    if row == len(board) {
        *result = append(*result, boardToStrings(board))
        return
    }
    
    for col := 0; col < len(board); col++ {
        if isValidQueen(board, row, col) {
            board[row][col] = 'Q'
            backtrackQueens(board, row+1, result)
            board[row][col] = '.' // Backtrack
        }
    }
}

func isValidQueen(board [][]byte, row, col int) bool {
    n := len(board)
    
    // Check column
    for i := 0; i < row; i++ {
        if board[i][col] == 'Q' {
            return false
        }
    }
    
    // Check diagonal (top-left)
    for i, j := row-1, col-1; i >= 0 && j >= 0; i, j = i-1, j-1 {
        if board[i][j] == 'Q' {
            return false
        }
    }
    
    // Check diagonal (top-right)
    for i, j := row-1, col+1; i >= 0 && j < n; i, j = i-1, j+1 {
        if board[i][j] == 'Q' {
            return false
        }
    }
    
    return true
}

func boardToStrings(board [][]byte) []string {
    result := make([]string, len(board))
    for i, row := range board {
        result[i] = string(row)
    }
    return result
}
// Time: O(n!), Space: O(n²)
```
