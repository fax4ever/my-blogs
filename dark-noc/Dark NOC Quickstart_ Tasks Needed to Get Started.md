# Dark NOC Quickstart: Tasks Needed to Get Started

# Executive Summary

[Dark NOC POC](http://github.com/msugur/auto-darknoc) represents exactly the kind of AI-driven innovation our telco customers need. The concept is proven, autonomous AI remediation actually works, and it has full integration across Red Hat / IBM portfolio (RHOAI, OpenShift, AAP, ACM, Granite LLM, Kafka).  
We would like to transform it into a production-quality Red Hat quickstart, trying to apply the same design patterns, principles and practices we’ve covered for instance in [Self-Service Agents quickstart](https://docs.redhat.com/en/learn/ai-quickstarts/rh-it-self-service-agent).   
Moreover, since this quickstart is designed to support the adoption for Telco partners and customers, we would like to cover different Telco use cases, extending the original scope of the POC.  
In the following you will find my recommendation about possible first / next steps. Please keep in mind that those would be based on my very personal experience, so they should be considered a starting point that will be fixed, improved and corrected by the team’s contributions and the business feedback.  
All in all, my take is that business needs and product promotion goals are already very well covered by [Dark NOC POC](http://github.com/msugur/auto-darknoc), while the AI Engineering team could help in improving the overall maintainability of the project.

# Recommended Practices

I suggest elevating the POC by decomposing it into discrete, independently testable modules, each with comprehensive test coverage and CI/CD, then composing into a maintainable, customer-ready system. This approach matches how we successfully delivered the IT in [Self-Service Agents quickstart](https://docs.redhat.com/en/learn/ai-quickstarts/rh-it-self-service-agent).

## Module Breakdown

Modular extraction is built on a foundational software engineering principle: the Single Responsibility Principle (SRP): A module should have one, and only one, reason to change. In practice it means that each module does one thing well and has one clear purpose, changes to requirements affect only one module, while testing focuses on one responsibility at a time.  
In general, we want to cut the responsibilities and the scopes of the modules to maximize the cohesion and minimize the coupling.  
In my opinion this activity should be done by the entire team during some shared sessions.

## Module ownership

Once a module has been clearly defined in terms of scope, responsibilities and design by contract can be assigned to a member of the team that becomes the maintainer of the module. In any case a contribution should be reviewed by another member of the team even if it is done by the maintainer of the module. Usually the maintainer should be the contributor or the reviewer of its modules.

## Module/Integration tests

We would like to introduce tests end-to-end scoped on each module. Those integration tests should try to cover as much as possible the source code. In this case it is very important to pay attention to the scope of the tests, unit tests scoped on a single class would be not useful and could limit the modification of the code. As a rule of thumb since each module should follow the design by contract principle, the test should use the level of API defined by the module contract.

## CI/CD pipelines

All tests should be executed automatically every time a pull request is made or when we do a new release. The same pipelines should also contain some code lint and validation steps done by some automatic code check tools. Other pipelines can be optionally configured to build and publish new containers or to execute special evaluations overnight, see [GitOps defined for IT self service agent](https://github.com/rh-ai-quickstart/it-self-service-agent/actions) for instance.

## UV project management

Python projects should be managed by [UV](https://github.com/astral-sh/uv), following all the best practices of the tool.

## Incremental / Iterative approach

The idea is to contribute to the project one step at a time, keeping the quality and the solidity very high at each iteration. We followed this principle when we developed the [Self-Service Agents quickstart](https://docs.redhat.com/en/learn/ai-quickstarts/rh-it-self-service-agent), for instance releasing a video at the end of each iteration in which all the members of the team described some areas of improvement in detail. 

## Daily standup meetings

I would follow the experience of [Self-Service Agents quickstart](https://docs.redhat.com/en/learn/ai-quickstarts/rh-it-self-service-agent) again, introducing a daily stand up bot message, plus having two meetings in presence periodically Monday or Tuesday, and Wednesday or Thursday at a time that is comfortable for all the team members.

## Support more Telco use cases

The logs analysis could be extended to support different and more Telco AI use cases. We can collaborate with the Telco Ecosystem organization for instance to identify, support and cover more use cases. This phase should be done after the test coverages, in order to make any additions / changes to the code base safe with the respect of the pre-existing use cases.  
Extensibility also requires manutenability, as proven again by the [Self-Service Agents quickstart](https://docs.redhat.com/en/learn/ai-quickstarts/rh-it-self-service-agent), in which we’re assisting in adding more functionalities these days.

# Next Steps

## Project Understanding Step

My suggestion for the first step would be to study in depth the [Dark NOC POC](http://github.com/msugur/auto-darknoc). So that all the members of the team could have a solid understanding of it in order to be proactive for the Module Breakdown phase.  
It is good for instance also to try to deploy the current code base solutions and see all difficulties and constraints.

## Module Breakdown Step

In this phase the entire team will proceed in defining and extracting the modules according to the principles stated before. Once a module is defined it can be assigned to a module maintainer.

## Integration Test Coverage Step

As mentioned above, once the modules are defined we want to define tests that cover most of the code and are automatically executed on CI/CD to verify the solidity of the further changes.

## Cover More Telco AI Use Cases Step

In collaboration with people from Telco organizations, we’re going to identify, cover and support more Telco AI Use cases.

# Final considerations

I would like to stress the fact that this document has to be intended as a starting point to address the topics that are covered. I’m expecting to be reviewed, changed and several things will be added by the team and the business needs.