# Phraseapp Orb

## Proposal
This orb was created to simplify the sync between the changes from the app.phrase and our repository.

## Configuration Needed

#### 1) Add [phrase.yml](.phrase.yml) to the root of your project.
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

Note that you could also use different `file_format` (like `simple_json`) too.


#### 2) Add the following environment variables on [circleci](https://app.circleci.com/settings/project/github/jobteaser/<repository>/environment-variables). 

Note that you can also specify then directly in each command (be careful about consistency between them).

The phrase token can be generated [here](https://app.phrase.com/settings/oauth_access_tokens).
- PHRASEAPP_ACCESS_TOKEN 
- PHRASEAPP_PROJECT_ID

The github user email is needed to create the PR:
- GITHUB_USER_EMAIL

If you want to receive slack notifications, you also need to add the channel webhook url. 
- SLACK_TRANSLATION_NOTIF_URL

To be able to create and merge PR, you will also need a token from Github
- GITHUB_PR_AUTOMATION_TOKEN

#### 3) Update the .circleci/config: 

```yml
orbs:
  service: "jobteaser/service@0.10.0"
  phraseapp: "jobteaser/phraseapp@dev:0.0.1"
  
workflows:
  main:
    jobs:
      - phraseapp/automerge-phraseapp-pr:
          name: "automerge-phraseapp-pr"
          context: "build"
          requires: [ "test" ]
          filters:
            branches:
              only: "phraseapp-locales"
              
              
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

