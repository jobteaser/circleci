# Phraseapp Orb

## Proposal
This orb was created to make the merge of the changes made in the app.phrase with the GitHub project repository simple.

## Configuration Needed
1) Add [pharaseapp.yml](.phraseapp-sample.yml) to the root of your project.
It will be used to know what locales we need to fetch from the app.phrase, the path to pull the changes and the format of the configuration file.
```
phraseapp:
  available_locales:
    en
    fr
    de
  pull:
    targets:
      - file: ./config/locales/
        params:
          file_format: yml
```


2) Add jobtomate as a user to the project collaborators in the github repository. 


3) Add the following environment variables to the circle:
- PHRASEAPP_ACCESS_TOKEN
- PHRASEAPP_PROJECT_ID


4) Update the .circleci/config: 

```
orbs:
  service: "jobteaser/service@0.10.0"
  phraseapp: "jobteaser/phraseapp@dev:0.0.0"

workflows:
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
