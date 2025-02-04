> [!WARNING]
> This repository is a work in progress and is not intended for external use at this time. Please follow the repository for future updates.

# Heroku Flow Action
GitHub Action to upload the source code to Heroku from a private GitHub repository using the [Heroku Source Endpoint API](https://devcenter.heroku.com/articles/build-and-release-using-the-api#sources-endpoint). The uploaded code is then built to either deploy an app (on push, workflow_dispatch and schedule events) or create a review app (on pull_request events).
The Review App is automatically removed when the pull request is closed (on pull_request events when the action is 'closed').
The action handles only the above mentioned events to prevent unexpected behavior, handling event-specific requirements and improving action reliability.

In a GitHub Workflow, this action requires to be preceeded by the [actions/checkout](https://github.com/actions/checkout) to work properly.

## Disclaimer
The author of this article makes any warranties about the completeness, reliability and accuracy of this information. **Any action you take upon the information of this website is strictly at your own risk**, and the author will not be liable for any losses and damages in connection with the use of the website and the information provided. **None of the items included in this repository form a part of the Heroku Services.**

## How to use it
Create a `.github/workflows` directory within your repository using the following YAML files (either an all-in-one file `push-and-pr.yml` or three separate files `push.yml`, `pr-opened.yml`, `pr-closed.yml` to deal with specific workflows) and configure them. If you need to filter files from your repository before deploy use the `sparse-checkout` option available with `actions/checkout`.<br/>
It's possible to configure the `review-app-creation-check-timeout` (Review App creation timeout) and `review-app-creation-check-sleep-time` (sleep time while polling) input params to tune the Review App creation check process.

This will be executed on push and pull_request events
```
# push-and-pr.yml
name: Deploy push and PR, delete Review App on PR closed

on:
  push:
    paths-ignore:
      - '.github/workflows/**'
    branches:
      - main

  pull_request:
    paths-ignore:
      - '.github/workflows/**'
    types: [opened, reopened, synchronize, closed]

jobs:
  push-and-pr:
    runs-on: self-hosted
    steps:
      - uses: actions/checkout@v4
        with:
          sparse-checkout: |
            /*
            !.gitignore
            !.github
      - uses: abernicchia-heroku/heroku-flow-action@v1
        with:
          heroku-api-key: ${{secrets.HEROKU_API_KEY}} # set it on GitHub as secret
          remove-git-folder: false # if you want to override the default (true) - it's usually recommended to avoid exposing the .git folder 
          heroku-app-name: ${{vars.HEROKU_APP_NAME}} # set it on GitHub as variable at repository level
          heroku-pipeline-id: ${{vars.HEROKU_REVIEWAPP_PIPELINE}} # set it on GitHub as variable at repository level
          review-app-creation-check-timeout: 600 # if you want to override the default (3600 seconds) 
          review-app-creation-check-sleep-time: 7 # if you want to override the default (10 seconds), min 5 seconds
```


This will be executed on push events
```
# push.yml
name: Deploy push

on:
  push:
    paths-ignore:
      - '.github/workflows/**'
    branches:
      - main

jobs:
  build-push:
    runs-on: self-hosted
    steps:
      - uses: actions/checkout@v4
        with:
          sparse-checkout: |
            /*
            !.gitignore
            !.github
      - uses: abernicchia-heroku/heroku-flow-action@v1
        with:
          heroku-api-key: ${{secrets.HEROKU_API_KEY}} # set it on GitHub as secret
          heroku-app-name: ${{vars.HEROKU_APP_NAME}} # set it on GitHub as variable at repository level
          remove-git-folder: false # if you want to override the default (true) - it's usually recommended to avoid exposing the .git folder 
```

This will be executed whenever a PR is [opened, reopened, synchronize]
```
# pr-opened.yml
name: Deploy PR (create Review App)

on:
  pull_request:
    paths-ignore:
      - '.github/workflows/**'
    types: [opened, reopened, synchronize]

jobs:
  build-pr:
    runs-on: self-hosted
    steps:
      - uses: actions/checkout@v4
        with:
          sparse-checkout: |
            /*
            !.gitignore
            !.github
      - uses: abernicchia-heroku/heroku-flow-action@v1
        with:
          heroku-api-key: ${{secrets.HEROKU_API_KEY}} # set it on GitHub as secret
          heroku-pipeline-id: ${{vars.HEROKU_REVIEWAPP_PIPELINE}} # set it on GitHub as variable at repository level
          remove-git-folder: false # if you want to override the default (true) - it's usually recommended to avoid exposing the .git folder
          review-app-creation-check-timeout: 600 # if you want to override the default (3600 seconds) 
          review-app-creation-check-sleep-time: 7 # if you want to override the default (10 seconds), min 5 seconds
```

This will be executed whenever a PR is [closed]
```
# pr-closed.yml
name: Close PR (delete Review App)

on:
  pull_request:
    paths-ignore:
      - '.github/workflows/**'
    types: [closed]

jobs:
  close-pr:
    runs-on: self-hosted
    steps:
      - uses: abernicchia-heroku/heroku-flow-action@v1
        with:
          heroku-api-key: ${{secrets.HEROKU_API_KEY}} # set it on GitHub as secret
          heroku-pipeline-id: ${{vars.HEROKU_REVIEWAPP_PIPELINE}} # set it on GitHub as variable at repository level
```