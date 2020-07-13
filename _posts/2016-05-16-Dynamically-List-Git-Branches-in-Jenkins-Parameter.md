---
layout: default
title: "Dynamically List Git Branches in Jenkins Parameter"
date: 2016-05-16
---

Jenkins git parameter plugin doesn’t list git branches if you work with bitbucket or at least it didn’t work for me, so I had to find other solution.

To get my git branches dynamically in a parameter I’m using Active Choices Plug-in with two scripts: one groovy that return the results to jenkins and the other one is a wrapper in bash which the groovy script use to get a list of git branches (because I don’t really know groovy:))

## Prerequisite

Before using this workaround you need to configure git on your jenkins server

## How To

* create get_git_branches.sh bash script that will list your git branches

```
vi /usr/local/bin/get_git_branches.sh
```

```
#!/bin/bash
GIT_URL=$1
git ls-remote --heads --tags ${GIT_URL} | awk -F" " '{print $NF}'
```

* make sure the script is executable

```
chmod +x /usr/local/bin/get_git_branches.sh
```

* In jenkins job configuration add “Active Choices Reactive Parameter”
* In the name field enter BRANCH (or what ever you want)
* Click on Groovy script and enter the following script

```
tags = []
text = "get_git_branches.sh https://user:pass@bitbucket.org/project/repo_name.git".execute().text
text.eachLine { tags.push(it) }
return tags
```

* In the “Source  Code Management” section in “Branch to build” enter ${BRANCH}

if you have better suggestions or better groovy script please write a comment
