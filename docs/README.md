# Golang Interview Preparation Guide

Complete guide for Golang interview preparation covering fundamentals, data structures, algorithms, concurrency, and best practices.

## 🔍 [Search Documentation](./search.html)

Open `search.html` in your browser for powerful search across all documents with highlighting, filtering, and regex support. See [Search Guide](./SEARCH-GUIDE.md) for details.

## 📚 [Documentation Overview](./00-overview.md)

Start here for a complete overview of all documents, navigation tips, and study paths.

## 🧭 [Quick Navigation](./NAVIGATION.md)

Find any topic quickly with our comprehensive navigation guide.

## Table of Contents

### 1. [Golang Basics](./01-golang-basics.md)
- Introduction to Go
- Variables and Data Types
- Control Structures
- Functions and Closures
- Pointers
- Structs and Interfaces
- Arrays, Slices, and Maps
- Defer, Panic, Recover

### 2. [Data Structures](./02-data-structures.md)
- Complexity Analysis
- Arrays and Slices
- Maps and Hash Tables
- Linked Lists (Singly, Doubly)
- Stacks and Queues
- Trees (Binary, BST)
- Heaps and Priority Queues
- Graphs (BFS, DFS, Topological Sort)
- Tries
- Union-Find
- Segment Trees
- Bloom Filters

### 3. [Algorithms](./03-algorithms.md)
- Algorithm Analysis
- Sorting Algorithms (Bubble, Selection, Insertion, Merge, Quick, Heap)
- Searching Algorithms (Linear, Binary)
- Recursion and Memoization
- Dynamic Programming
- Greedy Algorithms
- Two Pointers Technique
- Sliding Window
- Backtracking

### 4. [Concurrency](./04-concurrency.md)
- Concurrency vs Parallelism
- Goroutines and Scheduling
- Channels (Buffered, Unbuffered)
- Select Statement
- WaitGroup and Synchronization
- Mutex and RWMutex
- Once
- Context Package
- Atomic Operations
- Common Patterns (Fan-out/Fan-in, Pipeline, Worker Pool)
- Concurrency Pitfalls

### 5. [Error Handling](./05-error-handling.md)
- Error Handling Philosophy
- Basic Error Handling
- Custom Errors
- Error Wrapping (Go 1.13+)
- Panic and Recover
- Error Handling Patterns
- Concurrent Error Handling
- Best Practices

### 6. [Testing](./06-testing.md)
- Testing Philosophy
- Unit Testing
- Table-Driven Tests
- Test Fixtures
- Mocking and Interfaces
- Benchmarking
- Test Coverage
- Integration Testing
- Fuzzing (Go 1.18+)
- Testing Best Practices

### 7. [Best Practices](./07-best-practices.md)
- Code Organization
- Naming Conventions
- Error Handling
- Concurrency Patterns
- Performance Optimization
- Memory Management
- Interface Design
- Design Patterns (Functional Options, Builder, Strategy, Observer)
- Security Best Practices
- Logging and Configuration
- Profiling and Debugging

### 8. [Common Interview Questions](./08-interview-questions.md)
- Basic Concepts (40 questions)
- Deep Dive Questions
- Advanced Coding Problems
- System Design Questions
- Tricky Questions
- Performance Questions

### 9. [Quick Reference Guide](./09-quick-reference.md)
- Language Fundamentals Cheat Sheet
- Data Structures Complexity
- Algorithm Complexity
- Concurrency Patterns
- Common Idioms
- Performance Tips
- Common Pitfalls
- Go Commands
- Interview Preparation Checklist

### 10. [Practice Exercises](./10-practice-exercises.md)
- Easy, Medium, Hard Problems
- Concurrency Exercises
- System Design Exercises
- Testing Exercises

### 🔥 [Concurrency Deep Dive](./concurrency/README.md)
- **[Goroutines Deep Dive](./concurrency/01-goroutines-deep-dive.md)** - Complete goroutine guide
- **[Channels Deep Dive](./concurrency/02-channels-deep-dive.md)** - Complete channel guide
- **[Sync Package](./concurrency/04-sync-package.md)** - All sync primitives
- **[Context Package](./concurrency/05-context-package.md)** - Context management
- **[Summary](./concurrency/00-SUMMARY.md)** - Quick reference

