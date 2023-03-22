# Fn::ForEach intrinsic function to replicate similar CFN template fragments

* **Original Author(s):**: @arthurboghossian
* **Tracking Issue**: [Tracking Issue: Fn::ForEach](https://github.com/aws-cloudformation/cfn-language-discussion/issues/9)

## Summary

AWS CloudFormation is introducing a new `Fn::ForEach` intrinsic function. Customers can use this new feature to reduce verbosity of their CloudFormation templates, and improve their readability. This feature allows customers to declare multiple instances of similar resource type (ex. Subnets, VPCs) with a few lines of code. Customers can use this feature in Conditions, Resource, and Output sections of their template. Customers can declare any existing and future intrinsic functions such as `Fn::If`, `Fn::Join`, and more in `Fn::ForEach` intrinsic function. See [Examples](#examples) section for detailed use cases of `Fn::ForEach`, and sample templates with and without `Fn::ForEach` intrinsic function.

## Motivation

In a CloudFormation template, a single resource configures into one infrastructure object. A template, therefore, can become verbose when a customer manually declares similar resources. For example, customers want to declare a pool of `AWS::EC2::Instance` with the same configurations but different instance types, or `AWS::S3::Bucket NotificationConfiguration` with different `AWS::SNS::Topic`. Today, customers have to copy/paste the same lines of code with minor differences in resource properties.

## Solution

Customers can declare `Fn::ForEach` to iterate over a list of collections to generate desired fragments (ex. Resources).

### JSON

```yaml
{
    "Fn::ForEach::<UniqueLoopName>": [
        "Identifier",
        ["Value1", "Value2"], ## Collection
        {
            "OutputKey": "OutputValue" ## Ex: {"OutputKeyName${Identifier}": "OutputValue"}
            ## Multiple key-value pairs can be specified within this object
        }
    ]
}
```

### YAML

```yaml
'Fn::ForEach::<UniqueLoopName>':
  - Identifier
  - [Value1, Value2]  ## Collection
  - 'OutputKey': OutputValue  ## Ex: 'OutputKeyName${Identifier}': 'OutputValue'
    ## Multiple key-value pairs can be specified within this object
```

### Parameters

`Identifier` (String) → Identifier is used to refer to the current element we’re iterating over within the Collection (Array of Strings). Identifier can be used with Ref intrinsic function within OutputKey and OutputValue.

`Collection` (Array of Strings) → Array of values that the Identifier can take. Each element within a Collection when defined in OutputKey and OutputValue will be resolved and merged to the parent object.

`OutputKey` (String) →  The key of the resulting key-value pair for the given element in the collection that will be merged to the parent object. ${Identifier} must be included within the OutputKey. This ensures the resulting key is unique for each element in the collection. 

`OutputValue` (Any) → The value of the resulting key-value pair for the given element in the collection that will be merged to the parent object.

Note: the syntax of `Fn::ForEach` declaration has a suffix where the `UniqueLoopName` is used to identify the loop. This allows multiple `Fn::ForEach` function references to be declared on a given level.

## FAQ

### Which sections of a CloudFormation template can customers use Fn::ForEach?

`Fn::ForEach` intrinsic function can be used in the Resource, Conditions, Outputs, and Resource properties sections.

At launch, `Fn::ForEach` cannot be used it within the AWSTemplateFormatVersion, Description, Metadata, Transform, Parameters, Mappings, Rules, and Hooks sections.

### Which intrinsic functions are supported for Fn::ForEach?

All intrinsic functions (both current and future) are supported within the Fn::ForEach intrinsic function, which includes the following functions:

* Condition Functions (Fn::If, Fn::Equals, Fn::Not, Fn::And, and Fn::Or)
* Fn::Base64
* Fn::FindInMap
* Fn::GetAtt
* Fn::GetAZs
* Fn::ImportValue
* Fn::Join
* Fn::Length
* Fn::Select
* Fn::Sub
* Fn::ToJsonString
* Ref

### Can customers use Parameters to refer inputs to collections in Fn::ForEach?

Yes, customers can create a CommaDelimitedList parameter to refer as inputs for their Fn::ForEach function. This allows customers to reuse same list of strings across their template with ease.

```yaml
AWSTemplateFormatVersion: 2010-09-09
Transform: 'AWS::LanguageExtensions'
Parameters:
  InstanceList:
    Type: CommaDelimitedList
    Default: "InstanceA,InstanceB,InstanceC"
Resources:
  'Fn::ForEach::Instances':
    - LogicalId
    - !Ref InstanceList
    - '${LogicalId}':
        Type: AWS::EC2::Instance
        Properties:
          InstanceType: m5.xlarge
          ImageId: ami-id-default
          DisableApiTermination: true
```

### Do customers have to input customized Logical Ids for resources in Fn::ForEach?

Yes, customers are required to customize Names (Logical Id’s) of Resources, Conditions, and Outputs, and Property Names of Properties at the top-level key of the 3rd parameter of `Fn::ForEach`.  `Fn::ForEach` will treat this as an implicit `Fn::Sub`. Customers must make sure the resulting output keys are unique and alpha-numeric within their template. 

```yaml
AWSTemplateFormatVersion: 2010-09-09
Transform: 'AWS::LanguageExtensions'
Resources:
  'Fn::ForEach::Instances':
    - Identifier
    - [A, B, C]
    - 'Instance${Identifier}':
        Type: AWS::EC2::Instance
        Properties:
          InstanceType: m5.xlarge
```

Note: In the example above, the top-level key of the 3rd parameter is ‘Instance${Identifier}’. This results in Output Keys values ‘InstanceA’, ‘InstanceB’, and ‘InstanceC’.

Caveat: Only Strings at the top-level (OutputKey) will be treated as an implicit Fn::Sub, where only identifiers can be resolved. If a Parameter or Resource logical ID is passed within the OutputKey as an implicit Fn::Sub, the stack/changeset update/creation would fail

### Can customers use multiple nested Fn::ForEach?

Yes, customers can use up to 5 nested Fn::ForEach within a single parent loop. Customer must ensure the value of each loop’s Identifier field is unique, and referenced within the OutputKey to prevent key collisions and validation errors.

```yaml
AWSTemplateFormatVersion: 2010-09-09
Transform: 'AWS::LanguageExtensions'
Resources:
  VPC:
    Type: 'AWS::EC2::VPC'
    Properties:
      CidrBlock: 10.0.0.0/16
      EnableDnsSupport: 'true'
      EnableDnsHostnames: 'true'
  'Fn::ForEach::SubnetResources':
    - Prefix
    - [Transit, Public]
    - 'Nacl${Prefix}Subnet':
        Type: 'AWS::EC2::NetworkAcl'
        Properties:
          VpcId: !Ref VPC
      'Fn::ForEach::LoopInner':
        - Suffix
        - [A, B, C]
        - '${Prefix}Subnet${Suffix}':
            Type: 'AWS::EC2::Subnet'
            Properties:
              VpcId: !Ref VPC
          'Nacl${Prefix}Subnet${Suffix}Association':
            Type: 'AWS::EC2::SubnetNetworkAclAssociation'
            Properties:
              SubnetId: !Ref 
                'Fn::Sub': '${Prefix}Subnet${Suffix}'
              NetworkAclId: !Ref 
                'Fn::Sub': 'Nacl${Prefix}Subnet'
```

### Does CloudFormation service quota limits apply to Fn::ForEach?

Yes, customers are limited to CloudFormation service quota limits such as number of resources per stack, size of a template, and others. When using `Fn::ForEach`, customers must ensure that the final processed template has thresholds below quota limits for successful deployments. See our [user guide](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cloudformation-limits.html) for CloudFormation service quota limits.

### Why is the name of the intrinsic function “Fn::ForEach”?

CloudFormation considered other syntax names for this intrinsic function such as  `Fn::Map`, `Fn::Repeat`, and `Fn::Build`. Based on our research and community feedback, we have decided to choose `Fn::ForEach` as the looping function name. `Fn::ForEach` is easy to understand for non-programmers and first-time cloud customers. Additionally, for advanced CloudFormation customers `Fn::ForEach` represents that for each element in a given collection, a given template fragment would be replicated. For example, customers can replicate a collection [“Transit”, “Public”] to generate two fragments of `AWS::EC2::SubnetNetworkAclAssociation` resource type. See [Appendix A](#appendix-a-pros-and-cons-matrix-for-looping-function-names) for a pros and cons matrix of these function names.

### Can I use an Object for the Collection instead of a List?

This feature will not be available in the initial release (see [Potential Follow-up Features](#potential-follow-up-features)). If there's enough ask for this feature, then it can be added in the future (see [GitHub issue](https://github.com/aws-cloudformation/cfn-language-discussion/issues/118) with potential future enhancements to the `Fn::ForEach` intrinsic function).

### Can I output a List instead of a merging an Object to its parent object?

This feature will not be available in the initial release (see [Potential Follow-up Features](#potential-follow-up-features)). If there's enough ask for this feature, then it can be added in the future (see [GitHub issue](https://github.com/aws-cloudformation/cfn-language-discussion/issues/118) with potential future enhancements to the `Fn::ForEach` intrinsic function).

### Can I use Modules within Fn::ForEach?

This feature will not be available in the initial release (see [Potential Follow-up Features](#potential-follow-up-features)). If there's enough ask for this feature, then it can be added in the future (see [GitHub issue](https://github.com/aws-cloudformation/cfn-language-discussion/issues/118) with potential future enhancements to the `Fn::ForEach` intrinsic function).

## Examples

Here are examples of how customers can use `Fn::ForEach` in their CloudFormation templates.

### Fn::ForEach in Resources section

#### Use case 1: Replicate a single Resource

In this example, customer is creating four `AWS::DynamoDB::Table` with Logical IDs such as DynamoDBPoints, DynamoBDScore, and other.

#### JSON

```json
{
    "AWSTemplateFormatVersion": "2010-09-09",
    "Transform": "AWS::LanguageExtensions",
    "Resources": {
        "Fn::ForEach::Tables": [
            "TableName",
            ["Points", "Score", "Name", "Leaderboard"],
            {
                "DynamoDB${TableName}": {
                    "Type": "AWS::DynamoDB::Table",
                    "Properties": {
                        "TableName": {
                            "Ref": "TableName"
                        },
                        "AttributeDefinitions": [
                            {
                                "AttributeName": "id",
                                "AttributeType": "S"
                            }
                        ],
                        "KeySchema": [
                            {
                                "AttributeName": "id",
                                "KeyType": "HASH"
                            }
                        ],
                        "ProvisionedThroughput": {
                            "ReadCapacityUnits": "5",
                            "WriteCapacityUnits": "5"
                        }
                    }
                }
            }
        ]
    }
}
```

#### YAML

```yaml
AWSTemplateFormatVersion: 2010-09-09
Transform: 'AWS::LanguageExtensions'
Resources:
  'Fn::ForEach::Tables':
    - TableName
    - [Points, Score, Name, Leaderboard]
    - 'DynamoDB${TableName}':
        Type: 'AWS::DynamoDB::Table'
        Properties:
          TableName: !Ref TableName
          AttributeDefinitions:
            - AttributeName: id
              AttributeType: S
          KeySchema:
            - AttributeName: id
              KeyType: HASH
          ProvisionedThroughput:
            ReadCapacityUnits: '5'
            WriteCapacityUnits: '5'
```

#### The above templates would be equivalent to the below template in YAML

```yaml
AWSTemplateFormatVersion: 2010-09-09
Transform: 'AWS::LanguageExtensions'
Resources:
  DynamoDBPoints:
    Type: 'AWS::DynamoDB::Table'
    Properties:
      TableName: Points
      AttributeDefinitions:
        - AttributeName: id
          AttributeType: S
      KeySchema:
        - AttributeName: id
          KeyType: HASH
      ProvisionedThroughput:
        ReadCapacityUnits: '5'
        WriteCapacityUnits: '5'
  DynamoDBScore:
    Type: 'AWS::DynamoDB::Table'
    Properties:
      TableName: Score
      AttributeDefinitions:
        - AttributeName: id
          AttributeType: S
      KeySchema:
        - AttributeName: id
          KeyType: HASH
      ProvisionedThroughput:
        ReadCapacityUnits: '5'
        WriteCapacityUnits: '5'
  DynamoDBName:
    Type: 'AWS::DynamoDB::Table'
    Properties:
      TableName: Name
      AttributeDefinitions:
        - AttributeName: id
          AttributeType: S
      KeySchema:
        - AttributeName: id
          KeyType: HASH
      ProvisionedThroughput:
        ReadCapacityUnits: '5'
        WriteCapacityUnits: '5'
  DynamoDBLeaderboard:
    Type: 'AWS::DynamoDB::Table'
    Properties:
      TableName: Leaderboard
      AttributeDefinitions:
        - AttributeName: id
          AttributeType: S
      KeySchema:
        - AttributeName: id
          KeyType: HASH
      ProvisionedThroughput:
        ReadCapacityUnits: '5'
        WriteCapacityUnits: '5'
```

#### Use case 2: Replicate multiple types of Resources within a single Fn::ForEach function

In this example, customer is creating multiple instances of `AWS::EC2::NatGateway` and `AWS::EC2::EIP` while following a naming convention such as {ResourceType}${Identifier}. Customers can declare multiple resource types under one `Fn::ForEach` loop to take advantage of a single identifier.

Note: The example below assumes the “TwoNatGateways” and “ThreeNatGateways” Conditions exist, and “PublicSubnetA”, “PublicSubnetB”, and “PublicSubnetC” Resources are defined.

Note: Unique values for each element in the Collection are defined within the Mappings section, where the `Fn::FindInMap` intrinsic function is used to reference the corresponding value. If `Fn::FindInMap`  is unable to find the corresponding identifier, the Condition property will not be set resolving to `!Ref ‘AWS:::NoValue’`.

#### JSON

```json
{
    "AWSTemplateFormatVersion": "2010-09-09",
    "Transform": "AWS::LanguageExtensions",
    "Mappings": {
        "NatGateway": {
            "Condition": {
                "B": "TwoNatGateways",
                "C": "ThreeNatGateways"
            }
        }
    },
    "Resources": {
        "Fn::ForEach::NatGatewayAndEIP": [
            "Identifier",
            ["A", "B", "C"],
            {
                "NatGateway${Identifier}": {
                    "Type": "AWS::EC2::NatGateway",
                    "Properties": {
                        "AllocationId": {
                            "Fn::GetAtt": [
                                {
                                    "Fn::Sub": "NatGatewayAttachment${Identifier}"
                                },
                                "AllocationId"
                            ]
                        },
                        "SubnetId": {
                            "Ref": {
                                "Fn::Sub": "PublicSubnet${Identifier}"
                            }
                        }
                    },
                    "Condition": {
                        "Fn::FindInMap": ["NatGateway", "Condition", {"Ref": "Identifier"}, {"DefaultValue": {"Ref": "AWS::NoValue"}}]
                    }
                },
                "NatGatewayAttachment${Identifier}": {
                    "Type": "AWS::EC2::EIP",
                    "Properties": {
                        "Domain": "vpc"
                    },
                    "Condition": {
                        "Fn::FindInMap": ["NatGateway", "Condition", {"Ref": "Identifier"}, {"DefaultValue": {"Ref": "AWS::NoValue"}}]
                    }
                }
            }
        ]
    }
}
```

#### YAML

```yaml
AWSTemplateFormatVersion: 2010-09-09
Transform: 'AWS::LanguageExtensions'
Mappings:
  NatGateway:
    Condition: 
      B: TwoNatGateways
      C: ThreeNatGateways
Resources:
  'Fn::ForEach::NatGatewayAndEIP':
    - Identifier
    - [A, B, C]
    - 'NatGateway${Identifier}':
        Type: 'AWS::EC2::NatGateway'
        Properties:
          AllocationId: !GetAtt 
            - !Sub 'NatGatewayAttachment${Identifier}'
            - AllocationId
          SubnetId: !Ref 
            'Fn::Sub': 'PublicSubnet${Identifier}'
        Condition: !FindInMap [NatGateway, Condition, !Ref Identifier, DefaultValue: !Ref 'AWS::NoValue']
      'NatGatewayAttachment${Identifier}':
        Type: 'AWS::EC2::EIP'
        Properties:
          Domain: vpc
        Condition: !FindInMap [NatGateway, Condition, !Ref Identifier, DefaultValue: !Ref 'AWS::NoValue']
```

#### The above templates would be equivalent to the below template in YAML

```yaml
AWSTemplateFormatVersion: 2010-09-09
Transform: 'AWS::LanguageExtensions'
Resources:
  NatGatewayA:
    Type: 'AWS::EC2::NatGateway'
    Properties:
      AllocationId: !GetAtt 
        - NatGatewayAttachmentA
        - AllocationId
      SubnetId: !Ref 'PublicSubnetA'
  NatGatewayB:
    Type: 'AWS::EC2::NatGateway'
    Properties:
      AllocationId: !GetAtt 
        - NatGatewayAttachmentB
        - AllocationId
      SubnetId: !Ref 'PublicSubnetB'
    Condition: TwoNatGateways
  NatGatewayC:
    Type: 'AWS::EC2::NatGateway'
    Properties:
      AllocationId: !GetAtt 
        - NatGatewayAttachmentC
        - AllocationId
      SubnetId: !Ref 'PublicSubnetC'
    Condition: ThreeNatGateways
  NatGatewayAttachmentA:
    Type: 'AWS::EC2::EIP'
    Properties:
      Domain: vpc
  NatGatewayAttachmentB:
    Type: 'AWS::EC2::EIP'
    Properties:
      Domain: vpc
    Condition: TwoNatGateways
  NatGatewayAttachmentC:
    Type: 'AWS::EC2::EIP'
    Properties:
      Domain: vpc
    Condition: ThreeNatGateways
```

#### Use case 3: Replicate multiple Resources using a nested Fn::ForEach

In this example, customer is declaring two nested `Fn::ForEach` loop to map three resources (`AWS::EC2::NetworkAcl`, `AWS::EC2::Subnet`, `AWS::EC2::SubnetNetworkAclAssociation`) with each other.

#### JSON

```json
{
    "AWSTemplateFormatVersion": "2010-09-09",
    "Transform": "AWS::LanguageExtensions",
    "Resources": {
        "VPC": {
            "Type" : "AWS::EC2::VPC",
            "Properties" : {
               "CidrBlock" : "10.0.0.0/16",
               "EnableDnsSupport" : "true",
               "EnableDnsHostnames" : "true"
            }
        },
        "Fn::ForEach::SubnetResources": [
            "Prefix",
            ["Transit", "Public"],
            {
                "Nacl${Prefix}Subnet": {
                    "Type": "AWS::EC2::NetworkAcl",
                    "Properties": {
                        "VpcId": {"Ref": "VPC"}
                    }
                },
                "Fn::ForEach::LoopInner": [
                    "Suffix",
                    ["A", "B", "C"],
                    {
                        "${Prefix}Subnet${Suffix}": {
                            "Type": "AWS::EC2::Subnet",
                            "Properties": {
                                "VpcId": {"Ref": "VPC"}
                            }
                        },
                        "Nacl${Prefix}Subnet${Suffix}Association": {
                            "Type": "AWS::EC2::SubnetNetworkAclAssociation",
                            "Properties": {
                                "SubnetId": {
                                    "Ref": {
                                        "Fn::Sub": "${Prefix}Subnet${Suffix}"
                                    }
                                },
                                "NetworkAclId": {
                                    "Ref": {
                                        "Fn::Sub": "Nacl${Prefix}Subnet"
                                    }
                                }
                            }
                        }
                    }
                ]
            }
        ]
    }
}
```

#### YAML

```yaml
AWSTemplateFormatVersion: 2010-09-09
Transform: 'AWS::LanguageExtensions'
Resources:
  VPC:
    Type: 'AWS::EC2::VPC'
    Properties:
      CidrBlock: 10.0.0.0/16
      EnableDnsSupport: 'true'
      EnableDnsHostnames: 'true'
  'Fn::ForEach::SubnetResources':
    - Prefix
    - [Transit, Public]
    - 'Nacl${Prefix}Subnet':
        Type: 'AWS::EC2::NetworkAcl'
        Properties:
          VpcId: !Ref VPC
      'Fn::ForEach::LoopInner':
        - Suffix
        - [A, B, C]
        - '${Prefix}Subnet${Suffix}':
            Type: 'AWS::EC2::Subnet'
            Properties:
              VpcId: !Ref VPC
          'Nacl${Prefix}Subnet${Suffix}Association':
            Type: 'AWS::EC2::SubnetNetworkAclAssociation'
            Properties:
              SubnetId: !Ref 
                'Fn::Sub': '${Prefix}Subnet${Suffix}'
              NetworkAclId: !Ref 
                'Fn::Sub': 'Nacl${Prefix}Subnet'
```

#### The above templates would be equivalent to the below template in YAML

```yaml
AWSTemplateFormatVersion: 2010-09-09
Transform: 'AWS::LanguageExtensions'
Resources:
  VPC:
    Type: 'AWS::EC2::VPC'
    Properties:
      CidrBlock: 10.0.0.0/16
      EnableDnsSupport: 'true'
      EnableDnsHostnames: 'true'
  NaclTransitSubnet:
    Type: 'AWS::EC2::NetworkAcl'
    Properties:
      VpcId: !Ref VPC
  TransitSubnetA:
    Type: 'AWS::EC2::Subnet'
    Properties:
      VpcId: !Ref VPC
  NaclTransitSubnetAAssociation:
    Type: 'AWS::EC2::SubnetNetworkAclAssociation'
    Properties:
      SubnetId: !Ref TransitSubnetA
      NetworkAclId: !Ref NaclTransitSubnet
  TransitSubnetB:
    Type: 'AWS::EC2::Subnet'
    Properties:
      VpcId: !Ref VPC
  NaclTransitSubnetBAssociation:
    Type: 'AWS::EC2::SubnetNetworkAclAssociation'
    Properties:
      SubnetId: !Ref TransitSubnetB
      NetworkAclId: !Ref NaclTransitSubnet
  TransitSubnetC:
    Type: 'AWS::EC2::Subnet'
    Properties:
      VpcId: !Ref VPC
  NaclTransitSubnetCAssociation:
    Type: 'AWS::EC2::SubnetNetworkAclAssociation'
    Properties:
      SubnetId: !Ref TransitSubnetC
      NetworkAclId: !Ref NaclTransitSubnet
  NaclPublicSubnet:
    Type: 'AWS::EC2::NetworkAcl'
    Properties:
      VpcId: !Ref VPC
  PublicSubnetA:
    Type: 'AWS::EC2::Subnet'
    Properties:
      VpcId: !Ref VPC
  NaclPublicSubnetAAssociation:
    Type: 'AWS::EC2::SubnetNetworkAclAssociation'
    Properties:
      SubnetId: !Ref PublicSubnetA
      NetworkAclId: !Ref NaclPublicSubnet
  PublicSubnetB:
    Type: 'AWS::EC2::Subnet'
    Properties:
      VpcId: !Ref VPC
  NaclPublicSubnetBAssociation:
    Type: 'AWS::EC2::SubnetNetworkAclAssociation'
    Properties:
      SubnetId: !Ref PublicSubnetB
      NetworkAclId: !Ref NaclPublicSubnet
  PublicSubnetC:
    Type: 'AWS::EC2::Subnet'
    Properties:
      VpcId: !Ref VPC
  NaclPublicSubnetCAssociation:
    Type: 'AWS::EC2::SubnetNetworkAclAssociation'
    Properties:
      SubnetId: !Ref PublicSubnetC
      NetworkAclId: !Ref NaclPublicSubnet
```

### Fn::ForEach in Properties section

#### Use case 4: Replicate similar properties

In this example, the customer is using Fn::ForEach to duplicate the same inputs to `AWS::EC2::Instance` properties such as ImageId, InstanceType, and AvailabilityZone.

#### JSON

```json
{
    "AWSTemplateFormatVersion": "2010-09-09",
    "Transform": "AWS::LanguageExtensions",
    "Mappings": {
        "InstanceA": {
            "Properties": {
                "ImageId": "ami-id1",
                "InstanceType": "m5.xlarge"
            }
        },
        "InstanceB": {
            "Properties": {
                "ImageId": "ami-id2"
            }
        },
        "InstanceC": {
            "Properties": {
                "ImageId": "ami-id3",
                "InstanceType": "m5.2xlarge",
                "AvailabilityZone": "us-east-1a"
            }
        }
    },
    "Resources": {
        "Fn::ForEach::Instances": [
            "InstanceLogicalId",
            ["InstanceA", "InstanceB", "InstanceC"],
            {
                "${InstanceLogicalId}": {
                    "Type": "AWS::EC2::Instance",
                    "Properties": {
                        "DisableApiTermination": true,
                        "UserData": {
                            "Fn::Base64": {
                                "Fn::Join": ["", [
                                    "#!/bin/bash\n",
                                    "yum update -y\n",
                                    "yum install -y httpd.x86_64\n",
                                    "systemctl start httpd.service\n",
                                    "systemctl enable httpd.service\n",
                                    "echo \"Hello World from $(hostname -f)\" > /var/www/html/index.html\n"
                                ]]
                            }
                        },
                        "Fn::ForEach::Properties": [
                            "PropertyName",
                            ["ImageId", "InstanceType", "AvailabilityZone"],
                            {
                                "${PropertyName}": {"Fn::FindInMap": [{"Ref": "InstanceLogicalId"}, "Properties", {"Ref": "PropertyName"}, {"DefaultValue": {"Ref": "AWS::NoValue"}}]}
                            }
                        ]
                    }
                }
            }
        ]
    }
}
```

#### YAML

```yaml
AWSTemplateFormatVersion: 2010-09-09
Transform: 'AWS::LanguageExtensions'
Mappings:
  InstanceA:
    Properties:
      ImageId: ami-id1
      InstanceType: m5.xlarge
  InstanceB:
    Properties:
      ImageId: ami-id2
  InstanceC:
    Properties:
      ImageId: ami-id3
      InstanceType: m5.2xlarge
      AvailabilityZone: us-east-1a
Resources:
  'Fn::ForEach::Instances':
    - InstanceLogicalId
    - [InstanceA, InstanceB, InstanceC]
    - '${InstanceLogicalId}':
        Type: 'AWS::EC2::Instance'
        Properties:
          DisableApiTermination: true
          UserData:
            Fn::Base64: 
              !Sub |
                #!/bin/bash
                yum update -y
                yum install -y httpd.x86_64
                systemctl start httpd.service
                systemctl enable httpd.service
                echo "Hello World from $(hostname -f)" > /var/www/html/index.html
          'Fn::ForEach::Properties':
            - PropertyName
            - [ImageId, InstanceType, AvailabilityZone]
            - '${PropertyName}': !FindInMap [!Ref InstanceLogicalId, Properties, !Ref PropertyName, DefaultValue: !Ref 'AWS::NoValue']
```

#### The above templates would be equivalent to the below template in YAML

```yaml
AWSTemplateFormatVersion: 2010-09-09
Transform: 'AWS::LanguageExtensions'
Resources:
  InstanceA:
    Type: 'AWS::EC2::Instance'
    Properties:
      DisableApiTermination: true
      UserData:
        Fn::Base64: 
          !Sub |
            #!/bin/bash
            yum update -y
            yum install -y httpd.x86_64
            systemctl start httpd.service
            systemctl enable httpd.service
            echo "Hello World from $(hostname -f)" > /var/www/html/index.html
      ImageId: ami-id1
      InstanceType: m5.xlarge
  InstanceB:
    Type: 'AWS::EC2::Instance'
    Properties:
      DisableApiTermination: true
      UserData:
        Fn::Base64: 
          !Sub |
            #!/bin/bash
            yum update -y
            yum install -y httpd.x86_64
            systemctl start httpd.service
            systemctl enable httpd.service
            echo "Hello World from $(hostname -f)" > /var/www/html/index.html
      ImageId: ami-id2
  InstanceC:
    Type: 'AWS::EC2::Instance'
    Properties:
      DisableApiTermination: true
      UserData:
        Fn::Base64: 
          !Sub |
            #!/bin/bash
            yum update -y
            yum install -y httpd.x86_64
            systemctl start httpd.service
            systemctl enable httpd.service
            echo "Hello World from $(hostname -f)" > /var/www/html/index.html
      ImageId: ami-id3
      InstanceType: m5.2xlarge
      AvailabilityZone: us-east-1a
```

### Fn::ForEach in Outputs section

#### Use case 5: Reference Replicated Resources

In this example, customer is using two nested `Fn::ForEach` loops in Outputs section to reduce template length.

#### JSON

```json
{
    "AWSTemplateFormatVersion": "2010-09-09",
    "Transform": "AWS::LanguageExtensions",
    "Mappings": {
        "Buckets": {
            "Properties": {
                "Identifiers": ["A", "B", "C"]
            }
        }
    },
    "Resources": {
        "Fn::ForEach::Buckets": [
            "Identifier",
            {"Fn::FindInMap": ["Buckets", "Properties", "Identifiers"]},
            {
                "S3Bucket${Identifier}": {
                    "Type": "AWS::S3::Bucket",
                    "Properties": {
                        "AccessControl": "PublicRead",
                        "MetricsConfigurations": [
                            {
                                "Id": {"Fn::Sub": "EntireBucket${Identifier}"}
                            }
                        ],
                        "WebsiteConfiguration": {
                            "IndexDocument": "index.html",
                            "ErrorDocument": "error.html",
                            "RoutingRules": [
                                {
                                    "RoutingRuleCondition": {
                                        "HttpErrorCodeReturnedEquals": "404",
                                        "KeyPrefixEquals": "out1/"
                                    },
                                    "RedirectRule": {
                                        "HostName": "ec2-11-22-333-44.compute-1.amazonaws.com",
                                        "ReplaceKeyPrefixWith": "report-404/"
                                    }
                                }
                            ]
                        }
                    },
                    "DeletionPolicy": "Retain",
                    "UpdateReplacePolicy": "Retain"
                }
            }
        ]
    },
    "Outputs": {
        "Fn::ForEach::BucketOutputs": [
            "Identifier",
            {"Fn::FindInMap": ["Buckets", "Properties", "Identifiers"]},
            {
                "Fn::ForEach::GetAttLoop": [
                    "Property",
                    ["Arn", "DomainName", "WebsiteURL"],
                    {
                        "S3Bucket${Identifier}${Property}": {
                            "Value": {
                                "Fn::GetAtt": [{"Fn::Sub": "S3Bucket${Identifier}"}, {"Ref": "Property"}]
                            }
                        }
                    }
                ]
            }
        ]
    }
}
```

#### YAML

```yaml
AWSTemplateFormatVersion: 2010-09-09
Transform: 'AWS::LanguageExtensions'
Mappings:
  Buckets:
    Properties:
      Identifiers: [A, B, C]
Resources:
  'Fn::ForEach::Buckets':
    - Identifier
    - !FindInMap [Buckets, Properties, Identifiers]
    - 'S3Bucket${Identifier}':
        Type: 'AWS::S3::Bucket'
        Properties:
          AccessControl: PublicRead
          MetricsConfigurations:
            - Id: !Sub 'EntireBucket${Identifier}'
          WebsiteConfiguration:
            IndexDocument: index.html
            ErrorDocument: error.html
            RoutingRules:
              - RoutingRuleCondition:
                  HttpErrorCodeReturnedEquals: '404'
                  KeyPrefixEquals: out1/
                RedirectRule:
                  HostName: ec2-11-22-333-44.compute-1.amazonaws.com
                  ReplaceKeyPrefixWith: report-404/
        DeletionPolicy: Retain
        UpdateReplacePolicy: Retain
Outputs:
  'Fn::ForEach::BucketOutputs':
    - Identifier
    - !FindInMap [Buckets, Properties, Identifiers]
    - 'Fn::ForEach::GetAttLoop':
        - Property
        - [Arn, DomainName, WebsiteURL]
        - 'S3Bucket${Identifier}${Property}':
            Value: !GetAtt [!Sub 'S3Bucket${Identifier}', !Ref Property]
```

#### The above templates would be equivalent to the below template in YAML

```yaml
AWSTemplateFormatVersion: 2010-09-09
Transform: 'AWS::LanguageExtensions'
Resources:
  S3BucketA:
    Type: 'AWS::S3::Bucket'
    Properties:
      AccessControl: PublicRead
      MetricsConfigurations:
        - Id: EntireBucketA
      WebsiteConfiguration:
        IndexDocument: index.html
        ErrorDocument: error.html
        RoutingRules:
          - RoutingRuleCondition:
              HttpErrorCodeReturnedEquals: '404'
              KeyPrefixEquals: out1/
            RedirectRule:
              HostName: ec2-11-22-333-44.compute-1.amazonaws.com
              ReplaceKeyPrefixWith: report-404/
    DeletionPolicy: Retain
    UpdateReplacePolicy: Retain
  S3BucketB:
    Type: 'AWS::S3::Bucket'
    Properties:
      AccessControl: PublicRead
      MetricsConfigurations:
        - Id: EntireBucketB
      WebsiteConfiguration:
        IndexDocument: index.html
        ErrorDocument: error.html
        RoutingRules:
          - RoutingRuleCondition:
              HttpErrorCodeReturnedEquals: '404'
              KeyPrefixEquals: out1/
            RedirectRule:
              HostName: ec2-11-22-333-44.compute-1.amazonaws.com
              ReplaceKeyPrefixWith: report-404/
    DeletionPolicy: Retain
    UpdateReplacePolicy: Retain
  S3BucketC:
    Type: 'AWS::S3::Bucket'
    Properties:
      AccessControl: PublicRead
      MetricsConfigurations:
        - Id: EntireBucketC
      WebsiteConfiguration:
        IndexDocument: index.html
        ErrorDocument: error.html
        RoutingRules:
          - RoutingRuleCondition:
              HttpErrorCodeReturnedEquals: '404'
              KeyPrefixEquals: out1/
            RedirectRule:
              HostName: ec2-11-22-333-44.compute-1.amazonaws.com
              ReplaceKeyPrefixWith: report-404/
    DeletionPolicy: Retain
    UpdateReplacePolicy: Retain
Outputs:
  S3BucketAArn:
    Value: !GetAtt [S3BucketA, Arn]
  S3BucketADomainName:
    Value: !GetAtt [S3BucketA, DomainName]
  S3BucketAWebsiteURL:
    Value: !GetAtt [S3BucketA, WebsiteURL]
  S3BucketBArn:
    Value: !GetAtt [S3BucketB, Arn]
  S3BucketBDomainName:
    Value: !GetAtt [S3BucketB, DomainName]
  S3BucketBWebsiteURL:
    Value: !GetAtt [S3BucketB, WebsiteURL]
  S3BucketCArn:
    Value: !GetAtt [S3BucketC, Arn]
  S3BucketCDomainName:
    Value: !GetAtt [S3BucketC, DomainName]
  S3BucketCWebsiteURL:
    Value: !GetAtt [S3BucketC, WebsiteURL]
```

#### Use case 6: Reference a specific resource among replicated resources

In this example, customer is referencing to a resource created in a Fn::ForEach loop using the respective generated Logical IDs.

#### JSON

```json
{
    "AWSTemplateFormatVersion": "2010-09-09",
    "Transform": "AWS::LanguageExtensions",
    "Mappings": {
        "Instances": {
            "InstanceType": {
                "B": "m5.4xlarge",
                "C": "c5.2xlarge"
            },
            "ImageId": {
                "A": "ami-id1"
            }
        }
    },
    "Resources": {
        "Fn::ForEach::Instances": [
            "Identifier",
            ["A", "B", "C"],
            {
                "Instance${Identifier}": {
                    "Type": "AWS::EC2::Instance",
                    "Properties": {
                        "InstanceType": {
                            "Fn::FindInMap": ["Instances", "InstanceType", {"Ref": "Identifier"}, {"DefaultValue": "m5.xlarge"}]
                        },
                        "ImageId": {
                            "Fn::FindInMap": ["Instances", "ImageId", {"Ref": "Identifier"}, {"DefaultValue": "ami-id-default"}]
                        }
                    }
                }
            }
        ]
    },
    "Outputs": {
        "SecondInstanceId": {
            "Description": "Instance Id for InstanceB",
            "Value": {"Ref": "InstanceB"}
        },
        "SecondPrivateIp": {
            "Description": "Private IP for InstanceB",
            "Value": {
                "Fn::GetAtt": ["InstanceB", "PrivateIp"]
            }
        }
    }
}
```

#### YAML

```yaml
AWSTemplateFormatVersion: 2010-09-09
Transform: 'AWS::LanguageExtensions'
Mappings:
  Instances:
    InstanceType:
      B: m5.4xlarge
      C: c5.2xlarge
    ImageId:
      A: ami-id1
Resources:
  'Fn::ForEach::Instances':
    - Identifier
    - [A, B, C]
    - 'Instance${Identifier}':
        Type: 'AWS::EC2::Instance'
        Properties:
          InstanceType: !FindInMap [Instances, InstanceType, !Ref Identifier, DefaultValue: m5.xlarge]
          ImageId: !FindInMap [Instances, ImageId, !Ref Identifier, DefaultValue: ami-id-default]
Outputs:
  SecondInstanceId:
    Description: Instance Id for InstanceB
    Value: !Ref InstanceB
  SecondPrivateIp:
    Description: Private IP for InstanceB
    Value: !GetAtt [InstanceB, PrivateIp]
```

#### The above templates would be equivalent to the below template in YAML

```yaml
AWSTemplateFormatVersion: 2010-09-09
Transform: 'AWS::LanguageExtensions'
Resources:
  InstanceA:
    Type: 'AWS::EC2::Instance'
    Properties:
      InstanceType: m5.xlarge
      ImageId: ami-id1
  InstanceB:
    Type: 'AWS::EC2::Instance'
    Properties:
      InstanceType: m5.4xlarge
      ImageId: ami-id-default
  InstanceC:
    Type: 'AWS::EC2::Instance'
    Properties:
      InstanceType: c5.2xlarge
      ImageId: ami-id-default
Outputs:
  SecondInstanceId:
    Description: Instance Id for InstanceB
    Value: !Ref InstanceB
  SecondPrivateIp:
    Description: Private IP for InstanceB
    Value: !GetAtt [InstanceB, PrivateIp]
```

### Fn::ForEach in Conditions section

#### Use case 7: Replicate a single Condition

In this example, customer is using `Fn::ForEach` in the Conditions section to replicate multiple similar conditions with different properties.

#### JSON

```json
{
    "AWSTemplateFormatVersion": "2010-09-09",
    "Transform": "AWS::LanguageExtensions",
    "Parameters": {
        "ParamA": {
            "Type": "String",
            "AllowedValues": [
                "true",
                "false"
            ]
        },
        "ParamB": {
            "Type": "String",
            "AllowedValues": [
                "true",
                "false"
            ]
        },
        "ParamC": {
            "Type": "String",
            "AllowedValues": [
                "true",
                "false"
            ]
        },
        "ParamD": {
            "Type": "String",
            "AllowedValues": [
                "true",
                "false"
            ]
        }
    },
    "Conditions": {
        "Fn::ForEach::CheckTrue": [
            "Identifier",
            ["A", "B", "C", "D"],
            {
                "IsParam${Identifier}Enabled": {
                    "Fn::Equals": [
                        {"Ref": {"Fn::Sub": "Param${Identifier}"}},
                        "true"
                    ]
                }
            }
        ]
    },
    "Resources": {
        "WaitConditionHandle": {
            "Type": "AWS::CloudFormation::WaitConditionHandle"
        }
    }
}
```

#### YAML

```yaml
AWSTemplateFormatVersion: 2010-09-09
Transform: 'AWS::LanguageExtensions'
Parameters:
  ParamA:
    Type: String
    AllowedValues:
      - 'true'
      - 'false'
  ParamB:
    Type: String
    AllowedValues:
      - 'true'
      - 'false'
  ParamC:
    Type: String
    AllowedValues:
      - 'true'
      - 'false'
  ParamD:
    Type: String
    AllowedValues:
      - 'true'
      - 'false'
Conditions:
  'Fn::ForEach::CheckTrue':
    - Identifier
    - [A, B, C, D]
    - 'IsParam${Identifier}Enabled': !Equals 
        - !Ref 
          'Fn::Sub': 'Param${Identifier}'
        - 'true'
Resources:
  WaitConditionHandle:
    Type: 'AWS::CloudFormation::WaitConditionHandle'
```

#### The above templates would be equivalent to the below template in YAML

```yaml
AWSTemplateFormatVersion: 2010-09-09
Transform: 'AWS::LanguageExtensions'
Parameters:
  ParamA:
    Type: String
    AllowedValues:
      - 'true'
      - 'false'
  ParamB:
    Type: String
    AllowedValues:
      - 'true'
      - 'false'
  ParamC:
    Type: String
    AllowedValues:
      - 'true'
      - 'false'
  ParamD:
    Type: String
    AllowedValues:
      - 'true'
      - 'false'
Conditions:
  IsParamAEnabled: !Equals
    - !Ref ParamA
    - 'true'
  IsParamBEnabled: !Equals
    - !Ref ParamB
    - 'true'
  IsParamCEnabled: !Equals
    - !Ref ParamC
    - 'true'
  IsParamDEnabled: !Equals
    - !Ref ParamD
    - 'true'
Resources:
  WaitConditionHandle:
    Type: 'AWS::CloudFormation::WaitConditionHandle'
```





### Fn::ForEach with intrinsic functions such as Fn::FindInMap, Ref, and Fn::GetAtt

* Intrinsic functions that resolve to a String can now be specified within the Ref & Fn::GetAtt intrinsic functions
    * Note: the intrinsic function defined within Ref & Fn::GetAtt must be resolvable (i.e. should contain references to Parameters or Identifiers, and not Resource Logical ID’s)
* The Fn::FindInMap intrinsic function with its corresponding DefaultValue capability can be leveraged to define unique properties for each element in the collection

```yaml
AWSTemplateFormatVersion: 2010-09-09
Transform: 'AWS::LanguageExtensions'
Mappings:
  NatGateway:
    Condition: 
      B: TwoNatGateways
      C: ThreeNatGateways
Resources:
  'Fn::ForEach::NatGatewayAndEIP':
    - Identifier
    - [A, B, C]
    - 'NatGateway${Identifier}':
        Type: 'AWS::EC2::NatGateway'
        Properties:
          AllocationId: !GetAtt 
            - !Sub 'NatGatewayAttachment${Identifier}'
            - AllocationId
          SubnetId: !Ref 
            'Fn::Sub': 'PublicSubnet${Identifier}'
        Condition: !FindInMap [NatGateway, Condition, !Ref Identifier, DefaultValue: !Ref 'AWS::NoValue']
      'NatGatewayAttachment${Identifier}':
        Type: 'AWS::EC2::EIP'
        Properties:
          Domain: vpc
        Condition: !FindInMap [NatGateway, Condition, !Ref Identifier, DefaultValue: !Ref 'AWS::NoValue']
```

## Constraints

Customers must be aware of the following constraints of `Fn::ForEach` while authoring their CloudFormation templates.

### 1.  Customer cannot use short-form YAML notation for Fn::ForEach

Short-form notation is not supported for `Fn::ForEach` in YAML. The syntax of the intrinsic function `‘Fn::ForEach::<UniqueLoopName>’`, and the nature of YAML which does not allow Tags on the same level where other key-value pairs (Objects) are the reasons for not supporting short-form YAML notation with `Fn::ForEach`.

### 2. Customer cannot reference a NoEcho Parameter in a Collection to iterate with Fn::ForEach

A NoEcho Parameter value cannot be used as an argument to `Fn::ForEach`.

```yaml
AWSTemplateFormatVersion: 2010-09-09
Transform: 'AWS::LanguageExtensions'
Parameters:
   NoEchoList:
     Type: CommaDelimitedList
     NoEcho: True
Resources:
  'Fn::ForEach::SecurityGroups':
    - Identifier
    - !Ref NoEchoList
    - 'SecurityGroup${Identifier}':
        Type: AWS::EC2::SecurityGroup
        Properties:
          GroupDescription: 'Group Description'
```

### 3. “Identifier” and “UniqueLoopName” must not conflict with the name of any Parameters defined in the Parameters section, as well as any Resource logical ID’s defined in the Resources section

* The template below is invalid because the Identifier “Param” conflicts with a parameter with the same name.

```yaml
AWSTemplateFormatVersion: 2010-09-09
Transform: 'AWS::LanguageExtensions'
Parameters:
   Param:
     Type: String
Resources:
  'Fn::ForEach::SNSTopics':
    - Param
    - ['A', 'B', 'C']
    - 'SNSTopic${Param}':
        Type: AWS::SNS::Topic
```

* The template below is invalid because the UniqueLoopName “Param” conflicts with a parameter with the same name.

```yaml
AWSTemplateFormatVersion: 2010-09-09
Transform: 'AWS::LanguageExtensions'
Parameters:
   Param:
     Type: String
Resources:
  'Fn::ForEach::Param':
    - Identifier
    - ['A', 'B', 'C']
    - 'SNSTopic${Identifier}':
        Type: AWS::SNS::Topic
```

* The template below is invalid because the UniqueLoopName “SNS” conflicts with a resource with the same name.

```yaml
AWSTemplateFormatVersion: 2010-09-09
Transform: 'AWS::LanguageExtensions'
Resources:
  SNS:
    Type: AWS::SNS::Topic
  'Fn::ForEach::SNS':
    - Identifier
    - ['A', 'B', 'C']
    - 'SNSTopic${Identifier}':
        Type: AWS::SNS::Topic
```

### 4. The resulting “OutputKey” must not already exist within the parent object where this resulting Object is merged

* The template below is invalid because the resulting OutputKey “SNSTopicA” already exists in the parent object.

```yaml
AWSTemplateFormatVersion: 2010-09-09
Transform: 'AWS::LanguageExtensions'
Resources:
  'SNSTopicA':
    Type: AWS::SNS::Topic
  'Fn::ForEach::Topics':
    - Identifier
    - ['A', 'B', 'C']
    - 'SNSTopic${Identifier}':
        Type: AWS::SNS::Topic
```

### 5. “Identifier” name should be unique when nesting multiple Fn::ForEach loops

* The template below is invalid because the “Identifier” name in the nested `Fn::ForEach` intrinsic function “SameName” is the same as the “Identifier” name defined in the parent `Fn::ForEach` intrinsic function.

```yaml
AWSTemplateFormatVersion: 2010-09-09
Transform: 'AWS::LanguageExtensions'
Resources:
  'Fn::ForEach::SubnetResources':
    - SameName
    - [Transit, Public]
    - 'Fn::ForEach::LoopInner':
        - SameName
        - [A, B, C]
        - '${SameName}SNS${SameName}':
            Type: 'AWS::SNS::Topic'
```

### 6. The “Identifier” must be resolvable before Resource creation

* The value of the “Identifier” has to be *known* before resource provisioning, otherwise the Stack/ChangeSet creation/update. You can’t, for example, refer to an attribute of a Resource. Values must be known before CloudFormation performs any remote resource interactions.
* The template below is invalid because the “Identifier” cannot be resolved before Resource creation

```yaml
AWSTemplateFormatVersion: 2010-09-09
Transform: 'AWS::LanguageExtensions'
Resources:
  SNSTopic:
    Type: AWS::SNS::Topic
  'Fn::ForEach::Topics':
    - !Ref SNSTopic
    - ['A', 'B', 'C']
    - 'SNSTopic${Identifier}':
        Type: AWS::SNS::Topic
```

### 7. The “Collection” and each value in the “Collection” must be resolvable before Resource creation

* All the values in the list defined within the 2nd parameter have to be *known* before resource provisioning, otherwise the Stack/ChangeSet creation/update. You can’t, for example, refer to an attribute of a Resource. Values must be known before CloudFormation performs any remote resource interactions.
* The template below is invalid because an element in the “Collection” cannot be resolved before Resource creation.

```yaml
AWSTemplateFormatVersion: 2010-09-09
Transform: 'AWS::LanguageExtensions'
Resources:
  SNSTopic:
    Type: AWS::SNS::Topic
  'Fn::ForEach::Topics':
    - Identifier
    - ['A', 'B', !Ref SNSTopic]
    - 'SNSTopic${Identifier}':
        Type: AWS::SNS::Topic
```

* The template below is invalid because the “Collection” cannot be resolved before Resource creation.

```yaml
AWSTemplateFormatVersion: 2010-09-09
Transform: 'AWS::LanguageExtensions'
Resources:
  VPC:
    Type: 'AWS::EC2::VPC'
    Properties:
      CidrBlock: 10.0.0.0/16
      EnableDnsSupport: 'true'
      EnableDnsHostnames: 'true'
  TargetGroup:
    Type: AWS::ElasticLoadBalancingV2::TargetGroup
    Properties:
      Name: 'MyElasticGroup'
      Port: 80
      Protocol: 'HTTP'
      VpcId: !Ref VPC
  'Fn::ForEach::Topics':
    - Identifier
    - !GetAtt [TargetGroup, LoadBalancerArns]
    - 'SNSTopic${Identifier}':
        Type: AWS::SNS::Topic
```

## Potential Follow-up Features

Features like supporting iterating over a key-value pair, outputing a list instead of merging to an object or allowing Ref/Fn::GetAtt on the UniqueLoopName are out of scope for this RFC.

The GitHub issue below contains potential follow-up features, where based on customer demand, CloudFormation will create separate RFC’s for those customer pain-points:

https://github.com/aws-cloudformation/cfn-language-discussion/issues/118


## Appendix

### Appendix A: Pros and Cons matrix for looping function names

| FunctionName | Pros | Cons |
| ------------ | ---- | ---- |
| Fn::ForEach  | Addresses valid feedback in GitHub pull requestIntrinsic function name indicates it’s a functional concept and not an imperative oneForEach Illustrates that we’re operating on a template fragment over a set of valuesGives customers familiar with coding concepts the idea that some form of “looping” (or replication) is done | Intrinsic function name is potentially not unclear to customers who aren’t familiar with coding concepts; however, the syntax of Fn::ForEach::<ResourceLogicalId> would give users an idea that within the function, we define the template fragment "for each" resource |
| Fn::Map | Addresses valid feedback in GitHub pull requestIntrinsic function name indicates it’s a functional concept and not an imperative oneMap Illustrates that we’re operating on a template fragment over a set of valuesSimilar to the forEach function. However, the map function creates a new array with the results of calling a function for every element, whereas the forEach function doesn't return anything. | Intrinsic function name is likely not clear to customers who aren’t familiar with coding concepts (specifically the map function) |
| Fn::Build | Indicates a set of template fragments will be “built”/generatedIntrinsic function name is clear to customers who aren’t familiar with coding concepts | Does not imply iteration being done on a collection |
| Fn::Expand | Indicates a set of template fragments that are collapsed, will be expandedIntrinsic function name is clear to customers who aren’t familiar with coding concepts | Does not imply iteration being done on a collection |
| Fn::Repeat | Gives the idea that some form of “looping” (or replication) is doneIntrinsic function name is clear to customers who aren’t familiar with coding concepts | Does not imply iteration being done on a collectionImplies a template fragment will be repeated a set number of times (Note: this can be a separate on customer requests and demand) |
| Fn::Copy | Gives the idea that some form of “looping” (or replication) is doneIntrinsic function name is clear to customers who aren’t familiar with coding concepts | Vaguely implies iteration being done on a collectionCustomers will have to look at documentation to determine exactly what the function does |

