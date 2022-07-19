# Phraseapp Orb

## Proposal
This orb was created to make the merge of the changes made in the app.phrase with the GitHub project repository simple.

## Configuration Needed
1) Add [phrase.sample.yml](.phrase.sample.yml) to the root of your project.
It will be used to know what locales we need to fetch from the app.phrase, the path to pull the changes and the format of the configuration file.

- ruby projects:
```
phrase:
  push:
    sources:
      - file: ./config/locales/<locale_name>.yml
        params:
          file_format: yml
  pull:
    targets:
      - file: ./config/locales/<locale_name>.yml
        params:
          file_format: yml
          
```      

- go projects:
``` 
phrase:
  push:
    sources:
    - file: ./data/locales/<locale_name>.json
      params:
        file_format: go_i18n
  pull:
    targets:
    - file: ./data/locales/<locale_name>.json
      params:
        file_format: go_i18n
      
```

- js projects:
```
phrase:
  push:
    sources:
    - file: ./locales/<locale_name>.json
      params:
        file_format: nested_json
  pull:
    targets:
    - file: ./locales/<locale_name>.json
      params:
        file_format: nested_json
```        



2) Add the following environment variables to the project on circleCI. The path to add them could be found [here](https://app.circleci.com/settings/project/github/jobteaser/<repository>/environment-variables). The token for the project can be generated [here](https://app.phrase.com/settings/oauth_access_tokens). 
- PHRASEAPP_ACCESS_TOKEN 
- PHRASEAPP_PROJECT_ID
- PHRASEAPP_USER_EMAIL


3) Update the .circleci/config: 

```
orbs:
  service: "jobteaser/service@0.10.0"
  phraseapp: "jobteaser/phraseapp@dev:0.0.1"

```
- if you want to run the automerge on the build step of the main workflow

``` 
workflows:
  main 
    jobs:
      - phraseapp/automerge-phraseapp-pr:
          name: "automerge-phraseapp-pr"
          context: "build"
          requires: ["test"]
          filters:
            branches:
              only: phraseapp-locales

```
- if you want to run the automerge on a cronjob

```
  automerge_phraseapp_job:
    triggers:
      - schedule:
          cron: "00 13 * * 1-5"
          filters:
            branches:
              only:
                - phraseapp-locales
    jobs:
      - phraseapp/automerge-phraseapp-pr:
          name: "automerge-phraseapp-pr"
          context: "build"
          requires: ["test"]
```
- if you want to run the autocreate on a cronjob

```      
  autocreate_phraseapp_job:
    triggers:
      - schedule:
          cron: "00 12 * * 1-5"
          filters:
            branches:
              only:
                - master
    jobs:
      - phraseapp/autocreate-phraseapp-pr:
          name: "autocreate-phraseapp-pr"
          context: "build"
          
  
```

- if you want to run the notify failed automerge on a cronjob

```      
  notify_automerge_failed_phraseapp_job:
    triggers:
      - schedule:
          cron: "00 12 * * 1-5"
          filters:
            branches:
              only:
                - phraseapp-locales
    jobs:
      - phraseapp/notify-failed-automerge-phraseapp-pr:
          name: "notify-failed-automerge-phraseapp-pr"
          context: "build"
          
  
```
