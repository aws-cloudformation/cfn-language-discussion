# Allow Intrinsic Functions and Pseudo-Parameter References in `DeletionPolicy` and  `UpdateReplacePolicy` Resource attributes

* **Original Author(s):**: @mingujo
* **Tracking Issue**: [Tracking Issue](https://github.com/aws-cloudformation/cfn-language-discussion/issues/11)
* **Reviewer**:  @cfn-language-and-tools-team

# Summary

With Language Extensions, you can use CloudFormation [intrinsic functions](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference.html) and [Pseudo-Parameter references](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/pseudo-parameter-reference.html) to dynamically determine a value for DeletionPolicy and UpdateReplacePolicy Resource attributes

# Examples

### Use [Ref](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference-ref.html)
```
{
    "AWSTemplateFormatVersion" : "2010-09-09",
    "Parameters": {
        "DeletionPolicyParam": {
            "Type": "String",
            "AllowedValues": ["Delete", "Retain", "Snapshot"],
            "Default": "Delete"
        },
        "UpdateReplacePolicyParam": {
            "Type": "String",
            "AllowedValues": ["Delete", "Retain", "Snapshot"],
            "Default": "Delete"
        }
    },
    "Resources": {
        "Table": {
            "Type": "AWS::DynamoDB::Table",
            "Properties": {
              "KeySchema": [
               {
                 "AttributeName" : "primaryKey",
                 "KeyType" : HASH
               } 
              ]
            },
            "DeletionPolicy": { "Ref": "DeletionPolicyParameter" },
            "UpdateReplacePolicy": { "Ref": "UpdateReplacePolicyParam" }
        }
    }
}
```

### Use [Conditions](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/conditions-section-structure.html) and [Fn::If](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference-condition.html) based on Parameters
```
{
    "AWSTemplateFormatVersion" : "2010-09-09",
    "Parameters": {
        "Stage": {
            "Type": "String",
            "AllowedValues": ["Prod", "Staging", "Dev"]
        }
    },
    "Conditions": {
        "IsProd" : {"Fn::Equals" : [{"Ref" : "Stage"}, "Prod"]},
    },
    "Resources": {
        "Table": {
            "Type": "AWS::DynamoDB::Table",
            "Properties": {
              "KeySchema": [
               {
                 "AttributeName" : "primaryKey",
                 "KeyType" : HASH
               } 
              ]
            },
            "DeletionPolicy": { 
                "Fn::If": [
                    "IsProd",
                    "Retain",
                    "Delete"
                ]
            },
            "UpdateReplacePolicy": { 
                "Fn::If": [
                    "IsProd",
                    "Retain",
                    "Delete"
                ]
            }
        }
    }
}
```

### Use [Fn::FindInMap](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference-findinmap.html) and `AWS::Region` Pseudo-Parameter
```
{
    "AWSTemplateFormatVersion" : "2010-09-09",
    "Parameters": {
        "Stage": {
            "Type": "String",
            "AllowedValues": ["Prod", "Staging", "Dev"]
        }
    },
    "Mappings" : {
        "RegionMap" : {
            "us-east-1" : { 
                "DeletionPolicy": "Retain",
                "UpdateReplacePolicy": "Retain"
            },
            "us-west-1" : { 
                "DeletionPolicy": "Delete",
                "UpdateReplacePolicy": "Delete"
            },
            "eu-west-1" : { 
                "DeletionPolicy": "Retain",
                "UpdateReplacePolicy": "Delete"
            },
            "ap-southeast-1" : { 
                "DeletionPolicy": "Delete",
                "UpdateReplacePolicy": "Delete"
            }
        }
    },
    "Resources": {
        "Bucket": {
            "Type": "AWS::S3::Bucket",
            "DeletionPolicy": { 
                "Fn::FindInMap" : [
                    "RegionMap", 
                    {"Ref" : "AWS::Region" },
                    "DeletionPolicy"
                ]
            },
            "UpdateReplacePolicy": { 
                "Fn::FindInMap" : [
                    "RegionMap", 
                    {"Ref" : "AWS::Region" },
                    "UpdateReplacePolicy"
                ]
            }
        }
    }
}
```

# Motivation

CloudFormation customers expect to use conditions and intrinsic functions as a value for a `DeletionPolicy` and `UpdateReplacePolicy` Resource attributes as they can do elsewhere in the template. Main ask from our customers was to determine values for the policies using conditions based on Parameters. This RFC proposes to support not only such use case but also other intrinsic function usages and Pseudo-parameter references.

* References
    * https://github.com/aws-cloudformation/cloudformation-coverage-roadmap/issues/162 (148 thumbs up)
    * https://github.com/aws/aws-cli/issues/3825

# Details
* Following intrinsic functions will be supported.
    * [Ref](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference-ref.html)
    * [Fn::FindInMap](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference-findinmap.html)
    * [Fn::If](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference-conditions.html#intrinsic-function-reference-conditions-if)


* Following Pseudo-Parameters references will be supported
    * `AWS::AccountId`, `AWS::Region`, `AWS::Partition`, `AWS::NoValue`

# Limitation

If the expression contains references to *[Resource properties](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-template-resource-type-ref.html)* or *certain pseudo parameters*, Language Extensions will not be able to resolve the expression.

* Unsupported Pseudo-Parameter References
    * `AWS::StackId`, `AWS::StackName`, `AWS::NotificationArns`, `AWS::URLSuffix`


# FAQ
* How about other Resource attributes such as CreationPolicy, DependsOn, Metadata, UpdatePolicy?
    * They are out of scope for now. We will propose to support them in a separate RFC if there is enough demand from community.


* Will this be supported by CFN Lint (linter check for CFN template)?
    * The AWS CloudFormation Linter (cfn-lint) will be updated to validate function and Pseudo-parameter usages under Resource attributes.

# Appendix
## More examples
### Use [Fn::FindInMap](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference-findinmap.html) and  `AWS::Partition`
```
{
    "AWSTemplateFormatVersion" : "2010-09-09",
    "Parameters": {
        "Stage": {
            "Type": "String",
            "AllowedValues": ["Prod", "Staging", "Dev"]
        }
    },
    "Mappings" : {
        "PartitionMap" : {
            "aws" : { 
                "DeletionPolicy": "Retain",
                "UpdateReplacePolicy": "Retain"
            },
            "aws-cn" : { 
                "DeletionPolicy": "Delete",
                "UpdateReplacePolicy": "Delete"
            },
            "aws-us-gov" : { 
                "DeletionPolicy": "Retain",
                "UpdateReplacePolicy": "Delete"
            }
        }
    },
    "Resources": {
        "Table": {
            "Type": "AWS::DynamoDB::Table",
            "Properties": {
              "KeySchema": [
               {
                 "AttributeName" : "primaryKey",
                 "KeyType" : HASH
               } 
              ]
            },
            "DeletionPolicy": { 
                "Fn::FindInMap" : [
                    "PartitionMap", 
                    {"Ref" : "AWS::Partition"},
                    "DeletionPolicy"
                ]
            },
            "UpdateReplacePolicy": { 
                "Fn::FindInMap" : [
                    "PartitionMap", 
                    {"Ref" : "AWS::Partition" },
                    "UpdateReplacePolicy"
                ]
            }
        }
    }
}
```

### Use [Fn::FindInMap](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference-findinmap.html) and `AWS::Region` Pseudo-Parameter in YAML
```
AWSTemplateFormatVersion: "2010-09-09"
Parameters:
  Stage:
    Type: String
    AllowedValues:
      - Prod
      - Staging
      - Dev
Mappings: 
  RegionMap: 
    us-east-1:
      DeletionPolicy: Retain
      UpdateReplacePolicy: Retain
    us-west-1:
      DeletionPolicy: Delete
      UpdateReplacePolicy: Delete
    eu-west-1:
      DeletionPolicy: Retain
      UpdateReplacePolicy: Delete
    ap-northeast-1:
      DeletionPolicy: Delete
      UpdateReplacePolicy: Delete
Resources: 
  Bucket: 
    Type: "AWS::S3::Bucket"
    DeletionPolicy: !FindInMap [RegionMap, !Ref "AWS::Region", DeletionPolicy]
    UpdateReplacePolicy: !FindInMap [RegionMap, !Ref "AWS::Region", DeletionPolicy]
```

Use `AWS::NoValue`
```
{
    "AWSTemplateFormatVersion" : "2010-09-09",
    "Parameters": {
        "Stage": {
            "Type": "String",
            "AllowedValues": ["Prod", "Staging", "Dev"]
        }
    },
    "Conditions": {
        "IsProd" : {"Fn::Equals" : [{"Ref" : "Stage"}, "Prod"]},
    },
    "Resources": {
        "Bucket": {
            "Type": "AWS::S3::Bucket",
            "DeletionPolicy": { 
                "Fn::If": [
                    "IsProd",
                    "Retain",
                    {"Ref": "AWS::NoValue"}
                ]
            }
        }
    }
}
```