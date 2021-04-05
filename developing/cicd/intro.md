# [CI/CD](README.md) / Intro to CI/CD

The goal of CI/CD practice is to provide a workflow that can support
frequent updates, good testing, consistent builds, and prompt deploys.
Additionally issues with code should be found quickly and addressed before
it is released to customers.

<!-- toc -->

* [Delivery Pipeline Basics](#delivery-pipeline-basics)
  * [Build](#build)
    * [Notes about Versioning](#notes-about-versioning)
      * [Semantic Versioning](#semantic-versioning)
      * [Commit Hash](#commit-hash)
      * [Other versioning strategies](#other-versioning-strategies)
  * [Test](#test)
  * [Deployment](#deployment)
  * [Release](#release)
* [Continuous (Integration | Delivery | Deployment)](#continuous-integration--delivery--deployment)
  * [Continuous Integration](#continuous-integration)
  * [Continuous Delivery](#continuous-delivery)
  * [Continuous Deployment](#continuous-deployment)

<!-- Regenerate with "pre-commit run -a markdown-toc" -->

<!-- tocstop -->

## Delivery Pipeline Basics

There are usually four conceptual steps in a delivery pipeline:

1. `Build`: Where you build the code into a binary or other artifacts to eventually distribute.
2. `Test`: Where you validate or test the artifacts built in the previous step.
3. `Deploy`: Where you configure and deploy the artifacts to an environment. Could be a pre-prod or prod.
4. `Release`: Where you finally allow users access to that version of code you've configured and built.

In many cases you can merge or swap the steps `Release` and `Deploy`.

Philosophical ideas and notes about each step are outlined in the following sections.
These are not the be-all end-all of what we do at Truss but are a starting point for how we think about this process.

Note: These steps are all fungible.
They can be combined to a degree, you can have multiple deployment and validation steps.
You could merge deploy and release.
What is important is that each of those steps is well understood and documented.
That they are configurable and repeatable.

### Build

**Builds should be repeatable.**
This means you should be able to check out the code from your project at that same commit hash and build it again and get the same artifact(s).
_Note_: This means that dependencies should be versioned in your codebase.
To keep dependencies up to date in an automated way you can use [Dependabot](https://dependabot.com/) with our [documentation on configuring it](dependabot.md).
Please do whatever is "correct" in the context of the languages/frameworks your project is built in.

**Builds should be hermetic.** This means that the build should be isolated from other builds.
In CI, the build shouldn't share the same workspace or files from a previous build or from a build of a different project.

**Builds should output an immutable artifact.** Artifact outputs should not be rewritten or altered by subsequent builds.
This allows you to distribute or redeploy from these unaltered artifacts for different points in code.
Additionally, try to label the artifact with the appropriate versioning scheme for your project.

#### Notes about Versioning

Version your code and artifacts. Doesn't really matter how, just do it. It will make it easier to track down issues or deploy specific versions of your project.

In your code consider adding a command or an endpoint that surfaces that version for debugging.

Tag your versioned releases on your mainline branch.
This helps you find the state of mainline at the point in time your artifact was built.
Additionally, if you are using GitHub you can use their releases functionality to share release artifacts and notes.

Here is a preferential ordering of versioning schemes:

##### Semantic Versioning

Why semver? It tells you and your customer how much has changed since the last released code and sets expectations accordingly.
If you are tagging at mainline where you are building from, you can rebuild the artifact from the same point.

From [semver.org](https://semver.org):

```
Given a version number MAJOR.MINOR.PATCH, increment the:

MAJOR version when you make incompatible API changes,
MINOR version when you add functionality in a backwards compatible manner, and
PATCH version when you make backwards compatible bug fixes.
Additional labels for pre-release and build metadata are available as extensions to the MAJOR.MINOR.PATCH format.
```

##### Commit Hash

A commit hash is unique (with extremely few collisions) and is easily linked back to history in code.
However, how much has changed is opaque to your users and it is difficult to determine how old this version is in comparison to other versions.

##### Other versioning strategies

These are other versioning strategies we've seen. We do not recommend them.

* _Feature branch related names._ These should be short lived and maintained for no more than a few days. Most users will not find these useful.
* _Build id related names._ These are opaque to a user and harder to dig up history on when debugging.
* _Build date related names._ These are also opaque to a user and difficult to dig up history on when debugging. You at least get a sense of when these changes went in but are hard to tie to a commit in mainline.

### Test

We will not be writing deeply about testing methodologies or specifics around kinds of tests and the philosophies here.

In the context of a CI pipeline:

**Tests should be hermetic.** Ideally, multiple CI runs should be able to run at the same time without affecting each other.

**Tests should be idempotent.** Tests in CI should be able to run several times over without producing inconsistent effects.

**Tests should be reproducible in a developer's computer.** So developers can debug them more easily.

**Tests should run fast.** "Fast" is poorly defined here.
Generally speaking, you want tests to be fast so merging code changes is fast.
A developer shouldn't think they can start on a new piece of work in the hour or two to push a change through CI and context switch.

**Tests should fail fast.** Similar to the previous point, "fast" is relative but in CI you will want to give the waiting devloper signal on whether their work can be merged or requires changes as quick as possible.

### Deployment

Deployment will differ project to project and should take into account the requirements of the system being deployed to.

**Deployment should be repeatable.** The same code and configuration should be deployable again and again.

**Deployment process for software should be the same regardless of environment.** Using the same process and tooling means lower cognitive overhead in deployment and your non production environments should reflect production more closely.

**Deployment tools should be maintained separate from the application being deployed.** We've seen this happen where we need to fix some deployed code but also had a problem with the code that performs the deployment and all of it is a hairy mess of debugging.

### Release

There are a few different ways of implementing release.
No matter what, release is the step when you allow customers to have access to that particular version of your code.

If you're releasing a CLI, it may mean that you're releasing your binary to a Homebrew tap or just a zip file to GitHub releases.

If you have a service, you could have a loadbalancer manage traffic between different versions of your services.
Or you could use something like Launch Darkly to leverage feature flags to do something similar for different user cohorts.

## Continuous (Integration | Delivery | Deployment)

We throw around terms like CI or CD with the assumption that we know what the differences are between these practices.

For clarity, at Truss we'll refer to them as follows:

### Continuous Integration

`Continuous Integration` is when you validate on every push to mainline.
This helps folks find failures in the code before it reaches mainline.
Ideally, automated build and test steps validate changes and stop any new changes to mainline.
Mainline must always be in a good state to deploy from.

This may manifest as a process where developers merge changes into mainline as often as possible.
Small incremental changes to code and small discrete tests make changes easier to understand and facilitate frequent merges in CI.
This practice helps mitigate risks from long lived branches and helps validate small changes in quick succession.

That being said, in practice we see longer lived branches for features or versions.
These branches need to be kept up to date with mainline with frequent merges or rebases from mainline.

The point of either of those practices is to minimize drift between your working branch and mainline so validation in CI or manually is easier.

### Continuous Delivery

`Continuous Delivery` can be described as an extension to `Continuous Integration`.
In addition to automated build and test steps running on every proposed change to mainline, the delivery of the code is also automated.

You want the software built to be deployable at any given point in time.

Delivery may mean different things depending on your workflow.
It may mean that you have built your artifacts and validated them and delivered them to a centralized repository.
For example, Docker containers might go to Docker Hub or Amazon Elastic Container Registry (ECR).
Might mean that the code has been deployed to a staging environment.
All of these could be "delivered" depending on the project's workflow.

### Continuous Deployment

`Continuous Deployment` can be described as a further extension of `Continuous Delivery`.
In addition to automated build, test, and delivery, production deployment is also automated.

This is an advanced state that requires excellent, trustworthy, automated tests and monitoring. The complexity of continuous deployment system is correlated with the complexity of the delivery pipeline. It's much easier to have continous deployment of a single container image than a full-featured web application.
