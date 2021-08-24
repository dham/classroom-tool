# Classroom-tool

This package provides a script, `classroom-tool` which is designed to work with GitHub Classroom to automate some of the tasks an instructor might have in marking work submitted on that platform.

## Installation

Install the package using:

```console
$ pip install git+https://github.com/dham/classroom-tool
```

## GitHub personal access token

`classroom-tool` will need to access GitHub in order to work, and to do so it will need to be able to authenticate to GitHub. You will therefore need to set up a GitHub [personal access token](https://docs.github.com/en/github/authenticating-to-github/creating-a-personal-access-token) with `repo` permissions. Store the PAT in the `GITHUB_OAUTH` environment variable so that `classroom-tool` can access it.

## Anonymising the repository

Remapping the repository names to university identity numbers is not enough to
anonymise the submissions, because every commit has author information. 

Fortunately, [git-filter-repo](https://github.com/newren/git-filter-repo) can fix this. The following code will anonymise all commits in the repository:

```
git-filter-repo --email-callback 'return b"anon@anon.eu"' --force --name-callback 'return b"Anonymous"'
```