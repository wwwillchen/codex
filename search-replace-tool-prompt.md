# Search-Replace Edit Tool Implementation Prompt

You are implementing an edit tool that uses **context-based string matching** (not line numbers) to locate and apply changes to files. This approach is more robust than line-number-based editing because it works even when files have been modified.

## Core Format

Use a unified diff-like format wrapped in clear markers:

```
*** Begin Patch
[ one or more file operations ]
*** End Patch
```

## File Operations

### 1. Add File
```
*** Add File: <relative-path>
+first line of content
+second line of content
```

### 2. Delete File
```
*** Delete File: <relative-path>
```

### 3. Update File
```
*** Update File: <relative-path>
@@ [optional context header]
 context line (unchanged)
-line to remove
+line to add
 context line (unchanged)
```

### 4. Update + Rename File
```
*** Update File: <old-path>
*** Move to: <new-path>
@@
-old content
+new content
```

## Line Prefix Rules

Every line within a hunk MUST start with one of:
- ` ` (space) = context line, used for matching location, not modified
- `-` = line to be removed
- `+` = line to be added

## Context Matching Strategy

The tool locates where to apply changes by matching context lines against the file. Implement **cascading fuzzy matching** with decreasing strictness:

### Pass 1: Exact Match
```
file_line == pattern_line
```

### Pass 2: Trailing Whitespace Ignored
```
file_line.trim_end() == pattern_line.trim_end()
```

### Pass 3: All Edge Whitespace Ignored
```
file_line.trim() == pattern_line.trim()
```

### Pass 4: Unicode Normalization (Optional)
Normalize common Unicode variants to ASCII before comparing:
- En-dash `–`, em-dash `—`, etc. → `-`
- Smart quotes `"` `"` `'` `'` → `"` `'`
- Non-breaking space → regular space

## Context Header Disambiguation

When 3 lines of context aren't enough to uniquely identify a location (e.g., repeated patterns), use `@@` headers to specify the enclosing scope:

### Single Header
```
@@ class MyClass
 def method(self):
-    old_code
+    new_code
```

### Nested Headers (for deeper disambiguation)
```
@@ class MyClass
@@ def method():
 # context line
-old_code
+new_code
```

The matching algorithm should:
1. Find the context header (e.g., line containing "class MyClass")
2. Continue searching from that point for the actual change context
3. Apply the change at the matched location

## Algorithm Pseudocode

```python
def apply_patch(file_content: str, hunks: list[Hunk]) -> str:
    lines = file_content.split('\n')
    current_index = 0
    replacements = []

    for hunk in hunks:
        # If hunk has context header, find it first
        if hunk.context_header:
            idx = find_line_containing(lines, hunk.context_header, current_index)
            if idx is None:
                raise Error(f"Context header not found: {hunk.context_header}")
            current_index = idx + 1

        # Find the old_lines pattern in the file
        old_lines = [line for line in hunk.lines if line.startswith(' ') or line.startswith('-')]
        old_lines = [line[1:] for line in old_lines]  # strip prefix

        match_idx = seek_sequence(lines, old_lines, current_index)
        if match_idx is None:
            raise Error(f"Could not find pattern:\n{old_lines}")

        # Compute new lines
        new_lines = []
        for line in hunk.lines:
            if line.startswith(' '):
                new_lines.append(line[1:])
            elif line.startswith('+'):
                new_lines.append(line[1:])
            # skip '-' lines (they're removed)

        replacements.append((match_idx, len(old_lines), new_lines))
        current_index = match_idx + len(old_lines)

    # Apply replacements in reverse order to preserve indices
    for start_idx, old_len, new_lines in reversed(sorted(replacements)):
        lines[start_idx:start_idx + old_len] = new_lines

    return '\n'.join(lines)


def seek_sequence(lines: list[str], pattern: list[str], start: int) -> int | None:
    """Find pattern in lines starting at index start, using fuzzy matching."""

    # Try exact match
    for i in range(start, len(lines) - len(pattern) + 1):
        if lines[i:i+len(pattern)] == pattern:
            return i

    # Try with trimmed whitespace
    for i in range(start, len(lines) - len(pattern) + 1):
        if all(lines[i+j].strip() == pattern[j].strip() for j in range(len(pattern))):
            return i

    return None
```

## Best Practices for Patch Authors

1. **Include 3 lines of context** above and below each change by default
2. **Use context headers** (`@@`) when patterns repeat in the file
3. **Don't duplicate context** between adjacent hunks
4. **Use relative paths only**, never absolute paths
5. **Prefix all new content with `+`**, even in new files

## Example: Complete Patch

```
*** Begin Patch
*** Add File: src/utils/helper.py
+def helper():
+    return "hello"

*** Update File: src/main.py
@@ class Application
@@ def initialize():
 def initialize(self):
     self.config = {}
-    self.ready = False
+    self.ready = True
+    self.helper = helper()

*** Delete File: src/deprecated.py
*** End Patch
```

## Error Handling

The tool should produce clear errors:

1. **Pattern not found**: Show what pattern was being searched for
   ```
   Failed to find expected lines in src/main.py:
       self.config = {}
       self.ready = False
   ```

2. **Ambiguous match**: When pattern matches multiple locations
   ```
   Pattern matches multiple locations in src/main.py. Add more context or use @@ headers.
   ```

3. **Parse error**: Invalid patch syntax
   ```
   Invalid patch hunk on line 5: expected line to start with ' ', '-', or '+'
   ```

## Implementation Checklist

- [ ] Parse patch format (Begin/End markers, file operations, hunks)
- [ ] Handle Add/Delete/Update/Move file operations
- [ ] Implement context line matching with fuzzy fallbacks
- [ ] Support `@@` context headers for disambiguation
- [ ] Apply replacements in correct order (reverse index order)
- [ ] Preserve file trailing newlines correctly
- [ ] Handle edge cases (empty files, EOF changes, pure additions)
- [ ] Produce actionable error messages

## Testing Scenarios

1. Simple single-line replacement
2. Multi-line replacement with context
3. Multiple hunks in same file
4. Adding lines at end of file (`*** End of File` marker)
5. Whitespace variations (tabs vs spaces, trailing whitespace)
6. Unicode content matching
7. Repeated patterns requiring `@@` disambiguation
8. File creation, deletion, and renaming
