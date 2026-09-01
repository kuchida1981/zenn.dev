## 1. Issue Template Refinement

- [ ] 1.1 Update `.github/ISSUE_TEMPLATE/new_article.yml` to remove the default `title` value so the input starts empty with GitHub's default placeholder
- [ ] 1.2 Update the `emoji` input field in `.github/ISSUE_TEMPLATE/new_article.yml` to be optional (`required: false`) and set default initial value `value: "📝"`

## 2. GitHub Actions Workflow Refinement

- [ ] 2.1 Simplify title extraction in `.github/workflows/create_article.yml` to use `context.payload.issue.title` directly
- [ ] 2.2 Ensure fallback for emoji extraction in `.github/workflows/create_article.yml` defaults to `📝` if empty or unspecified
- [ ] 2.3 Add step in `.github/workflows/create_article.yml` to close the original issue via `gh issue close` after PR creation succeeds
