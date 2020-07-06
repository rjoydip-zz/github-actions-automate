# actions-automate

Collection of github actions automate.

## Table of content

- [Merge pull request based on specified label](#merge-pull-request-based-on-specified-label)

### Merge pull request based on specified label

It will create and add `database` label if only `database.json` changed. You can use alternative action module/lib for auto-label [Pull Request Labeler](https://github.com/actions/labeler). Create file `.mergepal.yml` in root folder of your repo

#### Dependency

- [github-script](https://github.com/actions/github-script)
- [Merge Pal](https://github.com/maxkomarychev/merge-pal-action)

```yml
# .mergepal.yml
whitelist:
    - database
blacklist:
method: squash
```

```yml
# .github/workflows/ci.yml
name: pr automate

on: [pull_request]

jobs:
  merge-database:
    runs-on: ubuntu-latest
    steps:
      - id: checkout
        uses: actions/checkout@v2
      - id: file_changes
        uses: trilom/file-changes-action@v1.2.3
      - id: add_label
        uses: actions/github-script@v2
        with:
          github-token: ${{secrets.GITHUB_TOKEN}}
          script: |
            const files = ${{steps.file_changes.outputs.files}}
            if (files.length === 1 && files.filter(f => f.includes('database.json')).length === 1) {
              await github.issues.addLabels({
                issue_number: context.issue.number,
                owner: context.repo.owner,
                repo: context.repo.repo,
                labels: ['database']
              })
              return true
            } else {
              return false
            }
      - name: merge-database
        if: steps.add_label.outputs.result == true
        uses: maxkomarychev/merge-pal-action@v0.5.1
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
```
