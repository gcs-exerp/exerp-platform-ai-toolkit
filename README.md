# exerp-platform-ai-toolkit

An experimental repo to play around with compound engineering workflow

## How to use

These skills are created for exerp-platform and are based on compound-engineering based workflow.

Steps to use:

- Add github repo `https://github.com/gcs-exerp/exerp-platform-ai-toolkit` as a git submodule inside exerp-platform repo.
git submodule add .exerp-platform-ai-toolkit https://github.com/gcs-exerp/exerp-platform-ai-toolkit
Go to .exerp-platform-ai-toolkit/config.yaml and change pipeline-prefix as per your pipeline prefix
Start implementing a ticket using Research-Plan-Implement flow
Research : Invoke $ep-brainstorm DEV-XXXXX (Input - Jira Id, Output - A research plan document saved at path defined by docs_root in config.yaml)
Plan : Invoke $ep-plan DEV-XXXXX @research-plan-doc.md (Input - Jira Id / generated research plan doc, Output - An implementation plan saved at path defined by docs_root in config.yaml)
Implement - Invoke $ep-work DEV-XXXXX @implementation-plan-doc.md (Input - Jira Id / generated implementation plan doc, Output - An open PR with code changes)
Compound - Invoke $ep-compound in the same context session as Implement and it will generate a solution document
