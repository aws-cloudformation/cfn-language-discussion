# Fn::NumberComparison

* **Original Author(s)**:: @minggon
* **Tracking Issue**: [Tracking Issue](https://github.com/aws-cloudformation/cfn-language-discussion/issues/80)

# Summary

We will create a numerical comparison function `Fn::NumberComparison` to support comparisons such as `“<”`, `“<=”`, `“==”`, `“>=”`, and `“>”` between numbers. The function would take in 3 parameters, the first number being compared, the comparison operator as a string, and the second number being compared, and output the boolean result of the comparison between the first and third parameter. 

# Motivation

CloudFormation users may want to do different things depending on the condition of a numeric variable. This RFC proposes to support the use case of comparing two numbers to create a boolean that can be used in the Fn::If intrinsic function. 

* Reference:
    * https://github.com/aws-cloudformation/cfn-language-discussion/issues/80

# Examples

## Example 1

Using `Fn::NumberComparison` in Conditions section to set resource properties

## YAML
```yaml
Parameters:
  stage:
    Description: Deployment stage
    Type: String
    Default: gamma
    AllowedValues:
      - gamma
      - prod
  retentionInDays:
    Description: Log retention in days
    Type: Number
Conditions:
  IsProd: !Equals
    - !Ref 'stage'
    - prod
  NotMeetRetentionRequirement: !NumberComparison
    - !Ref 'retentionInDays'
    - <=
    - 365
  ShouldEnforceLogRetention: !And
    - !Condition 'IsProd'
    - !Condition 'NotMeetRetentionRequirement'
Resources:
  cwLogGroup:
    Type: AWS::Logs::LogGroup
    Properties:
      LogGroupName: cwLogGroup
      RetentionInDays: !If
        - ShouldEnforceLogRetention
        - 365
        - !Ref 'retentionInDays'
```

## JSON 
```json
{
  "Parameters": {
    "stage": {
        "Description": "Deployment stage",
        "Type": "String",
        "Default": "gamma",
        "AllowedValues": [
          "gamma",
          "prod"
        ]
    },
    "retentionInDays": {
      "Description": "Log retention in days",
      "Type": "Number"
    }
  },
  "Conditions": {
    "IsProd": {
      "Fn::Equals": [
        { "Ref": "stage"},
        "prod"
      ]
    },
    "NotMeetRetentionRequirement": {
      "Fn::NumberComparison": [
        {"Ref": "retentionInDays"},
        "<=",
        365
      ]
    },
    "ShouldEnforceLogRetention": {
      "Fn::And": [
        {"Condition": "IsProd"},
        {"Condition": "NotMeetRetentionRequirement"}
      ]
    }
  },
  "Resources": {
    "cwLogGroup": {
      "Type" : "AWS::Logs::LogGroup",
      "Properties" : {
          "LogGroupName" : "cwLogGroup",
          "RetentionInDays" : {
            "Fn::If": [
              "ShouldEnforceLogRetention",
              365,
              {"Ref": "retentionInDays"}
            ]
          }
        }
    }
  }
}
```

# Limitation

The intrinsic function does not support reference to resource properties. It only supports references to Parameters and Mappings.

# Details

`Fn::NumberComparison` is an intrinsic function that takes a `Number`, a `String` with a comparison operator, and another `Number` and returns a `Boolean` representing the comparison between the first number and the second number. The number could be:

* a hardcoded `Number` in the template
* `Ref` to a Parameter of type `Number`
* a `Number` outputted by the use of a Function

The string could be:

* `"<"`
* `"<="`
* `"=="`
* `">"`
* `">="`

Supported Functions: 

* `Fn::Select`
* `Fn::Length`
* `Fn::Sub`
* `Fn::Join`
* `Fn::FindInMap`
* `Fn::If`

As this intrinsic function produces a `Boolean`, it can be used as the conditional argument in the `Fn::If` intrinsic function. `Fn::NumberComparison` can also be used outside the Conditions section to specify `Boolean` resource properties. 

# Alternatives considered

We considered creating several number comparison functions, i.e. `Fn::LessThan`, `Fn::GreaterThan`, etc. We settled on this approach as it improves readability, simplifies the development process, and allows users to set the comparator using a function. 

# Appendix

* https://github.com/aws-cloudformation/cfn-language-discussion/issues/80

