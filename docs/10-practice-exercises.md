# Practice Exercises

## Easy Level

### 1. Reverse a String
```go
// Write a function to reverse a string
// Input: "hello"
// Output: "olleh"

func reverseString(s string) string {
    // Your code here
}
```

<details>
<summary>Solution</summary>

```go
func reverseString(s string) string {
    runes := []rune(s)
    for i, j := 0, len(runes)-1; i < j; i, j = i+1, j-1 {
        runes[i], runes[j] = runes[j], runes[i]
    }
    return string(runes)
}
```
</details>

### 2. Find Maximum in Array
```go
// Find the maximum element in an array
// Input: []int{3, 7, 2, 9, 1}
// Output: 9

func findMax(arr []int) int {
    // Your code here
}
```

### 3. Count Vowels
```go
// Count the number of vowels in a string
// Input: "hello world"
// Output: 3

func countVowels(s string) int {
    // Your code here
}
```

### 4. Fibonacci Number
```go
// Return the nth Fibonacci number
// Input: 6
// Output: 8 (0, 1, 1, 2, 3, 5, 8)

func fibonacci(n int) int {
    // Your code here
}
```

### 5. Check Palindrome
```go
// Check if a string is a palindrome
// Input: "racecar"
// Output: true

func isPalindrome(s string) bool {
    // Your code here
}
```

## Medium Level

### 6. Two Sum
```go
// Find two numbers that add up to target
// Input: nums = [2,7,11,15], target = 9
// Output: [0,1]

func twoSum(nums []int, target int) []int {
    // Your code here
}
```

<details>
<summary>Solution</summary>

```go
func twoSum(nums []int, target int) []int {
    seen := make(map[int]int)
    for i, num := range nums {
        complement := target - num
        if j, exists := seen[complement]; exists {
            return []int{j, i}
        }
        seen[num] = i
    }
    return nil
}
```
</details>

### 7. Valid Parentheses
```go
// Check if parentheses are valid
// Input: "({[]})"
// Output: true

func isValid(s string) bool {
    // Your code here
}
```

### 8. Merge Sorted Arrays
```go
// Merge two sorted arrays
// Input: [1,3,5], [2,4,6]
// Output: [1,2,3,4,5,6]

func mergeSorted(arr1, arr2 []int) []int {
    // Your code here
}
```

### 9. Find Duplicate
```go
// Find the duplicate number in array
// Input: [1,3,4,2,2]
// Output: 2

func findDuplicate(nums []int) int {
    // Your code here
}
```

### 10. Longest Substring Without Repeating Characters
```go
// Find length of longest substring without repeating characters
// Input: "abcabcbb"
// Output: 3 ("abc")

func lengthOfLongestSubstring(s string) int {
    // Your code here
}
```

## Hard Level

### 11. LRU Cache
```go
// Implement an LRU (Least Recently Used) cache
// Operations: Get(key) and Put(key, value)

type LRUCache struct {
    // Your fields here
}

func Constructor(capacity int) LRUCache {
    // Your code here
}

func (lru *LRUCache) Get(key int) int {
    // Your code here
}

func (lru *LRUCache) Put(key, value int) {
    // Your code here
}
```

### 12. Median of Two Sorted Arrays
```go
// Find median of two sorted arrays
// Input: [1,3], [2]
// Output: 2.0

func findMedianSortedArrays(nums1, nums2 []int) float64 {
    // Your code here
}
```

### 13. Trapping Rain Water
```go
// Calculate how much water can be trapped
// Input: [0,1,0,2,1,0,1,3,2,1,2,1]
// Output: 6

func trap(height []int) int {
    // Your code here
}
```

### 14. Word Ladder
```go
// Find shortest transformation sequence
// Input: beginWord = "hit", endWord = "cog", 
//        wordList = ["hot","dot","dog","lot","log","cog"]
// Output: 5 ("hit" -> "hot" -> "dot" -> "dog" -> "cog")

func ladderLength(beginWord, endWord string, wordList []string) int {
    // Your code here
}
```

