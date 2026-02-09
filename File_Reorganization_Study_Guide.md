# File Reorganization System
## Complete Study Guide & Technical Reference

**Author:** Technical Study Material  
**Date:** February 9, 2026  
**Topics:** Bash Scripting • File Processing • Data Organization  

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Problem Statement & Requirements](#problem-statement--requirements)
3. [Technical Architecture](#technical-architecture)
4. [Complete Code Implementation](#complete-code-implementation)
5. [Detailed Code Explanation](#detailed-code-explanation)
6. [Bash Fundamentals Reference](#bash-fundamentals-reference)
7. [Advanced Concepts & Best Practices](#advanced-concepts--best-practices)
8. [Troubleshooting Guide](#troubleshooting-guide)
9. [Practical Exercises](#practical-exercises)

---

## 1. Executive Summary

### Overview

This document provides a comprehensive guide to understanding and implementing a file reorganization system using Bash scripting. The system reorganizes 30 text files from a nested directory structure into a flat, category-based structure.

### What This System Does

- **Reads category metadata from files** - Extracts categorization information from each file's content
- **Creates category-based directory structure** - Organizes files into directories by category
- **Renames and moves files systematically** - Preserves path information in new filenames
- **Handles Unicode characters, spaces, and special characters** - Robust support for international filenames
- **Generates cryptographic hash for verification** - SHA-256 hash ensures correctness

### Key Learning Outcomes

1. Master Bash scripting fundamentals and advanced techniques
2. Understand file processing with `find`, `grep`, and text manipulation
3. Learn proper handling of special characters and Unicode
4. Implement robust file system operations
5. Apply locale-aware sorting and hashing techniques

---

## 2. Problem Statement & Requirements

### The Challenge

You have 30 text files scattered across a complex nested directory structure. Each file contains category metadata in its first line. Your task is to reorganize these files into a flat structure based on their categories while preserving path information in the filename.

### Input Structure Example

```
archive/
  2024/
    advanced/
      file29.txt
    file17.txt
  módulo-3/
    intro/
      file01.txt
content/
  part 2/
    données/
      file-name/
        file23.txt
docs/
  café-2024/
    fіle27.txt
  chapter1/
    naïve/
      file03.txt
```

### File Content Format

Each file starts with a category declaration:

```
category: scripts

TECHNICAL DOCUMENTATION - File 05
...
```

### Requirements Table

| Requirement | Description |
|-------------|-------------|
| **Category Extraction** | Read FIRST line matching `category: ...` pattern |
| **Directory Creation** | Create directory for each unique category |
| **File Naming** | Format: `{category}/{path-with-dashes}-{filename}` |
| **Path Conversion** | Replace all `/` with `-` in directory path |
| **Special Characters** | Handle Unicode, spaces, accents, diacritics |
| **Verification** | Generate SHA-256 hash with `LC_ALL=C` sorting |

### Example Transformations

**Before:**
```
archive/2024/file17.txt (contains: category: café)
content/part 2/données/file-name/file23.txt (contains: category: scripts)
docs/chapter1/naïve/file03.txt (contains: category: 日本語)
```

**After:**
```
café/archive-2024-file17.txt
scripts/content-part 2-données-file-name-file23.txt
日本語/docs-chapter1-naïve-file03.txt
```

---

## 3. Technical Architecture

### System Workflow

The reorganization system follows a five-stage pipeline architecture:

```
┌─────────────────────────────────────────────────────────────┐
│ Stage 1: File Discovery                                     │
│ Tool: find with null-delimited output                       │
│ Output: List of all .txt files                              │
└────────────────────┬────────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────────┐
│ Stage 2: Metadata Extraction                                │
│ Tools: grep -m 1 + cut                                       │
│ Output: Category name for each file                         │
└────────────────────┬────────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────────┐
│ Stage 3: Path Processing                                    │
│ Tools: dirname + basename + tr                              │
│ Output: Transformed path components                         │
└────────────────────┬────────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────────┐
│ Stage 4: File Relocation                                    │
│ Tools: mkdir -p + mv                                         │
│ Output: Files moved to new structure                        │
└────────────────────┬────────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────────┐
│ Stage 5: Hash Generation                                    │
│ Tools: find + LC_ALL=C sort + sha256sum                     │
│ Output: Verification hash                                   │
└─────────────────────────────────────────────────────────────┘
```

### Data Flow Diagram

```
Input Files (archive/, content/, docs/, project/)
     ↓
[find -print0] → Null-delimited file list
     ↓
[while read -d ''] → Process each file
     ↓
[grep -m 1 + cut] → Extract category
     ↓
[dirname + basename + tr] → Transform path
     ↓
[mkdir -p] → Create category directory
     ↓
[mv] → Move file to new location
     ↓
Reorganized Files (category-based structure)
     ↓
[find | LC_ALL=C sort | sha256sum] → Hash
     ↓
Verification Hash: 1e1679f4cbe85cbbe90a94aaa36158e22724587ae7f05a9c82a44add694c23be
```

---

## 4. Complete Code Implementation

### Full Script with Detailed Comments

```bash
#!/bin/bash

# ================================================================
# File Reorganization Script
# ================================================================
# Purpose: Reorganize files based on category metadata
# Author: Study Guide Example
# Date: 2025-02-09
# ================================================================

# ----------------------------------------------------------------
# Configuration and Setup
# ----------------------------------------------------------------

echo "Starting file reorganization..."
echo "================================"
echo ""

# ----------------------------------------------------------------
# Main Processing Loop
# ----------------------------------------------------------------

# Find all .txt files in specified directories and process them
# -print0: Use null character as delimiter (handles spaces/special chars)
# IFS=: Clear Internal Field Separator to preserve whitespace
# read -r: Read raw input (don't interpret backslashes)
# -d '': Use null delimiter to match find's -print0

find archive content docs project -name "*.txt" -type f -print0 | \
while IFS= read -r -d '' filepath; do
    
    # ============================================================
    # Step 1: Extract Category from File
    # ============================================================
    # grep -m 1: Match only the FIRST occurrence
    # ^category:: Match lines starting with "category: "
    # cut -d' ' -f2-: Cut using space delimiter, get field 2 onwards
    
    category=$(grep -m 1 "^category: " "$filepath" | cut -d' ' -f2-)
    
    # ============================================================
    # Step 2: Parse File Path Components
    # ============================================================
    
    # Extract directory path (everything except filename)
    dirpath=$(dirname "$filepath")
    
    # Extract just the filename
    filename=$(basename "$filepath")
    
    # ============================================================
    # Step 3: Transform Path (Replace / with -)
    # ============================================================
    # tr '/' '-': Translate (replace) all slashes with dashes
    # This converts "content/part 2/données" to "content-part 2-données"
    
    path_with_dashes=$(echo "$dirpath" | tr '/' '-')
    
    # ============================================================
    # Step 4: Create Category Directory
    # ============================================================
    # mkdir -p: Create directory and all parent directories
    # -p flag: Don't error if directory already exists
    
    mkdir -p "$category"
    
    # ============================================================
    # Step 5: Build New Path and Move File
    # ============================================================
    # Format: {category}/{path-with-dashes}-{filename}
    
    newpath="${category}/${path_with_dashes}-${filename}"
    
    # Display progress
    echo "Moving: $filepath -> $newpath"
    
    # Move the file
    mv "$filepath" "$newpath"
    
done

# ----------------------------------------------------------------
# Post-Processing and Verification
# ----------------------------------------------------------------

echo ""
echo "================================"
echo "Reorganization complete!"
echo "================================"
echo ""
echo "Generating verification hash..."
echo ""

# ================================================================
# Generate SHA-256 Hash for Verification
# ================================================================
# LC_ALL=C: Use C locale for consistent byte-wise sorting
# find .: Find all files in current directory
# -type f: Only files (not directories)
# sort: Sort the list alphabetically
# sha256sum: Generate SHA-256 hash of the sorted list

find . -type f | LC_ALL=C sort | sha256sum

echo ""
echo "================================"
echo "Submit the hash (first part before the space/dash)"
echo "================================"
```

---

## 5. Detailed Code Explanation

### 5.1 The `find` Command

The `find` command is the foundation of our file discovery system. It recursively searches directories and outputs matching file paths.

#### Syntax Breakdown

```bash
find archive content docs project -name "*.txt" -type f -print0
```

| Component | Explanation |
|-----------|-------------|
| `archive content docs project` | Starting directories to search recursively |
| `-name "*.txt"` | Match only files with .txt extension (* is wildcard) |
| `-type f` | Only regular files (exclude directories, symlinks, etc.) |
| `-print0` | Use null character (\0) as delimiter instead of newline |

#### Why `-print0`?

The `-print0` flag is **CRITICAL** for handling filenames with special characters:

1. **Default delimiter (newline) breaks on filenames with spaces**
   - Example: `archive/part 2/file.txt` would be split into three parts
   
2. **Null character (\0) cannot appear in Unix filenames**
   - Guaranteed safe delimiter
   
3. **Enables safe processing of Unicode and special characters**
   - Works with: `café`, `données`, `模块`, `naïve`, etc.

#### Example Output Comparison

**Without `-print0` (WRONG):**
```
archive/part 2/file.txt
```
Would be split by the space into:
```
archive/part
2/file.txt
```

**With `-print0` (CORRECT):**
```
archive/part 2/file.txt\0
```
Treated as single complete path.

---

### 5.2 The `while` Loop with IFS

```bash
while IFS= read -r -d '' filepath; do
```

This line is deceptively powerful. Let's break it down:

| Component | Explanation |
|-----------|-------------|
| `IFS=` | Clear Internal Field Separator - prevents word splitting on spaces/tabs |
| `read -r` | Raw mode - don't interpret backslashes as escape characters |
| `-d ''` | Use null character as delimiter (matches find's -print0) |
| `filepath` | Variable name to store each file path |

#### Why Each Part Matters

**`IFS=` (Internal Field Separator)**
- Default IFS is space, tab, and newline
- Clearing it prevents splitting on spaces
- Essential for `part 2` in filenames

**`read -r` (Raw Mode)**
- Prevents backslash interpretation
- Important for paths like `test\file.txt`

**`-d ''` (Null Delimiter)**
- Matches `find -print0`
- Empty string between quotes represents null character

#### Example

```bash
# File path with space: "content/part 2/file.txt"

# WITHOUT IFS= (WRONG):
read filepath              # filepath = "content/part"
                           # "2/file.txt" goes to next iteration

# WITH IFS= (CORRECT):
IFS= read -r -d '' filepath  # filepath = "content/part 2/file.txt"
```

---

### 5.3 Category Extraction with `grep` and `cut`

```bash
category=$(grep -m 1 "^category: " "$filepath" | cut -d' ' -f2-)
```

This is a **pipeline** of two commands connected by pipe (`|`).

#### Step-by-Step Breakdown

**Step 1: `grep -m 1 "^category: " "$filepath"`**

| Component | Meaning |
|-----------|---------|
| `grep` | Search for pattern in file |
| `-m 1` | Maximum 1 match (stop after first) |
| `"^category: "` | Pattern to match |
| `^` | Regex anchor for "start of line" |
| `"$filepath"` | File to search (quotes protect spaces) |

**Output of grep:**
```
category: scripts
```

**Step 2: `cut -d' ' -f2-`**

| Component | Meaning |
|-----------|---------|
| `cut` | Extract portions of text |
| `-d' '` | Delimiter is space character |
| `-f2-` | Field 2 onwards (everything after first space) |

**Output of cut:**
```
scripts
```

**Step 3: `$( ... )` Command Substitution**
- Runs the commands inside
- Captures the output
- Stores in variable

#### Complete Example

**File content:**
```
category: naïve-bayes

TECHNICAL DOCUMENTATION
This is file 21.
```

**Execution:**
1. `grep -m 1 "^category: " file.txt` → `category: naïve-bayes`
2. `cut -d' ' -f2-` → `naïve-bayes`
3. `category="naïve-bayes"`

---

### 5.4 Path Manipulation

#### Extracting Directory and Filename

```bash
dirpath=$(dirname "$filepath")
filename=$(basename "$filepath")
```

| Full Path | `dirname` Result | `basename` Result |
|-----------|------------------|-------------------|
| `archive/2024/file17.txt` | `archive/2024` | `file17.txt` |
| `content/part 2/données/file23.txt` | `content/part 2/données` | `file23.txt` |
| `docs/café-2024/fіle27.txt` | `docs/café-2024` | `fіle27.txt` |

#### Path Transformation with `tr`

```bash
path_with_dashes=$(echo "$dirpath" | tr '/' '-')
```

The `tr` (translate) command replaces characters:

- **`tr '/' '-'`** means "translate all `/` to `-`"
- Each forward slash becomes a dash
- Preserves all other characters including spaces and Unicode

**Examples:**
```
archive/2024              → archive-2024
content/part 2/données    → content-part 2-données
docs/módulo-3/café-2024   → docs-módulo-3-café-2024
```

#### Why Use `echo` with `tr`?

`tr` reads from stdin (standard input), so we pipe the variable content to it:

```bash
# This works:
echo "$dirpath" | tr '/' '-'

# This would NOT work (tr expects file or stdin):
tr '/' '-' "$dirpath"  # WRONG!
```

---

### 5.5 Directory Creation and File Moving

```bash
mkdir -p "$category"
newpath="${category}/${path_with_dashes}-${filename}"
mv "$filepath" "$newpath"
```

#### `mkdir -p` Explained

The `-p` flag has two crucial behaviors:

1. **Creates parent directories as needed**
   - If category is `café`, creates `café/` directory
   - If category is `data/subset`, creates `data/` then `data/subset/`

2. **No error if directory already exists** (idempotent)
   - First file in category: creates directory
   - Subsequent files: silently succeeds
   - Makes script safe to run multiple times

**Why This Matters:**
```bash
# Without -p:
mkdir "café"     # Works first time
mkdir "café"     # ERROR: Directory exists

# With -p:
mkdir -p "café"  # Works first time
mkdir -p "café"  # Works every time (no error)
```

#### String Interpolation

```bash
newpath="${category}/${path_with_dashes}-${filename}"
```

**Variable expansion with `${}` syntax:**

- `${variable}` is preferred over `$variable` for clarity
- Allows concatenation without ambiguity
- Quotes preserve spaces and special characters

**Example:**
```bash
category="scripts"
path_with_dashes="content-part 2-données"
filename="file23.txt"

# Result:
newpath="scripts/content-part 2-données-file23.txt"
```

**Why curly braces?**
```bash
# Ambiguous without braces:
echo "$category_file"      # Looks for variable named "category_file"

# Clear with braces:
echo "${category}_file"    # Looks for "category" and appends "_file"
```

---

### 5.6 Hash Generation and Verification

```bash
find . -type f | LC_ALL=C sort | sha256sum
```

This pipeline generates a cryptographic hash of the file structure.

#### Why `LC_ALL=C`?

Locale settings affect how `sort` orders characters. Different locales produce different sort orders, which would generate different hashes for the same file set.

| Locale | Behavior | Example Sort Order |
|--------|----------|-------------------|
| `en_US.UTF-8` | Dictionary sort (ignores case, handles accents specially) | `café`, `Café`, `CAFÉ` |
| `C` | Byte-wise ASCII sort (predictable, consistent across all systems) | `CAFÉ`, `café`, `Café` |

**Why This Matters:**
```bash
# Different locales, same files, DIFFERENT hashes:
LC_ALL=en_US.UTF-8 find . -type f | sort | sha256sum
# Output: abc123...

LC_ALL=C find . -type f | sort | sha256sum  
# Output: def456...

# C locale ensures everyone gets the same hash:
LC_ALL=C find . -type f | sort | sha256sum
# Output: 1e1679f4... (always the same)
```

#### Pipeline Breakdown

```bash
find . -type f           # List all files recursively
|                        # Pipe to next command
LC_ALL=C sort            # Sort in C locale (byte order)
|                        # Pipe to next command
sha256sum                # Generate cryptographic hash
```

**Output Format:**
```
1e1679f4cbe85cbbe90a94aaa36158e22724587ae7f05a9c82a44add694c23be  -
                           ↑
                    64-character hex hash
```

The `-` indicates sha256sum read from stdin (piped input).

---

## 6. Bash Fundamentals Reference

### 6.1 Quoting and Escaping

| Type | Syntax | Behavior | Example |
|------|--------|----------|---------|
| **Double Quotes** | `"$variable"` | Variables expand, preserves spaces | `echo "Hello $name"` |
| **Single Quotes** | `'$variable'` | Literal string, no expansion | `echo 'Hello $name'` → `Hello $name` |
| **No Quotes** | `$variable` | Word splitting on spaces - **AVOID** | `echo $file` breaks on `my file.txt` |

**Best Practice:** Always quote variables that might contain spaces:
```bash
# WRONG:
mv $filepath $newpath      # Breaks on spaces

# CORRECT:
mv "$filepath" "$newpath"  # Works with all characters
```

---

### 6.2 Variable Assignment and Expansion

```bash
# Assignment (NO spaces around =)
name="value"              # Correct
count=42                  # Correct
path="/home/user"         # Correct

# WRONG:
name = "value"            # ERROR: command "name" not found

# Expansion
echo "$name"              # Basic expansion
echo "${name}"            # Explicit boundaries (preferred)
echo "${name}_file"       # Concatenation
echo "$name_file"         # WRONG: looks for variable "name_file"

# Command substitution
result=$(command)         # Modern style (preferred)
result=`command`          # Old style (avoid)
```

---

### 6.3 Common Bash Patterns

#### Conditional Execution

```bash
# AND operator (&&) - run second command if first succeeds
mkdir -p dir && cd dir

# OR operator (||) - run second command if first fails
command || echo "Failed!"

# Chaining
test -f file && echo "exists" || echo "missing"
```

#### File Tests

```bash
if [ -f "file.txt" ]; then    # File exists
if [ -d "dir" ]; then         # Directory exists  
if [ -r "file" ]; then        # File is readable
if [ -w "file" ]; then        # File is writable
if [ -x "script" ]; then      # File is executable
if [ -s "file" ]; then        # File exists and not empty
```

#### String Tests

```bash
if [ -z "$var" ]; then        # String is empty
if [ -n "$var" ]; then        # String is not empty
if [ "$a" = "$b" ]; then      # Strings are equal
if [ "$a" != "$b" ]; then     # Strings are not equal
```

---

### 6.4 Pipes and Redirection

```bash
# Pipe (|) - send output to next command
command1 | command2

# Redirect output to file
command > file.txt        # Overwrite
command >> file.txt       # Append

# Redirect error output
command 2> errors.txt     # Errors only
command &> all.txt        # Both stdout and stderr
command 2>&1              # Redirect stderr to stdout

# Redirect input from file
command < input.txt

# Here document
cat <<EOF
Multiple lines
of text
EOF
```

---

## 7. Advanced Concepts & Best Practices

### 7.1 Unicode and Character Encoding

Our script handles various Unicode characters because Bash treats strings as byte sequences. Key considerations:

#### UTF-8 Encoding
- Most common Unicode encoding
- Variable-length: 1-4 bytes per character
- ASCII characters (a-z, 0-9) = 1 byte
- Accented characters (é, ñ) = 2 bytes
- Chinese/Japanese (日, 本) = 3 bytes
- Emoji = 4 bytes

#### Why It Works
```bash
# These all work correctly:
category="café"          # French
category="münchen"       # German
category="naïve-bayes"   # Diacritics
category="日本語"         # Japanese
```

Bash doesn't interpret the bytes, just passes them through to:
- `mkdir` (creates directory with Unicode name)
- `mv` (moves file with Unicode name)
- File system (stores Unicode filename)

---

### 7.2 Error Handling Strategies

Production scripts should include robust error handling:

```bash
#!/bin/bash

# Strict error handling
set -euo pipefail
# -e: Exit immediately if any command fails
# -u: Error on undefined variables
# -o pipefail: Pipeline fails if ANY command fails

# Validate prerequisites
if [ ! -d "archive" ]; then
    echo "Error: archive directory not found" >&2
    exit 1
fi

# Check for required commands
for cmd in find grep cut tr mkdir mv sha256sum; do
    if ! command -v "$cmd" &>/dev/null; then
        echo "Error: required command '$cmd' not found" >&2
        exit 1
    fi
done

# Log errors to file
exec 2>> error.log

# Trap errors and cleanup
cleanup() {
    echo "Error occurred. Cleaning up..."
    # Restore original state if needed
}
trap cleanup ERR EXIT
```

---

### 7.3 Performance Considerations

| Approach | Speed | Use Case | Example |
|----------|-------|----------|---------|
| `while read` loop | Moderate | Small to medium datasets (< 10,000 files) | Our script |
| `xargs -P` parallel | Fast | Large datasets, CPU-bound tasks | `find ... -print0 \| xargs -0 -P 4 -I {} process {}` |
| `find -exec` | Slow | Simple operations only | `find . -exec echo {} \;` |

#### Optimizing Our Script

For large datasets (10,000+ files):

```bash
# Parallel processing (4 processes)
find archive content docs project -name "*.txt" -type f -print0 | \
xargs -0 -P 4 -I {} bash -c '
    category=$(grep -m 1 "^category: " "{}" | cut -d" " -f2-)
    # ... rest of processing
'
```

---

### 7.4 Debugging Techniques

```bash
# Enable debug mode (shows each command before execution)
bash -x script.sh

# Within script
set -x                     # Enable debugging
# ... commands to debug ...
set +x                     # Disable debugging

# Verbose variable inspection
echo "filepath='$filepath'"
echo "category='$category'" | cat -A  # Show hidden characters

# Test with single file
filepath="archive/2024/file17.txt"
category=$(grep -m 1 "^category: " "$filepath" | cut -d' ' -f2-)
echo "Extracted category: '$category'"
dirpath=$(dirname "$filepath")
echo "Directory: '$dirpath'"
filename=$(basename "$filepath")
echo "Filename: '$filename'"
path_with_dashes=$(echo "$dirpath" | tr '/' '-')
echo "Transformed: '$path_with_dashes'"
```

#### Common Debug Output

```bash
+ find archive content docs project -name '*.txt' -type f -print0
+ IFS=
+ read -r -d '' filepath
++ grep -m 1 '^category: ' 'archive/2024/file17.txt'
++ cut -d' ' -f2-
+ category=café
++ dirname 'archive/2024/file17.txt'
+ dirpath=archive/2024
++ basename 'archive/2024/file17.txt'
+ filename=file17.txt
```

---

## 8. Troubleshooting Guide

### Common Issues and Solutions

| Problem | Cause | Solution |
|---------|-------|----------|
| Files with spaces not processed | Missing `-print0` or `IFS` setting | Use `find -print0` with `IFS= read -d ''` |
| Wrong category extracted | Multiple `category:` lines in file | Use `grep -m 1` for first match only |
| Hash doesn't match expected | Wrong locale for sorting | Must use `LC_ALL=C sort` |
| Directory creation fails | Missing `-p` flag or permissions | Use `mkdir -p` and check write permissions |
| Unicode filenames broken | Improper quoting | Always quote variables: `"$filepath"` |
| Script fails on first error | No error handling | Add `set -e` or check return codes |
| Empty category variable | File doesn't have `category:` line | Add validation: `[ -z "$category" ] && continue` |

### Detailed Troubleshooting Steps

#### Issue: Files with Spaces Not Processed

**Symptoms:**
```
Moving: archive/part -> new-location
mv: cannot stat 'archive/part': No such file or directory
```

**Diagnosis:**
```bash
# Check if using correct delimiters
find . -name "*.txt" | head -1  # WRONG (newline delimited)
find . -name "*.txt" -print0 | head -1 | cat -A  # CORRECT (null delimited)
```

**Fix:**
```bash
# Before (WRONG):
find . -name "*.txt" | while read filepath; do

# After (CORRECT):
find . -name "*.txt" -print0 | while IFS= read -r -d '' filepath; do
```

---

#### Issue: Wrong Category Extracted

**Symptoms:**
```
Expected category: scripts
Got category: Category: scripts Documentation metadata
```

**Diagnosis:**
```bash
# Check file content
head -5 "file.txt"
# Output:
# Some header text
# category: documentation
# More text
# category: scripts  ← This is being picked up!
```

**Fix:**
```bash
# Before (gets last match):
grep "^category: " "$filepath" | cut -d' ' -f2-

# After (gets first match):
grep -m 1 "^category: " "$filepath" | cut -d' ' -f2-
```

---

#### Issue: Hash Doesn't Match

**Symptoms:**
```
Expected: 1e1679f4cbe85...
Got:      8a7b2c3d4e5f...
```

**Diagnosis:**
```bash
# Check locale
echo $LC_ALL
# Output: en_US.UTF-8  ← WRONG

# Test sort order
echo -e "café\nCafé\nCAFÉ" | sort
# Output depends on locale!
```

**Fix:**
```bash
# Before (locale-dependent):
find . -type f | sort | sha256sum

# After (consistent):
find . -type f | LC_ALL=C sort | sha256sum
```

---

## 9. Practical Exercises

### Exercise 1: Basic Path Manipulation

Extract components from these paths:

```bash
# Practice these:
filepath1="docs/chapter1/file.txt"
filepath2="content/part 2/données/test.txt"
filepath3="archive/módulo-3/intro/file.txt"

# Your task: Extract dirname, basename, and transformed path
# Expected outputs:
# filepath1: docs/chapter1, file.txt, docs-chapter1
# filepath2: content/part 2/données, test.txt, content-part 2-données
# filepath3: archive/módulo-3/intro, file.txt, archive-módulo-3-intro
```

**Solution:**
```bash
for filepath in "$filepath1" "$filepath2" "$filepath3"; do
    dirpath=$(dirname "$filepath")
    filename=$(basename "$filepath")
    path_with_dashes=$(echo "$dirpath" | tr '/' '-')
    echo "Dir: $dirpath | File: $filename | Transformed: $path_with_dashes"
done
```

---

### Exercise 2: Category Extraction

Create test files and extract categories:

```bash
# Create test file
mkdir -p test
cat > test/file1.txt <<EOF
category: scripts

This is a test file
EOF

cat > test/file2.txt <<EOF
Some header
category: data
More content
category: logs
EOF

# Extract categories
for file in test/*.txt; do
    category=$(grep -m 1 "^category: " "$file" | cut -d' ' -f2-)
    echo "$file -> category: $category"
done

# Expected output:
# test/file1.txt -> category: scripts
# test/file2.txt -> category: data
```

---

### Exercise 3: Build Complete Transformation

Given a file path and category, construct the new path:

```bash
# Inputs:
original_path="content/part 2/données/file-name/file23.txt"
category="scripts"

# Your task: Build newpath
# Expected: scripts/content-part 2-données-file-name-file23.txt

# Solution:
dirpath=$(dirname "$original_path")
filename=$(basename "$original_path")
path_with_dashes=$(echo "$dirpath" | tr '/' '-')
newpath="${category}/${path_with_dashes}-${filename}"
echo "Original: $original_path"
echo "New:      $newpath"
```

---

### Exercise 4: Unicode Handling

Test the script with Unicode filenames:

```bash
# Create test structure
mkdir -p "test/café-2024"
mkdir -p "test/módulo-3"
mkdir -p "test/naïve"

cat > "test/café-2024/fіle.txt" <<EOF
category: münchen
Test content
EOF

cat > "test/módulo-3/données.txt" <<EOF
category: 日本語
Test content
EOF

# Process them
find test -name "*.txt" -type f -print0 | \
while IFS= read -r -d '' filepath; do
    category=$(grep -m 1 "^category: " "$filepath" | cut -d' ' -f2-)
    dirpath=$(dirname "$filepath")
    filename=$(basename "$filepath")
    path_with_dashes=$(echo "$dirpath" | tr '/' '-')
    newpath="${category}/${path_with_dashes}-${filename}"
    echo "$filepath -> $newpath"
done

# Clean up
rm -rf test münchen 日本語
```

---

### Exercise 5: Error Handling

Add validation to catch edge cases:

```bash
# Test files with issues
cat > test/no_category.txt <<EOF
This file has no category line
EOF

cat > test/empty_category.txt <<EOF
category: 
EOF

cat > test/multi_word.txt <<EOF
category: complex category name
EOF

# Enhanced processing with validation
find test -name "*.txt" -type f -print0 | \
while IFS= read -r -d '' filepath; do
    category=$(grep -m 1 "^category: " "$filepath" | cut -d' ' -f2-)
    
    # Validation
    if [ -z "$category" ]; then
        echo "Warning: No category found in $filepath" >&2
        continue
    fi
    
    if [[ "$category" =~ [/\\:] ]]; then
        echo "Warning: Invalid characters in category: $category" >&2
        continue
    fi
    
    echo "Processing $filepath with category: $category"
done
```

---

## 10. Summary and Key Takeaways

### Critical Commands Mastered

1. **`find -print0`** - Null-delimited file discovery
2. **`IFS= read -r -d ''`** - Safe reading of null-delimited input
3. **`grep -m 1`** - Extract first matching line
4. **`cut -d' ' -f2-`** - Split and extract fields
5. **`dirname` / `basename`** - Path component extraction
6. **`tr '/' '-'`** - Character translation
7. **`mkdir -p`** - Idempotent directory creation
8. **`LC_ALL=C sort`** - Locale-independent sorting

### Best Practices Learned

✅ **Always quote variables** - `"$variable"` not `$variable`  
✅ **Use `-print0` and `-d ''` for special characters**  
✅ **Set `LC_ALL=C` for consistent sorting**  
✅ **Use `-m 1` to get first match only**  
✅ **Add error handling with `set -euo pipefail`**  
✅ **Validate inputs before processing**  
✅ **Test with Unicode and special characters**  

### Common Pitfalls to Avoid

❌ Not quoting variables with spaces  
❌ Using newline delimiters with filenames containing spaces  
❌ Forgetting `-m 1` when multiple matches possible  
❌ Not using `LC_ALL=C` for hash generation  
❌ Missing `-p` flag on `mkdir`  
❌ Not handling empty or missing category values  

---

## Appendix A: Complete Script (Production-Ready)

```bash
#!/bin/bash

# ================================================================
# File Reorganization Script - Production Version
# ================================================================

set -euo pipefail  # Strict error handling

# Validate prerequisites
for dir in archive content docs project; do
    if [ ! -d "$dir" ]; then
        echo "Error: Required directory '$dir' not found" >&2
        exit 1
    fi
done

# Check required commands
for cmd in find grep cut tr mkdir mv sha256sum; do
    if ! command -v "$cmd" &>/dev/null; then
        echo "Error: Required command '$cmd' not found" >&2
        exit 1
    fi
done

echo "Starting file reorganization..."
echo "================================"

# Statistics
total_files=0
processed_files=0
skipped_files=0

# Main processing
find archive content docs project -name "*.txt" -type f -print0 | \
while IFS= read -r -d '' filepath; do
    ((total_files++))
    
    # Extract category
    category=$(grep -m 1 "^category: " "$filepath" | cut -d' ' -f2- || echo "")
    
    # Validate category
    if [ -z "$category" ]; then
        echo "Warning: No category in $filepath - skipping" >&2
        ((skipped_files++))
        continue
    fi
    
    # Parse path components
    dirpath=$(dirname "$filepath")
    filename=$(basename "$filepath")
    path_with_dashes=$(echo "$dirpath" | tr '/' '-')
    
    # Create directory and move file
    mkdir -p "$category"
    newpath="${category}/${path_with_dashes}-${filename}"
    
    echo "Moving: $filepath -> $newpath"
    mv "$filepath" "$newpath"
    ((processed_files++))
done

echo ""
echo "================================"
echo "Summary:"
echo "  Total files found: $total_files"
echo "  Successfully processed: $processed_files"
echo "  Skipped: $skipped_files"
echo "================================"
echo ""

# Generate hash
echo "Generating verification hash..."
find . -type f | LC_ALL=C sort | sha256sum
echo ""
echo "================================"
```

---

## Appendix B: Quick Reference Card

### One-Liners for Common Tasks

```bash
# Count files by extension
find . -type f | sed 's/.*\.//' | sort | uniq -c

# Find files modified in last 24 hours
find . -type f -mtime -1

# Find large files (>100MB)
find . -type f -size +100M

# Rename files (remove spaces)
find . -name "* *" -type f | while IFS= read -r f; do mv "$f" "${f// /_}"; done

# Find duplicate files by hash
find . -type f -exec sha256sum {} + | sort | uniq -d -w 64

# Count lines in all .txt files
find . -name "*.txt" -exec wc -l {} + | tail -1

# Extract categories from all files
find . -name "*.txt" -print0 | xargs -0 grep -h "^category:" | sort | uniq
```

---

**End of Study Guide**

This comprehensive reference covers everything needed to understand, implement, and debug the file reorganization system. Use it as both a learning resource and a quick reference for future projects.
