![Servers](../../assets/tech-blog/devops/github-shared-action/roman-synkevych-wX2L8L-fGeA-unsplash.jpg)

##### originally posted at [LinkedIn](https://www.linkedin.com/pulse/github-shareable-action-janus-chung) at July 7, 2023


## Background

I need to set up 3 GitHub workflows for 10 repo. For each workflow, it will run a similar bash script.

## First trial - Create Separate Bash Script and Workflow

On my first trial, I did the following:

1. work on a single repo
1. create three separate bash scripts
1. create three separate workflows

Here are the three bash scripts:

_run-dependabot-alert.sh_
``` bash
#!/bin/sh

TOKEN=$1
REPO=$2

get_response(){
  curl -sL \
    -H "Accept: application/vnd.github+json" \
    -H "Authorization: Bearer $TOKEN"\
    -H "X-GitHub-Api-Version: 2022-11-28" \
    https://api.github.com/repos/januschung/"$REPO"/dependabot/alerts
}

do_something(){
    echo "Running Dependabot!"
}

get_response
do_something
```
_run-secret-scanning.sh_
``` bash
#!/bin/sh

TOKEN=$1
REPO=$2

get_response(){
  curl -sL \
    -H "Accept: application/vnd.github+json" \
    -H "Authorization: Bearer $TOKEN"\
    -H "X-GitHub-Api-Version: 2022-11-28" \
    https://api.github.com/repos/januschung/"$REPO"/secret-scanning/alerts
}

do_something(){
    echo "Running Secret Scanning!"
}

get_response
do_something
```

_run-codeql-alert.sh_
``` bash
#!/bin/sh

TOKEN=$1
REPO=$2

get_response(){
  curl -sL \
    -H "Accept: application/vnd.github+json" \
    -H "Authorization: Bearer $TOKEN"\
    -H "X-GitHub-Api-Version: 2022-11-28" \
    https://api.github.com/repos/januschung/"$REPO"/code-scanning/alerts
}

do_something(){
    echo "Running CodeQL Scanning!"
}

get_response
do_something
```
Here is one of the three workflow, as they are more like the same:

_dependabot-workflow.yml_
``` yaml
name: 'Dependabot Alert Check'
on:
  push:
    branches:
      - 'main'

jobs:
  dependabot-scanning-alert-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
        with:
          token: ${{ secrets.GH_TOKEN }}
      - name: Run Dependabot Alert Check
        shell: bash
        run: |
          ./scripts/run-dependabot-alert.sh ${{ secret.GH_TOKEN }} ${{ github.event.repository.name }}
```

As you can see, there are a lot of similar logic from both of the bash scripts and the workflows. It will be tedious to repeat the work across the other repos.

Can I do better? Sure!

## Second Trial - Create Shareable Composite Action

Since all of the 10 repo will share the same logic, I converted the workflow into a shareable composite action in a dedicated repo. In so doing, the other repo can reference the same action. Here are the benefits:

- a single source of truth
- easier to maintain one set of bash scripts and action
- easier to get the rest of the repo onboard

Here is the shareable composite action:

``` yaml
name: 'Alert Check'
description: 'Run alert check'
inputs:
  alert-type:
    description: 'Alert type to run - code-scanning, dependabot or secret-scanning'
    required: true
    default: 'dependabot'
  gh-token:
    required: true

runs:
  using: "composite"
  steps:
    - uses: actions/checkout@v3
      with:
        token: ${{ inputs.gh-token }}
    - shell: bash
      run: echo "${{ github.action_path }}" >> $GITHUB_PATH
    - shell: bash
      run: |
        ${{ alert-type }}.sh ${{ inputs.gh-token }} ${{ github.event.repository.name }}
```

Note that the bash scripts are now migrated from `scripts` folder into `.github/actions/alerts`

The corresponding workflow that reference the shareable composite action:

``` yaml
on: [push]

jobs:
  run_alert:
    runs-on: ubuntu-latest
    name: Run Dependabot Alert
    steps:
      - uses: actions/checkout@v3
      - uses: januschung/shared-action-repo/.github/actions/alerts@main
        with:
          gh-token: ${{ secrets.GH_TOKEN }}
          alert-type: ${{ matrix.type }}
```

Looking way much better now. But, can I do even better?

## Third Trial - Consolidate Into a Single Bash Script and a Single Action

Since all three bash scripts share 80% of the same logic, I consolidated them into a single one:

_alert.sh_
``` bash
#!/bin/sh

TOKEN=$1
REPO=$2
TYPE=$3

get_response(){
  curl -sL \
    -H "Accept: application/vnd.github+json" \
    -H "Authorization: Bearer $TOKEN"\
    -H "X-GitHub-Api-Version: 2022-11-28" \
    https://api.github.com/repos/januschung/"$REPO"/"$TYPE"/alerts
}

do_something(){
    if [ "TYPE" == 'code-scanning' ]; then
        do_something_code_scanning
    elif [ "TYPE" == 'dependabot' ]; then
        do_something_dependabot
    elif [ "TYPE" == 'secret-scanning' ]; then
        do_something_secret_scanning
    else
        exit 1
    fi
}

do_something_code_scanning(){
    echo "Running Secret Scanning!"
}

do_something_dependabot(){
    echo "Running Dependabot!"
}

do_something_secret_scanning(){
    echo "Running Secret Scanning!"
}

get_response
do_something
```

The consolidated _action.yml_

``` yaml
name: 'Alert Check'
description: 'Run alert check'
inputs:
  alert-type:
    description: 'Alert type to run - code-scanning, dependabot or secret-scanning'
    required: true
    default: 'dependabot'
  gh-token:
    required: true

runs:
  using: "composite"
  steps:
    - uses: actions/checkout@v3
      with:
        token: ${{ inputs.gh-token }}
    - shell: bash
      run: echo "${{ github.action_path }}" >> $GITHUB_PATH
    - shell: bash
      run: |
        alert.sh ${{ inputs.gh-token }} ${{ github.event.repository.name }} {{ inputs.alert-type }}
```

I do not want to create three separate workflows from the external repo either, so I use GitHub Matrix in a single workflow:

``` yaml
on: [push]

jobs:
  run_alert:
    runs-on: ubuntu-latest
    name: Run Alert
    strategy:
      matrix:
        type: ['code-scanning', 'dependabot', 'secret-scanning']
    steps:
      - uses: actions/checkout@v3
      - uses: januschung/shared-action-repo/.github/actions/alerts@main
        with:
          gh-token: ${{ secrets.GH_TOKEN }}
          alert-type: ${{ matrix.type }}
```

That is it! I hope this helps if you are into similar situation in the coming future.

## Links

[Composite Action](https://docs.github.com/en/actions/creating-actions/creating-a-composite-action)

[Matrix](https://docs.github.com/en/actions/using-jobs/using-a-matrix-for-your-jobs)