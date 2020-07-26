# actions-automate

Collection of github actions helps to automate gihub CI/CD.

## Table of content

- [Get list of files in PR](#get-list-of-files-in-pr)
- [Create or Update googlesheet data](#create-or-update-googlesheet-data)
- [Merge PR](#merge-pr)
  - [Merge pull request](#merge-pull-request)
  - [Merge pull request based on specified label with dependency](#merge-pull-request-based-on-specified-label-with-dependency)
  - [Merge pull request based on specified label without dependency](#merge-pull-request-based-on-specified-label-without-dependency)

### Get list of files in PR

Get list of files changed in pull request.

```yml
steps:
  - uses: actions/github-script@v2
    if: github.event_name == 'pull_request'
    with:
    github-token: ${{secrets.GITHUB_TOKEN}}
    script: |
      let isDatabaseChanges = false, isArtworkChanges = false;
      const repo = context.repo;
      const pull_number = context.issue.number;
      const options = await github.pulls.listFiles({
        ...repo,
        pull_number: pull_number,
      });
      const files = (
        (await github.paginate(options, (response) => response.data)) || []
      ).map((x) => x.filename).filter(Boolean);
```

### Create or Update googlesheet data

Create or update googlesheet data. But it should do on schedule job.

```yml
name: CI
on:
  push:
    branches: [master]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - id: googlesheet_actions
        uses: rjoydip/googlesheet-actions@0.1.1
        with:
          sheet-id: ${{ secrets.SHEET_ID }}
      - uses: actions/github-script@v2
        with:
          github-token: ${{secrets.GITHUB_TOKEN}}
          script: |
            let sha = null;
            const path = "data.json";
            const branchName = "master";
            const ref = `heads/${branchName}`;
            const message = "Data updateed - commit from github actions";
            const content = Buffer.from(
                JSON.stringify(
                    ${{steps.googlesheet_actions.outputs.result}}
                )
            ).toString("base64");

            // Get the current "master" reference, to get the current master's sha
            const getRef = await github.git.getRef({
                ...context.repo,
                ref,
            });

            // Get the tree associated with master, and the content
            // of the template file to open the PR with.
            const tree = await github.git.getTree({
                ...context.repo,
                tree_sha: getRef.data.object.sha,
            });

            // Create a new blob with the existing template content
            const blob = await github.git.createBlob({
                ...context.repo,
                content,
                encoding: "utf8",
            });

            const newTree = await github.git.createTree({
                ...context.repo,
                tree: [{
                    path,
                    sha: blob.data.sha,
                    mode: "100644",
                    type: "blob",
                }],
                base_tree: tree.data.sha,
            });

            try {
                const getContents = await github.repos.getContents({
                    ...context.repo,
                    path,
                    ref: "refs/heads/master",
                });
                sha = getContents.data.sha
            } catch (_) {
                sha = newTree.data.sha
            }

            github.repos.createOrUpdateFile({
                ...context.repo,
                path,
                message,
                content,
                sha,
                committer: content.repo,
            });

```

## Merge PR

### Merge pull request

Automatically merge pull request. Below actions code will merge PR automatically. Zero configuration/dependency and it runs on top of `github-script`.

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

### Merge pull request based on specified label with dependency

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

### Merge pull request based on specified label without dependency

Automatically merge pull request based on specified label.

Below actions code will create `database` label and will add it if and only if (iff) `database.json` is changed. Zero configuration/dependency and it runs on top of `github-script`.

```yml
- uses: actions/checkout@v2
- uses: actions/github-script@v2
  with:
    github-token: ${{secrets.GITHUB_TOKEN}}
    script: |
      const repo = context.repo;
      const labels = ["database"];
      const validate_file = "database.json";
      const pull_issue_number = context.issue.number;
      const options = await github.pulls.listFiles({
        ...repo,
        pull_number: pull_issue_number,
      });
      const files = (await github.paginate(options, (response) => response.data) || []).map((x) => x.filename).filter(Boolean);
      if (
        validate_file &&
        files.length === 1 &&
        files.filter((f) => f.includes(validate_file)).length === 1
      ) {
        await github.issues.addLabels({
          issue_number: pull_issue_number,
          ...repo,
          labels,
        });
        await github.pulls.merge({
          ...repo,
          pull_number: pull_issue_number,
        });
      }
```