### 🔥 [Go Fundamentals Deep Dive](./go-fundamentals/README.md)
- **[Pointers](./go-fundamentals/01-pointers-deep-dive.md)** - Memory model, escape analysis, unsafe, pointer receivers
- **[Structs](./go-fundamentals/02-structs-deep-dive.md)** - Memory layout, padding, embedding, tags, patterns
- **[Interfaces](./go-fundamentals/03-interfaces-deep-dive.md)** - iface internals, type assertions, composition, pitfalls
- **[Functions & Closures](./go-fundamentals/04-functions-closures-deep-dive.md)** - Stack frames, closures, defer, panic/recover
- **[Types & Generics](./go-fundamentals/05-types-generics-deep-dive.md)** - Named types, aliases, generics, constraints, reflection
- **[Error Handling](./go-fundamentals/06-error-handling-deep-dive.md)** - error internals, wrapping, Is/As, patterns

### 🔥 [Data Structures Deep Dive](./data-structures/README.md)
- **[Arrays & Slices](./data-structures/01-arrays-slices-deep-dive.md)** - Slice header, growth strategy, memory leaks
- **[Maps](./data-structures/02-maps-deep-dive.md)** - Hash table internals, buckets, rehashing
- **[Linked Lists](./data-structures/03-linked-lists-deep-dive.md)** - Singly, doubly, circular, two-pointer techniques
- **[Stacks & Queues](./data-structures/04-stacks-queues-deep-dive.md)** - Implementations, monotonic stack/queue
- **[Trees](./data-structures/05-trees-deep-dive.md)** - BST, AVL, Red-Black, all traversals
- **[Heaps](./data-structures/06-heaps-deep-dive.md)** - Min/max heap, container/heap, heap sort
- **[Graphs](./data-structures/07-graphs-deep-dive.md)** - DFS, BFS, Dijkstra, MST, topological sort
- **[Tries](./data-structures/08-tries-deep-dive.md)** - Prefix tree, radix tree, binary trie
- **[Union-Find, Segment Trees & Bloom Filters](./data-structures/09-union-find-segment-tree-bloom-filter.md)** - Advanced structures

## How to Use This Guide

1. **For Beginners**: Start with Golang Basics, then move to Data Structures and Algorithms
2. **For Intermediate**: Focus on Concurrency, Error Handling, and Testing
3. **For Advanced**: Study Best Practices, System Design, and Advanced Interview Questions
4. **Before Interview**: Review Quick Reference Guide and practice coding problems

## Study Plan

### Week 1-2: Fundamentals
- Go basics and syntax
- Data structures implementation
- Basic algorithms

### Week 3-4: Concurrency
- Goroutines and channels
- Synchronization primitives
- Common concurrency patterns

### Week 5-6: Advanced Topics
- Error handling patterns
- Testing strategies
- Best practices and design patterns

### Week 7-8: Interview Preparation
- Practice coding problems
- System design questions
- Mock interviews

## Practice Resources

- **LeetCode**: Practice algorithmic problems
- **HackerRank**: Go-specific challenges
- **Exercism**: Mentored practice
- **Go by Example**: Learn by examples
- **The Go Playground**: Test code snippets

## Additional Resources

### Official Documentation
- [Go Documentation](https://go.dev/doc/)
- [Effective Go](https://go.dev/doc/effective_go)
- [Go Blog](https://go.dev/blog/)
- [Go Wiki](https://github.com/golang/go/wiki)

### Books
- "The Go Programming Language" by Donovan & Kernighan
- "Concurrency in Go" by Katherine Cox-Buday
- "Go in Action" by William Kennedy
- "Learning Go" by Jon Bodner

### Online Courses
- [Go Tour](https://go.dev/tour/)
- [Go by Example](https://gobyexample.com/)
- [Gophercises](https://gophercises.com/)

## Contributing

Feel free to contribute by:
- Adding more examples
- Fixing errors
- Improving explanations
- Adding new topics

## Tips for Success

1. **Practice Regularly**: Code every day, even if just for 30 minutes
2. **Understand, Don't Memorize**: Focus on understanding concepts
3. **Build Projects**: Apply knowledge in real projects
4. **Read Code**: Study well-written Go code on GitHub
5. **Join Community**: Participate in Go forums and discussions
6. **Mock Interviews**: Practice with peers or online platforms
7. **Time Yourself**: Practice solving problems within time limits
8. **Review Mistakes**: Learn from errors and edge cases

## Interview Day Checklist

- [ ] Review Quick Reference Guide
- [ ] Practice common patterns (two pointers, sliding window, etc.)
- [ ] Review concurrency patterns
- [ ] Understand time/space complexity
- [ ] Prepare questions to ask interviewer
- [ ] Test your setup (if remote interview)
- [ ] Get good sleep
- [ ] Stay calm and think aloud

Good luck with your interviews! 🚀
