---
title: "Shellscript Notes"
date: 2020-02-20T15:25:00+01:00
draft: false
tags:
- cheatsheet
---

Useful shellscript snippets
<!--more-->

## Commonly used

```bash
# Check if variable contains mystring
if [[ $var == *mystring*  ]]; then # The double closed brackets are essential for wildcard match
    echo "it does";
fi

# Split on ":" and get 1st
echo "docker-image:0.1" | cut -d ":" -f1 # returns docker-image

# Find and replace all via pipe
echo "Hello I'm Dom" | sed 's/Dom/<void>/g' # "/" could be any char


# Redirect stdout and stderr to the void
./run_script.sh >/dev/null 2>&1
```

## Basics

```bash
# Read top lines of file
head -n 3 myfile

# Read bottom lines of file
tail -n 3 myfile

# Watch all updates to file
tail -f myfile

# Read file in window (not cat)
less myfile

# Make all directories if not exist
mkdir -p exist/not/dir

# Visualise dir structure
tree -L 2

# Remove env variable
unset myvar

# Alias command to short one
alias lc="ls -laht" # unalias to remove

# Piping output
echo "overwrite" > file # Same as 1>
echo "append" >> file
```

## System

```bash
# See disk space usage (human readable)
df -h

# See file usage for dirs
du (--max-depth=2)

# Show running processes, a: show other user, u: show username, x: show non terminal
ps -aux

# Show usage
top
```

## Tricks

```bash
# Use command output as if it's file
diff <(grep string file1) <(grep string file2)

# Repeat last 3 command
!!

# Repeat last arg of the last command
!$

# Take all args from previous and use
!:1-$
```

# IO

```bash
# Secure input
read -s -p "Password: " password

# Default value
read -p "Address [http://127.0.0.1]: " ADDR
default="http://127.0.0.1"
ADDR=${ADDR:-$default}
```

# jq

```bash
# Get value from object within array within object
jq -rec ".parent.children | .[] | .name" # raw:no quotes, e:null exit0, compact

# Get object attr and value and format "Key=$attr,Value=$val"
jq -r '.data | keys[] as $k | "Key=\($k),Value=\(.[$k])"'

# Use output in for loop
for object in $(echo "${JSON}" | jq -rec ".entries[] | .whole | @base64"); do
  echo "This is the value: $(echo ${val} | base64 -d | jq .attribute)"
done

# Args
curl localhost:8500/v1/catalog/nodes | \
    jq \
        --arg ip $(./get_host.sh) \
        '.[] | select(.Address==$ip)'
        
# Output mulitple entries in list
jq -rec '[.importItems[].itemId]'

# Create object
jq -rec '{} +{ Items:[.importItems[].itemId]}'

```

## Behaviour

```bash
# Exit script if commands return nonzero
set -e

# Output command as they run (debug)
set -x
```