### 15. Serialize and Deserialize Binary Tree
```go
// Serialize and deserialize a binary tree

type Codec struct{}

func Constructor() Codec {
    return Codec{}
}

func (codec *Codec) serialize(root *TreeNode) string {
    // Your code here
}

func (codec *Codec) deserialize(data string) *TreeNode {
    // Your code here
}
```

## Concurrency Exercises

### 16. Parallel Sum
```go
// Calculate sum of array using multiple goroutines
// Input: []int{1,2,3,4,5,6,7,8,9,10}
// Output: 55

func parallelSum(nums []int, numWorkers int) int {
    // Your code here
}
```

<details>
<summary>Solution</summary>

```go
func parallelSum(nums []int, numWorkers int) int {
    chunkSize := (len(nums) + numWorkers - 1) / numWorkers
    results := make(chan int, numWorkers)
    
    for i := 0; i < numWorkers; i++ {
        start := i * chunkSize
        end := start + chunkSize
        if end > len(nums) {
            end = len(nums)
        }
        
        go func(chunk []int) {
            sum := 0
            for _, num := range chunk {
                sum += num
            }
            results <- sum
        }(nums[start:end])
    }
    
    total := 0
    for i := 0; i < numWorkers; i++ {
        total += <-results
    }
    
    return total
}
```
</details>

### 17. Producer-Consumer
```go
// Implement producer-consumer pattern
// Producer generates numbers, consumer processes them

func producerConsumer(n int) {
    // Your code here
}
```

### 18. Concurrent Web Scraper
```go
// Scrape multiple URLs concurrently
// Input: []string{"url1", "url2", "url3"}
// Output: map[string]string (url -> content)

func scrapeURLs(urls []string) map[string]string {
    // Your code here
}
```

### 19. Rate Limited API Client
```go
// Implement rate-limited API client
// Max 10 requests per second

type RateLimitedClient struct {
    // Your fields here
}

func NewRateLimitedClient(rps int) *RateLimitedClient {
    // Your code here
}

func (c *RateLimitedClient) Request(url string) (string, error) {
    // Your code here
}
```

### 20. Concurrent Map Reduce
```go
// Implement map-reduce pattern
// Map: Transform each element
// Reduce: Combine results

func mapReduce(data []int, mapper func(int) int, reducer func(int, int) int) int {
    // Your code here
}
```

## System Design Exercises

### 21. Design URL Shortener
```
Requirements:
- Shorten long URLs
- Redirect short URLs to original
- Track click statistics
- Handle 1000 requests/second

Design the system and implement core functionality.
```

### 22. Design Rate Limiter
```
Requirements:
- Limit requests per user
- Support multiple rate limit strategies (fixed window, sliding window, token bucket)
- Distributed system support

Implement token bucket algorithm.
```

### 23. Design Cache System
```
Requirements:
- LRU eviction policy
- TTL support
- Thread-safe
- Memory limit

Implement the cache.
```

### 24. Design Task Scheduler
```
Requirements:
- Schedule tasks at specific times
- Support recurring tasks
- Priority queue
- Concurrent execution

Implement the scheduler.
```

### 25. Design Message Queue
```
Requirements:
- FIFO ordering
- Multiple producers/consumers
- Persistence
- Acknowledgments

Implement basic message queue.
```

## Testing Exercises

### 26. Write Tests for Calculator
```go
type Calculator struct{}

func (c *Calculator) Add(a, b int) int {
    return a + b
}

func (c *Calculator) Divide(a, b int) (int, error) {
    if b == 0 {
        return 0, errors.New("division by zero")
    }
    return a / b, nil
}

// Write comprehensive tests including:
// - Table-driven tests
// - Error cases
// - Edge cases
```

### 27. Write Benchmarks
```go
// Benchmark different string concatenation methods
// - Using + operator
// - Using strings.Builder
// - Using bytes.Buffer

func BenchmarkStringConcat(b *testing.B) {
    // Your code here
}
```

