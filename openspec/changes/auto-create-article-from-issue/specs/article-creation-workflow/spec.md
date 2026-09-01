## ADDED Requirements

### Requirement: Issue template for new articles
The repository SHALL provide a GitHub Issue Form template allowing authors to input the title, article type (tech or idea), and emoji for a new article.

#### Scenario: User opens new article issue form
- **WHEN** user selects the "New Article" template on GitHub Issues
- **THEN** the form presents fields for Title (text input, required), Type (dropdown: tech/idea, required), and Emoji (text input, 1 character, required)

### Requirement: Automated draft generation and pull request workflow
The system SHALL trigger a GitHub Actions workflow when an Issue using the new article template is opened, creating a draft article via Zenn CLI and opening a Pull Request.

#### Scenario: Workflow executes on issue creation
- **WHEN** an issue is opened with the article template labels or format
- **THEN** the workflow extracts title, type, and emoji from the issue body
- **THEN** the workflow creates a new git branch `article/issue-<issue_number>`
- **THEN** the workflow runs `npx zenn new:article` with the specified title, type, and emoji (allowing slug to be auto-generated)
- **THEN** the workflow commits and pushes the newly created article file to the branch
- **THEN** the workflow creates a Pull Request targeted to `main` with `Resolves #<issue_number>` in the body
