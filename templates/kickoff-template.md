# Kickoff Template
(Include below 3 sections i.e. Kick-off notes, Test and Documentation while generating the notes)

## Kick-off notes
- Functionality, rules or special cases: Summarize the baselined behavior, constraints, compatibility requirements, and special cases.
- New/reuse action, system property, backward/feature flags/toggles or license: Do not mention all reused components. Only list if any new value is created or old value is reused from SystemPropertyKey.java or Actions.java or BackwardsCompatibilityFlag.java or FeatureFlag.java or LicenseType.java is involved in code changes or mentioned in research document.
- Should this ticket be hot-fixed? Which version(s)? Always add as TBD
- Dependencies to API, events or other tickets: List confirmed API dependencies. api-soap-endpoint project has endpoints exposed in API. Do not include other classes that were used or impacted
- Does this solution requires expert input from another department in the organization ? Identify required or suggested input and the relevant department(like data team, consulting, client services, architecture) when supported by the research. If could not find any use cases just mention as ?NA?
## Test
- Relevant configuration that must be considered to enable proper testing Identify configuration needed to exercise the behavior.
- Automatic testing: Identify existing automated checks to extend or the scenarios that should be automated. Include feature file name or junit class name, if could not be resolved then do not include, just add as TBD
- Specific things to acceptance test? List specific behavior and compatibility scenarios for acceptance testing.
- Specific things to regression test? List affected existing flows and risks that require regression coverage.
- Should requester do final acceptance test? Always add as TBD
## Documentation
- Where in Confluence? Always add as TBD 
- Does this ticket require a feature video? Always add as TBD


# Writing rules
Kick-off notes mentioned above is the template and the notes should always follow that pattern
Keep notes concise, specific, and ready to paste into Jira; use plain language and include technical names only when they clarify impact, configuration, dependencies, or testing.
Preserve technical names, module names, versions, and compatibility boundaries exactly as stated in the research.
Call out risks, open questions, and missing decisions in the most relevant bullet rather than resolving them by inference.
If the research describes impacts outside the primary module, identify those impacts explicitly.
Do not make repository changes, contact external teams, update Jira, or begin follow-up implementation unless the user separately requests and authorizes it.