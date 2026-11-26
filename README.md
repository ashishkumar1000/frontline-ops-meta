# Frontline Operations Platform

## What is this project?

This project has two main purposes:

1. **Manage all project code in one place:** It helps developers manage different parts of the
   platform, such as the mobile app, website and backend services. Each part has its own code and
   history, but developers can manage them together.
2. **Define how the product should work:** It contains all the plans, features, business rules and
   technical details needed to build a platform for managing daily work in businesses.

> **Important:** This project currently contains plans, instructions and tools for developers. It
> does not contain the actual mobile app, website or backend application code yet.

## The main idea

The platform is designed to help businesses manage their daily work from one place.

For example, it can be used by:

- Retail stores
- Restaurants
- Warehouses
- Factories
- Offices and other facilities

Today, many businesses use different apps, spreadsheets and paper records to manage their work.
This platform brings everything together.

Here is how it works:

1. **Set the rules:** Head office decides how work should be done.
2. **Assign work:** Employees receive their daily tasks on their phones.
3. **Complete work:** Employees complete their assigned tasks.
4. **Share proof:** Employees upload photos, forms or other information to show that work is
   completed.
5. **Report problems:** If something goes wrong, employees can report it.
6. **Fix problems:** Managers assign someone to solve the problem.
7. **Check again:** Managers make sure the problem has been fixed.
8. **Learn and improve:** The business uses this information to improve how work is done.

AI will also help with tasks such as checking photos, finding problems, answering questions and
preparing reports. However, important business and safety rules will always be checked before AI
takes action.

## Where can you find information?

This project contains different documents. Each document explains a different part of the product.

| If you want to know... | Open this document | What it explains |
|---|---|---|
| What are we building? | [`docs/00-PRD.md`](docs/00-PRD.md) | Product idea, goals, users and future plans |
| How will the platform work? | [`docs/01-FSD.md`](docs/01-FSD.md) | How different parts of the platform work together |
| What needs to be developed? | [`docs/02-Epics-and-Stories.md`](docs/02-Epics-and-Stories.md) | 15 major work areas and 154 smaller tasks |
| What are all the features? | [`docs/modules/README.md`](docs/modules/README.md) | Simple explanation of all 25 features |
| What rules should developers follow? | [`docs/_context/README.md`](docs/_context/README.md) | Common rules and important decisions |
| Where is everything documented? | [`docs/README.md`](docs/README.md) | List of all project documents |
| How are different projects managed? | [`METAREPO.md`](METAREPO.md) | How to manage all parts of the project together |
| How do AI tools help developers? | [`CLAUDE.md`](CLAUDE.md) / [`AGENTS.md`](AGENTS.md) | Rules for using AI while building the product |

## The 25 main features of the platform

The platform is divided into 25 features, grouped by what they do.

### 1. Basic features (01–05)

These are the main features needed to run the platform.

- **Employee and company management:** Manage employees, companies, branches and teams.
- **Task management:** Create tasks and assign them to employees.
- **Form builder:** Create forms and checklists without writing code.
- **Mobile app:** Employees can use the app to complete their work, even without internet.
- **Proof and records:** Safely store photos, documents and other proof of completed work.

### 2. Problem tracking and learning (06–09)

These features help businesses find and fix problems.

- **Problem tracking:** Report problems, assign them to people and track until they are fixed.
- **Messages and information:** Share important updates and instructions with employees.
- **Employee training:** Give employees short training lessons based on their work.
- **Photo checking:** Use AI to check photos and identify possible mistakes or rule violations.

### 3. Reports and automation (10–13)

These features help managers understand what is happening and save time.

- **AI assistants:** AI helpers that can perform approved tasks and answer questions.
- **Management reports:** Show business performance, problems and future predictions.
- **Connect with other software:** Share information with existing accounting, HR and business
  systems.
- **Security and privacy:** Protect employee and business information.

### 4. Employees and equipment (14–18)

These features help manage people and business equipment.

- **Staff scheduling:** Plan employee shifts and working hours.
- **Attendance tracking:** Record when employees start and finish work.
- **Equipment maintenance:** Track machines and equipment that need repair or regular servicing.
- **Smart sensors:** Collect information from connected devices and machines.
- **Employee engagement:** Help businesses communicate with employees and collect their feedback.

### 5. Additional business features (19–25)

These features support different business needs.

- **Stock and purchasing:** Manage stock and purchase supplies.
- **Health and safety:** Track workplace safety rules and incidents.
- **Food safety:** Help restaurants and food businesses follow food safety rules.
- **Rewards and points:** Reward employees for completing tasks and achieving goals.
- **New store setup:** Manage the work needed to open a new store or branch.
- **Add-on store:** Allow businesses to add more tools and features.
- **Vendor management:** Manage outside companies and service providers.

## How do we build the product?

We follow a two-step process using AI development tools.

### Step 1: Plan everything first (BMAD)

Before developers start building something, we first decide what needs to be built.

- Understand the business requirement.
- Discuss how the feature should work.
- Decide what information needs to be stored.
- Define the rules and expected behaviour.
- Prepare clear instructions for developers.

The planning team prepares the documents and designs. It does not write the actual application
code.

### Step 2: Build and test the product (GSD)

Once the plan is ready, developers start building the feature.

The process is:

1. Discuss the work and clarify any doubts.
2. Break the work into small tasks.
3. Build each task separately.
4. Run tests to check that everything works.
5. Check the code for errors and problems.
6. Mark the task as complete only after all checks pass.

This helps ensure that developers build what was planned and that the work is properly tested.

## How to get started with the project

The project uses Bun to install and manage development tools.

Developers need to run these commands:

```bash
bun install              # installs everything needed to work on the project
bun run workspace:setup  # downloads all registered projects to your computer
bun run workspace:status # shows which projects have changes
bun run workspace:push   # checks and sends changes to the correct project repositories
bun run typecheck        # checks the project management tools for coding errors
```

> **Important:** The list of child projects is currently empty. These commands will manage
> projects as they are added.

## One important rule for developers

Each part of the platform has its own separate code repository.

For example:

- Mobile app has its own repository.
- Backend has its own repository.
- Website has its own repository.

When developers make changes, they must save those changes in the correct project.

They should never save application code in the main project folder.

This keeps the code of each project separate and makes it easier to manage.

## Other important information

- `docs/repos/` stores important information about each project.
- `docs/initiatives/` stores information about work that affects multiple projects.
- Some older documents are kept only for reference. Developers should use the newer documents
  inside the correct feature folder.
- The technology choices for building the platform have already been decided. Developers should
  follow those choices instead of selecting different technologies.

---

**In simple words:** This project is the master plan and management centre for building a complete
business operations platform. It explains what to build, how every feature should work, what rules
to follow and how AI tools should help developers build and test it.
