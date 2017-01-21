---
layout: post
tags:
  - Software Engineering
---

Reviewing code becomes second-nature to most engineers that have worked a significant amount of time. Reviewers will catch minor bugs and stylistic issues, but code review slowly becomes a chore that’s necessary to deploy a change. Code review is actually a way to improve, and reviewing time can be used to better yourself in addition to the team. This is an ongoing list to do just that.

## Structuring a Team to Review

Segment your team into squads with each engineer having 1-2 principal reviewers. CC the whole team when the pull request is opened, but make sure the whole team does not receive notification spam when comments are made. This is the greatest balance I’ve found with increasing knowledge transfer and lowering review turnaround time. Having two principal reviewers for larger diffs will expose newer engineers to different styles and arguments.

## Things to Look for as a Reviewer

While the onus is on the reviewer to bugs and argue stylistic issues, the reviewer can get plenty out of a review.

### Review for Understanding

One of the first questions the reviewer should ask is: “Does the code satisfy the goals of the engineer and their system?” If the answer is no, then all the stylistic errors or bugs you can catch are going to be for naught.

Don’t feel bad about leaving a few or no comments if you’ve satisfied this most important goal.

### Modularization and Readability

How much sharing of code do you need, and what is the versioning story for when your component grows? There are right and wrong times to share. A common pattern when writing state machines is to define the state transitions in one versioned class while using shared hooks to be called before or after state transitions. This makes the code much more modular and clever, but at the cost of readability (not everyone might be a grepping wizard like you). Shared Examples in RSpec do a great job of DRYing up statements, but they incur both performance and readability penalties when running the tests themselves. Shared Examples look attractive, but are a pain to debug.

Effective code review, much like software engineering, is all about tradeoffs.

### The Beginner’s Mind

Approach the components from a newcomer’s mindset. Why is something done this way? Write questions on the pull request if you need additional context — not asking hurts both you and the team.

### Review Now

Get to pull requests early.

### Best Practices for Opening Pull Requests

The engineer can also do his/her part in making the code reviewable and clear.

### Small Diffs

The activation energy to review is directly proportional to the size of the diff. Splitting up a complicated pull request into individual libraries or functionalities will greatly improve turnaround times.

### Write a Description and Comments

Why is this change being made? Providing historical context will help others, especially newer engineers, stay on the same page. This is also effectively you picking your own brain to see whether or not your pull request accomplishes your intent.

Write comments on the pull request to clarify parts of the code for reviewers. A decision you made might be clear to you, but others will wonder how you arrived at it.
