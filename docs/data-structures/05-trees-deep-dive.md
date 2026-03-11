# Trees Deep Dive

## Table of Contents
- [Tree Fundamentals](#tree-fundamentals)
- [Binary Tree](#binary-tree)
- [Binary Search Tree (BST)](#binary-search-tree-bst)
- [AVL Tree](#avl-tree)
- [Red-Black Tree](#red-black-tree)
- [Tree Traversals](#tree-traversals)
- [Level Order Traversal](#level-order-traversal)
- [BST Operations](#bst-operations)
- [Tree Properties](#tree-properties)
- [Performance Characteristics](#performance-characteristics)
- [Common Patterns](#common-patterns)
- [Interview Problems](#interview-problems)

---

## Tree Fundamentals

### Terminology
```
         1          ← Root
        / \
       2   3        ← Internal nodes
      / \   \
     4   5   6      ← Leaf nodes

- Root: topmost node (1)
- Parent: node with children (1 is parent of 2,3)
- Child: node with a parent (2,3 are children of 1)
- Leaf: node with no children (4,5,6)
- Height: longest path from root to leaf = 2
- Depth of node: distance from root (depth of 4 = 2)
- Level: depth + 1 (4 is at level 3)
- Subtree: any node and its descendants
```

### Node Structure
```go
type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func NewNode(val int) *TreeNode {
    return &TreeNode{Val: val}
}
```

---

## Binary Tree

### Types
```
Full Binary Tree:        Complete Binary Tree:    Perfect Binary Tree:
Every node has 0 or 2   All levels filled         All levels completely
children                except last (left-fill)   filled

    1                       1                         1
   / \                     / \                       / \
  2   3                   2   3                     2   3
 / \                     / \  /                    / \ / \
4   5                   4  5 6                    4  5 6  7
```

### Memory Layout
```
Tree stored as array (for complete binary tree):
Index:  0   1   2   3   4   5   6
Value: [1] [2] [3] [4] [5] [6] [7]

Parent of i: (i-1)/2
Left child:  2*i + 1
Right child: 2*i + 2
```

### Array-based Binary Tree
```go
type ArrayTree struct {
    data []int
}

func (t *ArrayTree) Parent(i int) int      { return (i - 1) / 2 }
func (t *ArrayTree) LeftChild(i int) int   { return 2*i + 1 }
func (t *ArrayTree) RightChild(i int) int  { return 2*i + 2 }
func (t *ArrayTree) IsLeaf(i int) bool     { return t.LeftChild(i) >= len(t.data) }
```

---

## Binary Search Tree (BST)

### Property
For every node N:
- All values in left subtree < N.Val
- All values in right subtree > N.Val

```
        8
       / \
      3   10
     / \    \
    1   6    14
       / \   /
      4   7 13

BST property holds at every node
```

### Insert
```go
func insert(root *TreeNode, val int) *TreeNode {
    if root == nil {
        return &TreeNode{Val: val}
    }
    if val < root.Val {
        root.Left = insert(root.Left, val)
    } else if val > root.Val {
        root.Right = insert(root.Right, val)
    }
    return root
}
// Time: O(h) where h = height, O(log n) balanced, O(n) worst
```

### Search
```go
func search(root *TreeNode, val int) *TreeNode {
    if root == nil || root.Val == val {
        return root
    }
    if val < root.Val {
        return search(root.Left, val)
    }
    return search(root.Right, val)
}
// Time: O(h)
```

### Delete
```go
func delete(root *TreeNode, val int) *TreeNode {
    if root == nil {
        return nil
    }
    if val < root.Val {
        root.Left = delete(root.Left, val)
    } else if val > root.Val {
        root.Right = delete(root.Right, val)
    } else {
        // Node found — 3 cases
        if root.Left == nil {
            return root.Right   // Case 1: no left child
        }
        if root.Right == nil {
            return root.Left    // Case 2: no right child
        }
        // Case 3: two children — replace with inorder successor
        minNode := findMin(root.Right)
        root.Val = minNode.Val
        root.Right = delete(root.Right, minNode.Val)
    }
    return root
}

func findMin(root *TreeNode) *TreeNode {
    for root.Left != nil {
        root = root.Left
    }
    return root
}
// Time: O(h)
```

### Delete Visualization
```
Delete 3 from:          Step 1: Find inorder       Step 2: Replace 3
        8               successor of 3 (4)         with 4, delete 4
       / \                      8                        8
      3   10                   / \                      / \
     / \    \                 4   10                   4   10
    1   6    14              / \    \                 / \    \
       / \   /              1   6    14              1   6    14
      4   7 13                 / \   /                  / \   /
                              4   7 13                 _   7 13
```

---

## AVL Tree

### Definition
Self-balancing BST where height difference between left and right subtrees ≤ 1 for every node.

### Balance Factor
```go
type AVLNode struct {
    Val    int
    Left   *AVLNode
    Right  *AVLNode
    Height int
}

func height(n *AVLNode) int {
    if n == nil {
        return 0
    }
    return n.Height
}

func balanceFactor(n *AVLNode) int {
    if n == nil {
        return 0
    }
    return height(n.Left) - height(n.Right)
}

func updateHeight(n *AVLNode) {
    n.Height = 1 + max(height(n.Left), height(n.Right))
}
```

### Rotations
```go
// Right Rotation (LL case)
//     z                y
//    / \              / \
//   y   T4    →      x   z
//  / \              / \ / \
// x   T3           T1 T2 T3 T4
func rightRotate(z *AVLNode) *AVLNode {
    y := z.Left
    T3 := y.Right
    y.Right = z
    z.Left = T3
    updateHeight(z)
    updateHeight(y)
    return y
}

// Left Rotation (RR case)
//   z                  y
//  / \                / \
// T1   y      →      z   x
//     / \           / \ / \
//    T2   x        T1 T2 T3 T4
func leftRotate(z *AVLNode) *AVLNode {
    y := z.Right
    T2 := y.Left
    y.Left = z
    z.Right = T2
    updateHeight(z)
    updateHeight(y)
    return y
}
```

### AVL Insert
```go
func avlInsert(root *AVLNode, val int) *AVLNode {
    if root == nil {
        return &AVLNode{Val: val, Height: 1}
    }
    if val < root.Val {
        root.Left = avlInsert(root.Left, val)
    } else if val > root.Val {
        root.Right = avlInsert(root.Right, val)
    } else {
        return root // Duplicate
    }

    updateHeight(root)
    bf := balanceFactor(root)

    // LL case
    if bf > 1 && val < root.Left.Val {
        return rightRotate(root)
    }
    // RR case
    if bf < -1 && val > root.Right.Val {
        return leftRotate(root)
    }
    // LR case
    if bf > 1 && val > root.Left.Val {
        root.Left = leftRotate(root.Left)
        return rightRotate(root)
    }
    // RL case
    if bf < -1 && val < root.Right.Val {
        root.Right = rightRotate(root.Right)
        return leftRotate(root)
    }
    return root
}
```

---

## Red-Black Tree

### Properties
1. Every node is Red or Black
2. Root is Black
3. Every leaf (nil) is Black
4. Red node's children are Black (no two consecutive reds)
5. All paths from node to leaves have same number of Black nodes

```
        B:7
       /   \
     R:3   B:18
     / \   /  \
   B:1 B:5 R:10 R:22
           / \
         B:8 B:11

B = Black, R = Red
```

### Why Red-Black Trees?
```
AVL Tree:
- Strictly balanced (height diff ≤ 1)
- Faster lookups
- More rotations on insert/delete

Red-Black Tree:
- Loosely balanced (height ≤ 2*log n)
- Faster insert/delete
- Used in: Go's map (not exactly), Java TreeMap, Linux kernel
```

### Go's Built-in Usage
```go
// Go's sort package uses pdqsort (pattern-defeating quicksort)
// Go's map uses hash tables, not trees
// For ordered map, use a third-party library or implement BST

// Example: Ordered map using sorted slice
type OrderedMap struct {
    keys   []int
    values map[int]interface{}
}

func (om *OrderedMap) Set(key int, val interface{}) {
    if _, exists := om.values[key]; !exists {
        idx := sort.SearchInts(om.keys, key)
        om.keys = append(om.keys, 0)
        copy(om.keys[idx+1:], om.keys[idx:])
        om.keys[idx] = key
    }
    om.values[key] = val
}
```

---

## Tree Traversals

### Inorder (Left → Root → Right)
```go
// Recursive
func inorder(root *TreeNode) []int {
    if root == nil {
        return nil
    }
    result := inorder(root.Left)
    result = append(result, root.Val)
    result = append(result, inorder(root.Right)...)
    return result
}

// Iterative
func inorderIterative(root *TreeNode) []int {
    result := []int{}
    stack := []*TreeNode{}
    current := root

    for current != nil || len(stack) > 0 {
        for current != nil {
            stack = append(stack, current)
            current = current.Left
        }
        current = stack[len(stack)-1]
        stack = stack[:len(stack)-1]
        result = append(result, current.Val)
        current = current.Right
    }
    return result
}
// BST inorder gives sorted output
```

### Preorder (Root → Left → Right)
```go
// Recursive
func preorder(root *TreeNode) []int {
    if root == nil {
        return nil
    }
    result := []int{root.Val}
    result = append(result, preorder(root.Left)...)
    result = append(result, preorder(root.Right)...)
    return result
}

// Iterative
func preorderIterative(root *TreeNode) []int {
    if root == nil {
        return nil
    }
    result := []int{}
    stack := []*TreeNode{root}

    for len(stack) > 0 {
        node := stack[len(stack)-1]
        stack = stack[:len(stack)-1]
        result = append(result, node.Val)
        if node.Right != nil {
            stack = append(stack, node.Right)
        }
        if node.Left != nil {
            stack = append(stack, node.Left)
        }
    }
    return result
}
```

### Postorder (Left → Right → Root)
```go
// Recursive
func postorder(root *TreeNode) []int {
    if root == nil {
        return nil
    }
    result := postorder(root.Left)
    result = append(result, postorder(root.Right)...)
    result = append(result, root.Val)
    return result
}

// Iterative
func postorderIterative(root *TreeNode) []int {
    if root == nil {
        return nil
    }
    result := []int{}
    stack := []*TreeNode{root}

    for len(stack) > 0 {
        node := stack[len(stack)-1]
        stack = stack[:len(stack)-1]
        result = append([]int{node.Val}, result...)
        if node.Left != nil {
            stack = append(stack, node.Left)
        }
        if node.Right != nil {
            stack = append(stack, node.Right)
        }
    }
    return result
}
```

### Traversal Visualization
```
Tree:       1
           / \
          2   3
         / \
        4   5

Inorder:   4 2 5 1 3  (Left→Root→Right)
Preorder:  1 2 4 5 3  (Root→Left→Right)
Postorder: 4 5 2 3 1  (Left→Right→Root)
```

---

## Level Order Traversal

### BFS Level by Level
```go
func levelOrder(root *TreeNode) [][]int {
    if root == nil {
        return nil
    }
    result := [][]int{}
    queue := []*TreeNode{root}

    for len(queue) > 0 {
        levelSize := len(queue)
        level := []int{}

        for i := 0; i < levelSize; i++ {
            node := queue[0]
            queue = queue[1:]
            level = append(level, node.Val)
            if node.Left != nil {
                queue = append(queue, node.Left)
            }
            if node.Right != nil {
                queue = append(queue, node.Right)
            }
        }
        result = append(result, level)
    }
    return result
}
// Output for above tree: [[1],[2,3],[4,5]]
```

### Zigzag Level Order
```go
func zigzagLevelOrder(root *TreeNode) [][]int {
    if root == nil {
        return nil
    }
    result := [][]int{}
    queue := []*TreeNode{root}
    leftToRight := true

    for len(queue) > 0 {
        size := len(queue)
        level := make([]int, size)

        for i := 0; i < size; i++ {
            node := queue[0]
            queue = queue[1:]
            if leftToRight {
                level[i] = node.Val
            } else {
                level[size-1-i] = node.Val
            }
            if node.Left != nil {
                queue = append(queue, node.Left)
            }
            if node.Right != nil {
                queue = append(queue, node.Right)
            }
        }
        result = append(result, level)
        leftToRight = !leftToRight
    }
    return result
}
```

---

## BST Operations

### Find Kth Smallest
```go
func kthSmallest(root *TreeNode, k int) int {
    stack := []*TreeNode{}
    current := root
    count := 0

    for current != nil || len(stack) > 0 {
        for current != nil {
            stack = append(stack, current)
            current = current.Left
        }
        current = stack[len(stack)-1]
        stack = stack[:len(stack)-1]
        count++
        if count == k {
            return current.Val
        }
        current = current.Right
    }
    return -1
}
```

### Validate BST
```go
func isValidBST(root *TreeNode) bool {
    return validate(root, math.MinInt64, math.MaxInt64)
}

func validate(node *TreeNode, min, max int) bool {
    if node == nil {
        return true
    }
    if node.Val <= min || node.Val >= max {
        return false
    }
    return validate(node.Left, min, node.Val) &&
        validate(node.Right, node.Val, max)
}
```

### Lowest Common Ancestor (BST)
```go
func lcaBST(root *TreeNode, p, q *TreeNode) *TreeNode {
    if root == nil {
        return nil
    }
    if p.Val < root.Val && q.Val < root.Val {
        return lcaBST(root.Left, p, q)
    }
    if p.Val > root.Val && q.Val > root.Val {
        return lcaBST(root.Right, p, q)
    }
    return root
}
```

### Lowest Common Ancestor (Binary Tree)
```go
func lcaBinaryTree(root, p, q *TreeNode) *TreeNode {
    if root == nil || root == p || root == q {
        return root
    }
    left := lcaBinaryTree(root.Left, p, q)
    right := lcaBinaryTree(root.Right, p, q)
    if left != nil && right != nil {
        return root
    }
    if left != nil {
        return left
    }
    return right
}
```

---

## Tree Properties

### Height of Tree
```go
func height(root *TreeNode) int {
    if root == nil {
        return 0
    }
    return 1 + max(height(root.Left), height(root.Right))
}
```

### Diameter of Tree
```go
func diameterOfBinaryTree(root *TreeNode) int {
    maxDiam := 0
    var depth func(*TreeNode) int
    depth = func(node *TreeNode) int {
        if node == nil {
            return 0
        }
        l, r := depth(node.Left), depth(node.Right)
        if l+r > maxDiam {
            maxDiam = l + r
        }
        return 1 + max(l, r)
    }
    depth(root)
    return maxDiam
}
```

### Check Balanced
```go
func isBalanced(root *TreeNode) bool {
    var check func(*TreeNode) int
    check = func(node *TreeNode) int {
        if node == nil {
            return 0
        }
        l := check(node.Left)
        if l == -1 {
            return -1
        }
        r := check(node.Right)
        if r == -1 {
            return -1
        }
        if abs(l-r) > 1 {
            return -1
        }
        return 1 + max(l, r)
    }
    return check(root) != -1
}
```

### Count Nodes
```go
func countNodes(root *TreeNode) int {
    if root == nil {
        return 0
    }
    return 1 + countNodes(root.Left) + countNodes(root.Right)
}
```

---

## Performance Characteristics

### Time Complexity
| Operation | BST Average | BST Worst | AVL | Red-Black |
|-----------|-------------|-----------|-----|-----------|
| Search | O(log n) | O(n) | O(log n) | O(log n) |
| Insert | O(log n) | O(n) | O(log n) | O(log n) |
| Delete | O(log n) | O(n) | O(log n) | O(log n) |
| Min/Max | O(log n) | O(n) | O(log n) | O(log n) |
| Traversal | O(n) | O(n) | O(n) | O(n) |

### Space Complexity
- O(n) for all tree types
- O(h) for recursive traversal stack (h = height)
- O(n) worst case for skewed tree

---

## Common Patterns

### Path Sum
```go
func hasPathSum(root *TreeNode, target int) bool {
    if root == nil {
        return false
    }
    if root.Left == nil && root.Right == nil {
        return root.Val == target
    }
    return hasPathSum(root.Left, target-root.Val) ||
        hasPathSum(root.Right, target-root.Val)
}
```

### All Root-to-Leaf Paths
```go
func binaryTreePaths(root *TreeNode) []string {
    result := []string{}
    var dfs func(*TreeNode, string)
    dfs = func(node *TreeNode, path string) {
        if node == nil {
            return
        }
        if path != "" {
            path += "->"
        }
        path += strconv.Itoa(node.Val)
        if node.Left == nil && node.Right == nil {
            result = append(result, path)
            return
        }
        dfs(node.Left, path)
        dfs(node.Right, path)
    }
    dfs(root, "")
    return result
}
```

### Serialize and Deserialize
```go
func serialize(root *TreeNode) string {
    if root == nil {
        return "null"
    }
    return fmt.Sprintf("%d,%s,%s", root.Val,
        serialize(root.Left), serialize(root.Right))
}

func deserialize(data string) *TreeNode {
    parts := strings.Split(data, ",")
    idx := 0
    var build func() *TreeNode
    build = func() *TreeNode {
        if parts[idx] == "null" {
            idx++
            return nil
        }
        val, _ := strconv.Atoi(parts[idx])
        idx++
        return &TreeNode{Val: val, Left: build(), Right: build()}
    }
    return build()
}
```

### Mirror / Invert Tree
```go
func invertTree(root *TreeNode) *TreeNode {
    if root == nil {
        return nil
    }
    root.Left, root.Right = invertTree(root.Right), invertTree(root.Left)
    return root
}
```

---

## Interview Problems

### 1. Maximum Path Sum
```go
func maxPathSum(root *TreeNode) int {
    maxSum := math.MinInt32
    var dfs func(*TreeNode) int
    dfs = func(node *TreeNode) int {
        if node == nil {
            return 0
        }
        left := max(0, dfs(node.Left))
        right := max(0, dfs(node.Right))
        maxSum = max(maxSum, node.Val+left+right)
        return node.Val + max(left, right)
    }
    dfs(root)
    return maxSum
}
```

### 2. Right Side View
```go
func rightSideView(root *TreeNode) []int {
    result := []int{}
    if root == nil {
        return result
    }
    queue := []*TreeNode{root}
    for len(queue) > 0 {
        size := len(queue)
        for i := 0; i < size; i++ {
            node := queue[0]
            queue = queue[1:]
            if i == size-1 {
                result = append(result, node.Val)
            }
            if node.Left != nil {
                queue = append(queue, node.Left)
            }
            if node.Right != nil {
                queue = append(queue, node.Right)
            }
        }
    }
    return result
}
```

### 3. Construct BST from Preorder
```go
func bstFromPreorder(preorder []int) *TreeNode {
    if len(preorder) == 0 {
        return nil
    }
    root := &TreeNode{Val: preorder[0]}
    i := 1
    for i < len(preorder) && preorder[i] < preorder[0] {
        i++
    }
    root.Left = bstFromPreorder(preorder[1:i])
    root.Right = bstFromPreorder(preorder[i:])
    return root
}
```

---

## Summary

### Key Concepts
1. BST property: left < root < right
2. Inorder traversal of BST gives sorted output
3. AVL and Red-Black trees guarantee O(log n) operations
4. Height determines performance — keep trees balanced
5. Use dummy nodes and recursion for clean implementations

### When to Use
- BST: Ordered data, range queries, floor/ceiling operations
- AVL: Read-heavy workloads needing strict balance
- Red-Black: Write-heavy workloads (used in most standard libraries)
- Trie: String prefix operations (see separate document)
- Heap: Priority queue operations (see separate document)
