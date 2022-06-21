# New Intrinsic Function for looping: `Fn::Map`

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
    "index": "Index",
    "value": "Value",
    "collection": "CollectionToIterate",
    "fragment": { },
    "key": "Key"
  }
}
```

### Yaml
```yaml
Fn::Map: # or !Map
  index: "Index"
  value: "Value"
  collection: "CollectionToIterate"
  fragment: { }
  key: "Key"
```

### Parameters

Fn::Map supports 5 parameters.

* `index` (optional)
    * The name of an iterator variable which stores a positional index of each value in a given List.
    * The index number sequence starts from 0.
    * String type
    * the default value is "index"
* `value` (optional)
    * The name of an iterator variable which stores each value in a List.
    * String type
    * the default value is "value"
* `collection`
    * The collection to iterate on. It must be a List. You can either in-place the collection itself or reference List type parameter values from the Parameters section.
* `fragment`
    * The fragment to replicate for each item in iteration
* `key` (conditional)
    * The json or yaml key for replicated objects. For Resources and Outputs this will be the logical id.
    * **This field is only required when Fn::Map should generate an object of key value pairs, for example in the Resources or Outputs section of a template. It should not be specified when using Fn::Map to generate an array, for example a list type attribute for a Resource Property.** 
    * The value **should be interpolated with an index** using `Fn::Sub` intrinsic function.


## Fn::Merge
Fn::Merge takes an array of objects as input and merges them into the parent object
```json
{
  "Fn::Merge": [
    {
      "key1": { },
      "key2": { }
    },
    {
      "key3": { }
    }
  ],
  "key4": { }
}
```

Result:
```json
{
  "key1": { },
  "key2": { },
  "key3": { },
  "key4": { }
}
```

Fn::Merge will throw an error when there is a key collision. The below examples will cause an error:

```json
{
  "Fn::Merge": [
    {
      "key1": {}
    },
    {
      "key1": { }
    }
  ]
}
```
```json
{
  "Fn::Merge": [
    {
      "key1": {}
    }
  ],
  "key1": { }
}
```

# Examples
### Usage in Resources Section
#### Use case: Replicate a single Resource
* Note that the **`key`** parameter is provided.

In Json:
```json
{
  "AWSTemplateFormatVersion": "2010-09-09",
  "Description": "EC2 Instances with different AMIs",
  "Resources": {
    "Fn::Merge": [
      {
        "Fn::Map": {
          "index": "i",
          "value": "x",
          "collection": ["ami-1", "ami-2", "ami-3"],
          "fragment": {
            "Type": "AWS::EC2::Instance",
            "Properties": {
              "InstanceType": "m1.small",
              "ImageId": {"Ref": "x"}
            }
          },
          "key": {
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
        index: i
        value: x
        collection: [ "ami-1", "ami-2", "ami-3" ]
        fragment:
          Type: AWS::EC2::Instance
          Properties:
            InstanceType: m1.small
            ImageId: !Ref x
        key:
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
```
#### Use case: Replicate Multiple Resources
Note also the usage of defaults for index and value
* In Json
```json
{
  "AWSTemplateFormatVersion": "2010-09-09",
  "Description": "VPCs and Subnets",
  "Resources": {
    "Fn::Merge": [
      {
        "Fn::Map": {
          "collection": ["172.16.0.0/16", "172.17.0.0/16", "172.18.0.0/16"],
          "fragment": {
            "Type": "AWS::EC2::VPC",
            "Properties": {
              "CidrBlock": {"Ref": "value"}
            }
          },
          "key": {
            "Fn::Sub": "Vpc${index}"
          }
        }
      },
      {
        "Fn::Map": {
          "collection": ["172.16.0.0/16", "172.17.0.0/16", "172.18.0.0/16"],
          "fragment": {
            "Type": "AWS::EC2::Subnet",
            "Properties": {
              "VpcId": {"Ref": {"Fn::Sub":  "Vpc${index}"}}
            }
          },
          "key": {
            "Fn::Sub": "Subnet${index}"
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
        collection:
          - 172.16.0.0/16
          - 172.17.0.0/16
          - 172.18.0.0/16
        fragment:
          Type: AWS::EC2::VPC
          Properties:
            CidrBlock: !Ref value
        key: !Sub Vpc${i}
    - !Map:
        collection:
          - 172.16.0.0/16
          - 172.17.0.0/16
          - 172.18.0.0/16
        fragment:
          Type: AWS::EC2::Subnet
          Properties:
            VpcId: 
              Ref: !Sub "Vpc${index}"
        key: !Sub Subnet${index}
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
            "collection": {"Ref": "InstanceIpv6Address"},
            "fragment": {
              "Ipv6Address": {"Ref": "value"}
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
          collection: !Ref InstanceIpv6Address
          fragment:
            Ipv6Address: !Ref value
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
          "collection": ["ami-1", "ami-2", "ami-3"],
          "fragment": {
            "Type": "AWS::EC2::Instance",
            "Properties": {
              "InstanceType": "m1.small",
              "ImageId": {
                "Ref": "value"
              }
            }
          },
          "key": {
            "Fn::Sub": "Instance${index}"
          }
        }
      }
    ],
    "Outputs": {
      "Fn::Merge": [
        {
          "Fn::Map": {
            "collection": ["Instance0", "Instance1"],
            "fragment": {
              "Description": {
                "Fn::Sub": "Instance Id for ${value}"
              },
              "Value": {
                "Fn::Ref": {
                  "Fn::Ref": "value"
                }
              }
            },
            "key": {
              "Fn::Sub": "InstanceId${index}"
            }
          }
        },
        {
          "PrivateIps": {
            "Fn::Map": {
              "collection": ["Instance0", "Instance1"],
              "fragment": {
                "Description": {"Fn::Sub": "Private IP for ${x}"},
                "Value": {
                  "Fn::GetAtt": [
                    {
                      "Fn::Ref": "value"
                    },
                    "PrivateIp"
                  ]
                }
              },
              "key": {
                "Fn::Sub": "PrivateIp${index}"
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
        collection:
          - ami-1
          - ami-2
          - ami-3
        fragment:
          Type: AWS::EC2::Instance
          Properties:
            InstanceType: m1.small
            ImageId:
              Ref: value
        key:
          Fn::Sub: Instance${index}
  Outputs:
    Fn::Merge:
      - Fn::Map:
          collection:
            - Instance0
            - Instance1
          fragment:
            Description:
              Fn::Sub: Instance Id for ${value}
            Value:
              Fn::Ref:
                Fn::Ref: value
          key:
            Fn::Sub: InstanceId${i}
      - PrivateIps:
          Fn::Map:
            collection:
              - Instance0
              - Instance1
            fragment:
              Description:
                Fn::Sub: Private IP for ${value}
              Value:
                Fn::GetAtt:
                  - Fn::Ref: value
                  - PrivateIp
            key:
              Fn::Sub: PrivateIp${i}
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
              "collection": ["ami-1", "ami-2", "ami-3"],
              "fragment": {
                "Type": "AWS::EC2::Instance",
                "Properties": {
                  "InstanceType": "m1.small",
                  "ImageId": {"Ref": "value"}
                }
              },
              "key": {"Fn::Sub": "Instance${index}"}
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
        collection:
          - ami-1
          - ami-2
          - ami-3
        fragment:
          Type: AWS::EC2::Instance
          Properties:
            InstanceType: m1.small
            ImageId: !Ref value
        key:
          Fn::Sub: Instance${index}
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
        collection: !Ref AmiIds
        fragment:
          Type: AWS::EC2::Instance
          Properties:
            InstanceType: m1.small
            ImageId: !Ref value
        key:
          Fn::Sub: Instance${index}
  Bucket:
    Type: AWS::S3::Bucket
    Properties:
      Name: "bucket"
```

#### Logical Id Customization
* You are required to customize Names (Logical Ids) of Resource and Outputs. Specify the key parameter by using Fn::Sub with Index or Value. Make sure the resulting Logical Ids are unique within the template and alphanumerical.
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
        collection: !Ref AmiIds
        fragment:
          Type: AWS::EC2::Instance
          Properties:
            InstanceType: m1.small
            ImageId: !Ref value
        key:
          Fn::Sub: Instance${index}
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
        collection: !Ref AmiIds
        fragment:
          Type: AWS::EC2::Instance
          Properties:
            ImageId: !Ref value
        key: !Sub Instance${index}
```

#### Customize value and index variable names
The optional index and value parameters can be used to customize the corresponding variable names
```yaml
Parameters:
  AmiIds:
    Type: AWS::SSM::Parameter::Value<List<String>>
Resources:
  Fn::Merge:
    - Fn::Map:
        index: amiCounter
        value: amiId
        collection: !Ref AmiIds
        fragment:
          Type: AWS::EC2::Instance
          Properties:
            ImageId: !Ref amiId
        key: !Sub Instance${amiCounter}
```

#### Nested Fn::Map
* Using customized names for index and value is particularly useful when nesting declarations of Fn::Map
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
        index: i1
        value: x1
        collection: !Ref InstanceSizes
        key: !Sub "Instance${i1}"
        fragment:
          Type: AWS::EC2::Instance
          Properties:
            InstanceType: !Ref x1
            Ipv6Addresses:
              - Fn::Map:
                  index: i2
                  value: x2
                  collection: !Ref Ipv6Addresses
                  fragment:
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
        index: i1
        value: x1
        collection: !Ref Subnets
        key: !Sub "Instance${i1}"
        fragment:
          Type: AWS::EC2::Instance
          InstanceType: "m1.small"
          SubnetId: !Ref x1
          Tags:
            - Fn::Map:
                index: i2
                value: x2
                collection: !Ref TagValues
                fragment:
                  Key: !Sub "${x1}"
                  Value: !Sub "${x2}"
```

# Potential Follow-up Features

The following features are out of scope of this RFC. Separate RFC could be created in the future if there is customer demand.

### Modules
* `Fn::Map` should support replicating Modules

### Replicate a fragment for certain number of times
* For a number parameter x, replicate a given template block x number of times.

Users can easily work around this limitation by iterating over a hardcoded list:
```yaml
Resources:
  Fn::Merge:
    - Fn::Map:
        collection: [1, 2, 3, 4]
        fragment:
          Type: AWS::EC2::Instance
          Properties:
            ImageId: "ami-123"
        key: !Sub Instance${index}
```

### Zipping
* `Fn::Map` could take in multiple lists and generate an aggregated list of Resources or Outputs.

# Limitations
#### Replicate multiple Resource objects in a single Fn::Map usage
* To replicate a Resource/Output object, a custom key interpolated with an index has to be provided under the `key` parameter. This means an `Fn::Map` loop can be used to replicate only one Resource at a time. To replicate multiple Resources, `Fn::Map` has to be defined for each Resource.

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
              collection: [!Ref SecurityGroup]
              fragment: !Ref value
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
            collection: !Ref NoEchoList
            fragment:
              Endpoint: !Ref value
```
* In this case the value of the collection would be used to replicate topic subscriptions, which means the values of the no echo list would be exposed in the processed template. So NoEcho lists cannot be used to iterate over.
