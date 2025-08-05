# Bash Scripting Project - Ubuntu 20.04

This project includes Bash scripts that demonstrate manipulation of environment variables, basic arithmetic, path configuration, and shell behavior. All scripts comply with the following constraints:

- All scripts are exactly 2 lines long.
- First line of each file is `#!/bin/bash`.
- No use of `&&`, `||`, `;`, `bc`, `sed`, or `awk`.
- Scripts are created using `vi`, `vim`, or `emacs`.
- All scripts are executable.
- Designed for Ubuntu 20.04 LTS.

---

## 0-alias.sh
Creates an alias named `ls` with the value `rm *`.

## 1-hello_user.sh
Prints `Hello` followed by the current Linux user.

## 2-path_append.sh
Adds `/action` to the end of the `PATH` environment variable.

## 3-count_path.sh
Counts and prints the number of directories in the `PATH`.

## 4-list_env.sh
Lists all environment variables.

## 5-list_all.sh
Lists all local variables, environment variables, and functions.

## 6-create_local.sh
Creates a new local variable named `BEST` with the value `School`.

## 7-create_global.sh
Creates a new global (exported) variable named `BEST` with the value `School`.

## 8-add_trueknowledge.sh
Adds 128 to the value of the environment variable `TRUEKNOWLEDGE` and prints the result.

## 9-divide.sh
Divides the value of `POWER` by `DIVIDE` and prints the result.

## 10-power.sh
Calculates `BREATH` to the power of `LOVE` and prints the result.

## 11-bin_to_dec.sh
Converts a binary number stored in `BINARY` to decimal and prints the result.

## 12-no_oo.sh
Prints all possible 2-letter lowercase combinations from `aa` to `zz`, excluding `oo`. One per line. Script length is within 64 characters.
