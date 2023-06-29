## Background

I have to run a js script at repo B from repo A which has a python app. While I can spend the time to reinvent the wheel and fork the logic into python, it is best to keep thing simple by reusing what is already out there and proved to be working fine.

The js script in repo B is already wired up with a Github workflow. What if I can kick off that workflow remotely from repo A?

Github repository dispatch is the answer.

## About Github Repository Dispatch

Github repository dispatch is a feature to trigger custom events or actions programmatically. It allows you to create a webhook endpoint that can receive external HTTP requests and generate a repository-level event. This event can then be used to trigger workflows, CI/CD pipelines, or any other custom actions defined in your repository.

## Sample Workflow

This sample workflow will print out the client payload message.

``` yaml
name: dispatch receiver test

on: 
  repository_dispatch:
    types: [my-type]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v1
    - name: dispatch trigger
      env:
        MESSAGE: ${{ github.event.client_payload.message }}
      run: |
        echo "$MESSAGE"
```

## Github Personal Token

You will need to create a personal token for the repo, in this case, repo B, with the following setting:

`permissions contents: write`

Find more about [how to create a token](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens#creating-a-fine-grained-personal-access-token).

## Sample Code To trigger the event

Curl

``` bash
curl -H "Accept: application/vnd.github.everest-preview+json" \
    -H "Authorization: token $TOKEN" \
    --request POST \
    --data '{"event_type": "my-type", "client_payload": {"message": "Testing Github repository dispatch"}}' \
    https://api.github.com/repos/<org>/<repo>/dispatches
```

Python
``` python
import json
import requests
from loguru import logger

ORG = "my-org"
REPO = "my-repo"
TOKEN = "my-secret-token"

def get_github_dispatch_response(team_name):
        external_repo = f"https://api.github.com/repos/{ORG}/{REPO}/dispatches"
        data = {"event_type": "my-type"}
        message = {"message": f"Triggered by workflow dispatch"}
        payload = {"client_payload": message}
        data.update(payload)
        headers = {"Authorization": f"Bearer {TOKEN}"}
        response = requests.post(external_repo, data=json.dumps(data), headers=headers)
        if response.status_code == 204:
            return response
        logger.error(f"Workflow dispatch failed. Error code and message: {response.status_code} - {response.text}")
        response.raise_for_status()
```

## Link
[Repository Dispatch](https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows#repository_dispatch)

[How to create a token](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens#creating-a-fine-grained-personal-access-token)