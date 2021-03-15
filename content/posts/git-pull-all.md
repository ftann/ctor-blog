---
title: "Pull all the changes"
keywords: ["git"]
tags: ["git", "monday"]
date: 2021-03-15T15:21:14+01:00
---

It's Monday after an extended weekend. Git go fetch all changes my colleagues committed since I
left. Place this script in the folder where all the projects are and run it.

```shell
#!/usr/bin/env bash

REPOSITORIES=$(pwd)

is_git_repo() {
  [[ -d "$1" && -d "$1/.git" ]]
}

git_update() {
  git fetch --all --prune
  git pull --all --quiet
}

ssh-add

for REPO in "$REPOSITORIES"/*; do
  if is_git_repo "$REPO"; then
    cd "$REPO" && git_update
  fi
done
```

# Monday is when the weekend starts

Do you like the guide and want to give feedback or found a mistake? Then send me a mail
to `f4ntasyland /at/ protonmail /dot/ com`

You can always buy me a beer.
`（ ^_^）o自自o（^_^ ）`

_[xmr][0]:
473WTZ1gWFdjdEyioCQfbGQKRurZQoPDJLhuFJyubCrk4TRogKCtRum63bFMx2dP2y4AN1vf2fN6La7V7eB2cZ4vNJgMAcG_

[0]: https://www.getmonero.org/
