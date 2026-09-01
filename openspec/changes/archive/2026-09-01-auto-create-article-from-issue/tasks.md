## 1. Issue Template Setup

- [x] 1.1 Create GitHub Issue Form template at `.github/ISSUE_TEMPLATE/new_article.yml` with fields for title, type (tech/idea dropdown), and emoji

## 2. GitHub Actions Workflow Implementation

- [x] 2.1 Create `.github/workflows/create_article.yml` configured to trigger on issues opened with the `article-draft` label
- [x] 2.2 Add step to parse title, type, and emoji from issue content
- [x] 2.3 Add step to run `npx zenn new:article` and capture the generated article file path
- [x] 2.4 Add step to create new branch `article/issue-<number>`, commit article file, and push to origin
- [x] 2.5 Add step to create Pull Request linked to the issue using `gh pr create`
