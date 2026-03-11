# Search Functionality Guide

## 🔍 How to Use the Search Feature

### Option 1: HTML Search Interface (Recommended)

Open `search.html` in your browser for a full-featured search experience.

**Features:**
- ✅ Search across all documents or specific ones
- ✅ Real-time highlighting of search terms
- ✅ Case-sensitive search option
- ✅ Whole word matching
- ✅ Regular expression support
- ✅ Context preview for each result
- ✅ Section and line number information
- ✅ Beautiful, responsive UI

**How to Use:**
1. Open `docs/search.html` in any web browser
2. Enter your search term (e.g., "goroutine", "interface", "binary search")
3. Select which documents to search (or search all)
4. Click "Search" or press Enter
5. View results with highlighted matches

**Search Options:**
- **Case Sensitive**: Match exact case
- **Whole Word**: Match complete words only
- **Regular Expression**: Use regex patterns

**Search Tips:**
- Use quotes for exact phrases: `"error handling"`
- Use OR for multiple terms: `goroutine OR channel`
- Use - to exclude: `concurrency -mutex`
- Enable regex for patterns: `\b\w+able\b`

### Option 2: Browser's Built-in Search (Ctrl+F / Cmd+F)

For quick searches within a single document:

1. Open any markdown file in your browser or editor
2. Press `Ctrl+F` (Windows/Linux) or `Cmd+F` (Mac)
3. Type your search term
4. Navigate through results with Enter or arrow keys

**Pros:**
- Fast and simple
- Works offline
- No setup required

**Cons:**
- Only searches current document
- No advanced features
- No context preview

### Option 3: GitHub Search

If viewing on GitHub:

1. Press `/` to activate GitHub search
2. Type your search term
3. Filter by file or content
4. View results in GitHub interface

### Option 4: VS Code Search

If using VS Code:

1. Press `Ctrl+Shift+F` (Windows/Linux) or `Cmd+Shift+F` (Mac)
2. Enter search term
3. Optionally use regex, case-sensitive, or whole word options
4. View results across all files

**Advanced VS Code Search:**
```
# Search in specific files
goroutine *.md

# Exclude files
interface -file:README.md

# Regex search
\bfunc\s+\w+\(
```

### Option 5: Command Line Search (grep)

For terminal users:

```bash
# Search in all markdown files
grep -r "goroutine" docs/*.md

# Case-insensitive search
grep -ri "interface" docs/*.md

# Show line numbers
grep -rn "channel" docs/*.md

# Show context (2 lines before and after)
grep -rn -C 2 "mutex" docs/*.md

# Search with regex
grep -rE "\bfunc\s+\w+\(" docs/*.md

# Count occurrences
grep -rc "error" docs/*.md

# Search multiple terms
grep -rE "goroutine|channel" docs/*.md
```

## 📊 Search Comparison

| Feature | HTML Search | Browser Ctrl+F | GitHub | VS Code | grep |
|---------|-------------|----------------|--------|---------|------|
| Multi-document | ✅ | ❌ | ✅ | ✅ | ✅ |
| Highlighting | ✅ | ✅ | ✅ | ✅ | ❌ |
| Context preview | ✅ | ❌ | ✅ | ✅ | ✅ |
| Regex support | ✅ | ❌ | ✅ | ✅ | ✅ |
| Offline | ✅ | ✅ | ❌ | ✅ | ✅ |
| Setup required | ❌ | ❌ | ❌ | ✅ | ❌ |
| Speed | Fast | Instant | Medium | Fast | Very Fast |

## 🎯 Common Search Queries

### Basics
```
variable
function
pointer
struct
interface
slice
map
```

### Concurrency
```
goroutine
channel
select
mutex
waitgroup
context
atomic
```

### Data Structures
```
array
linked list
stack
queue
tree
graph
heap
```

### Algorithms
```
sort
search
recursion
dynamic programming
greedy
two pointers
sliding window
```

### Patterns
```
error handling
testing
benchmark
design pattern
best practice
```

## 💡 Pro Tips

1. **Start Broad, Then Narrow**
   - First search: "concurrency"
   - Then: "goroutine channel"
   - Finally: "goroutine leak"

2. **Use Document-Specific Search**
   - Select specific document in HTML search
   - Faster and more relevant results

3. **Combine Search Methods**
   - HTML search to find documents
   - Ctrl+F to navigate within document

4. **Save Common Searches**
   - Bookmark HTML search with query parameters
   - Create VS Code search presets

5. **Use Table of Contents First**
   - Check TOC before searching
   - Often faster for known topics

## 🚀 Quick Start

**For Beginners:**
1. Open `search.html` in browser
2. Search for topic (e.g., "interface")
3. Click result to see context
4. Open full document for details

**For Advanced Users:**
1. Use VS Code workspace search
2. Enable regex for complex patterns
3. Use grep for command-line workflow
4. Combine with git grep for version control

## 📝 Search Examples

### Example 1: Find All Error Handling Patterns
```
HTML Search: "error handling"
Select: 05-error-handling.md
Options: Whole Word
```

### Example 2: Find Goroutine Examples
```
HTML Search: goroutine
Select: 04-concurrency.md
View: All code examples with goroutines
```

### Example 3: Find Interview Questions About Interfaces
```
HTML Search: interface
Select: 08-interview-questions.md
Filter: Questions only
```

## 🔧 Troubleshooting

**Search not working?**
- Ensure markdown files are in same directory as search.html
- Check browser console for errors
- Try different browser

**No results found?**
- Check spelling
- Try synonyms (e.g., "function" vs "method")
- Disable case-sensitive option
- Search in all documents

**Too many results?**
- Enable "Whole Word" option
- Search in specific document
- Use more specific terms
- Use quotes for exact phrases

## 📚 Additional Resources

- [Markdown Guide](https://www.markdownguide.org/)
- [Regex Tutorial](https://regexr.com/)
- [VS Code Search Tips](https://code.visualstudio.com/docs/editor/codebasics#_search-across-files)
- [grep Manual](https://www.gnu.org/software/grep/manual/grep.html)

---

**Happy Searching! 🔍**
