# GitHub Actions and the CI process
GitHub Actions is a powerful tool. I don't think there's any other way to put it. 

A CI/CD tool, fully native to the GitHub platform, running on Microsoft Azure VMs, with the ability to perform a myriad of tasks in your dev cycle. Learning about Actions has been a pleasure.

I wanted to talk about something specific today though - Actions and the CI process. How would I use Actions as a substitute to, say Jenkins/Circle CI? You hear the value point, but what does this _actually_ look like? Let's take a look.

## Before we start
Things I'm expecting you to know:
- [ ] You're familiar with the [CI/CD process in software development](https://resources.github.com/ci-cd/)
- [ ] You have a "101" understanding of [GitHub Actions](https://github.com/features/actions)
- [ ] You know what a ['runner'](https://docs.github.com/en/actions/learn-github-actions/understanding-github-actions#runners) is
- [ ] (Optional) You have heard of Javascript/Node.js

That's kind of it! We'll be learning about how to integrate testing into our development workflow. We WILL NOT be learning about how to run Actions on self-hosted runners, scaling policies, how to deploy your code to a Cloud Provider, or any of the other things you can do on Actions (which is a lot). I might talk about these things in future posts.

Also wanted to call out that I used the similarly named course from the [GitHub Learning Lab](https://lab.github.com/githubtraining/github-actions:-continuous-integration) to walk through this exercise. One of the reason I wrote this is because our [Learning Lab is getting sunset](https://github.com/github/releases/issues/2016) (at the time of writing this), so you might not even be able to access the link above. There are also a ton of other resources to learning about Actions and the CI process which I'll link at the bottom of this post. Alright let's jump in!

## The Basics <sub><sub>10,000m view</sub></sub>
Conceptually, we can break the CI process into 3 parts:

1) **Coding the feature**
2) **Building your program**
3) **Testing it**


<details>
<summary>I think this is self explanatory.</summary>
<br>
<b>MORE CONTEXT:</b>

Let's say I want to push a new feature to my company's website: a questionnaire that appears at the top of the page. 

Before pushing it into the main code I need to actually develop the feautre (code), run the website with my new code (build), and then make sure it works as expected given certain condition (test). Within the CI process, and what we try to do at GitHub, is really automate away the "build" and "test" portion in the cleanest way possible, so that developers really only have to focus on "coding". 
</details>

## The Setup <sub><sub>1,000m view</sub></sub>
The best way to learn about this is by looking at some code. Let's say I have simple `Node.js` app. I'd want to build a workflow that goes through the steps mentioned above, so that everytime I open a PR my code is automatically tested. I'll be using [`Jest`](https://jestjs.io/), a Javascript testing framework.

