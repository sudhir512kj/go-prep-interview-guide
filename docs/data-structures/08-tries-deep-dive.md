# Tries Deep Dive

## Table of Contents
- [Trie Fundamentals](#trie-fundamentals)
- [Internal Implementation](#internal-implementation)
- [Trie Operations](#trie-operations)
- [Compressed Trie (Radix Tree)](#compressed-trie-radix-tree)
- [Ternary Search Tree](#ternary-search-tree)
- [Performance Characteristics](#performance-characteristics)
- [Common Patterns](#common-patterns)
- [Interview Problems](#interview-problems)

---

## Trie Fundamentals

### Definition
A trie (prefix tree) is a tree data structure used to store strings where each node represents a character. Strings with common prefixes share nodes.

```
Words: ["cat", "car", "card", "care", "bat"]

         root
        /    \
       c      b
       |      |
       a      a
      / \     |
     t   r    t
         |
         d,e
```

### Why Tries?
- O(m) search where m = word length (independent of n = number of words)
- Efficient prefix matching
- Autocomplete, spell checking, IP routing
- Better than hash map for prefix queries

---

## Internal Implementation

### Node Structure
```go
type TrieNode struct {
    children [26]*TrieNode  // For lowercase letters
    isEnd    bool
    count    int            // Optional: word frequency
}

type Trie struct {
    root *TrieNode
}

func NewTrie() *Trie {
    return &Trie{root: &TrieNode{}}
}
```

### Memory Layout
```
Trie storing "cat", "car":

root
└── c (children['c'-'a'])
    └── a
        ├── t (isEnd=true)  ← "cat"
        └── r (isEnd=true)  ← "car"

Each node: [26 pointers + isEnd bool]
Size per node: 26*8 + 1 = 209 bytes (64-bit)
```

### Map-based Node (Memory Efficient)
```go
type TrieNodeMap struct {
    children map[rune]*TrieNodeMap
    isEnd    bool
}

type TrieMap struct {
    root *TrieNodeMap
}

func NewTrieMap() *TrieMap {
    return &TrieMap{root: &TrieNodeMap{children: make(map[rune]*TrieNodeMap)}}
}
// Better for large alphabets (Unicode)
// Slightly slower due to map overhead
```

---

## Trie Operations

### Insert
```go
func (t *Trie) Insert(word string) {
    node := t.root
    for _, ch := range word {
        idx := ch - 'a'
        if node.children[idx] == nil {
            node.children[idx] = &TrieNode{}
        }
        node = node.children[idx]
    }
    node.isEnd = true
}

// Time: O(m) where m = word length
// Space: O(m) worst case (new nodes)
```

### Search
```go
func (t *Trie) Search(word string) bool {
    node := t.root
    for _, ch := range word {
        idx := ch - 'a'
        if node.children[idx] == nil {
            return false
        }
        node = node.children[idx]
    }
    return node.isEnd
}

// Time: O(m)
```

### StartsWith (Prefix Search)
```go
func (t *Trie) StartsWith(prefix string) bool {
    node := t.root
    for _, ch := range prefix {
        idx := ch - 'a'
        if node.children[idx] == nil {
            return false
        }
        node = node.children[idx]
    }
    return true  // Prefix exists (regardless of isEnd)
}

// Time: O(m)
```

### Delete
```go
func (t *Trie) Delete(word string) bool {
    return t.deleteHelper(t.root, word, 0)
}

func (t *Trie) deleteHelper(node *TrieNode, word string, depth int) bool {
    if node == nil {
        return false
    }
    if depth == len(word) {
        if !node.isEnd {
            return false  // Word not found
        }
        node.isEnd = false
        return t.isEmpty(node)  // Can delete if no children
    }

    idx := word[depth] - 'a'
    if t.deleteHelper(node.children[idx], word, depth+1) {
        node.children[idx] = nil  // Delete child
        return !node.isEnd && t.isEmpty(node)
    }
    return false
}

func (t *Trie) isEmpty(node *TrieNode) bool {
    for _, child := range node.children {
        if child != nil {
            return false
        }
    }
    return true
}

// Time: O(m)
```

### Get All Words with Prefix
```go
func (t *Trie) GetWordsWithPrefix(prefix string) []string {
    node := t.root
    for _, ch := range prefix {
        idx := ch - 'a'
        if node.children[idx] == nil {
            return nil
        }
        node = node.children[idx]
    }

    result := []string{}
    t.collectWords(node, prefix, &result)
    return result
}

func (t *Trie) collectWords(node *TrieNode, current string, result *[]string) {
    if node.isEnd {
        *result = append(*result, current)
    }
    for i, child := range node.children {
        if child != nil {
            t.collectWords(child, current+string(rune('a'+i)), result)
        }
    }
}

// Time: O(p + n) where p=prefix length, n=output size
```

### Count Words with Prefix
```go
func (t *Trie) CountWordsWithPrefix(prefix string) int {
    node := t.root
    for _, ch := range prefix {
        idx := ch - 'a'
        if node.children[idx] == nil {
            return 0
        }
        node = node.children[idx]
    }
    return t.countWords(node)
}

func (t *Trie) countWords(node *TrieNode) int {
    if node == nil {
        return 0
    }
    count := 0
    if node.isEnd {
        count++
    }
    for _, child := range node.children {
        count += t.countWords(child)
    }
    return count
}
```

### Longest Common Prefix
```go
func (t *Trie) LongestCommonPrefix() string {
    node := t.root
    prefix := ""

    for {
        // Count non-nil children
        childCount := 0
        var nextNode *TrieNode
        var nextChar rune

        for i, child := range node.children {
            if child != nil {
                childCount++
                nextNode = child
                nextChar = rune('a' + i)
            }
        }

        // Stop if multiple children or current node is end of word
        if childCount != 1 || node.isEnd {
            break
        }

        prefix += string(nextChar)
        node = nextNode
    }
    return prefix
}
```

---

## Compressed Trie (Radix Tree)

### Definition
Compresses chains of single-child nodes into single edges with string labels.

```
Regular Trie for ["interview", "internal", "inter"]:
i→n→t→e→r (isEnd)
              →v→i→e→w (isEnd)
              →n→a→l (isEnd)

Compressed Trie:
"inter" (isEnd)
  ├── "view" (isEnd)
  └── "nal" (isEnd)

Saves memory for sparse tries
```

```go
type RadixNode struct {
    children map[string]*RadixNode
    isEnd    bool
}

type RadixTree struct {
    root *RadixNode
}

func (rt *RadixTree) Insert(word string) {
    node := rt.root
    for word != "" {
        matched := false
        for key, child := range node.children {
            commonLen := commonPrefixLen(word, key)
            if commonLen == 0 {
                continue
            }
            if commonLen == len(key) {
                // Full key match, continue with rest
                word = word[commonLen:]
                node = child
                matched = true
                break
            }
            // Partial match — split the edge
            newNode := &RadixNode{children: map[string]*RadixNode{
                key[commonLen:]: child,
            }}
            node.children[key[:commonLen]] = newNode
            delete(node.children, key)
            if commonLen < len(word) {
                newNode.children[word[commonLen:]] = &RadixNode{isEnd: true}
            } else {
                newNode.isEnd = true
            }
            return
        }
        if !matched {
            node.children[word] = &RadixNode{isEnd: true}
            return
        }
    }
    node.isEnd = true
}

func commonPrefixLen(a, b string) int {
    i := 0
    for i < len(a) && i < len(b) && a[i] == b[i] {
        i++
    }
    return i
}
```

---

## Ternary Search Tree

### Definition
Space-efficient alternative to trie. Each node has three children: less, equal, greater.

```go
type TSTNode struct {
    char        rune
    left, mid, right *TSTNode
    isEnd       bool
}

type TST struct {
    root *TSTNode
}

func (t *TST) Insert(word string) {
    t.root = t.insert(t.root, []rune(word), 0)
}

func (t *TST) insert(node *TSTNode, word []rune, idx int) *TSTNode {
    if idx >= len(word) {
        return node
    }
    ch := word[idx]
    if node == nil {
        node = &TSTNode{char: ch}
    }
    if ch < node.char {
        node.left = t.insert(node.left, word, idx)
    } else if ch > node.char {
        node.right = t.insert(node.right, word, idx)
    } else {
        if idx+1 == len(word) {
            node.isEnd = true
        } else {
            node.mid = t.insert(node.mid, word, idx+1)
        }
    }
    return node
}

func (t *TST) Search(word string) bool {
    return t.search(t.root, []rune(word), 0)
}

func (t *TST) search(node *TSTNode, word []rune, idx int) bool {
    if node == nil || idx >= len(word) {
        return false
    }
    ch := word[idx]
    if ch < node.char {
        return t.search(node.left, word, idx)
    } else if ch > node.char {
        return t.search(node.right, word, idx)
    } else {
        if idx+1 == len(word) {
            return node.isEnd
        }
        return t.search(node.mid, word, idx+1)
    }
}
```

---

## Performance Characteristics

### Time Complexity
| Operation | Trie | Hash Map | BST |
|-----------|------|----------|-----|
| Insert | O(m) | O(m) | O(m log n) |
| Search | O(m) | O(m) | O(m log n) |
| Delete | O(m) | O(m) | O(m log n) |
| Prefix search | O(p) | O(n*m) | O(n*m) |
| All with prefix | O(p+k) | O(n*m) | O(n*m) |

m = word length, n = number of words, p = prefix length, k = output size

### Space Complexity
- Standard Trie: O(ALPHABET_SIZE * m * n)
- Map-based Trie: O(m * n) — more memory efficient
- Radix Tree: O(n) — most compact

### Trie vs Hash Map
```
Trie advantages:
- Prefix queries: O(p) vs O(n*m)
- Sorted iteration
- No hash collisions
- Autocomplete support

Hash Map advantages:
- Less memory for exact lookups
- Simpler implementation
- Better cache performance
```

---

## Common Patterns

### Autocomplete System
```go
type AutoComplete struct {
    trie  *Trie
    words map[string]int  // word → frequency
}

func (ac *AutoComplete) AddWord(word string, freq int) {
    ac.trie.Insert(word)
    ac.words[word] += freq
}

func (ac *AutoComplete) GetSuggestions(prefix string, k int) []string {
    candidates := ac.trie.GetWordsWithPrefix(prefix)

    // Sort by frequency
    sort.Slice(candidates, func(i, j int) bool {
        return ac.words[candidates[i]] > ac.words[candidates[j]]
    })

    if len(candidates) > k {
        return candidates[:k]
    }
    return candidates
}
```

### Spell Checker
```go
func (t *Trie) SpellCheck(word string) []string {
    suggestions := []string{}
    t.findSimilar(t.root, word, "", 0, 2, &suggestions)
    return suggestions
}

func (t *Trie) findSimilar(node *TrieNode, target, current string, pos, maxEdits int, result *[]string) {
    if maxEdits < 0 {
        return
    }
    if node.isEnd && pos >= len(target)-maxEdits {
        *result = append(*result, current)
    }
    for i, child := range node.children {
        if child == nil {
            continue
        }
        ch := string(rune('a' + i))
        if pos < len(target) && rune('a'+i) == rune(target[pos]) {
            t.findSimilar(child, target, current+ch, pos+1, maxEdits, result)
        } else {
            t.findSimilar(child, target, current+ch, pos+1, maxEdits-1, result)
        }
    }
}
```

### IP Routing (Binary Trie)
```go
type IPTrie struct {
    children [2]*IPTrie
    route    string
}

func (t *IPTrie) Insert(ip uint32, prefix int, route string) {
    node := t
    for i := 31; i >= 32-prefix; i-- {
        bit := (ip >> uint(i)) & 1
        if node.children[bit] == nil {
            node.children[bit] = &IPTrie{}
        }
        node = node.children[bit]
    }
    node.route = route
}

func (t *IPTrie) LongestPrefixMatch(ip uint32) string {
    node := t
    best := ""
    for i := 31; i >= 0; i-- {
        bit := (ip >> uint(i)) & 1
        if node.children[bit] == nil {
            break
        }
        node = node.children[bit]
        if node.route != "" {
            best = node.route
        }
    }
    return best
}
```

---

## Interview Problems

### 1. Word Search II (Trie + DFS)
```go
func findWords(board [][]byte, words []string) []string {
    trie := NewTrie()
    for _, w := range words {
        trie.Insert(w)
    }

    rows, cols := len(board), len(board[0])
    result := map[string]bool{}

    var dfs func(node *TrieNode, r, c int, path string)
    dfs = func(node *TrieNode, r, c int, path string) {
        if r < 0 || r >= rows || c < 0 || c >= cols || board[r][c] == '#' {
            return
        }
        ch := board[r][c]
        next := node.children[ch-'a']
        if next == nil {
            return
        }
        path += string(ch)
        if next.isEnd {
            result[path] = true
        }
        board[r][c] = '#'
        dfs(next, r+1, c, path)
        dfs(next, r-1, c, path)
        dfs(next, r, c+1, path)
        dfs(next, r, c-1, path)
        board[r][c] = ch
    }

    for r := 0; r < rows; r++ {
        for c := 0; c < cols; c++ {
            dfs(trie.root, r, c, "")
        }
    }

    words = words[:0]
    for w := range result {
        words = append(words, w)
    }
    return words
}
```

### 2. Replace Words
```go
func replaceWords(dictionary []string, sentence string) string {
    trie := NewTrie()
    for _, root := range dictionary {
        trie.Insert(root)
    }

    words := strings.Split(sentence, " ")
    for i, word := range words {
        node := trie.root
        for j, ch := range word {
            idx := ch - 'a'
            if node.children[idx] == nil {
                break
            }
            node = node.children[idx]
            if node.isEnd {
                words[i] = word[:j+1]
                break
            }
        }
    }
    return strings.Join(words, " ")
}
```

### 3. Maximum XOR of Two Numbers
```go
// Binary trie for XOR maximization
type XORTrie struct {
    children [2]*XORTrie
}

func (t *XORTrie) Insert(num int) {
    node := t
    for i := 31; i >= 0; i-- {
        bit := (num >> i) & 1
        if node.children[bit] == nil {
            node.children[bit] = &XORTrie{}
        }
        node = node.children[bit]
    }
}

func (t *XORTrie) MaxXOR(num int) int {
    node := t
    xor := 0
    for i := 31; i >= 0; i-- {
        bit := (num >> i) & 1
        want := 1 - bit  // We want opposite bit for max XOR
        if node.children[want] != nil {
            xor |= 1 << i
            node = node.children[want]
        } else {
            node = node.children[bit]
        }
    }
    return xor
}

func findMaximumXOR(nums []int) int {
    trie := &XORTrie{}
    for _, n := range nums {
        trie.Insert(n)
    }
    maxXOR := 0
    for _, n := range nums {
        maxXOR = max(maxXOR, trie.MaxXOR(n))
    }
    return maxXOR
}
```

---

## Summary

### Key Concepts
1. Each path from root to isEnd node represents a word
2. Nodes with common prefixes share the same path
3. O(m) operations independent of number of words
4. Excellent for prefix-based queries

### When to Use
- Autocomplete / type-ahead
- Spell checking
- IP routing (longest prefix match)
- Word games (Boggle, Scrabble)
- Dictionary implementations
- XOR maximization (binary trie)

### Variants
| Variant | Use Case |
|---------|----------|
| Standard Trie | General string storage |
| Compressed/Radix | Memory-efficient storage |
| Ternary Search Tree | Space-efficient, sorted |
| Binary Trie | Bit manipulation, XOR |
| Suffix Trie | Substring search |
