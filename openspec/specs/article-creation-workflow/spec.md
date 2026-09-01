# article-creation-workflow Specification

## Purpose
Issue template and automated GitHub Actions workflow to create draft article Pull Requests from GitHub Issues.

## Requirements
### Requirement: Issue template for new articles
The repository SHALL provide a GitHub Issue Form template allowing authors to input the title, article type (tech or idea), and emoji for a new article.

#### Scenario: User opens new article issue form
- **WHEN** user selects the "New Article" template on GitHub Issues
- **THEN** the form presents an empty Title input field (using GitHub standard placeholder), a Type dropdown (tech/idea, required), and an Emoji input field (optional, default value `📝`)

### Requirement: Automated draft generation and pull request workflow
The system SHALL trigger a GitHub Actions workflow when an Issue using the new article template is opened, creating a draft article via Zenn CLI, opening a Pull Request, and closing the original Issue.

#### Scenario: Workflow executes on issue creation
- **WHEN** an issue is opened with the article template format
- **THEN** the workflow extracts title, type, and emoji from the issue (falling back to `📝` if emoji is not specified)
- **THEN** the workflow creates a new git branch `article/issue-<issue_number>`
- **THEN** the workflow runs `npx zenn new:article` with the specified title, type, and emoji
- **THEN** the workflow commits and pushes the newly created article file to the branch
- **THEN** the workflow creates a Pull Request targeted to `main`
- **THEN** the workflow closes the original issue with a completion message referencing the created Pull Request