### 28. Mock HTTP Client
```go
// Write tests for function that makes HTTP requests
// Use httptest to mock server

func fetchData(url string) ([]byte, error) {
    resp, err := http.Get(url)
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()
    return io.ReadAll(resp.Body)
}

// Write tests with mocked HTTP server
```

## Real-World Scenarios

### 29. Log Aggregator
```
Build a log aggregator that:
- Reads logs from multiple sources concurrently
- Filters logs by level (INFO, WARN, ERROR)
- Aggregates statistics
- Writes to output file

Implement with proper error handling and testing.
```

### 30. File Processor
```
Build a file processor that:
- Reads large files in chunks
- Processes each chunk concurrently
- Aggregates results
- Handles errors gracefully

Implement with context for cancellation.
```

## Challenge Problems

### 31. Implement sync.Map
```go
// Implement your own concurrent map similar to sync.Map
// Support: Load, Store, Delete, Range

type ConcurrentMap struct {
    // Your implementation
}
```

### 32. Implement Context
```go
// Implement your own context package
// Support: WithCancel, WithTimeout, WithValue

type Context interface {
    Deadline() (deadline time.Time, ok bool)
    Done() <-chan struct{}
    Err() error
    Value(key interface{}) interface{}
}
```

### 33. Implement Worker Pool
```go
// Implement a generic worker pool
// Support: Dynamic worker count, graceful shutdown

type WorkerPool struct {
    // Your implementation
}
```

### 34. Implement Circuit Breaker
```go
// Implement circuit breaker pattern
// States: Closed, Open, Half-Open

type CircuitBreaker struct {
    // Your implementation
}
```

### 35. Implement Retry Mechanism
```go
// Implement retry with exponential backoff
// Support: Max retries, backoff strategy, timeout

func RetryWithBackoff(fn func() error, maxRetries int) error {
    // Your implementation
}
```

## Tips for Practice

1. **Start Simple**: Begin with easy problems, gradually increase difficulty
2. **Time Yourself**: Practice under time constraints
3. **Test Your Code**: Write tests for your solutions
4. **Optimize**: After solving, think about optimization
5. **Review Solutions**: Study multiple approaches
6. **Explain Aloud**: Practice explaining your solution
7. **Handle Edge Cases**: Consider empty inputs, large inputs, etc.
8. **Use Debugger**: Learn to debug effectively
9. **Read Others' Code**: Learn from well-written solutions
10. **Build Projects**: Apply concepts in real projects

## Practice Schedule

### Week 1: Basics
- Day 1-2: Arrays and strings (5 problems)
- Day 3-4: Linked lists (5 problems)
- Day 5-6: Stacks and queues (5 problems)
- Day 7: Review and practice

### Week 2: Intermediate
- Day 1-2: Trees (5 problems)
- Day 3-4: Graphs (5 problems)
- Day 5-6: Dynamic programming (5 problems)
- Day 7: Review and practice

### Week 3: Advanced
- Day 1-2: Concurrency (5 problems)
- Day 3-4: System design (3 problems)
- Day 5-6: Mixed problems (10 problems)
- Day 7: Mock interview

### Week 4: Polish
- Day 1-3: Weak areas
- Day 4-5: Speed practice
- Day 6: Mock interview
- Day 7: Rest and review

## Resources for More Practice

- **LeetCode**: 1000+ problems with Go support
- **HackerRank**: Go-specific track
- **Exercism**: Mentored practice
- **CodeWars**: Gamified practice
- **Project Euler**: Mathematical problems
- **Advent of Code**: Annual coding challenge

## Evaluation Criteria

When solving problems, evaluate:
- **Correctness**: Does it work for all cases?
- **Efficiency**: Time and space complexity
- **Readability**: Is code clean and clear?
- **Error Handling**: Are errors handled properly?
- **Testing**: Are edge cases covered?
- **Concurrency**: Is it thread-safe if needed?

Good luck with your practice! 🚀
