# `litparse` - Basic literate programming using Markdown

## Why?
`litparse` is a Bash program that [extracts](#extraction), [concatenates](#processing), and [executes](#display--execution) code blocks from [Markdown] files.

```bash
#!/usr/bin/env bash

```

The intermingling of natural language with code enables a basic form of [literate programming] similar to [Docco], [Pycco], and [codedown]. Note that the purpose of `litparse` is to extract and run code rather than generate documentation.

[codedown]: https://github.com/earldouglas/codedown
[Docco]: https://github.com/jashkenas/docco
[literate programming]: https://en.wikipedia.org/wiki/Literate_programming
[Markdown]: https://daringfireball.net/projects/markdown/syntax
[Pycco]: https://github.com/pycco-docs/pycco

## Build
This `README` contains the literate source code for `litparse` and it can build itself. We extract all the code blocks in this file with `bash` as the language (`-l bash`) and we don't execute the result, but merely print the result (`-p`). You can read about [how these arguments work](#arguments) below.

You can use this output to create a temporary file called `litparse.sh` and then install it in `/usr/bin` without an extension so that it can be called without an extension in the shell.

Note that the following code block is not fenced with a language so that these lines do not appear in the final source code.

    litparse -l bash -p README.md > dist/litparse.sh
    sudo install dist/litparse.sh /usr/bin/litparse

## Examples
There are simple examples in the [test directory](./test). Here are some additional examples:

**Python**
```python
#!/usr/bin/env python
def hello():
    print "hello world"

if __name__ == '__main__':
    hello()  # this line also contains leading space
```

**Ruby**
```ruby
#!/usr/bin/env ruby
puts "Hello World"
```

## License
Licensed under the [MIT License].

```bash
# Copyright 2017 Metaist LLC <http://metaist.com/>
# MIT License <https://opensource.org/licenses/mit-license>

```

[MIT License]: https://opensource.org/licenses/mit-license

## Strict Mode
There are many ways in which Bash programs are hard to debug. In order to make development easier, we enable certain [Bash options] that prevent many inadvertent errors:

- `-u` - Prevents us from using a variable that has not been declared.
- `-o pipefail` - Prevents a pipe from continuing if part of the pipeline has an error.

```bash
set -uo pipefail
```

Another common problem is Bash's notorious [Internal Field Separator]. This internal variable is sometimes used to split strings, but because it defaults to whitespace (including spaces), it is often more headache than it is worth. We set it to newline and tab to handle the common cases we care about.

```bash
IFS=$'\n\t'

```

[Bash options]: http://tldp.org/LDP/abs/html/options.html
[Internal Field Separator]: http://tldp.org/LDP/abs/html/internalvariables.html

## Script Information
When we display the [script usage](#usage) we also display the name and [version](#arguments) of the script. In general, we compute these values instead of hard-coding them to avoid errors when copying these lines into other scripts.

  - `SCRIPT_SOURCE` holds the full path of the current script.
  - `SCRIPT_NAME` is the name of the script
  - `SCRIPT_VERSION` is the [semantic version][semver] of the script. Note that this value is manually updated before each release.

```bash
SCRIPT_SOURCE=$(readlink -f ${BASH_SOURCE[0]})
SCRIPT_NAME=$(basename $SCRIPT_SOURCE)
SCRIPT_VERSION='0.1.0'

```

[semver]: http://semver.org/

## Usage
The `usage` function displays a string with the description of the script and the possible options. See [Arguments][#arguments] for more details of how these options are used.

```bash
# Display the script usage.
usage() {
  printf "\
$SCRIPT_NAME ($SCRIPT_VERSION) - Extract executable code from Markdown files.

Usage:
  $SCRIPT_NAME [INPUT] [-h|--help] [-v|--version] [-l|--lang LANG] [-p|--print]

Options:
  INPUT                 markdown file to parse (default: STDIN)
  -h, --help            show this message and exit
  -v, --version         print script version and exit
  -l, --lang LANG       language name to extract from fenced blocks
  -p, --print           print the script instead of executing
"
}

```

## Regular Expressions
We use a [regular expressions][regex] (aka regex) to detect and extract code blocks. We also us a regex to detect the [shebang] that a program uses to parse the resulting output.

- `REGEX_SHEBANG` matches the `#!` and captures the command to execute at the end of processing.
- `REGEX_BLOCK` matches four consecutive spaces at the start of a line.
- `REGEX_FENCE` matches three backticks and it captures the name of the language for the code fence.

```bash
REGEX_SHEBANG='^#!([ /a-zA-Z]*)'
REGEX_BLOCK='^\s\s\s\s'
REGEX_FENCE='^```([a-zA-Z]*)'

```

[regex]: https://en.wikipedia.org/wiki/Regular_expression
[shebang]: https://en.wikipedia.org/wiki/Shebang_(Unix)

## Extraction
In order to extract code blocks we need to know the path to the file and which language (if any) to extract.

```bash
# Process a single file.
process_file() {
  local in_path=${1:-''}
  local in_lang=${2:-''}

```

As we process each line, we will also need to keep track of whether we are currently in a fenced code block and its associated language.

```bash
  local fence_start=false
  local fence_lang=''

```

We iterate over each line using reading from the given file [without escaping backslash characters][tldp-read] (`-r`). To handle files piped in via `STDIN`, we use file handle number ten (`-u 10`).

```bash
  while read -ru 10 line; do
```

When we reach a fenced code block, extract the name of the language (at the start of a code block) and toggle whether we are in a fenced code block. We do not want to do anything else with this line, so we `continue`.

```bash
    if [[ $line =~ $REGEX_FENCE ]]; then
      fence_lang="${BASH_REMATCH[1]}"
      if $fence_start; then fence_start=false; else fence_start=true; fi
      continue
    fi # fence detected

```

But which lines should we include in the output? If no language was specified, then all code blocks (fenced or otherwise) should be included. Otherwise, we only output those fenced code blocks with the given langauge.

For code blocks, but not _fenced_ code blocks, we remove the leading 4 spaces using `sed`.

```bash
    if [[ "$in_lang" == "" || "$in_lang" == "$fence_lang" ]]; then
      if $fence_start; then
        echo $line
      elif [[ $line =~ $REGEX_BLOCK ]]; then
        echo $line | sed "s/$REGEX_BLOCK//"
      fi
    fi
  done 10<$in_path
}

```

[tldp-read]: http://tldp.org/LDP/Bash-Beginners-Guide/html/sect_08_02.html

## Arguments
We set default values for our arguments:

- `ARG_INPUT` - read from `STDIN`
- `ARG_LANG` - extract all blocks (i.e. don't specify a language)
- `ARG_PRINT` - try to execute the resulting script

```bash
ARG_INPUT='/dev/stdin'
ARG_LANG=''
ARG_PRINT=false

```

Now we iterate over each argument to check its value.

```bash
while [[ "$#" > 0 ]]; do
  case "$1" in
```

For `--help` and `--version` we echo the appropriate output and exit.

```bash
    -h|--help) usage; exit 0;;
    -v|--version) echo "$SCRIPT_NAME $SCRIPT_VERSION"; exit 0;;
```

For optional arguments we consume the option as well as any corresponding arguments.

```bash
    -l|--lang) ARG_LANG="$2"; shift 2;;
    -p|--print) ARG_PRINT=true; shift 1;;
```

For unknown arguments, the first is treated as the input file while subsequent arguments cause and error.

```bash
    *)
      if [[ "$ARG_INPUT" == "/dev/stdin" ]]; then
        ARG_INPUT="$1"
        shift 1
      else
        echo "unknown option: [$1]" >&2
        usage
        exit 1
      fi;;
  esac
done # args parsed

```

## Processing
Once we have all of our arguments parsed, we can now pass those arguments to the [extraction](#extraction) process. We save the content of the extraction in a single string so that we can do some post-processing on it.

```bash
CONTENT=$(process_file "$ARG_INPUT" "$ARG_LANG")
```

If the extracted blocks begin with a [shebang], we save it separately.

```bash
SHEBANG=''
if [[ $CONTENT =~ $REGEX_SHEBANG ]]; then
  SHEBANG="${BASH_REMATCH[1]}"
fi # extracted the shebang

```

## Display / Execution
Unless we are explicitly told to print the script, we will try to execute it using the [shebang] we extracted.

```bash
if [[ "$ARG_PRINT" == "true" || "$SHEBANG" == "" ]]; then
  echo "$CONTENT"
else
  echo "$CONTENT" | eval $SHEBANG
fi # printed or executed the script
```

And that's all there is.
