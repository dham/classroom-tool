# Classroom-tool

This package provides a script, `classroom-tool` which is designed to work with
[GitHub Classroom](https://classroom.github.com) to automate some of the tasks
an instructor might have in marking work submitted on that platform.

The assumed workflow is that students undertake a GitHub classroom assignment,
which they push to GitHub by a given date and time. The instructor(s) mark the
work on a separate, private, GitHub repository. This repository contains one
pull request per student, which enables the instructor(s) to mark the work by
commenting on the pull request. The instructors' comments are returned to the
students by creating PDFs of the annotated pull requests.


## Installation

Install the package using:

```console
$ pip install git+https://github.com/dham/classroom-tool
```

## GitHub personal access token

`classroom-tool` will need to access GitHub in order to work, and to do so it will need to be able to authenticate to GitHub. You will therefore need to set up a GitHub [personal access token](https://docs.github.com/en/github/authenticating-to-github/creating-a-personal-access-token) with `repo` permissions. Store the PAT in the `GITHUB_PAT` environment variable so that `classroom-tool` can access it.

## Setting up the GitHub assignment

When configuring the assignment on GitHub Classroom, ensure that you select
`Enable feedack pull requests`. `classroom-tool` will need these as the base
for the marking pull requests in the marking repository.

## Configuration file

Configuration information which is constant for a particular marking exercise
needs to be provided in a configuration file in
[ConfigParser](https://docs.python.org/3/library/configparser.html) format. A
simple example follows.

```
[students]
roster = classroom_roster.csv

[github]
organization = my-org
basename = exam-2021
marking-repo = marking-exam-2021

[assignment]
deadline = 2021-05-26T12:00:00+01:00
```

### students

This section contains information about the students taking the exercise.

`roster` is a `.csv` (comma-separated values) file with at least the follwing
columns. These are present in the roster you can download from GitHub classroom:

`identifier`: The identifier of the student at your institution, such as a
student number or username.

`github_username`: The GitHub username of the student.

Any other columns will be preserved in output. The following optional column
has special meaning:

`extra_time`: The additional time allowed to a student in minutes (for example
as a result of a disability). This can be blank for students with no additional
time.

### github

This section contains the (non-sensitive) GitHub information about the
assignment.

`organization`: The name of the GitHub organization containing the classroom.

`basename`: The GitHub classroom prefix for the current assignment.

`marking-repo`: The name of the repository in which the marking branches and
pul requests will be created. This is in the same organization. The marking
repository should be an initially empty, **private** repository. Its name
should not start with `basename`.

### assignment

Information specific to this assignment.

`deadline`: The submission deadline in iso 8601 time and data format.

Note that GitHub does not record push times, so this is the timestamp on the
last commit which will be accepted.

## classroom-tool usage

`classroom-tool` assumes a sequential processing of the submissions, with
potential manual intervention at each stage to cater for anomolous inputs or
circumstances.

The basic usage is:
```
classroom-tool --config-file CONFIG_FILE
```
with `CONFIG_FILE` replaced by the name of the configuration file. By itself,
this will do nothing. Actual processing is achieved by adding the options for
one or more of the following, stages. A help message is available by passing `--help`.

### Fetching repositories

Passing `--fetch` will fetch all the existant student branches from GitHub.

### Creating local branches

Passing `--create-branches` will create local branches with name
`identifier-main` (or `identifier-master`) and `identifier-feedback`. Here,
`identifier` is taken from the class roster in the configuration file, as
opposed to the GitHub username. If the identifier is anonymous (such as a
student number), this forms the first stage in anonymising the submissions.

### Imposing the deadline

Passing `--impose-deadline` will create local branches with names of the form
`--identifier-mark`. These branches will point to the last commit before the
deadline specified in the configuration file. In the current version of
`classroom-tool`, extra time is not automatically accounted for at this stage:
you will need to manually move the branch pointer in these cases. For example
if student 123456 had additional time and you wish to mark their very last
commit rather than the last commit before the deadline, you would run:
```
git branch -f 123456-mark 123456-main
```

### Creating a submission report

Passing `--create-report` will create a `csv` file with a name of the form
`marking-repo-report.csv` containing all of the columns from the roster file,
plus the following:

`commit_time`: The timestamp of the last commit on the student's `main` (or
`master`) branch.

`late`: True if `commit_time` is after the deadline. This field **does** take
into account extra time.

`cloned`: True if the student accepted the assignment on GitHub.

`submitted`: True if the student pushed at least one commit.

This report is indended to assist in identifying students who may have had
technical or other issues.

### Pushing to the marking repository.

Passing `--push` will push all of the local, `-main`, `-master`, `-feedback`,
and `-mark` branches to the marking repository on `GitHub.

### Creating marking pull requests.

Passing `--pull-requests` will create a pull request for each student in the
marking repository on GitHub. This pull request will compare the `-mark` branch
with `-feedback`. Creating pull requests is intentionally throttled to once per
three seconds in order to avoid triggering GitHub rate limits when working with
large classes.

## Anonymising the repository

Remapping the repository names to university identity numbers is not enough to
anonymise the submissions, because every commit has author information. 

Fortunately, [git-filter-repo](https://github.com/newren/git-filter-repo) can fix this. The following code will anonymise all commits in the repository:

```
git-filter-repo --email-callback 'return b"anon@anon.eu"' --force --name-callback 'return b"Anonymous"'
```

Note that this action will rewrite *all* the commits in the repository. This
means that none of the branches in the marking repository will share any
history with the student repositories from which they were copied. This means
that you probably only want to take this step once you are sure that all the
other steps have been successfully completed. If you have already pushed the
branches to GitHub then you will need to force push the anonymised branches:

```
classroom-tool --push --force
```
