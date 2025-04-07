### Basic Syntax

The basic syntax of the grep command is:

```
grep [options] pattern [file...]
```

- pattern: The text pattern to search for.
- [file...]: One or more files to search. If no file is specified, grep searches the standard input (stdin).

#### Common Options

-i: Ignore case distinctions in patterns and input data.

-v: Invert the match, i.e., select non-matching lines.

-c: Count the number of matching lines.

-l: Print the names of files with matching lines (not the matching lines).

-L: Print the names of files with no matching lines.

-n: Prefix each line of output with the line number within its input file.

-H: Print the filename for each match (useful when searching multiple files).

-r or -R: Recursively search directories.

-w: Match whole words only.

-x: Match whole lines only.

-A NUM: Print NUM lines of trailing context after matching lines.

-B NUM: Print NUM lines of leading context before matching lines.

-C NUM: Print NUM lines of output context (both before and after).

### Examples

#### Basic Search

Make the files "file.txt ..."

```
ls  /lib > file.txt
ls  /usr/lib > file2.txt
touch file1.txt
```

Search for the pattern "os-" in the file file.txt:

```bash
grep 'os-release' file.txt
```

Search for the pattern "os-release" in multiple files:

```bash
grep 'os-release' file.txt file2.txt
```

Case-Insensitive Search

Search for the pattern "os-release" ignoring case:

```bash
grep -i 'os-release' file.txt
```

Invert Match

Search for lines that do not contain the pattern "os-release":

```bash
grep -v 'os-release' file.txt
```

Count Matches

Count the number of lines that match the pattern "os-release":

```bash
grep -c 'os-release' file.txt
```

Print Filenames

Print the names of files that contain the pattern "os-release":

```bash
grep -l 'os-release' file.txt file2.txt
```

Print the names of files that do not contain the pattern "os-release":

```bash
grep -L 'os-release' file1.txt file2.txt
```

Line Numbers

Print the matching lines with their line numbers:

```bash
grep -n 'os-release' file.txt
```

Recursive Search

Recursively search for the pattern "os-release" in the current directory and all subdirectories:

```bash
grep -r 'os-release' .
```

Whole Word Match

Search for lines that contain the whole word "os-release":

```bash
grep -w 'os-release' file.txt
```

Whole Line Match

Search for lines that exactly match the pattern "os-release":

```bash
grep -x 'os-release' file.txt
```

Context Lines

Print 2 lines of trailing context after matching lines:

```bash
grep -A 2 'os-release' file.txt
```

Print 2 lines of leading context before matching lines:

```bash
grep -B 2 'os-release' file.txt
```

Print 2 lines of context (before and after) around matching lines:

```bash
grep -C 2 'os-release' file.txt
```

### Advanced Usage

Using Regular Expressions

grep supports extended regular expressions with the -E option (or use egrep):

```bash
grep -E 'os-|world' file.txt
```

Search for lines that start with "os-release":

```bash
grep '^os-release' file.txt
```

Search for lines that end with "os-release":

```bash
grep 'os-release$' file.txt
```

Search for lines containing any digit:

```bash
grep '[0-9]' file.txt
```

Piping with Other Commands

You can use grep in combination with other commands using pipes:

Filter the output of ls to show only files containing "log":

```bash
ls /var | grep 'log'
```

Filter the output of ps to show only processes containing "bash":

```bash
ps aux | grep 'bash'
```

Using grep with Multiple Patterns

Search for lines containing either "os-release" or "world":

```bash
grep -E 'os-release|world' file.txt
```

Using multiple -e options:

```bash
grep -e 'os-release' -e 'world' file.txt
```

Combining grep with find

Search for a pattern in files found by find:

```bash
find . -type f -name '*.txt' | xargs grep 'os-release'
```

Or using -exec:

```bash
find . -type f -name '*.txt' -exec grep 'os-release' {} +
```

### Basic Syntax

Basic Syntax The basic syntax of the sed command is:

```bash
sed [options] 'script' file
```

- options: Various options that control the behavior of sed.
- script: The sed script that specifies the operations to perform.
- file: The input file to process.

Common Options

`-e script: Add the script to the commands to be executed.`

`-f script-file: Add the contents of the script-file to the commands to be executed.`

`-i[SUFFIX]: Edit files in place (makes backup if SUFFIX is supplied).
`
`-n: Suppress automatic printing of pattern space (useful with p command).`

Common Commands

`s/pattern/replacement/flags: Substitute replacement for pattern.
`
`d: Delete the pattern space (delete lines).`

`p: Print the pattern space`.

`a\text: Append text after the current line.`

`i\text: Insert text before the current line.`

`q: Quit sed.`

### Examples

Make the file "file.txt"

```bash
ls  /lib > file.txt
ls  /usr/lib > file2.txt
touch file1.txt
```

#### Substitution

The most common use of sed is to substitute text. The s command is used for this purpose.

```bash
sed 's/apt/apt-get/' file.txt | head 
```

Example:

Replace the first occurrence of "lib" in each line of a file:

```bash
sed 's/lib/NEW/' file.txt 
```

Replace all occurrences of "-" with "!!!" in each line of a file:

```bash
sed 's/-/!!!/g' file.txt
```

### Deletion

The d command deletes lines from the input.

```bash
sed 'nd' file.txt
```

Example:

Delete the third line of a file:

```bash
sed '3d' file.txt
```

Delete lines that match a pattern:

```bash
sed '/sys/d' file.txt
```

### Insertion and Appending

The i command inserts text before the matched line, and the a command appends text after the matched line.

Insert text before the third line:

```bash
sed '3i\This is inserted before line 3' file.txt
```

Append text after the third line:

```bash
sed '3a\This is appended after line 3' file.txt
```

Printing

The p command prints the pattern space. This is often used with the -n option to suppress automatic printing.

Print only lines that match a pattern:

```bash
sed -n '/sys/p' file.txt
```

Print the first three lines of a file:

```bash
sed -n '1,3p' file.txt
```

In-place Editing

Edit a file in place (i.e., modify the original file directly):

```bash
sed -i 's/sys/SYS/' file.txt
cat file.txt
```

  
### Advanced Usage

Print lines 5 to 10:

```bash
sed -n '5,10p' file.txt
```

Insert "Hello" before every line that matches "pattern":

```bash
sed '/python/i\Hello' file.txt
```

Append "Goodbye" after every line that matches "pattern":

```bash
sed '/x/a\Goodbye' file.txt
```

Number of files in each directory

```bash
nano a.sh
```

```bash
#!/bin/bash
mypath=$(echo $PATH | sed 's/:/ /g')
count=0
for directory in $mypath
 do
  check=$(ls $directory)
  for item in $check
   do
    count=$[ $count + 1 ]
   done
  echo "$directory - $count"
  count=0
done
```

```yaml
chmod 755 a.sh
./a.sh
```

sed is a versatile and powerful tool for text processing and manipulation. Understanding its basic commands and options allows you to perform complex text transformations with ease. Experiment with different sed commands to get comfortable with its capabilities and syntax.