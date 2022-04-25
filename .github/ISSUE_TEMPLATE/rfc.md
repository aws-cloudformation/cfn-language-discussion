---
name: RFC
about: Language Enhancement Request For Comment
title: ''
labels: ''
assignees: ''

---

To start an RFC process, RFC author will create a tracking issue and follow the 
instruction in tracking issue. It includes a checklist of the various stages 
an RFC goes through.

## 1. Tracking issue

Each RFC starts with a tracking issue. This issue is the hub for conversation, 
community signal(+1s) and a unique identifier for the RFC.

> Before creating any issue, please make sure there are no similar issues in issue list or RFCs in RFC table.

The tracking issue includes a checklist helps the RFC owner drive the RFC 
through out the process.


## 2. Reviewer

Reach out to [cfn-language-discussion#](cfn-language-discussion@amazon.com) to 
get **TWO** reviewers.

For each RFC, CloudFormation leadership will assign 2 reviewers who will review 
and approve the CloudFormation language RFC. An RFC will not be considered approved 
unless both reviewers approve.

## 3. Kick off

Before diving into writing the RFC, it is highly recommended to have a kick-off 
meeting that include reviewers and some stakeholders who might be interested in 
this RFC or can contribute ideas and direction. The goal of this meeting is to 
have a preliminary discussion of the proposal: does it have breaking changes? 
What is the scope of this proposal? Can this new feature be part of an existing RFC?


## 4. RFC Document

Now you can start drafting your RFC document itself.

Create a file under `text/NNNN-my-feature` based on the template under 
`0000-template.md`, where `NNNN` is your tracking issue number and `my-feature` 
is descriptive. Follow the template which includes guidance on completing the RFC.

## 5. Feedback

Once you have your initial draft of your RFC ready, please submit as a pull 
request and start collecting feedback.

Contact [#cfn-language-discussion](cfn-language-discussion@amazon.com) and 
reach out to the public via various channels, Twitter and other relevant forums. 
Also make sure RFC reviewers assigned are notified so they can provide their feedback.

## 6. Sign-off

Before you can merge your RFC, you will need both reviewers sign-off on the RFC. 

Once signed off, reviewers will add `approved` label to the RFC pull request.


## 7. Final Comments Period

At this stage, you have reached consensus about the majority concerns/questions 
brought up during review period. This is a period for the broader community to 
get a chance to look into this RFC, which usually takes about a week. If no 
major concerns are raised, your pull request is ready to be merged.

## 8. Execution

It is highly recommended to have an execution plan which list all the tasks required. 
Breaking down a large tasks into multiple smallish tasks helps prioritization. 

The execution plan can be submitted through a separate pull request as an 
appendix to the RFC document. You can ping the reviewers of the RFC to review 
the execution plan.

During execution, update the tracking issue accordingly.
- Execution plan submitted (label: `status/planning`)
- Execution plan approved and merged (label: `status/executing`)
- Execution plan compelted (label: `status/complete`)

## 9. Complete

Once execution complete, change status label to `status/complete` and close the tracking issue.
