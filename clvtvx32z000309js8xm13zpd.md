---
title: "Semantics of good commit message"
datePublished: Sun May 05 2024 18:47:09 GMT+0000 (Coordinated Universal Time)
cuid: clvtvx32z000309js8xm13zpd
slug: a-case-for-good-commit-message
cover: https://cdn.hashnode.com/res/hashnode/image/stock/unsplash/8tem2WpFPhM/upload/2d68f06df313ae85d856021d506ee6c3.jpeg
tags: documentation, git, commit-messages

---

[Studies observed that 14% of commit messages in over 23,000 OSS projects were completely empty, 66% of the messages contained only a few words, and only 10% of commits had messages containing “normal” descriptive English sentences.](https://ieeexplore.ieee.org/document/6606588)

Common fallacies while writing commit messages are

* Underestimating the importance of commit messages and clean log history
    
* It is obvious to you and your team members (At the moment)
    
* You have already put in all the efforts to make the code changes. Now you lack the motivation to spend more time & effort.
    
* Vague and generic explanation. “Cleanup of x files”, “Minor changes to tests”, “Made changes to X files”.
    

The commit message should describe what changes the commit made to the behavior of the code and not what changed in the code. What changed in the code is apparent by looking at the diff. Explaining what changed, why those changes were made, and how it fit in the grand scheme of things is important for the reader to get the context.

Communication is key to collaborative software development and context is king. Context is built by recording the evolution of the software project, an audit log of commits. Every new commit message should sound like a good build-up to the existing story written with previous commit messages.

We can say that a commit message should address the following questions

1. What was changed in the codebase?
    
2. Why those changes were made?
    

We’ll look at some of the examples that may occur while writing commit messages.

### Code changes with direct co-relation to business outcome

* A change to save infra cost → More money to spend on the right things
    
* Built table stakes feature → Less customer churn
    
* Critical security fix → Reputation / Goodwill
    
* Performance improvements to meet service level agreements → Better UX → Happy customers
    

While working on such changes, the commit log should look like a story building towards achieving the expected business outcome.

### Department-specific code changes

In this case, you should mention the reason from the department's perspective. For example, the design team has decided to change colour of the warning message from X to Y. Commit message can describe the change and link to the design document.

### Tech debt items

Commits related to tech debt items should describe the developer's motivation. Such commits involve fixing edge cases, fixing warnings given by developer & build tools, and improving over a shortcoming of existing implementation. In such commits, the developer’s motivation to fix the issue is a vital part of providing the context.

### Building on top of existing work

While building features, and amending existing features, changes are made in relation to prior commits. Messages of such commits should establish the current state of the software, what is missing, and how new code changes fill that gap.

### Exceptions

When the rationale for making the change is trivial, it is okay to skip the why, and what part. Examples would be fixing a typo or indentation fix.

### Conclusion

In conclusion, focusing on what has changed and why the change has been made are important questions to answer while writing a commit message.