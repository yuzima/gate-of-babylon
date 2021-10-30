# Use Grep for Fast Search from Command Line

Summary of this vidio: [https://egghead.io/courses/use-grep-for-fast-search-from-the-command-line](https://egghead.io/courses/use-grep-for-fast-search-from-the-command-line)

Search text `version` in package.json 

```bash
grep version package.json
```

Find all headings in all markdown files.

```bash
grep "#" *.md
```

Find all phrase `toggle` in directory recursively

```bash
grep -r toggle .

grep -r toggle /example/react/js
```

Find all files end with `jsx`

```bash
find . -name "*jsx"
```

Find all files end with `Sepc.js` in example directory

```bash
find examples/ -name "*Spec.js"
```

Pipe the find result to xargs. By default, xargs will use echo so it just print the output.

```bash
find examples -name "*Spec.js" | xargs
find examples -name "*Spec.js" | xargs echo
```

```bash
find examples -name "*Spec.js" | xargs grep "describe"
```

It has same output as this command.

```bash
grep -r --include="*Spec.js" "describe" examples/
```

Only search in the git repository, ignore node_modules.

```bash
git grep bind
```

Give the matching phrases colour highlight.

```bash
grep -r --color bind
```

Add line number to each file in the output.

```bash
grep -n "#" *.md
```

Print num lines of trailing context after or before or both each match

```bash
grep -A 2 "#" *.md
grep -B 2 "#" *.md
grep -C 2 "#" *.md
```

Dot will match any single character. Use `\.` to match the exact dot.

```bash
grep --color "h." readme.md
grep --color "https." readme.md
grep --color "\.com" readme.md
```

Search for optional patterns

```bash
echo "is it grey or gray?" | grep --color -E "grey|gray"
echo "is it grey or gray?" | grep --color "grey\|gray"
grep --color -rE "grey|gray"
```

Only search exact `grey'` or `gray'` 

```bash
grep --color -rE "(grey|gray)\'" 
```

Search with group 

```bash
grep --color -rE "(grey|gray)(\'|\"): (\'|\")#?[[:xdigital:]]+" . 
```

Exclude `node_modules`

```bash
find examples/angular.js -name "*.js" | grep -v "node_modules"
find examples/angular.js -name "*.js" | grep -vE "node_modules|Spec"
```