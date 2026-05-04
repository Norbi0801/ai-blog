+++
title = "Running a software project - what nobody teaches you in CS class"
date = 2025-12-20
description = "A practical guide to executing software projects: scope, estimation, Git workflows, communication, and the failure patterns that kill most student and junior projects."

[taxonomies]
tags = ["software-engineering", "project-management", "git", "teamwork"]
+++

You can write a binary search from memory. You know what a hash map is. You survived the algorithms final. None of that prepares you for the moment four people stare at a shared repository and realize nobody knows who's building what, the deadline is in three weeks, and the README still says "TODO."

Writing code is a skill. Shipping software is a different one. Most CS programs spend four years on the first and maybe one afternoon on the second. This post is about the second part - the unglamorous work of actually getting a project from idea to working product with a team of real humans.

<!-- more -->

## The numbers are bad

The Standish Group has been tracking IT project outcomes since 1994 in their [CHAOS Report](https://opencommons.org/CHAOS_Report_on_IT_Project_Outcomes). The latest data: **only 31% of software projects succeed**. 50% are "challenged" (late, over budget, or missing features). 19% fail outright - cancelled, never delivered.

These aren't student projects. These are professional teams with budgets, project managers, and years of experience. If teams getting paid to do this fail two thirds of the time, what does that tell you about how hard project execution actually is?

Project size matters dramatically here. Small projects (under $1M) succeed about 90% of the time. Large projects (over $10M) are **ten times more likely to be cancelled** than small ones. The lesson: keep your scope small. A small project that ships beats an ambitious one that doesn't.

The most common killers, [according to PMI research](https://www.pmi.org/learning/library/top-five-causes-scope-creep-6675):

- **37% fail due to unclear objectives and milestones** - the team doesn't know what "done" looks like
- **Scope creep** - "quick tweaks" pile up until the project is twice its original size
- **Communication gaps** - developers code one vision while stakeholders expect another
- **Estimation failures** - timelines based on optimism instead of evidence

Notice something? None of these are technical problems. Nobody's project failed because they picked the wrong sorting algorithm. Projects fail because of people problems - unclear goals, bad communication, unrealistic plans.

## Methodologies - the map, not the territory

Before getting into the practical stuff, it helps to know the major approaches people use to organize software work. You'll hear these terms in every job interview and on every team. None of them is a silver bullet - they're frameworks, not religions.

### Waterfall

The oldest approach. You go through phases sequentially: Requirements -> Design -> Implementation -> Testing -> Deployment. Each phase must be complete before the next begins.

```
Requirements ──> Design ──> Implementation ──> Testing ──> Deployment
     │                                                         │
     └──────── no going back (in theory) ─────────────────────┘
```

**When it works:** When requirements are truly fixed and well-understood upfront. Building a bridge. Implementing a specification that won't change. Government contracts where the scope is legally binding.

**When it fails:** Almost every software project where humans are involved. Requirements change. You discover halfway through implementation that the design has a flaw. The client sees the first demo and says "that's not what I meant." By the time you reach the testing phase, the bugs are expensive to fix because they were introduced months ago.

Waterfall gets a bad reputation, and often deservedly so. But the core idea - think before you build - is sound. The problem is assuming you can think of everything upfront.

### Agile

Agile isn't a methodology - it's a set of values from the [Agile Manifesto](https://agilemanifesto.org/) (2001). The key ideas:

- **Individuals and interactions** over processes and tools
- **Working software** over comprehensive documentation
- **Customer collaboration** over contract negotiation
- **Responding to change** over following a plan

The manifesto doesn't tell you how to run your project. It tells you what to prioritize when trade-offs arise. Scrum and Kanban are specific implementations of these ideas.

### Scrum

The most popular Agile framework. Work happens in **sprints** - fixed time boxes, usually 2 weeks. Each sprint has:

- **Sprint planning** - team picks items from the backlog for this sprint
- **Daily standup** - 15-minute sync (what did I do, what will I do, any blockers)
- **Sprint review** - demo what was built to stakeholders
- **Sprint retrospective** - what went well, what didn't, what to change

Roles:
- **Product Owner** - decides what to build and in what order (the backlog)
- **Scrum Master** - facilitates the process, removes blockers
- **Development Team** - builds the thing

**For student projects:** You don't need formal Scrum. But the sprint concept is gold. Pick a cadence (1 or 2 weeks), plan what you'll deliver by the end of that sprint, demo it, reflect on what went wrong. That rhythm alone will keep your project on track.

### Kanban

Simpler than Scrum. No sprints, no roles. Just a board with columns:

```
┌──────────┬──────────────┬───────────┬──────────┐
│ Backlog  │ In Progress  │ Review    │   Done   │
├──────────┼──────────────┼───────────┼──────────┤
│ Task E   │ Task B       │ Task A    │ Task X   │
│ Task F   │ Task C       │           │ Task Y   │
│ Task G   │              │           │ Task Z   │
└──────────┴──────────────┴───────────┴──────────┘
```

The key rule: **limit work in progress (WIP).** If your WIP limit is 2 per person, you can't start Task D until you finish Task B or C. This prevents the common failure mode where everyone has 5 things "in progress" and nothing is actually getting done.

Kanban works well when work items arrive continuously (bug reports, support tickets) or when the team is small and doesn't need the ceremony of Scrum.

### What to actually use

For a student team of 3-5 people working on a project for a few weeks? **Kanban with weekly syncs.** A GitHub Projects board or Trello, WIP limits of 2, and a 30-minute meeting once a week to check progress. That's it. You don't need a Scrum Master. You don't need sprint ceremonies. You need a board everyone looks at and a regular heartbeat.

For larger or longer projects, Scrum's sprint structure adds valuable rhythm. But start simple. You can always add process. Removing process that's not working is politically harder.

## Start with scope, not code

The biggest mistake in student projects (and honestly, in professional ones too): opening your editor before you know what you're building.

Scope answers three questions:

1. **What does this thing do?** Not in technical terms - in user terms. "A user can create an account, list their expenses, and see a monthly summary." Concrete, verifiable.
2. **What does it NOT do?** This is more important than the first question. Explicitly listing what's out of scope prevents the "oh, we should also add..." conversations later. Write it down: "No mobile app. No real-time sync. No multi-currency support. V1 only."
3. **What does done look like?** A checklist of features that, when all checked, means you ship.

Write this down. In a document, a GitHub issue, a shared note - anywhere the whole team can see it. A scope that lives only in someone's head is not a scope at all.

Here's what a simple scope doc looks like in practice:

```markdown
# Expense Tracker - V1 Scope

## In scope
- User registration and login (email + password)
- Add expense (amount, category, date, note)
- List expenses with filters (date range, category)
- Monthly summary with total per category
- REST API + basic web frontend

## Out of scope
- Mobile app
- Multi-currency
- Receipt scanning / OCR
- Shared expenses / groups
- Export to CSV/PDF

## Done when
- [ ] User can register and log in
- [ ] User can CRUD expenses
- [ ] Expense list shows filters working
- [ ] Monthly summary page renders correctly
- [ ] Deployed to a publicly accessible URL
```

That took five minutes to write. It will save you five hours of arguing later.

### Requirements vs. scope

Requirements are more detailed than scope. A requirement says *how* something should work: "The login form must show an error message after 3 failed attempts." Scope says *what* exists: "Users can log in."

For student projects, a scope doc is usually enough. For professional work, you'll write requirements documents, user stories with acceptance criteria, API specifications. The principle is the same: write down what you're building before you build it, with enough detail that two developers reading the same document would build roughly the same thing.

## Choosing a tech stack

This decision happens early and is hard to reverse. A few principles:

**Use what the team already knows.** If three out of four team members know Python and one knows Rust, use Python. The project will move faster. Learning a new language while building a project under deadline is a recipe for frustration and half-baked code.

**Avoid resume-driven development.** Picking Kubernetes, GraphQL, and a microservice architecture for a four-person project that serves 10 users is not engineering - it's padding your LinkedIn. Pick the simplest stack that solves the problem. A monolith with SQLite will take you further than you think.

**Pick boring technology.** There's a [famous blog post by Dan McKinley](https://mcfunley.com/choose-boring-technology) about this. Every new or trendy technology comes with unknown failure modes. PostgreSQL has been around since 1996 and its failure modes are well-documented on Stack Overflow. That brand-new database you read about on Hacker News last week? Not so much. You get a limited budget of "new things" per project - spend it on the parts that actually need novelty.

**Decide on the API contract early.** If you have frontend and backend teams, agree on the API shape before writing code. A simple OpenAPI spec or even a shared document listing endpoints, request bodies, and response shapes prevents the "your API returns `user_name` but I expected `username`" problem.

```yaml
# Even this rough sketch saves hours of integration pain
POST /api/expenses
  Request:  { amount: int, category: string, date: string, note?: string }
  Response: { id: string, amount: int, category: string, date: string, note: string, created_at: string }
  Errors:   400 (validation), 401 (not logged in)

GET /api/expenses?from=2026-01-01&to=2026-03-31&category=food
  Response: { expenses: [...], total: int }
```

**Document the stack decision.** Why did you pick this database? Why this framework? Not for posterity - for the team member who joins two weeks in and asks "why are we using X?" A few sentences in the README is enough.

## Break work into pieces that can actually be done

A task that says "build the backend" is not a task. It's a wish. Tasks need to be small enough that one person can finish one in a day or two.

The breakdown looks like this:

**Epic** (the big goal): "Users can track expenses"

**Stories** (user-visible chunks):
- User can create an account
- User can add an expense
- User can see their expense list
- User can view monthly summary

**Tasks** (actual work items):
- Set up project structure and database schema
- Implement user registration endpoint
- Implement login endpoint with JWT
- Create expense model and migration
- Implement POST /expenses endpoint
- Implement GET /expenses with query params
- Build monthly aggregation query
- Wire up frontend login form
- Build expense form component
- Build expense list view

Each task has a clear definition of done. "Implement POST /expenses endpoint" is done when you can `curl` it and get an expense back in the database. No ambiguity.

### Dependencies matter

Some tasks can't start until others finish. You can't build the expense list frontend until the GET /expenses API exists. You can't implement login until the user model and database are set up.

Map these dependencies. Not in a fancy Gantt chart - just think about it for ten minutes:

```
Database setup ──> User model ──> Auth endpoints ──> Protected routes
                       │
                       └──> Expense model ──> Expense endpoints ──> Frontend views
```

Start with the tasks that unblock the most other work. Database setup and basic models first. Then API endpoints. Then frontend. This is called the **critical path** - the longest chain of dependent tasks that determines your minimum project duration.

Use whatever tool your team agrees on - GitHub Issues, a Trello board, a Notion table, sticky notes on a wall. The tool matters less than the habit. Every morning, everyone should be able to look at the board and know: what's in progress, what's done, what's next.

## Git workflow - the non-negotiable

If your team is committing directly to `main`, stop. Today. Here's the workflow that works for teams of 2 to 20:

**1. `main` is always deployable.** Nothing gets merged that breaks the build.

**2. Feature branches for everything.**

```bash
git checkout -b feature/add-expense-endpoint
# do your work
git add src/routes/expenses.rs src/models/expense.rs
git commit -m "Add POST /expenses endpoint with validation"
```

Branch names should describe what's in them. `feature/login`, `fix/expense-date-parsing`, `chore/update-dependencies`. Not `johns-branch` or `test123`.

**3. Commit messages that mean something.**

Bad:
```
fix stuff
wip
asdfasdf
final version
final version 2
FINAL final version
```

Good:
```
Add expense creation endpoint with category validation
Fix date parsing for expenses with timezone offset
Add monthly summary aggregation query
```

A good commit message completes the sentence "If applied, this commit will..." - *add expense creation endpoint with category validation*. Future you (and your teammates) will thank present you when `git log` actually tells a story.

There's a widely adopted standard called [Conventional Commits](https://www.conventionalcommits.org/) that formalizes this:

```
feat: add expense creation endpoint
fix: handle timezone offset in date parsing
docs: add API endpoint documentation to README
chore: update dependency versions
refactor: extract validation logic into shared module
```

The prefix makes it immediately clear what type of change this is. You don't have to use Conventional Commits, but whatever format you pick, be consistent across the team.

**4. Pull requests before merging.**

```bash
git push -u origin feature/add-expense-endpoint
# then open a PR on GitHub
```

PRs serve two purposes: **code review** (someone else looks at your code before it hits `main`) and **documentation** (the PR description explains *why* this change exists). Even in a two-person team, review each other's code. You'll catch bugs, learn from each other, and avoid the "I have no idea what this code does" problem two weeks later.

A PR description doesn't need to be long:

```markdown
## What
Adds the POST /expenses endpoint.

## Why
Users need to be able to create expenses - this is the core write path.

## How to test
curl -X POST http://localhost:8080/api/expenses \
  -H "Content-Type: application/json" \
  -d '{"amount": 1500, "category": "food", "date": "2026-04-01"}'
```

### What to look for in code review

When you're reviewing someone's PR, focus on:

- **Does it do what it says?** Read the description, then read the code. Do they match?
- **Edge cases** - what happens with empty input, zero values, missing fields?
- **Naming** - can you understand what a function does from its name? Are variable names clear?
- **Complexity** - is there a simpler way to do this? A 50-line function that could be 15?
- **Tests** - are the important paths tested?

What NOT to nitpick in review: formatting (use a formatter), personal style preferences, things that "could be slightly better" but work fine. Review is about catching bugs and sharing knowledge, not about proving you're smarter.

**5. Pull before you push. Rebase or merge, but stay current.**

```bash
git fetch origin
git rebase origin/main
# resolve conflicts if any, then push
```

Merge conflicts are inevitable. They're not a bug - they're a signal that two people touched the same area. Resolve them carefully, test after resolving, and move on.

### When things go wrong with Git

Everyone messes up Git at some point. The most common disasters and their fixes:

**"I committed to main instead of a branch":**
```bash
git branch feature/my-work        # save your commits to a new branch
git checkout main
git reset --hard origin/main      # reset main to match remote
git checkout feature/my-work      # continue on your branch
```

**"I need to undo my last commit but keep the changes":**
```bash
git reset --soft HEAD~1           # uncommit, but keep changes staged
```

**"I have merge conflicts and I'm panicking":**
Don't panic. Open the conflicted file. Look for the `<<<<<<<`, `=======`, `>>>>>>>` markers. The top section is your code, the bottom is theirs. Decide which version to keep (or combine both), remove the markers, save, `git add`, and continue.

The golden rule: **if you're about to run a Git command you found on Stack Overflow and don't fully understand, ask someone first.** A bad `git reset --hard` or `git push --force` can destroy work. `git reflog` can often save you, but prevention is better than recovery.

## Estimation - the hardest skill in software

Developers are terrible at estimation. This isn't a personal failing - it's a [well-documented cognitive bias](https://en.wikipedia.org/wiki/Planning_fallacy) called the planning fallacy. We estimate based on the best-case scenario and forget about debugging, integration issues, unclear requirements, and that afternoon we'll lose to a package upgrade breaking everything.

### Why estimation is hard

There are structural reasons estimation fails:

**The cone of uncertainty.** At the start of a project, your estimates can be off by 4x in either direction. After requirements are defined, maybe 2x. After design, 1.5x. Estimates get better as you learn more, but early estimates are guesses wearing a suit.

```
Project start:      0.25x ──────────────── 4x
Requirements done:  0.5x  ────────── 2x
Design done:        0.67x ────── 1.5x
Code started:       0.8x  ──── 1.25x
```

**Novel vs. familiar work.** You can estimate "build a CRUD endpoint" because you've done it before. You can't estimate "integrate with this payment API we've never used" because you don't know what you don't know. Unknown work has unknown duration.

**Context switching.** You estimate 4 hours for a task assuming 4 focused hours. In reality you'll have a meeting, answer some messages, help a teammate debug something, and get 2.5 focused hours. Estimate in elapsed time, not focused time.

### Techniques that help

**Break it down first, estimate second.** You can't estimate "build the backend." You can estimate "write the expense creation endpoint" - maybe 3-4 hours. Estimate the small tasks, then add them up.

**Multiply by two.** Seriously. Whatever you think it'll take, double it. If you think a task is 4 hours, plan for 8. This isn't pessimism - it's accounting for the reality that you'll hit at least one unexpected problem per task.

**Planning poker.** Each team member independently estimates a task (using t-shirt sizes or story points), then everyone reveals at the same time. If estimates vary wildly - say, one person says S and another says XL - you discuss. The person who said XL usually knows about a complexity the others missed. This technique surfaces hidden assumptions.

**Track actuals.** After you finish a task, note how long it actually took. After a few weeks, you'll see a pattern: you consistently underestimate by 40%, or database tasks take twice as long as API tasks. This data makes your next estimates better.

**Use t-shirt sizes for rough planning.** When you don't need hour-level precision, categorize tasks as S (half day), M (one day), L (two-three days), XL (needs to be broken down further). If a task feels XL, that's a signal you don't understand it well enough yet.

**Never estimate something you've never done before as "easy."** Setting up CI/CD for the first time? That's not a 30-minute task. Integrating a payment API you've never used? Not "probably an afternoon." Unknown work has unknown duration. Give it buffer.

## Communication patterns that actually work

The failure mode here is predictable: everyone works in isolation for two weeks, then comes together and discovers three people built overlapping features, one person is blocked on something nobody knew about, and the integration doesn't work.

**Daily standups (async or sync).** Takes 5 minutes. Three questions per person:
1. What did I do since last time?
2. What am I doing next?
3. Am I blocked on anything?

For student projects where you're not meeting daily, a Slack/Discord message every other day works fine. The point is that everyone knows what everyone else is doing. No surprises.

**Weekly sync.** One longer meeting per week where you look at the board together:
- Are we on track for the deadline?
- Does anything need to be cut or reprioritized?
- Are there technical decisions that need the whole team?

**Document decisions.** "We decided to use PostgreSQL instead of SQLite because we need concurrent writes" - put it in a decision log, a GitHub discussion, or even a pinned message. Two weeks later when someone asks "why Postgres?", you point to the doc instead of trying to reconstruct the conversation from memory.

For important architectural decisions, teams use **Architecture Decision Records (ADRs)** - a lightweight format:

```markdown
# ADR-001: Use PostgreSQL instead of SQLite

## Status
Accepted

## Context
Our app needs to handle concurrent writes from multiple users.
SQLite uses file-level locking which creates contention under load.

## Decision
Use PostgreSQL for the primary database.

## Consequences
- Need to run a Postgres instance (Docker in dev, managed in prod)
- Team needs basic SQL and Postgres knowledge
- Better concurrent performance and query capabilities
```

You probably don't need formal ADRs for a student project. But the habit of writing down "what we decided and why" will save you when someone asks "why did we do it this way?" in week four and nobody remembers.

**Raise blockers immediately.** If you're stuck and have been for more than an hour - say something. In a team chat, in a PR comment, in person. The worst thing you can do is sit silently on a blocker for three days and then announce it at the deadline. Your team can't help you if they don't know you need help.

## CI/CD - automation that pays for itself

CI/CD stands for Continuous Integration and Continuous Deployment. In practice:

- **CI** = every time someone pushes code or opens a PR, automated checks run (tests, linting, building)
- **CD** = when code is merged to `main`, it's automatically deployed

You don't need CD for a student project. But CI is a force multiplier even for a team of two.

### Why CI matters

Without CI, the conversation goes like this: "It works on my machine." "Well, it doesn't work on mine." "Did you install the dependencies?" "Which ones?" "The ones in the README." "The README is from two weeks ago."

With CI, every PR triggers a build on a clean machine. If the build fails, the PR can't be merged. This catches:
- Missing dependencies that exist on your machine but not in the project
- Tests that pass locally but fail with a different seed or environment
- Code that compiles on macOS but not Linux
- Formatting and linting issues

### Setting up GitHub Actions

GitHub Actions is free for public repos and gives you 2,000 minutes/month for private repos. A basic CI config:

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Node.js        # adjust for your stack
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install dependencies
        run: npm ci

      - name: Run linter
        run: npm run lint

      - name: Run tests
        run: npm test

      - name: Build
        run: npm run build
```

For a Rust project:

```yaml
jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install Rust
        uses: dtolnay/rust-toolchain@stable

      - name: Run clippy
        run: cargo clippy -- -D warnings

      - name: Run tests
        run: cargo test

      - name: Build
        run: cargo build --release
```

This takes 15 minutes to set up and will catch problems for the entire duration of the project. It's one of the highest-return investments you can make.

### Branch protection

After setting up CI, enable branch protection on `main` in your GitHub repo settings:
- Require PR reviews before merging (at least 1 approval)
- Require status checks to pass (your CI workflow)

Now nobody can accidentally push broken code to `main`. The process is enforced by the tool, not by discipline alone.

## Technical debt - the tax on shortcuts

Technical debt is the gap between "the code we have" and "the code we wish we had." It's the quick hack you wrote to make the demo work, the test you skipped because the deadline was tomorrow, the hardcoded config that should be an environment variable.

Some technical debt is fine. Deliberate shortcuts to hit a deadline are a valid trade-off - as long as you know you're making them and plan to pay them back. The dangerous kind is accidental: debt you don't know you're accumulating.

### Good debt vs. bad debt

**Acceptable (deliberate, documented):**
- "We're hardcoding the admin email for now. TODO: move to config before launch."
- "This query is O(n^2) but n is always under 100. Optimize if data grows."
- "Skipping input validation on this internal-only endpoint. Add before making it public."

**Dangerous (accidental, invisible):**
- Copy-pasted code in five places that will need to be updated in five places
- No tests on the payment flow because "it works and we're afraid to touch it"
- A global variable that three modules depend on in undocumented ways

The rule: **if you take a shortcut, leave a comment.** A `// TODO: this is a hack because X, fix when Y` is infinitely better than a mystery hack that the next developer (possibly future you) has to reverse-engineer.

### When to pay it back

The answer is usually "before it blocks you." If that hardcoded config is fine for the demo but will break in production - fix it before production. If the O(n^2) query is fine for 100 items but you're about to have 10,000 - fix it now.

For student projects: don't obsess over perfect code. Ship first. But keep a list of known shortcuts so you can address them if time allows, and so your future maintainer (or grader) knows they were intentional.

## Roles and ownership

Even without formal titles, someone needs to own things. "Everyone is responsible" means nobody is responsible.

**Code ownership** doesn't mean gatekeeping - it means accountability. If Person A owns the authentication module, that means:
- They review PRs that touch it
- They're the first person to ask when something breaks
- They make sure it stays working as other parts of the system change

For a student team of four, a simple split works:

```
Alice: Backend API + database
Bob: Frontend
Carol: Auth + deployment
Dave: Testing + CI/CD + documentation
```

This doesn't mean Dave can't write backend code. It means when the CI pipeline breaks at 11 PM, Dave is the person who knows how to fix it.

**Rotate ownership** on longer projects. If one person always does deployment, they're the only one who knows how. That's a single point of failure. Pair on ops tasks, document procedures, and swap responsibilities between milestones.

## What actually kills projects

After watching dozens of student projects and being part of enough professional ones, these are the patterns that show up over and over:

### The "we'll figure it out as we go" approach

No scope doc, no task breakdown, no shared understanding of what you're building. Everyone has a slightly different mental model. Two weeks in, you discover that one person built a REST API and another built a GraphQL one. Starting without a plan doesn't save time - it costs time. Every ambiguity you skip in planning becomes a conflict in execution.

### The hero developer

One person does 80% of the work. They're the only one who understands the codebase. When they get sick, go on vacation, or burn out - the project stops. Distribute knowledge. Pair program. Review each other's code. No single point of failure, in infrastructure or in people.

### Scope creep by committee

"Wouldn't it be cool if we also added..." Yes, it would be cool. It would also be another week of work when you have two weeks left. Every new feature has a cost, and that cost isn't just the coding time - it's testing, documentation, integration with everything else, and the bugs it'll introduce. Say no to features. Say it often. You can always add them in v2.

### Integration at the end

Frontend team builds their side for four weeks. Backend team builds their side for four weeks. Week five: "Let's put it together." Nothing works. The API contract was never agreed on. Field names don't match. Date formats are different. Auth works differently than the frontend expected.

Integrate early and often. Get a "hello world" flowing from frontend through backend to database in week one, even if it's ugly. Then build features on top of that working skeleton. This is sometimes called the **walking skeleton** approach - a tiny end-to-end implementation that proves the architecture works, then you flesh it out feature by feature.

### Not testing until the last day

"We'll test it before the presentation." Famous last words. Then you find 15 bugs, fix 10, introduce 3 new ones, and demo with your fingers crossed.

You don't need 100% test coverage. But you need:
- A way to run the app locally with one command
- At least some automated tests for the critical paths
- Manual testing after every merge to `main`

If `git pull && cargo run` (or `npm start`, or `docker compose up`) doesn't work at any point in the project, you have a problem. Fix it before writing new features.

### The "we'll deploy later" fallacy

The project works on localhost. It has never run anywhere else. Two days before the deadline, someone tries to deploy it. Environment variables are missing. The database connection string is hardcoded to `localhost:5432`. The frontend has hardcoded `http://localhost:3000` URLs. File paths use Windows backslashes but the server runs Linux.

Deploy on day one. Even if it's just a "hello world" page. Services like [Railway](https://railway.app/), [Fly.io](https://fly.io/), or [Render](https://render.com/) have free tiers that take minutes to set up. Once you have a deployment pipeline working, every subsequent deploy is a `git push`.

## Retrospectives - learning from what happened

A retrospective is a structured conversation about how the work went. It happens at the end of a sprint, a milestone, or the project. Format:

**What went well?** Things to keep doing. "Code reviews caught two critical bugs." "The weekly sync kept everyone aligned."

**What didn't go well?** Things that caused pain. "We underestimated the auth integration by 3x." "Nobody tested on mobile until the last day."

**What will we change?** Concrete actions. Not "communicate better" - that's a wish. "Post standup updates in #dev-updates by 10am every Monday, Wednesday, Friday."

The point isn't blame. The point is that the same team making the same mistakes twice is a process failure, not a people failure. Write down what you learn. Apply it to the next sprint or project.

For student projects that are one-off: do the retro anyway. Write it down. You'll carry those lessons to your next team project, your internship, your first job. The people who improve fastest are the ones who reflect deliberately.

## A minimal project checklist

Here's the bare minimum for a team project that doesn't want to join the 69% failure statistic:

**Week 0 - Setup:**
- [ ] Scope doc with in/out of scope and definition of done
- [ ] Repo with README explaining how to run the project
- [ ] Task board with initial breakdown
- [ ] API contract documented (if frontend + backend)
- [ ] CI pipeline running on every PR
- [ ] Branch protection on `main` (require PR reviews)
- [ ] Deploy a "hello world" to a public URL
- [ ] Assign ownership areas

**Every week:**
- [ ] Standups (async or sync, at least 3x per week)
- [ ] Weekly sync to check progress against scope
- [ ] Integration testing - does `main` still work?
- [ ] Update task board - close done tasks, add new ones if needed
- [ ] At least one deployment to the live environment

**Before delivery:**
- [ ] All scope checklist items are done (or explicitly cut with a reason)
- [ ] README has setup and run instructions that actually work
- [ ] Known technical debt is documented
- [ ] Quick retrospective completed
- [ ] Demo/presentation prepared from working software, not slides

## The uncomfortable truth

Project management isn't glamorous. Nobody got excited about writing a scope doc or updating a task board. But the projects that ship are the ones where someone did the boring work. Someone wrote down the scope. Someone broke the tasks apart. Someone nagged about the standups. Someone said "no, we're not adding that feature."

The 31% of projects that succeed aren't using magic technology or hiring ten times better developers. They're doing the basics consistently. They have clear scope, realistic plans, regular communication, and the discipline to cut what doesn't fit.

The gap between a computer science education and professional software development isn't algorithms or data structures - you've got those covered. The gap is everything in this post: scoping, estimating, communicating, integrating, deploying, and saying no. These are skills like any other. They're learnable. But they require practice, and the best time to start practicing is right now, on a project where the stakes are a grade instead of someone's livelihood.

The code is the easy part. The project is the hard part. Start acting accordingly.