>Try following along by using **[this repo template](https://github.com/Eldrick19/github-actions-for-ci)**. Feel free to fork it as well.

 Remember that a GitHub Actions workflow will usually have the following structure: 

![Workflow Structure](img/workflow_structure.png "Workflow Structure Diagram")

Ok.. now let's take a look at the starter CI workflow provided by Node.js

```yaml
# This workflow will do a clean installation of node dependencies, cache/restore them, build the source code and run tests across different versions of node
# For more information see: https://help.github.com/actions/language-and-framework-guides/using-nodejs-with-github-actions

name: Node.js CI

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  build:

    runs-on: ubuntu-latest

    strategy:
      matrix:
        node-version: [12.x, 14.x, 16.x]
        # See supported Node.js release schedule at https://nodejs.org/en/about/releases/

    steps:
    - uses: actions/checkout@v2
    - name: Use Node.js ${{ matrix.node-version }}
      uses: actions/setup-node@v2
      with:
        node-version: ${{ matrix.node-version }}
        cache: 'npm'
    - run: npm ci
    - run: npm run build --if-present
    - run: npm test
```

This is a great starting point. You can see that the workflow will run when a push or pull request is made off our `main` branch. We also have one job called `build` that runs different Node.js versions, and goes through specific steps:
- [Checkout](https://github.com/actions/checkout): this gives our runner availability to the codebade. A very common action
- [Setup Node](https://github.com/actions/setup-node): Sets up a node environment (making packages like npm available for use)
- Expresses commands to the runner

We'll want to edit this to work better in our situation, but for now copy this code to a file called `node.js.yml` under the `.github/workflows` directory if you're following along.


## Things we want to do

1. Tell Actions to run our specified tests during PRs

2. Specify Language / OS versions we want to test

3. Separate `test` and `build` jobs

4. Use artifacts to pass objects through jobs

---

### (1) Tell Actions to run our specified tests during PRs <sub><sub>100m</sub></sub>

As you can see from our starter workflow above, we run the command `npm test`. In the sample repo provided, this will run fine - but only because we put in a `__test__` folder and are using the Jest framework. How you test will depend on your testing framework and tools. I have included links to additional reading on testing frameowrks at the bottom.

For our example, this step is complete already, but keep in mind this will differ depending on your situation.

### (2) Specify Language / OS versions we want to test <sub><sub>100m</sub></sub>

Our team is deploying this app to Node versions 12 and 14. We also want to support deploying the app to a Windows environment, on top of Linux. To do this we'd create Matrix builds: builds that run across multiple VM instances. In this scenario, our Matrix build will look something like this:

| | Node 12.x | Node 14.x |
| --- | --- | --- |
| Ubuntu | Node 12 / Ubuntu | Node 14 / Ubuntu |
| Windows | Node 12 / Windows | Node 14 / Windows |

i.e. Four builds being run.

Let's change `node.js.yml` to reflect this:

````yaml
strategy:
      matrix:
        os: [ubuntu-latest, windows-2016]
        node-version: [12.x, 14.x]
````

### (3) Separate `test` and `build` jobs <sub><sub>100m</sub></sub>

We want to follow the CI step framework we mentioned above (code, build, test). Let's reflect this by changing `node.js.yml` to be broken down into a `build` job followed by a `test` job:

````yaml
name: Node CI

on: [push]

jobs:
  build:
    ...
  test:
    ...
````

Under `build` include the following portion of the existing workflow: 

````yaml
build:
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v2
    - name: npm install and build webpack
      run: |
        npm install
        npm run build
````

And under `test`, include the following:

````yaml
test:
  runs-on: ubuntu-latest
  strategy:
    matrix:
      os: [ubuntu-latest, windows-2016]
      node-version: [12.x, 14.x]
  steps:
  - uses: actions/checkout@v2
  - name: Use Node.js ${{ matrix.node-version }}
    uses: actions/setup-node@v1
    with:
      node-version: ${{ matrix.node-version }}
  - name: npm install, and test
    run: |
      npm install
      npm test
    env:
      CI: true
````

### (4) Use artifacts to pass objects through jobs <sub><sub>100m</sub></sub>

We'll be using artifacts in this workflow to facilitate passing of the build container from the `build` job to the `test` job 

<details>
<summary>More on GitHub artifacts.</summary>
<br>
When a workflow produces something other than a log entry, it's called an artifact. For example, the Node.js build will produce a Docker container that can be deployed when we run the action <i>actions/setup-node@v1</i>. This artifact, the container, can be uploaded to storage using the action <a href ="https://github.com/actions/upload-artifact">actions/upload-artifact</a> and downloaded from storage using the action <a href ="https://github.com/actions/download-artifact">actions/download-artifact</a>.

Storing an artifact helps to preserve it between jobs. Each job uses a fresh instance of a VM, so you can't reuse the artifact by saving it on the VM. If you need your artifact in a different job, you can upload the artifact to storage in one job, and download it for the other job.

Artifacts are stored in storage space on GitHub. The space is free for public repositories and some amount is free for private repositories, depending on the account.
</details>


We'll add the following block as a step to our `build` job:

````yaml
- uses: actions/upload-artifact@main
      with:
        name: webpack artifacts
        path: public/
````

This will allow our `build` job to upload all the artifacts it creates to GitHub storage under the `public/` path.

Next, we'll add the following block to our `test` job:

(1) _Under `test`_:

````yaml
needs: build
````

(2) _As a step following `- uses: actions/checkout@v2`_:

````yaml
- uses: actions/download-artifact@main
      with: 
        name: webpack artifacts
        path: public
````

As you can see, in this job we've added an instructions for it to wait for our `build` job to complete prior to running, as well as an instruction to download the artifacts from available storage.

Your final workflow should look like this:

````yaml
name: Node CI

on: [push]

jobs:
  build:

    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v2
      - name: npm install and build webpack
        run: |
          npm install
          npm run build
      - uses: actions/upload-artifact@main
        with:
          name: webpack artifacts
          path: public

  test:
    needs: build
    runs-on: ubuntu-latest

    strategy:
      matrix:
        os: [ubuntu-latest, windows-2016]
        node-version: [12.x, 14.x]

    steps:
    - uses: actions/checkout@v2
    - uses: actions/download-artifact@main
      with:
        name: webpack artifacts
        path: public
    - name: Use Node.js ${{ matrix.node-version }}
      uses: actions/setup-node@v1
      with:
        node-version: ${{ matrix.node-version }}
    - name: npm install, and test
      run: |
        npm install
        npm test
      env:
        CI: true
````


## Wrap-Up

We come to an end of using GitHub Actions in a simple CI process! If you haven't already, try copying **[this repo template](https://github.com/Eldrick19/github-actions-for-ci)** and pasting this workflow as a new PR under `.github/workflows/node.js.yml`. You'll see the tests run and (hopefully) pass.

I also wanted to call out again that depending on your testing strategy and the tools you use to do CI your workflow could look different. For example instead of running `npm build` it is possible for your workflow to spin up your app using Maven (`mvn --batch-mode --update-snapshots verify`).

Or, let''s say you have a rails app and part of your CI includes [linting](https://www.freecodecamp.org/news/what-is-linting-and-how-can-it-save-you-time/): you can include this is a specific step in your `test` job:

````yaml
 - uses: ruby/setup-ruby@477b21f02be01bcb8030d50f37cfec92bfa615b6
        with:
          ruby-version: 2.6
      - run: bundle install
      - name: Rubocop
        run: rubocop
````

These workflows can really be configured as you need.

Finally, there are other tools and actions available on GitHub to have a well-rounded CI process:
1. In the [repo template](https://github.com/Eldrick19/github-actions-for-ci) add the code block below into a new file: `.github/workflows/approval-workflow.yml`. 
````yaml
name: Team awesome's approval workflow
on: pull_request_review
jobs:
  labelWhenApproved:
    runs-on: ubuntu-latest
    steps:
    - name: Label when approved
      uses: pullreminders/label-when-approved-action@master
      env:
        APPROVALS: "1"
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        ADD_LABEL: "we're approved!"
        REMOVE_LABEL: "awaiting%20review"
````
This labels approved workflows, which can be used for further automation (e.g. auto-merging or even deployment!). <sub>You can turn approval requirements on under 'Settings' > 'Branches and editing the branch protection rules'</sub>

2. Requiring status checks before merging ensures your CI process is validated prior to ever merging

3. Allowing auto-merge to automate away the "merge branch" manual step

4. etc.

Hope you enjoyed - this was my first try at a blog post! Going to try and refine the process a little going forward, but excited to see where this will go. Until next time :wave:

## References / Extra Reading

- [Continuous Integration Overview](https://docs.github.com/en/actions/automating-builds-and-tests/about-continuous-integration)
- More on [Build Automation and Top 10 Tools](https://lightrun.com/dev-tools/top-10-build-automation-tools/)
- More on [Test Automation and Top 10 Frameworks](https://dzone.com/articles/top-10-test-automation-frameworks-in-2020)
- [GitHub Actions Workflow Syntax](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions)
- [Billing of GitHub Storage Space](https://docs.github.com/en/billing/managing-billing-for-github-packages/about-billing-for-github-packages)



