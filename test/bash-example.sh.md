# Bash Example
This is an example of literate programming using bash.

Code block without fencing:

    #!/usr/bin/env bash

Fenced code block without language:
```

# To check that the generated code is correct run:
# $ litparse -p test/bash-example.sh.md | diff test/bash-example.sh -

printf "Hello "

```

Fenced code block with language:
```bash
# To check that the output is correct run:
# $ litparse test/bash-example.sh.md

printf World
```

The output should be:
> Hello World
