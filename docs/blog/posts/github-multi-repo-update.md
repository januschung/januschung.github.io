---
date: 2023-07-15
categories:
  - shell script
  - github
  - devops
---
![Servers](../../assets/blog/github-multi-repo-update/pexels-alberlan-barros-16018144.jpg)

##### originally posted at [LinkedIn](https://www.linkedin.com/pulse/github-multi-repos-update-janus-chung) at July 15, 2023

## Background

This is related to the other blog post about [GitHub Shared Action](./github-shared-action.md).

I want to make the same change to multiple GitHub repos and I want to limit manual steps as much as possible. As an example, I would like to push the the files of a source folder (source) to three of my repos. Here is one of the file:

<!-- more -->

_`source/somefile`_
``` yaml
Hello World!
```

The following file contains the repo name that I want to update:

_`repos.txt`_

```
mock-flask
job-winner
rock-paper-scissors
```

## Original Plan

My first thought was to implement a single bash script to do all the work:

1. read the repos.txt file line by line
1. for each line, run all the require git commands one by one

Here is the snippet for the original plan:

``` bash
REPO=$1
REPO_ROOT="/Users/janus/workspace/"
SOURCE_DIR="source"

while read -r line; do
    cd "$line"
    cp $SOURCE_DIR/* $REPO_ROOT/$REPO/
    cd $REPO_ROOT/$REPO/
    git checkout -b TEST-0000
    git add .
    git commit -m "TEST-0000 patching"
    git push origin TEST-0000
done < repos.txt

```

This can get the job done, but what if I have to do similar update in the future?


## Build a Reusable Script

As a DevOps engineerer, I expect a similar task to patch multiple repos will come over and over again. Therefore, I break down the script into two parts:

1. The reusable part - one that read the repos.txt file and perform the while loop
    - no need to reinvent the wheel anymore
    - no need to test if the looping the repos feature in the future
1. The pluggable part - the script that does the GitHub update
    - can be versionized
    - can be tested easier as it can be tested against a single repo as a standalone script

Here are the new implementation:

_`github-looping.sh`_
``` bash
#!/bin/sh

MY_SCRIPT=$1

while read -r line; 
    do ./$MY_SCRIPT "$line"; 
done < repos.txt
```

_`TEST-0000.sh`_ - while TEST-0000 is the jira ticket number
``` bash
#!/bin/sh

GH_BRANCH=$(basename -- $0 | cut -d. -f1)
REPO=$1
SOURCE_DIR="source"
REPO_ROOT="/Users/janus/workspace/"
COMMIT_MESSAGE="mass patching"

echo "processing repo "$REPO" ..."
cp $SOURCE_DIR/* $REPO_ROOT/$REPO/
cd $REPO_ROOT/$REPO/
git checkout -b $GH_BRANCH
git add .
git commit -m "$GH_BRANCH $COMMIT_MESSAGE"
git push origin $GH_BRANCH
echo "finish processing repo "$REPO" ..."
```

To run the update across all the repos, simply run the following:
`./github-looping.sh TEST-0000.sh`

The idea is to have a seperate script being called by `github-looping.sh`. 

In so doing, I can versionize the TEST-XXXX.sh script and have an idea of what exactly was being run.


## Conclusion

There are probably a thousand better ways out there to do similar thing. I found couple packages out there that take user inputs from the command lines interactively. Some of them would print out a nice report too. There is certainly room for improvement. However, as long as I can save myself from the headache of manaully update indivdual repos, I am happy with what I have so far.

Hope this works for you too if you are facing a similar problem.