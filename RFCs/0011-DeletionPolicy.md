# Allow Intrinsic Functions and Pseudo-Parameter References in `DeletionPolicy` and  `UpdateReplacePolicy` Resource attributes

* **Original Author(s):**: @mingujo
* **Tracking Issue**: [Tracking Issue](https://github.com/aws-cloudformation/cfn-language-discussion/issues/11)

# Summary

Use CloudFormation [intrinsic functions](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference.html) and [Pseudo-Parameter references](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/pseudo-parameter-reference.html) to dynamically determine a value for DeletionPolicy and UpdateReplacePolicy Resource attributes

# Examples

### Use [Ref](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference-ref.html)
```json
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
                 "KeyType" : "HASH"
               } 
              ]
            },
            "DeletionPolicy": { "Ref": "DeletionPolicyParameter" },
            "UpdateReplacePolicy": { "Ref": "UpdateReplacePolicyParam" }
        }
    }
}
```

### Use [Conditions](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/conditions-section-structure.html) and [Fn::If](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference-conditions.html#intrinsic-function-reference-conditions-if) based on Parameters
```json
{
    "AWSTemplateFormatVersion" : "2010-09-09",
    "Parameters": {
        "Stage": {
            "Type": "String",
            "AllowedValues": ["Prod", "Staging", "Dev"]
        }
    },
    "Conditions": {
        "IsProd" : {"Fn::Equals" : [{"Ref" : "Stage"}, "Prod"]}
    },
    "Resources": {
        "Table": {
            "Type": "AWS::DynamoDB::Table",
            "Properties": {
              "KeySchema": [
               {
                 "AttributeName" : "primaryKey",
                 "KeyType" : "HASH"
               }
              ]
            },
            "DeletionPolicy": {
                "Fn::If": [ "IsProd", "Retain", "Delete" ]
            },
            "UpdateReplacePolicy": {
                "Fn::If": [ "IsProd", "Retain", "Delete" ]
            }
        }
    }
}
```

### Use [Fn::FindInMap](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference-findinmap.html) and `AWS::Region` Pseudo-Parameter
```json
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
                    { "Ref" : "AWS::Region" },
                    "DeletionPolicy"
                ]
            },
            "UpdateReplacePolicy": {
                "Fn::FindInMap" : [
                    "RegionMap",
                    { "Ref" : "AWS::Region" },
                    "UpdateReplacePolicy"
                ]
            }
        }
    }
}
```

# Motivation

CloudFormation customers expect to use conditions and intrinsic functions as values for `DeletionPolicy` and `UpdateReplacePolicy` Resource attributes as they can do elsewhere in the template.
The main ask from our customers was to allow them to vary DeletionPolicy and UpdateReplacePolicy values based on development stage, which tends to be passed in as a parameter, e.g., allow resource deletion in Dev stages but not Prod. This RFC proposes to support not only such use case but also other intrinsic function usages and Pseudo-parameter references.

* References
    * https://github.com/aws-cloudformation/cfn-language-discussion/issues/58 (149 thumbs up)
    * https://github.com/aws/aws-cli/issues/3825

# Details
* The following intrinsic functions will be supported.
    * [Ref](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference-ref.html)
    * [Fn::FindInMap](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference-findinmap.html)
    * [Fn::If](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference-conditions.html#intrinsic-function-reference-conditions-if)


* The following Pseudo-Parameters will be supported
    * `AWS::AccountId`, `AWS::Region`, `AWS::Partition`

# Limitation

**Note:** This limitation is due to underlying implementation constraints. In the fullness of time, this limitation should be removed.

Does not support expressions containing references to *[Resource properties](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-template-resource-type-ref.html)* or *certain pseudo parameters*
* Unsupported Pseudo-Parameters
    * `AWS::StackId`, `AWS::StackName`, `AWS::NotificationArns`, `AWS::URLSuffix`, `AWS::NoValue`


# FAQ
* How about other Resource attributes such as CreationPolicy, DependsOn, Metadata, UpdatePolicy?
    * They are out of scope of this RFC.


* Will this be supported by CFN Lint (linter check for CFN template)?
    * The AWS CloudFormation Linter (cfn-lint) will be updated to validate function and Pseudo-parameter usages under Resource attributes.

# Appendix
## More examples
### Use [Fn::FindInMap](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference-findinmap.html) and  `AWS::Partition`
```json
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
                 "KeyType" : "HASH"
               }
              ]
            },
            "DeletionPolicy": {
                "Fn::FindInMap" : [
                    "PartitionMap",
                    { "Ref" : "AWS::Partition" },
                    "DeletionPolicy"
                ]
            },
            "UpdateReplacePolicy": {
                "Fn::FindInMap" : [
                    "PartitionMap",
                    { "Ref" : "AWS::Partition" },
                    "UpdateReplacePolicy"
                ]
            }
        }
    }
}
```

### Use [Fn::FindInMap](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference-findinmap.html) and `AWS::Region` Pseudo-Parameter in YAML
```yaml
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
