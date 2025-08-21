---
title: "Automate testing in the technical doc workflow"
date: 2025-08-21
categories: [blog]
tags: [doc-as-code]
layout: single
author_profile: true
share: true
---

Automate testing in the technical doc workflow
Keeping technical documentation valid is usually a challenging job. It depends on a mechanism that can detect discrepancies between the product and the docs. Without a procedural guarantee, it's easy to be ignored, as documenting new features gets higher priority.
One advantage of doc-as-code is that it brings documentation closer to the product code, thereby making it more likely to be built and tested automatically. 
Background
I worked as a technical writer at a database company last year. Occasionally, code updates would break key procedures described in the documentation. 
As we just shifted our docs to Gitlab and started adopting a doc-as-code workflow, I experimented with a automation testing pipeline inspired by our development team’s CI (Continuous Integration), which  automatically runs database regression tests on a VM whenever an updates was pushed to the master branch.
This is not a step-by-step tutorial but more of a project summary, as the process contains steps that only apply to my case. But I think the idea can be generalized and potentially applied to doc workflows that face  similar maintainence issue.

What’s CI
In case you’re unfamiliar with Continuous Integration: it's a process that allows developers to merge changes into the main codebase frequently. CI includes automated tests that verify the changes and ensure no conflicts arise.
In addition, my project leverages the PostgreSQL regression testing tool, pg_regress, to do the actual testing work. With pg_regress, you just need to create test cases in SQL, the tool can run the test and return the result and information needed to debug.
Flow
Here’s how the flow works:

DB updated → CI pipeline triggered → (In VM) Pull test cases → Install the latest DB →  Run the tests → Return test results → Clean up the env
Below is a breakdown of how it was established:
Prerequisites
Setting up a full test framework or pipeline from scratch isn’t realistic. So this trial relies on a few prerequisites:
Your development team already has a CI pipeline.


Test cases are not difficult to be created. In my case, the test tool uses SQL syntax, which isn’t too complex for a database tech writer.


You have access and permissions to required resources (e.g., VM creation in the dev/test environment).

Step 1: Set Up a VM for Testing
First, you need an environment to run tests. For my case, the pipeline uses pg_regress to execute SQL scripts on a live database. So I needed a VM where the pipeline could install the latest DB, add test cases, and run the tests.
With the help of a test engineer, I got a copy of the script that set up and initialize the regression test flow.  (This can be done by a writer too, through reading the code of the test pipeline. But it would take much longer.)
You’ll need to check with your dev team about how they manage test environments and mirror the configuration and the launch method. 

Step 2: Prepare the test script
Next, you’ll need to create a script to do the actual job described in the above diagram.
The Gitlab CI will pick up and execute the script when the contidition you predefined is met. 

My script does the following:
Download test cases from the documentation GitLab repo and move them to the specified folder in the VM.
Download and install the latest database version.
Run the tests and print the results.
Clean up the environment.

Step 3: Register a GitLab Runner
GitLab uses runners to execute CI/CD tasks in a given env defined in the .gitlab-ci.yml file in each repo. You define how the runner is triggered and what it would do in your test env.
Install GitLab Runner on the VM:
# Add the GitLab Runner repository
curl -L https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh | sudo bash

# Install GitLab Runner
sudo apt-get install gitlab-runner

Register the Runner:
Get a registration token from your GitLab project:
 Settings > CI/CD > Runners > Expand "Set up a specific Runner manually"
Run: sudo gitlab-runner register
And then follow the prompts to complete the registration:
GitLab instance URL: e.g., https://gitlab.com or your self-hosted instance


Registration token: from the UI above


Description: a name to identify the runner


Tags: optional


Executor: shell, docker, etc. (Use docker for isolation)


Docker image: if using Docker executor (e.g., alpine, python:3)



Step 4: Create a .gitlab-ci.yml File
This file defines your CI pipeline. Place it in the root of your documentation repo. YOu need to put the script you created in Step 2 to the `script` section in the yml file.
(code example)

By now, the pipeline has been built up. The next thing is to create test cases and get the pipeline run when DB is updated!

Write the Test Cases
pg_regress makes it simple to write tests. For each test case, it requires two files:
A .sql file that contains the SQL commands to be run


A .out file that contains the expected output
To verify the test cases:
Check regression.diffs to compare expected vs. actual output (watch out for extra spaces at line edges)


Inspect test output in the results folder


After a test case is created, you can just add them to the doc repo, the pipeline will automatically pick them up when triggered.

Wrap-up
At this point, the initial version of the test pipeline is set up. The last thing is to set up a proper trigger for the pipeline, for example, when the new DB version is out, the pipeline is triggered and also pull the URL to the new DB’s installation package.
This pipeline is far from perfect and could be improved in all aspects. It’s just a proof-of-concept of a development-styled test process for docs. When manual maintenance has become un-manageable, it would be worth a try to set up a such automation to help.
