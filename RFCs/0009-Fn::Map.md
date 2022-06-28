# Looping functionality for CFN Templates

* **Original Author(s):**: @mingujo
* **Tracking Issue**: [Tracking Issue: Fn::Map](https://github.com/aws-cloudformation/cfn-language-discussion/issues/9)

# Summary
We are proposing to introduce a new intrinsic function called `Fn::Map`. It will allow users to iterate over a given list to replicate a particular template fragment thereby avoiding repetitively writing a similar declaration for each object (e.g. Resource, Output, or Resource Properties).

`Fn::Map` traverses a single given List of Strings from first to last, and over each iteration, it references each item under a given template snippet to be replicated.

We additionally introduce `Fn::Merge` which can be used to merge the result of Fn::Map into its parent object.
# Motivation

In a CloudFormation template, a single Resource configures one infrastructure object. Therefore, a template can become verbose when you manually declare multiple similar resources. Some examples would include a pool of EC2 instances with the same configurations but different instance types, or S3 Bucket Notification configurations with different SNS Topics.

# Declaration
## Fn::Map
### Json
```json
{
  "Fn::Map": {
    "Index": "Index",
    "Value": "Value",
    "Collection": "CollectionToIterate",
    "Fragment": { },
    "Key": "Key"
  }
}
```

### Yaml
```yaml
Fn::Map: # or !Map
  Index: "Index"
  Value: "Value"
  Collection: "CollectionToIterate"
  Fragment: { }
  Key: "Key"
```

### Parameters

Fn::Map supports 5 parameters. Note, that the parameters are named, not positional.

* `Index` (optional)
    * The name of an iterator variable which stores a positional index of each value in a given List.
    * The index number sequence starts from 0.
    * String type
    * the default value is "Index"
* `Value` (optional)
    * The name of an iterator variable which stores each value in a List.
    * String type
    * the default value is "Value"
* `Collection`
    * The collection to iterate on. It must be a List. You can either in-place the collection itself or reference List type parameter values from the Parameters section.
* `Fragment`
    * The fragment to replicate for each item in iteration
* `Key` (conditional)
    * The json or yaml key for replicated objects. For Resources and Outputs this will be the logical id.
    * **This field is only required when Fn::Map should generate an object of key value pairs, for example in the Resources or Outputs section of a template. It should not be specified when using Fn::Map to generate an array, for example a list type attribute for a Resource Property.** 
    * The value **should be interpolated with an index** using `Fn::Sub` intrinsic function.


## Fn::Merge
Fn::Merge takes an array of objects as input and merges them into the parent object
```json
{
  "Fn::Merge": [
    {
      "Key1": { },
      "Key2": { }
    },
    {
      "Key3": { }
    }
  ],
  "Key4": { }
}
```

Result:
```json
{
  "Key1": { },
  "Key2": { },
  "Key3": { },
  "Key4": { }
}
```

Fn::Merge will throw an error when there is a key collision. The below examples will cause an error:

```json
{
  "Fn::Merge": [
    {
      "Key1": {}
    },
    {
      "Key1": { }
    }
  ]
}
```
```json
{
  "Fn::Merge": [
    {
      "Key1": {}
    }
  ],
  "Key1": { }
}
```

# Examples
### Usage in Resources Section
#### Use case: Replicate a single Resource
* Note that the **`Key`** parameter is provided.

In Json:
```json
{
  "AWSTemplateFormatVersion": "2010-09-09",
  "Description": "EC2 Instances with different AMIs",
  "Resources": {
    "Fn::Merge": [
      {
        "Fn::Map": {
          "Index": "i",
          "Value": "x",
          "Collection": ["ami-1", "ami-2", "ami-3"],
          "Fragment": {
            "Type": "AWS::EC2::Instance",
            "Properties": {
              "InstanceType": "m1.small",
              "ImageId": {"Ref": "x"}
            }
          },
          "Key": {
            "Fn::Sub": "Instance${i}"
          }
        }
      }
    ],
    "MyS3Bucket": {
      "Type": "AWS::S3::Bucket"
    },
    "MyQueue": {
      "Type": "AWS::SQS::Queue"
    }
  }
}
```
* In Yaml
```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: EC2 Instances with different AMIs
Resources:
  Fn::Merge:
    - Fn::Map:
        Index: i
        Value: x
        Collection: [ "ami-1", "ami-2", "ami-3" ]
        Fragment:
          Type: AWS::EC2::Instance
          Properties:
            InstanceType: m1.small
            ImageId: !Ref x
        Key:
          Fn::Sub: Instance${i}
  MyS3Bucket:
    Type: AWS::S3::Bucket
  MyQueue:
    Type: AWS::SQS::Queue

```
* The above templates would be equivalent to the one below:
```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: "EC2 Instances with different AMIs"
Resources:
  Instance0:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: "m1.small"
      ImageId: "ami-1"
  Instance1:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: "m1.small"
      ImageId: "ami-2"
  Instance2:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: "m1.small"
      ImageId: "ami-3"
  MyS3Bucket:
    Type: AWS::S3::Bucket
  MyQueue:
    Type: AWS::SQS::Queue
```
#### Use case: Replicate Multiple Resources
Note also the usage of defaults for Index and Value
* In Json
```json
{
  "AWSTemplateFormatVersion": "2010-09-09",
  "Description": "VPCs and Subnets",
  "Resources": {
    "Fn::Merge": [
      {
        "Fn::Map": {
          "Collection": ["172.16.0.0/16", "172.17.0.0/16", "172.18.0.0/16"],
          "Fragment": {
            "Type": "AWS::EC2::VPC",
            "Properties": {
              "CidrBlock": {"Ref": "Value"}
            }
          },
          "Key": {
            "Fn::Sub": "Vpc${Index}"
          }
        }
      },
      {
        "Fn::Map": {
          "Collection": ["172.16.0.0/16", "172.17.0.0/16", "172.18.0.0/16"],
          "Fragment": {
            "Type": "AWS::EC2::Subnet",
            "Properties": {
              "VpcId": {"Ref": {"Fn::Sub":  "Vpc${Index}"}}
            }
          },
          "Key": {
            "Fn::Sub": "Subnet${Index}"
          }
        }
      }
    ],
    "MyS3Bucket": {
      "Type": "AWS::S3::Bucket"
    },
    "MyQueue": {
      "Type": "AWS::SQS::Queue"
    }
  }
}
```
* In Yaml
```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: VPCs and Subnets
Resources:
  Fn::Merge:
    - !Map:
        Collection:
          - 172.16.0.0/16
          - 172.17.0.0/16
          - 172.18.0.0/16
        Fragment:
          Type: AWS::EC2::VPC
          Properties:
            CidrBlock: !Ref Value
        Key: !Sub Vpc${Index}
    - !Map:
        Collection:
          - 172.16.0.0/16
          - 172.17.0.0/16
          - 172.18.0.0/16
        Fragment:
          Type: AWS::EC2::Subnet
          Properties:
            VpcId: 
              Ref: !Sub "Vpc${Index}"
        Key: !Sub Subnet${Index}
  MyS3Bucket:
    Type: AWS::S3::Bucket
  MyQueue:
    Type: AWS::SQS::Queue

```
* The above templates would be equivalent to the below:
```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: "EC2 Instances with different AMIs"
Resources:
  Vpc0:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: "172.16.0.0/16"
  Vpc1:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: "172.17.0.0/16"
  Vpc2:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: "172.18.0.0/16"
  Subnet0:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref Vpc0
  Subnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref Vpc1
  Subnet2:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref Vpc2
  MyS3Bucket:
    Type: AWS::S3::Bucket
  MyQueue:
    Type: AWS::SQS::Queue
```

#### Use case: Concisely define a list type property of Resource using parametrization
* Note that the **`Key`** parameter is not provided.
* In Json
```json
{
  "AWSTemplateFormatVersion": "2010-09-09",
  "Description": "EC2 Instance with list of Ipv6Addresses",
  "Parameters": {
    "InstanceIpv6Address": {
      "Type": "CommaDelimitedList",
      "Default": "ipv6-1,ipv6-2,ipv6-3"
    }
  },
  "Resources": {
    "Instance": {
      "Type": "AWS::EC2::Instance",
      "Properties": {
        "InstanceType": "m1.small",
        "Ipv6Addresses": {
          "Fn::Map": {
            "Collection": {"Ref": "InstanceIpv6Address"},
            "Fragment": {
              "Ipv6Address": {"Ref": "Value"}
            }
          }
        }
      }
    }
  }
}
```
* In Yaml
```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: EC2 Instance with list of Ipv6Addresses
Parameters:
  InstanceIpv6Address:
    Type: CommaDelimitedList
    Default: ipv6-1,ipv6-2,ipv6-3
Resources:
  Instance:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: m1.small
      Ipv6Addresses:
        Fn::Map:
          Collection: !Ref InstanceIpv6Address
          Fragment:
            Ipv6Address: !Ref Value
```
* The Above templates would be equivalent to the below:
```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: "EC2 Instances with Ipv6Addresses"
Parameters:
  Ipv6Addresses:
    Type: CommaDelimitedList
    Default: "ipv6-1,ipv6-2,ipv6-3"
Resources:
  Instance:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: "m1.small"
      Ipv6Addresses:
        - Ipv6Address: "ipv6-1"
        - Ipv6Address: "ipv6-2"
        - Ipv6Address: "ipv6-3"
```

### Usage in Outputs Section

#### Use case: Reference Replicated Resource(s)

* In Json
```json
{
  "AWSTemplateFormatVersion": "2010-09-09",
  "Description": "EC2 Instances with different AMIs",
  "Resources": {
    "Fn::Merge": [
      {
        "Fn::Map": {
          "Collection": ["ami-1", "ami-2", "ami-3"],
          "Fragment": {
            "Type": "AWS::EC2::Instance",
            "Properties": {
              "InstanceType": "m1.small",
              "ImageId": {
                "Ref": "Value"
              }
            }
          },
          "Key": {
            "Fn::Sub": "Instance${Index}"
          }
        }
      }
    ],
    "Outputs": {
      "Fn::Merge": [
        {
          "Fn::Map": {
            "Collection": ["Instance0", "Instance1"],
            "Fragment": {
              "Description": {
                "Fn::Sub": "Instance Id for ${Value}"
              },
              "Value": {
                "Fn::Ref": {
                  "Fn::Ref": "Value"
                }
              }
            },
            "Key": {
              "Fn::Sub": "InstanceId${Index}"
            }
          }
        },
        {
          "PrivateIps": {
            "Fn::Map": {
              "Collection": ["Instance0", "Instance1"],
              "Fragment": {
                "Description": {"Fn::Sub": "Private IP for ${Value}"},
                "Value": {
                  "Fn::GetAtt": [
                    {
                      "Fn::Ref": "Value"
                    },
                    "PrivateIp"
                  ]
                }
              },
              "Key": {
                "Fn::Sub": "PrivateIp${Index}"
              }
            }
          }
        }
      ]
    }
  }
}
```
* In Yaml
```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: EC2 Instances with different AMIs
Resources:
  Fn::Merge:
    - Fn::Map:
        Collection:
          - ami-1
          - ami-2
          - ami-3
        Fragment:
          Type: AWS::EC2::Instance
          Properties:
            InstanceType: m1.small
            ImageId:
              Ref: Value
        Key:
          Fn::Sub: Instance${Index}
  Outputs:
    Fn::Merge:
      - Fn::Map:
          Collection:
            - Instance0
            - Instance1
          Fragment:
            Description:
              Fn::Sub: Instance Id for ${Value}
            Value:
              Fn::Ref:
                Fn::Ref: Value
          Key:
            Fn::Sub: InstanceId${Index}
      - PrivateIps:
          Fn::Map:
            Collection:
              - Instance0
              - Instance1
            Fragment:
              Description:
                Fn::Sub: Private IP for ${Value}
              Value:
                Fn::GetAtt:
                  - Fn::Ref: Value
                  - PrivateIp
            Key:
              Fn::Sub: PrivateIp${Index}
```
* The above templates would be equivalent to the below:
```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: "AutoScaling LaunchConfiguration"
Parameters:
  AmiIds:
    Type: CommaDelimitedList
    Default: "ami-1,ami-2,ami-3"
Resources:
  Instance0:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: "m1.small"
      ImageId: "ami-1"
  Instance1:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: "m1.small"
      ImageId: "ami-2"
  Instance2:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: "m1.small"
      ImageId: "ami-3"
Outputs:
  InstanceId0:
    Description: !Sub "Instance Id for InstanceId0"
    Value:
      Fn::Ref: Instance0
  InstanceId1:
    Description: !Sub "Instance Id for InstanceId1"
    Value:
      Fn::Ref: Instance1
  PrivateIP0:
    Description: !Sub "Private IP for PrivateIP0"
    Value:
      Fn::GetAtt:
        - Instance0
        - PrivateIp
  PrivateIP1:
    Description: !Sub "Private IP for PrivateIP1"
    Value:
      Fn::GetAtt:
        - Instance1
        - PrivateIp
```

#### Use case: Reference a specific resource among replicated resources
* In Json
```json
{
    "AWSTemplateFormatVersion": "2010-09-09",
    "Description": "EC2 Instances with different AMIs",
    "Resources": {
        "Fn::Merge": [
          {
            "Fn::Map": {
              "Collection": ["ami-1", "ami-2", "ami-3"],
              "Fragment": {
                "Type": "AWS::EC2::Instance",
                "Properties": {
                  "InstanceType": "m1.small",
                  "ImageId": {"Ref": "Value"}
                }
              },
              "Key": {"Fn::Sub": "Instance${Index}"}
            }
          }
        ]
    },
    "Outputs": {
        "SecondInstanceId": {
            "Description": "Instance Id for Instance1",
            "Value": { "Fn::Ref": "Instance1" }
        },
        "SecondPrivateIp": {
            "Description": "Private ip for Instance1",
            "Value": {
                "Fn::GetAtt": [ "Instance1", "PrivateIp" ]
            }
        }
    }
}
```
* In Yaml
```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: EC2 Instances with different AMIs
Resources:
  Fn::Merge:
    - Fn::Map:
        Collection:
          - ami-1
          - ami-2
          - ami-3
        Fragment:
          Type: AWS::EC2::Instance
          Properties:
            InstanceType: m1.small
            ImageId: !Ref Value
        Key:
          Fn::Sub: Instance${Index}
Outputs:
  SecondInstanceId:
    Description: Instance Id for Instance1
    Value:
      Fn::Ref: Instance1
  SecondPrivateIp:
    Description: Private ip for Instance1
    Value:
      Fn::GetAtt:
        - Instance1
        - PrivateIp

```
* Above templates are equivalent to a below template in Yaml
```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: "AutoScaling LaunchConfiguration"
Parameters:
  AmiIds:
    Type: CommaDelimitedList
    Default: "ami-1,ami-2,ami-3"
Resources:
  Instance0:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: "m1.small"
      ImageId: "ami-1"
  Instance1:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: "m1.small"
      ImageId: "ami-2"
  Instance2:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: "m1.small"
      ImageId: "ami-3"
Outputs:
  SecondInstanceId:
    Description: "Instance Id for Instance1"
    Value: !Ref Instance1
  SecondPrivateIp:
    Description: "Private ip for Instance1"
    Value: !GetAtt Instance1.PrivateIp
```

# Properties of `Fn::Map`

#### Reference source list from Parameters
* You can reference a source list from Parameters
```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: "EC2 Instances with different AMIs"
Parameters:
  AmiIds:
    Type: CommaDelimitedList
    Default: "ami-1,ami-2,ami-3"
Resources:
  Fn::Merge:
    - Fn::Map:
        Collection: !Ref AmiIds
        Fragment:
          Type: AWS::EC2::Instance
          Properties:
            InstanceType: m1.small
            ImageId: !Ref Value
        Key:
          Fn::Sub: Instance${Index}
  Bucket:
    Type: AWS::S3::Bucket
    Properties:
      Name: "bucket"
```

#### Logical Id Customization
* You are required to customize Names (Logical Ids) of Resource and Outputs. Specify the Key parameter by using Fn::Sub with Index or Value. Make sure the resulting Logical Ids are unique within the template and alphanumerical.
```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: "EC2 Instances with different AMIs"
Parameters:
  AmiIds:
    Type: CommaDelimitedList
    Default: "ami-1,ami-2,ami-3"
Resources:
  Fn::Merge:
    - Fn::Map:
        Collection: !Ref AmiIds
        Fragment:
          Type: AWS::EC2::Instance
          Properties:
            InstanceType: m1.small
            ImageId: !Ref Value
        Key:
          Fn::Sub: Instance${Index}
  Bucket:
    Type: AWS::S3::Bucket
    Properties:
      Name: "bucket"
```
* Caveat: Make sure Logical Ids are alphanumeric characters

#### SSM/AWS List Type Parameters
* SSM/AWS List type parameters can be referenced in `Fn::Map`. Only List type parameter is allowed.
```yaml
Parameters:
  AmiIds:
    Type: AWS::SSM::Parameter::Value<List<String>>
Resources:
  Fn::Merge:
    - Fn::Map:
        Collection: !Ref AmiIds
        Fragment:
          Type: AWS::EC2::Instance
          Properties:
            ImageId: !Ref Value
        Key: !Sub Instance${Index}
```

#### Customize Value and Index variable names
The optional Index and Value parameters can be used to customize the corresponding variable names
```yaml
Parameters:
  AmiIds:
    Type: AWS::SSM::Parameter::Value<List<String>>
Resources:
  Fn::Merge:
    - Fn::Map:
        Index: amiCounter
        Value: amiId
        Collection: !Ref AmiIds
        Fragment:
          Type: AWS::EC2::Instance
          Properties:
            ImageId: !Ref amiId
        Key: !Sub Instance${amiCounter}
```

#### Nested Fn::Map
* Using customized names for Index and Value is particularly useful when nesting declarations of Fn::Map
```yaml
Parameters:
  InstanceSizes:
    Type: CommaDelimitedList
    Default: "m1.small,m1.medium"
  Ipv6Addresses:
    Type: CommaDelimitedList
    Default: "ipv6-1,ipv6-2,ipv6-3"
Resources:
  Fn::Merge:
    - Fn::Map:
        Index: i1
        Value: x1
        Collection: !Ref InstanceSizes
        Key: !Sub "Instance${i1}"
        Fragment:
          Type: AWS::EC2::Instance
          Properties:
            InstanceType: !Ref x1
            Ipv6Addresses:
              - Fn::Map:
                  Index: i2
                  Value: x2
                  Collection: !Ref Ipv6Addresses
                  Fragment:
                    - Ipv6Address: !Ref x2
```
* The above template is equivalent to the below:
```yaml

Parameters:
  InstanceSizes:
    Type: CommaDelimitedList
    Default: "m1.small,m1.medium"
  Ipv6Addresses:
    Type: CommaDelimitedList
    Default: "ipv6-1,ipv6-2,ipv6-3"
Resources:
  Instance0:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: "m1.small"
      Ipv6Addresses:
        - Ipv6Address: "ipv6-1"
        - Ipv6Address: "ipv6-2"
        - Ipv6Address: "ipv6-3"
  Instance1:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: "m1.medium"
      Ipv6Addresses:
        - Ipv6Address: "ipv6-1"
        - Ipv6Address: "ipv6-2"
        - Ipv6Address: "ipv6-3"
```

* Another example of using an outer function’s value variable inside the inner one function
```yaml
Parameters:
  Subnets:
      Type: CommaDelimitedList
      Default: "subnet1,subnet2"
  TagValues:
      Type: CommaDelimitedList
      Default: "tag1,tag2"
Resources:
  Fn::Merge:
    - Fn::Map:
        Index: i1
        Value: x1
        Collection: !Ref Subnets
        Key: !Sub "Instance${i1}"
        Fragment:
          Type: AWS::EC2::Instance
          InstanceType: "m1.small"
          SubnetId: !Ref x1
          Tags:
            - Fn::Map:
                Index: i2
                Value: x2
                Collection: !Ref TagValues
                Fragment:
                  Key: !Sub "${x1}"
                  Value: !Sub "${x2}"
```

# Potential Follow-up Features

The following features are out of scope of this RFC. Separate RFC could be created in the future if there is customer demand.

### Modules
* `Fn::Map` should support replicating Modules

### Replicate a Fragment a certain number of times
* For a number parameter x, replicate a given template block x number of times.

Users can easily work around this limitation by iterating over a hardcoded list:
```yaml
Resources:
  Fn::Merge:
    - Fn::Map:
        Collection: [1, 2, 3, 4]
        Fragment:
          Type: AWS::EC2::Instance
          Properties:
            ImageId: "ami-123"
        Key: !Sub Instance${Index}
```

### Zipping
* `Fn::Map` could take in multiple lists and generate an aggregated list of Resources or Outputs.

# Limitations
#### Replicate multiple Resource objects in a single Fn::Map usage
* To replicate a Resource/Output object, a custom key interpolated with an Index has to be provided under the `Key` parameter. This means an `Fn::Map` loop can be used to replicate only one Resource at a time. To replicate multiple Resources, `Fn::Map` has to be defined for each Resource.

#### A Collection to Iterate on has to have known values
* All the values in a list have to be *known* before resource provisioning, otherwise it will throw an error message that Fn::Map received a value which cannot be resolved. You can’t, for example, refer to an attribute of a Resource. Values must be known before CFN performs any remote resource interactions.
* For  example, the below would not work
```yaml
Resources:
  SecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: '22'
          ToPort: '22'
          CidrIp: 0.0.0.0/0    
  Instance:
     Type: AWS::EC2::Instance
     Properties:
        SecurityGroups:
          - Fn::Map:
              Collection: [!Ref SecurityGroup]
              Fragment: !Ref Value
```

#### You can’t reference NoEcho Parameter
* A NoEcho Parameter value cannot be used as an argument to Fn::Map.
```yaml
Parameters:
   NoEchoList:
     Type: CommaDelimitedList
     NoEcho: True
Resources:
  ResourceReferringBucket:
     Type: AWS::SNS::Topic
     Properties:
        Subscription:
          - Fn::Map:
              Collection: !Ref NoEchoList
              Fragment:
                Endpoint: !Ref Value
```
* In this case the value of the collection would be used to replicate topic subscriptions, which means the values of the no echo list would be exposed in the processed template. So NoEcho lists cannot be used to iterate over.
