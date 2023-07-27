## Using Git Mob
Git mob is a command line tool to copilot with your team when you collaborate on code. Read more about [Git mob](https://www.npmjs.com/package/git-mob).


To setup Git Mob, first you have to definte your information:
``` bash
git config --global user.name "Janus Chung"
git config --global user.email "janus.chung@dummy-domain.com"
```

Add your teammate to `.git-coauthors` file
``` bash
$ cat <<-EOF > ~/.git-coauthors
{
  "coauthors": {
    "rh": {
      "name": "Robin Hood",
      "email": "rhood@dummy-domain.com"
    },
    "ab": {
      "name": "Astro Boy",
      "email": "aboy@dummy-domain.com"
    }
  }
}
EOF
```

Say if you want to pair with only Astro Boy
``` bash
git mob ab
```

If you want to pair with both Robin Hood and Astro Boy
``` bash
git mob rh ab
```

If you decide to code solo
``` bash
git solo
```

## To tag a build
``` bash
git checkout main
git pull
git tag x.x.x
git push origin x.x.x
```

## Rebase
``` bash
git checkout main
git pull
git checkout feature-branch
git rebase main feature-branch
git push --force origin feature-branch
```

## Bash helper

### Checkout main and pull
``` bash
gm(){
  git checkout main && git pull
}
```

### Commit with feature branch as the prefix
``` bash
gcm(){
  if [[ $# -eq 0 ]] ; then
    echo "add a git comment"
  else
    branch=$(git branch | grep '*' | awk '{print $2}')
    echo $branch
    echo "$branch $*"
    git commit -m "$branch $*"
  fi    
}
``` 

### Push to feature branch without typing it out
``` bash
gp(){
  branch=$(git branch | grep '*' | awk '{print $2}')
  git push origin $branch
}
```

### auto tag and push a new minor build
``` bash
gt(){
  git checkout main && git pull
  LAST_TAG_SHA=$(git show-ref | tail -n 1 | awk '{print $1}')
  LAST_TAG=$(git show-ref | tail -n 1 | awk '{print $2}' | cut -d '/' -f 3)
  LAST_TAG_PATCH_VERSION=$(echo "${LAST_TAG%%-*}" | cut -d '.' -f 3)
  NEW_TAG_PATCH_VERSION=$((LAST_TAG_PATCH_VERSION + 1))
  NEW_TAG=$(echo "$LAST_TAG" | cut -d '.' -f 1,2).$NEW_TAG_PATCH_VERSION

  git tag "$NEW_TAG" "$LAST_TAG_SHA"
  git push origin "$NEW_TAG"
}
```