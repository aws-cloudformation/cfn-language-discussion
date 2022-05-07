# CFN Language Discussion

## What is it?
This repo is a place to propose and discuss new language features for the CloudFormation template language. It is also a great place to request CloudFormation template language features, report CloudFormation template language bugs or any general discussion related to the CloudFormation template language.

Go to: [What is an RFC](#what-is-an-rfc) | [RFC Process](#rfc-process)

<!--BEGIN_TABLE-->
\#|Title|Owner|Status|
---|-----|-----|------|
[9](https://github.com/aws-cloudformation/cfn-language-discussion/issues/9)|[Adding looping functionality in CFN Template](https://github.com/aws-cloudformation/cfn-language-discussion/pull/19)|[@MalikAtalla-AWS](https://github.com/MalikAtalla-AWS)|✍️ review
[14](https://github.com/aws-cloudformation/cfn-language-discussion/issues/14)|[Convert Template Block to JSON String](https://github.com/aws-cloudformation/cfn-language-discussion/pull/15)|[@mluk-aws](https://github.com/mluk-aws)|✍️ review
[11](https://github.com/aws-cloudformation/cfn-language-discussion/issues/11)|[Allow Intrinsic Functions and Pseudo-Parameter References in DeletionPolicy and UpdateReplacePolicy](https://github.com/aws-cloudformation/cfn-language-discussion/pull/21)|[@MalikAtalla-AWS](https://github.com/MalikAtalla-AWS)|✍️ review
<!--END_TABLE-->

## What is an RFC?

An RFC (request for comment) is a text document proposing a change to the CloudFormation template language. RFCs will only contain customer-facing syntax and behavior, not implementation details, and while we welcome all customer feedback on the proposal, the CloudFormation team will have final authority on decisions around the proposal.

### When should I write an RFC?

You will need to draft an RFC when you propose a new language feature in the CloudFormation template language. Examples are:

   - New intrinsic functions
   - New parameter types
   - New functionality for an existing intrinsic funtion.

### Who should submit an RFC?

An RFC can be submitted by anyone. In most cases, RFCs are authored by CloudFormation team members, but contributors are more than welcome to submit RFCs as well. However, RFCs can be a bit formal and time-consuming, so we highly recommend first creating an issue to discuss a new language feature proposal at a high level before diving into writing a detailed RFC document for it.

## RFC Process

### 1. Tracking issue

Each RFC starts with a [tracking issue](.github/ISSUE_TEMPLATE/rfc_tracking_issue.md). This issue is the hub for conversation, community signal(+1s) and a unique identifier for the RFC. CloudFormation team members will comment on the issue to help clarify problems, stablize syntax and make sure no breaking changes are introduced. The goal of this discussion to make sure there is no major disagreenment on high-level proposal.

> Before creating any issue, please make sure there are no similar issues in issue list or RFCs in RFC table.

The tracking issue includes a checklist helps the RFC owner drive the RFC
throughout the process.

### 2. RFC Document

Now you can start drafting your RFC document itself.

Create a file under `RFCs/NNNN-my-feature` based on the template under
`0000-template.md`, where `NNNN` is your tracking issue number and `my-feature`
is descriptive. Follow the template which includes guidance on completing the RFC.

### 3. Feedback

Once you have your initial draft of your RFC ready, please submit as a pull
request and start collecting feedback. CloudFormation team will assign two reviewers within 5 business days.

### 4. Sign-off

Before you can merge your RFC, you will need both reviewers sign-off on the RFC.

Once signed off, reviewers will add `status/approved` label to the RFC pull request.


### 5. Final Comments Period

At this stage, you have reached consensus about the majority concerns/questions
brought up during review period. This is a period for the broader community to
get a chance to look into this RFC, which usually takes about a week. If no
major concerns are raised, your pull request is ready to be merged.

### 6. Implementation

Once CloudFormation Language team picks up the RFC and starts implementation, we will update the tracking issue accordingly
- Started implementation (label: `status/implementing`)

### 7. Complete

Once Implementation complete, CloudFormation Language team will change status label to `status/done` and close the tracking issue.

## Report bugs or suggest features in the CloudFormation Template Language

We welcome you to use the GitHub issue tracker to report bugs or suggest features.

When filing an issue, please check existing open or recently closed issues to make sure somebody else hasn't already reported the issue. 

Please use the [bug report template](.github/ISSUE_TEMPLATE/bug_report.md) for reporting bugs and the [feature request template](.github/ISSUE_TEMPLATE/feature_request.md) for requesting new features.

## Security

If you discover a potential security issue in this project we ask that you notify AWS/Amazon Security via our [vulnerability reporting page](http://aws.amazon.com/security/vulnerability-reporting/). Please do **not** create a public github issue.

## License

This project is licensed under the Apache-2.0 License.

---

## Credit
AWS CloudFormation Language Improvement RFC process owes its inspiration to [AWS CDK's RFC Process](https://github.com/aws/aws-cdk-rfcs)
