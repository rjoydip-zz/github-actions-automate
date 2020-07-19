# actions-automate

Collection of github actions helps to automate gihub CI/CD.

## Table of content

- [Merge pull request](#merge-pull-request)
- [Merge pull request based on specified label](#merge-pull-request-based-on-specified-label)

### Merge pull request

Automatically merge pull request. 

Below actions code will merge PR automatically. Zero configuration/depedencies and it runs on top of `github-script`.

```yml
- uses: actions/github-script@v2
  with:
    github-token: ${{secrets.GITHUB_TOKEN}}
    script: |
      const pr_number = context.issue.number;
      const options = await github.pulls.listFiles({
        ...context.repo,
        pull_number: pr_number,
      });
      const fileRes = await github.paginate(options, (response) => response.data);
      const files = fileRes.map(x => x.filename);
      console.log(files);
      await github.pulls.merge({
        ...context.repo,
        pull_number: pr_number,
      });
```

### Merge pull request based on specified label

Automatically merge pull request based on specified label.

#### Dependency

- [github-script](https://github.com/actions/github-script)
- [Merge Pal](https://github.com/maxkomarychev/merge-pal-action)

Create file `.mergepal.yml` in root folder of your repo.

```yml
# .mergepal.yml
whitelist:
    - database
blacklist:
method: squash
```

Below actions code will create `database` label and will add it if and only if (iff) `database.json` is changed. You can use alternative action module/lib for auto-label [Pull Request Labeler](https://github.com/actions/labeler).

```yml
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
