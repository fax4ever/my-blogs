# Dark NOC Quickstart: Tasks Needed to Get Started

## Executive Summary

The [Dark NOC POC](https://github.com/msugur/auto-darknoc) is a strong example of the kind of AI-driven innovation our telco customers need. The concept is proven, autonomous AI remediation works in practice, and it integrates across the Red Hat / IBM portfolio (RHOAI, OpenShift, AAP, ACM, Granite LLM, Kafka).

We want to evolve it into a production-quality Red Hat quickstart, applying the same design patterns, principles, and practices we used for the [Self-Service Agents quickstart](https://docs.redhat.com/en/learn/ai-quickstarts/rh-it-self-service-agent).

Because this quickstart is intended to support adoption with telco partners and customers, we also want to cover additional telco use cases and extend the original POC scope.

The sections below are my recommended first / next steps. They are based on my experience and should be treated as a starting point to refine with team input and business feedback.

Overall, the business narrative and product promotion goals are already well served by the [Dark NOC POC](https://github.com/msugur/auto-darknoc). The AI Engineering team can add the most value by improving maintainability and engineering quality so the project can scale safely.

## Recommended practices

I recommend elevating the POC by decomposing it into discrete, independently testable modules, each with strong automated testing and CI/CD, then composing them into a maintainable, customer-ready system. This matches how we delivered the IT-focused [Self-Service Agents quickstart](https://docs.redhat.com/en/learn/ai-quickstarts/rh-it-self-service-agent).

### Module breakdown

Modular extraction is grounded in a foundational software engineering principle: the Single Responsibility Principle (SRP): a module should have one, and only one, reason to change. In practice, each module should do one thing well with a clear purpose; requirement changes should map cleanly to specific modules; and testing should focus on one responsibility at a time.

In general, we want to narrow module responsibilities and scopes to maximize cohesion and minimize coupling.

This work should be done collaboratively by the whole team in shared working sessions.

### Module ownership

Once a module’s scope, responsibilities, and contracts are clear, it can be assigned to a team member who becomes the module maintainer. Even then, every change should be reviewed by another team member, including contributions from the maintainer. Typically, the maintainer is either the primary author or the reviewer for changes in their module.

### Module / integration tests

We should introduce end-to-end tests scoped to each module. These integration tests should aim to cover most of the module’s behavior through its public contract.

It is important to keep test scope aligned with the module boundary: tests that are overly tied to a single internal class can become brittle and discourage useful refactors. As a rule of thumb, if modules follow design-by-contract thinking, tests should exercise the module at the API level defined by that contract.

### CI/CD pipelines

All tests should run automatically on every pull request and on every release. The same pipelines should include linting and static validation steps.

Additional pipelines can optionally build and publish container images or run heavier evaluations on a schedule (for example overnight). See the GitOps / automation setup used for the IT self-service agent: [GitHub Actions for IT self-service agent](https://github.com/rh-ai-quickstart/it-self-service-agent/actions).

### UV project management

Python projects should be managed with [uv](https://github.com/astral-sh/uv), following the tool’s recommended practices.

### Incremental / iterative approach

We should improve the project in small, reviewable steps, keeping quality high at each iteration. We followed this approach for the [Self-Service Agents quickstart](https://docs.redhat.com/en/learn/ai-quickstarts/rh-it-self-service-agent), including releasing a short video at the end of each iteration where team members called out concrete improvement areas in detail.

### Daily standup meetings

I would mirror the [Self-Service Agents quickstart](https://docs.redhat.com/en/learn/ai-quickstarts/rh-it-self-service-agent) operating model: a daily standup bot message, plus periodic in-person meetings (for example twice weekly on Monday or Tuesday, and Wednesday or Thursday) at a time that works across time zones.

### Support more telco use cases

Log analysis can be extended to support additional telco AI use cases. We can collaborate with the telco ecosystem organization to identify and prioritize them.

This expansion should come after meaningful automated test coverage is in place, so changes remain safe for existing use cases.

Extensibility depends on maintainability. The [Self-Service Agents quickstart](https://docs.redhat.com/en/learn/ai-quickstarts/rh-it-self-service-agent) is a good example: we are continuing to add functionality now that the engineering foundation supports it.

## Next steps

### Project understanding

Start with a deep review of the [Dark NOC POC](https://github.com/msugur/auto-darknoc) so the whole team builds shared context before module breakdown.

It is also valuable to deploy the current solution end-to-end and capture operational constraints, gaps, and rough edges early.

### Module breakdown

In this phase, the team defines and extracts modules using the principles above. Once a module is defined, assign a maintainer.

### Integration test coverage

After modules are defined, add integration tests that cover most behavior and run automatically in CI/CD to protect future changes.

### Cover more telco AI use cases

Working with telco ecosystem stakeholders, identify, design, and implement support for additional telco AI use cases.

## Final considerations

This document is intentionally a starting point. I expect it to be revised, expanded, and corrected based on team input and evolving business priorities.
