---
title: "A case for good commit message"
datePublished: Sun May 05 2024 18:47:09 GMT+0000 (Coordinated Universal Time)
cuid: clvtvx32z000309js8xm13zpd
slug: a-case-for-good-commit-message

---

## \[Writing in progress\]

Studies observed that 14% of commit messages in over 23,000 OSS projects were completely empty, 66% of the messages contained only a few words, and only 10% of commits had messages containing “normal” descriptive english sentences.

The commit message should describe what changes commit made to the behaviour of the code and not what changed in the code. What changed in the code is apparent by looking at the diff. Explaining what changed in the code and how it related to the expected outcome is essential for reader to understand the context.

Based on this principle, we can say that a commit message should address the following questions

1. What was changed in the codebase?
    
2. Why those changes were made?
    

A common fallacies while writing commit message are

* It is hard to explain
    
* No one reads commit log
    
* It is obvious to you and your team members (At the moment)
    
* You have already put in all the efforts to make the code changes. Now you lack motivation to spend more time & effort.
    
* It is trivial, I just want to get done with it. “Cleanup of x files”, “Minor changes to tests”, “Made changes to X files”. All these commit messages shows lack of accountability.
    

Communication is key to collaborative software development and context is king. Context is built by recording evolution of the software project, an audit log of commits. Every new commit message should sound like a good build-up on top of the story written with previous commit messages.

Sometimes there is a direct co-relation between code changes and expected business outcome.

A change to save infra cost → More money to spend on the right things

Built table stakes feature → Less customer churn

Critical security fix → Reputation / Goodwill

Performance improvements to meet service level agreements → Better UX → Happy customers

While working on such changes, commit log should look like a story building towards achieving expected business outcome.

Some changes are department specific. In this case, you should mention the reason from the lens of the department. For example, design team has decided to change colour for warning message from X to Y. Commit message can have a link to the design document explaining the change.

Commits that are related to tech debt items should describe the motivation. Such commits are often describe the error scenario, followed by options, followed by fix. Another category is fixing warnings given by different developer & build tools, Improving over a shortcoming of existing implementation. In such commits, developer’s motivation to fix the issue is vital part of providing the context.

Commits that are in relation to previous work

When rationale for making the change is trivial, it is okay to skip the why, and what part. Examples would be fixing a typo, indentation fix.

## \[Writing in progress\]