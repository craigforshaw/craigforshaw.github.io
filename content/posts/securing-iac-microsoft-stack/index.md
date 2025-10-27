+++
title = 'Securing infrastructure as code (IaC) with the Microsoft technology stack'
date = '2023-09-25T09:00:00+00:00'
draft = false
slug = 'securing-iac-microsoft-stack'
tags = ['Azure', 'Bicep', 'IaC', 'DevSecOps', 'Github']
description = 'I thought it would be interesting to see how a secure IaC deployment would work using only Microsoft products. As it turns out.. pretty secure!'
+++

## The Challenge

Ok so the challenge is end-to-end infrastructure security with IaC using only the Microsoft technology stack. Do I really need any third party tooling or does Microsoft have the products to support securing the entire DevSecOps process?

## Bicep

Lets start with your IaC configuration files. Microsoft launched its own IaC declarative languages tool called [Bicep](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/overview?tabs=bicep) on August 31, 2020. It is a domain specific language (DSL) for infrastructure deployments in Azure.

### State

So How does Bicep handle state? Bicep is a set of configuration files similar to most other IaC tools but one of the advantages it has over its main rival is that it is stateless in nature. The state is what is actually deployed and Bicep runs a differential comparison of the actual configuration compared to the configuration files being executed. Why is this an advantage from a security perspective? A stateful tool such as Terrafom requires a separate [state file](https://developer.hashicorp.com/terraform/language/state) which needs careful planning when it comes to security. The state should be stored securely and encrypted as it contains sensitive information in clear text. This file is critical to infrastructure deployments and is a value asset for any threat actor so a secure storage, authentication and access design needs to be done. Bicep however reads directly from resource manager and coverts its templates automatically into JSON format before deploying resources.

### Parameters

Looking at the code, what type secure coding practices does Bicep support then? Well firstly marking string or object parameters as secure is a start as this these values are not saved to the deployment history and aren't logged.

```bicep
@secure()
param demoPassword string

@secure()
param demoSecretObject object
```

### Secrets

The most important security coding practice that you need to follow is placing secrets in a password vault. Microsoft has its own solution that supports this, Azure KeyVault. Bicep uses the [getSecret](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/bicep-functions-resource#getsecret) function to return a secret from KeyVault once the pre-requisites are in place ([Key Vault secret with Bicep](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/key-vault-parameter?tabs=azure-cli))

**Note:** Github and Azure DevOps also support secret storage for actions/pipelines which we will cover.

What about ARM templates i hear you ask. Well, you can pass secret parameters to ARM templates for sure! But for most cases you should be using Bicep which calls ARM anyway (those JSON files i mentioned earlier) so let's stick with that since its declarative and is in a human-readable format.

What else can we use as a security tool for secure coding practices then? Copilot of course!

## Github copilot & chat (beta)

Github copilot is an AI pair programmer that offers autocomplete-style suggestions as you code, so this tool is of huge benefit to have from a security perspective. Inline suggestions will not only fix code formatting but also make suggestions around secure code.

The recently released [Github copilot chat beta](https://docs.github.com/en/copilot/getting-started-with-github-copilot) now takes it to the next level so that you can ask copilot to suggest recommendations for securing code vulnerabilities. Infrastructure as code vulnerabilities can be analysed within chat and recommendations given to ensure a developer can remediate issues before it goes to pull request. This tool will only get better over time based on the inputs of potentially millions of developers worldwide so this will become an invaluable tool to a developer for secure coding.

So that's our IaC tool covered as well as Azure KeyVault for secret storage and some help from Copilot. Where do we put the code?

## Version Control System

Placing the config files in a VCS is a given. Luckily Microsoft has two versions to choose from, GitHub and Azure DevOps.

### GitHub Advanced Security

Microsoft aquired GitHub in 2018 and now is seen as a preferred platform over Azure DevOps at least for IaC deployments. GitHub has a feature called advanced security which is available for Enterprise accounts as well as some features available to public repositorys (further info can be found here: [GitHub plans](https://docs.github.com/en/get-started/learning-about-github/githubs-plans)).

A GitHub advanced security licence includes [code scanning](https://docs.github.com/en/code-security/code-scanning/introduction-to-code-scanning/about-code-scanning), [secret scanning](https://docs.github.com/en/code-security/secret-scanning/about-secret-scanning) and [dependancy reviews](https://docs.github.com/en/code-security/supply-chain-security/understanding-your-software-supply-chain/about-dependency-review) that give a next level of security features that are critcal in todays security landscape and especially for those of us who are new to coding in the infrastructure world. GitHub provides starter workflows with ready made security features to get started with too!

### Other security features

Security policies are available to allow for reporting of security vulnerabilities in code by adding a SECURITY.md file to the root of your repo.

Additionally, GitHub has a security advisory database with known vulnerabilties and malware, grouped in two categories: GitHub-reviewed advisories and unreviewed advisories.

Finally it supports Entra ID SSO integration for user and role management and also there are some [security hardening features](https://docs.github.com/en/organizations/keeping-your-organization-secure) available to organisations on GitHub (several 2FA options for example) as well as a host of security settings to control, log and monitor access to repositorys with, of course, it's own secret storage for actions.

### Azure DevOps

Out of the box Azure DevOps security features are restricted to permissions, access and security grouping with some pipeline secret storage available. For features such as code scanning then you would need to use the [GitHub advanced security integration feature](https://learn.microsoft.com/en-us/azure/devops/repos/security/configure-github-advanced-security-features?view=azure-devops&tabs=yaml) which has recently just become [generally available](https://devblogs.microsoft.com/devops/now-generally-available-github-advanced-security-for-azure-devops-is-ready-for-you-to-use/). This extends the GitHub advanced security features to Azure repos however worth noting that there is no plan for this to be made available to the DevOps server edition.

Additionally, a license is required on GitHub advanced security for this integration to work.

Both these version control systems ship with the standard branching pull request, peer review processes that you find in most VCS systems and are fundamental to good DevSecOps practices.

## Pipelines/actions

So your now ready to deploy code using a pipeline or as GitHub calls them a GitHub action. There are a host of [security hardening for Github actions](https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions) considerations that GitHub has published guidance for. Secrets management is the main consideration but also a lot of other important considerations need to be taken such as risk assessment and using Open ID connect when authenticating to Azure: [Configuring OpenID Connect in Azure](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-azure).

Before you run your pipeline or action you should run some Bicep validation checks:

- **Linting** - Linting validation will check for things like unused parameters, unused variables, interpolation, secure parameters and more. You can tell Bicep to verify your file by manually building the Bicep file through the Bicep CLI: `bicep build main.bicep`
- **Validate** - You can use the `AzureResourceManagerTemplateDeployment` task to submit a Bicep file for preflight validation.
- **What-if** - Previews the changes that will happen. The what-if operation doesn't make any changes to existing resources. Instead, it predicts the changes if the specified Bicep file is deployed.

## Defender for DevOps

So you've created the code, put it in GitHub or Azure DevOps, used GitHub copilot to help quality check before you create a pull request, you've validated the deployment and your ready to deploy with your pipelines or github actions.

We now need to check the code via an analyser and also ensure that your security team is aware of any security vulnerabilities that might exist in your repository. Luckily Microsoft have an answer to that too.. Defender for DevOps.

[Defender for DevOps](https://learn.microsoft.com/en-us/azure/defender-for-cloud/defender-for-devops-introduction) provides comprehensive visibility, posture management, and threat protection across multi-cloud environments including Azure. Initial set up involves connecting the service to a GitHub org or repository as well as enabling the GitHub advanced security features on the target (which we mentioned earlier).

### Configure the Microsoft security github action

This action incorporates static analysis tools into a github action to be run on your IaC code repository. It has a host of open-source tools for analysis but in our case its [template analyser](https://github.com/Azure/template-analyzer) that covers analysis of Bicep configuration files.

The steps involve simply incorporating this [sample action workflow](https://github.com/microsoft/security-devops-action/blob/main/.github/workflows/sample-workflow.yml) into your actions as a pre-requisite for code deployment. With all the GitHub advanced security features enabled this can report security vulnerabilities to github under the security tab and auto-create pull requests for remediation. Additionally, this data is reported back to Defender for DevOps for visibility to the security teams for remediation and reporting.

Its also worth mentioning that there is an equivalent for Azure DevOps pipelines too. Marvellous!

## Summary

In summary Microsoft has matured its offerings around IaC and DevSecOps with its tooling over the past few years and most recently the offerings around GitHub advanced security, GitHub Copilot and Defender for DevOps are now introducing even more advanced capabilities in this space to compete with Hashicorp, Sonar and CyberArk among others.

As more sophisticated supply chain threats emerge its important to have full end-to-end security coverage that continuously evolves over time. Microsoft is building its capability in this area which is exciting to see and will benefit its customers who need to implement DevSecOps with the latest emerging security tools to protect their environments.
