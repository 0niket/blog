---
title: "A case for good commit message"
datePublished: Sun May 05 2024 18:47:09 GMT+0000 (Coordinated Universal Time)
cuid: clvtvx32z000309js8xm13zpd
slug: a-case-for-good-commit-message

---

## \[Writing in progress\]

Commit messages usually express what has changed, but rarely explain the why.

Why?

Because it is hard to explain.

Because who reads commit logs?

Because it is obvious to you and your team members (At the moment)

Because you or co-pilot has written doc strings for new & updated functions

Because you are going to write high level documentation in confluence, right?

Because you have already put in all the efforts to make the code changes. And don’t want to exert more force.

Communication is key to collaborative software development and context is king for effective communication.

VCS already has a way to build up the context. In git, it is `git log` , a log of commits.

The commit message is the face of a commit.

Every new commit message should sound like a good build-up on top of the story written with previous commit messages. Instead of just writing what has changed, mention the reason for the change. The best way is to co-relate it with the business outcome.

A change to save infra cost → More money to spend on the right things

Built table stakes feature → Less customer churn

Critical security fix → Reputation / Goodwill

Performance improvements to meet service level agreements → Better UX → Happy customers

Having said that, not all commits directly impact the business outcome. In this case, you should reason about change from the lens of the department that requires that change. For example, design team has decided to change colour for warning message from X to Y followed by their reasoning to do so. It could be a link to the document.

But but.. you are piling up small commits to build up the feature.  
Each commit still contributes to the part of the story. Mention that. Mention the decisions you and your team have made. Don’t forget to answer the why question.

## \[Writing in progress\]