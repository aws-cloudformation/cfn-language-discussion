# New Intrinsic Function for looping: `Fn::For`

* **Original Author(s):**: @mingujo
* **Tracking Issue**: [Tracking Issue: Fn::For](https://github.com/aws-cloudformation/cfn-language-discussion/issues/9)

# Summary
We introduce `Fn::For` intrinsic function. You can iterate a given list to replicate a certain template snippet thereby avoiding repetitively writing a similar declaration for each object (e.g. Resource, Output, or Resource Properties) just like using a loop idiom in any programming languages.

`Fn::For` traverses a single given List of Strings from first to last, and over each iteration, it references each item under a given template snippet to be replicated.

# Motivation

In CloudFormation template, a single Resource configures one infrastructure object. Therefore, a template can become verbose when you manually declare multiple similar resources. Some examples would include a pool of EC2 instances with the same configurations but different instance types, or S3 Bucket Notification configurations with different SNS Topics.

# Declaration
### Json
```
{
    "Fn::For" : [
        ["Index", "Value"],
        "CollectionToIterate",
        "TemplateSnippet",
        "LogicalId"
    ]
}
```

### Yaml

For the full function name
```
Fn::For:
    - ["Index", "Value"],
    - "CollectionToIterate",
    - "TemplateSnippet"
    - "LogicalId"
```

For the short form
```
!For:
    - ["Index", "Value"],
    - "CollectionToIterate",
    - "TemplateSnippet"
    - "LogicalId"
```


# Parameters

Fn::For requires a list of 4 positional parameters.

* `Index`
    * The name of an iterator variable which stores a positional index of each value in a given List.
    * The index number sequence starts from 0.
    * String type
* `Value`
    * The name of an iterator variable which stores each value in a List.
    * String type
* `CollectionToIterate`
    * The collection to iterate on. It should be a List. You can either in-place the collection itself or reference List type parameter values from Parameters section.
* `TemplateSnippet`
    * The template snippet to replicate for each item in iteration.
* (Conditional) `LogicalId`
    * The logical Id of replicated Resource or Output objects.
    * **This field is only required for usage on a Resource/Output to name replicated Resource/Output objects. It should not be specified when using Fn::For to generate a list type attribute for a Resource Property.**
    * The value **should be interpolated with an index** using `Fn::Sub` intrinsic function.


# Examples

### Usage in Resources Section

#### Use case: Replicate a single Resource
* Notice that **`LogicalId`** positional parameter is provided.
* In Json
    ```
    {
       "AWSTemplateFormatVersion":"2010-09-09",
       "Description":"EC2 Instances with different AMIs",
       "Resources":{
          "Instances":{
             "Fn::For":[
                ["i", "x"],
                ["ami-1", "ami-2", "ami-3"],
                {
                   "Type":"AWS::EC2::Instance",
                   "Properties":{
                      "InstanceType":"m1.small",
                      "ImageId":{
                         "Ref": "x"
                      }
                   }
                },
                {
                   "Fn::Sub": "Instance${i}"
                }
             ]
          }
       }
    }
    ```
* In Yaml
    ```
    AWSTemplateFormatVersion: '2010-09-09'
    Description: "EC2 Instances with different AMIs"
    Resources: 
      Instances:
        Fn::For:
          - ["i", "x"]
          - ["ami-1", "ami-2", "ami-3"]
          - Type: AWS::EC2::Instance
            Properties:
              InstanceType: "m1.small"
              ImageId: !Ref x
          - !Sub Instance${i}
    ```
* Above templates are equivalent to a below template in Yaml
    ```
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

* In Json
    ```
    {
        "AWSTemplateFormatVersion": '2010-09-09',
        "Description": "EC2 Instances with different AMIs",
        "Resources": {
            "Vpcs": {
                "Fn::For": [
                    ["i", "x"],
                    ["172.16.0.0/16","172.17.0.0/16","172.18.0.0/16"],
                    {
                        "Type": "AWS::EC2::VPC",
                        "Properties": {
                            "CidrBlock": {
                                "Ref": "x"
                            }
                        }
                    },
                    {
                        "Fn::Sub": "Vpc${i}"
                    }
                ]
            },
            "Subnets": {
                "Fn::For": [
                    ["i", "x"],
                    ["Vpc0", "Vpc1", "Vpc2"]
                    {
                        "Type": "AWS::EC2::Subnet",
                        "Properties": {
                            "VpcId": {
                                "Ref": "x"
                            }
                        }
                    },
                    {
                        "Fn::Sub": "Subnet${i}"
                    }
                ]
            }
        }
    }
    ```
* In Yaml
    ```
    AWSTemplateFormatVersion: '2010-09-09'
    Description: "EC2 Instances with different AMIs"
    Resources: 
      Vpcs:
        Fn::For:
            - ["i", "x"]
            - ["172.16.0.0/16","172.17.0.0/16","172.18.0.0/16"],
            - Type: AWS::EC2::VPC
              Properties: 
                CidrBlock: !Ref x
            - !Sub Vpc${i}
      Subnets:
        Fn::For:
            - ["i", "x"]
            - ["Vpc0", "Vpc1", "Vpc2"],
            - Type: AWS::EC2::Subnet
              Properties:
                VpcId: !Ref Vpc${i}
            - !Sub Subnet${i}
    ```
* Above templates are equivalent to a below template in Yaml
    ```
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
* Notice that **`LogicalId`** positional parameter is not provided.
* In Json
    ```
    {
        "AWSTemplateFormatVersion": '2010-09-09',
        "Description": "EC2 Instance with list of Ipv6Addresses",
        "Parameters": {
            "InstanceIpv6Address": {
                "Type": "CommaDelimitedList"
                "Default": "ipv6-1,ipv6-2,ipv6-3"
            }
        },
        "Resources": {
            "Instance": {
                "Type": "AWS::EC2::Instance",
                "Properties": {
                    "InstanceType": "m1.small",
                    "Ipv6Addresses": [
                        "Fn::For": [
                            ["i", "x"],
                            {"Ref": "InstanceIpv6Address"},
                            {
                                "Ipv6Address": {"Ref": "x"}
                            }
                        ]
                    ]
                }
            }
        }
    }
    ```
* In Yaml
    ```
    AWSTemplateFormatVersion: '2010-09-09'
    Description: "EC2 Instances with Ipv6Addresses"
    Parameters:
      InstanceIpv6Address:
        Type: CommaDelimitedList
        Default: "ipv6-1,ipv6-2,ipv6-3"
    Resources:
      Instance:
        Type: AWS::EC2::Instance
        Properties:
          InstanceType: "m1.small"
          Ipv6Addresses:
            - Fn::For:
              - ["i", "x"]
              - !Ref InstanceIpv6Address 
              - Ipv6Address: !Ref x
    ```
* Above templates are equivalent to a below template in Yaml
    ```
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
    ```
    {
        "AWSTemplateFormatVersion": '2010-09-09',
        "Description": "EC2 Instances with different AMIs",
        "Resources": {
            "Instances":{
                "Fn::For": [
                    ["i", "x"],
                    ["ami-1","ami-2","ami-3"],
                    {
                        "Type": "AWS::EC2::Instance",
                        "Properties": {
                            "InstanceType": "m1.small",
                            "ImageId": {
                                "Ref": "x"
                            }
                        }
                    },
                    {
                        "Fn::Sub": "Instance${i}"
                    }
                ]
            }
        },
        "Outputs": {
            "InstanceIds": {
                "Fn::For": [
                    ["i", "x"],
                    ["Instance0", "Instance1"], // have to be explicitly indicated.
                    {
                        "Description": {
                            "Fn::Sub": "Instance Id for ${x}"
                        },
                        "Value": {
                            "Fn::Ref": {
                                "Fn::Ref": {
                                    "x"
                                }
                            }
                        }
                    },
                    {
                        "Fn::Sub": "InstanceId${i}"
                    }
                ]
            },
            "PrivateIps": {
                "Fn::For": [
                    ["i", "x"],
                    ["Instance0", "Instance1"], // have to be explicitly indicated.
                    {
                        "Description": !Sub "Private IP for ${x}",
                        "Value": {
                            "Fn::GetAtt": [
                                {
                                    "Fn::Ref": "x"
                                },
                                "PrivateIp"
                            ]
                        }
                    },
                    {
                        "Fn::Sub": "PrivateIp${i}"
                    } 
                ]
            }
        }
    }
    ```
* In Yaml
    ```
    AWSTemplateFormatVersion: '2010-09-09'
    Description: "EC2 Instances with different AMIs"
    Resources: 
      Instances:
        Fn::For:
            - ["i", "x"]
            - ["ami-1","ami-2","ami-3"]
            - Type: AWS::EC2::Instance
              Properties:
                InstanceType: "m1.small"
                ImageId: !Ref x
            - !Sub Instance${i}
    Outputs:
      InstanceIds:
        Fn::For:
          - ["i", "x"]
          - ["Instance0", "Instance1"]
          - Description: !Sub "Instance Id for ${x}"
            Value:
              Fn::Ref:
                Fn::Ref: x
          - !Sub InstanceId${i}
      PrivateIps:
        Fn::For:
          - ["i", "x"]
          - ["Instance0", "Instance1"]
          - Description: !Sub "Private IP for ${x}"
            Value:
              Fn::GetAtt:
                - !Ref: x
                - PrivateIp
          - !Sub PrivateIp${i}
    ```
* Above templates are equivalent to a below template in Yaml
    ```
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
    ```
    {
        "AWSTemplateFormatVersion": '2010-09-09',
        "Description": "EC2 Instances with different AMIs",
        "Resources": {
            "Instances": {
                "Fn::For": [
                    ["i", "x"],
                    ["ami-1","ami-2","ami-3"],
                    {
                        "Type": "AWS::EC2::Instance",
                        "Properties": {
                            "InstanceType": "m1.small",
                            "ImageId": {
                                "Ref": "x"
                            }
                        }
                    },
                    {
                        "Fn::Sub": "Instance${i}"
                    }
                ]
            }
        },
        "Outputs": {
            "SecondInstanceId": {
                "Description": "Instance Id for Instance1"
                "Value": {
                    "Fn::Ref": "Instance1"
                }
            },
            "SecondPrivateIp": {
                "Description": "Private ip for Instance1"
                "Value": {
                    "Fn::GetAtt": [
                        "Instance1",
                        "PrivateIp"
                    ]
                }
            }
        }
    }
    ```
* In Yaml
    ```
    AWSTemplateFormatVersion: '2010-09-09'
    Description: "EC2 Instances with different AMIs"
    Resources:
      Instances:
        Fn::For:
          - ["i", "x"]
          - ["ami-1,ami-2,ami-3"]
          - Type: AWS::EC2::Instance
            Properties:
              InstanceType: "m1.small"
              ImageId: !Ref x
          - !Sub Instance${i}
    Outputs:
      SecondInstanceId: 
        Description: "Instance Id for Instance1"
        Value: !Ref Instance1
      SecondPrivateIp:
        Description: "Private ip for Instance1"
        Value: !GetAtt Instance1.PrivateIp
    ```
* Above templates are equivalent to a below template in Yaml
    ```
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

# Properties of `Fn::For`

#### Reference source list from Parameters
* You can reference a source list from Parameters
    ```
    AWSTemplateFormatVersion: '2010-09-09'
    Description: "EC2 Instances with different AMIs"
    Parameters:
      AmiIds:
        Type: CommaDelimitedList
        Default: "ami-1,ami-2,ami-3"
    Resources:
      Instances:
        Fn::For:
          - ["i", "x"]
          - !Ref AmiIds
          - Type: AWS::EC2::Instance
            Properties:
              InstanceType: "m1.small"
              ImageId: !Ref x
          - !Sub Instance${i}
      Bucket: 
        Type: AWS::S3::Bucket
        Properties:
          Name: "bucket"
    ```

#### Logical Id Customization
* You are required to customize Names (Logical Ids) of Resource and Outputs using Index
    ```
    AWSTemplateFormatVersion: '2010-09-09'
    Description: "EC2 Instances with different AMIs"
    Parameters:
      AmiIds:
        Type: CommaDelimitedList
        Default: "ami-1,ami-2,ami-3"
    Resources:
      Instances:
        Fn::For:
          - ["i", "x"]
          - !Ref AmiIds
          - Type: AWS::EC2::Instance
            Properties:
              InstanceType: "m1.small"
              ImageId: !Ref x
          - !Sub Instance${i}
      Bucket: 
        Type: AWS::S3::Bucket
        Properties:
          Name: "bucket"
    ```
* Caveat: Make sure Logical Ids are alphanumeric characters

#### Enumerate with Index
* Like a general For statement in programming languages, `Fn::For` maintains an explicit counter (positional index) for each item in given list which you can reference it to use within your template snippet
    ```
    Parameters:
      Stages:
        Type: CommaDelimitedList
        Default: "Prod,Gamma,Beta"
    Resources:
      Instance:
        Type: AWS::EC2::Instance
        Properties:
          KeyName: !Ref x
          InstanceType: "m1.small"
          Tags: 
            - Fn::For:
               - ["i", "x"]
               - !Ref Ipv6Addresses
               - Key: !Sub "key${i}"
                 Value: !Ref x
          - !Sub Instance${i1}
    ```

#### SSM/AWS List Type Parameters
* SSM/AWS List type parameters can be referenced in `Fn::For`. Only List type parameter is allowed.
  ```
  Parameters:
    AmiIds:
      Type: AWS::SSM::Parameter::Value<List<String>>
      Default: "TestList"
  Resources:
    Instances:
      Fn::For:
        - ["i", "x"]
        - !Ref AmiIds
        - Type: AWS::EC2::Instance
          Properties:
            ImageId: !Ref x
  ```

#### Nested Fn::For
* There could be the case where you would want to take nested loop approach
    ```
    Parameters:
      Stages:
        Type: CommaDelimitedList
        Default: "Prod,Gamma,Beta"
      Ipv6Addresses:
        Type: CommaDelimitedList
        Default: "ipv6-1,ipv6-2,ipv6-3"
    Resources: 
      Instances:
        Fn::For:
          - ["i1", "x1"]
          - !Ref Stages
          - Type: AWS::EC2::Instance
            Properties:
              KeyName: !Ref x1
              InstanceType: "m1.small"
              Ipv6Addresses: 
                - Fn::For:
                  - ["i2", "x2"]
                  - !Ref Ipv6Addresses
                  - Ipv6Address: !Ref x2
          - !Sub Instance${i1}
    ```
* Above templates are equivalent to a below template
    ```
        Parameters:
          Stages:
            Type: CommaDelimitedList
            Default: "Prod,Gamma,Beta"
          Ipv6Addresses:
            Type: CommaDelimitedList
            Default: "ipv6-1,ipv6-2,ipv6-3"
        Resources:
          Instance0:
            Type: AWS::EC2::Instance
            Properties:
              KeyName: "Prod"
              InstanceType: "m1.small"
              Ipv6Addresses:
                - Ipv6Address: "ipv6-1"
                - Ipv6Address: "ipv6-2"
                - Ipv6Address: "ipv6-3"
          Instance1:
            Type: AWS::EC2::Instance
            Properties:
              KeyName: "Gamma"
              InstanceType: "m1.small"
              Ipv6Addresses:
                - Ipv6Address: "ipv6-1"
                - Ipv6Address: "ipv6-2"
                - Ipv6Address: "ipv6-3"
          Instance2:
            Type: AWS::EC2::Instance
            Properties:
              KeyName: "Beta"
              InstanceType: "m1.small"
              Ipv6Addresses:
                - Ipv6Address: "ipv6-1"
                - Ipv6Address: "ipv6-2"
                - Ipv6Address: "ipv6-3"
    ```
* Another example of using an outer loop’s variable inside inner loop
    ```
        Parameters:
          Stages:
              Type: CommaDelimitedList
              Default: "Prod,Gamma,Beta"
          Ipv6Addresses:
              Type: CommaDelimitedList
              Default: "ipv6-1,ipv6-2,ipv6-3"
        Resources:
          Instances:
              Fn::For:
                  - ["i1", "x1"]
                  - !Ref Stages
                  - Type: AWS::EC2::Instance
                    Properties:
                      KeyName: !Ref x
                      InstanceType: "m1.small"
                      Tags: 
                        - Fn::For:
                          - ["i2", "x2"]
                          - !Ref Ipv6Addresses
                          - Key: !Sub "${x1}${i2}"
                            Value: !Sub "${x2}"
                  - !Sub Instance${i1}
    ```
* Above templates are equivalent to a below template
    ```
    Parameters:
      Stages:
        Type: CommaDelimitedList
        Default: "Prod,Gamma,Beta"
      Ipv6Addresses:
        Type: CommaDelimitedList
        Default: "ipv6-1,ipv6-2,ipv6-3"
    Resources:
      Instance0:
        Type: AWS::EC2::Instance
        Properties:
          KeyName: "Prod"
          InstanceType: "m1.small"
          Tags:
            - Key: "Prod0"
              Value: "ipv6-1"
            - Key: "Prod1"
              Value: "ipv6-2"
            - Key: "Prod2"
              Value: "ipv6-3"
      Instance1:
        Type: AWS::EC2::Instance
        Properties:
          KeyName: "Gamma"
          InstanceType: "m1.small"
          Tags:
            - Key: "Gamma0"
              Value: "ipv6-1"
            - Key: "Gamma1"
              Value: "ipv6-2"
            - Key: "Gamma2"
              Value: "ipv6-3"
      Instance2:
        Type: AWS::EC2::Instance
        Properties:
          KeyName: "Beta"
          InstanceType: "m1.small"
          Tags:
            - Key: "Beta0"
              Value: "ipv6-1"
            - Key: "Beta1"
              Value: "ipv6-2"
            - Key: "Beta2"
              Value: "ipv6-3"
    ```

# Follow-up Features

Following features are not prioritized, but might potentially be supported in subsequent releases.

### Indexing
* You may want to reference a specific object among iteratively created objects in any place within template. Currently this is not supported yet.
    ```
    Parameters:
       Endpoints:
         Type: CommaDelimitedList
         Default: "endpoint1,endpoint2,endpoint3"
    Resources:
      Topic:
        Fn::For:
          - ["i", "x"]
          - !Ref Endpoints
          - Type: AWS::SNS::Topic
            Properties:
              Subscription: 
                - Endpoint: !Ref x
    Outputs:
      TopicOutput:
        Value: !Select [1, Topic]
    ```

### Modules
* `Fn::For` should also work on Module type Resource seamlessly. This will be supported in subsequent releases.

### Replicate a template snippet for certain number of times
* If parameter was a number, then replicate a given template block x number of times.

### Iterate on Map type collection
* This feature will be supported once JSON type parameter is allowed
    ```
    AWSTemplateFormatVersion: '2010-09-09'
    Description: "EC2 Instances with different AMIs"
    Parameters:
      InstanceTypeAmiIdMap:
        Type: JSON
        Default: '{"m1.small": "ami-1", "t2.micro": "ami-2"}'
    Resources: 
      Instances:
        Fn::For:
          - ["key", "value"]
          - !Ref InstanceTypeAmiIdMap
          - Type: AWS::EC2::Instance
            Properties:
              InstanceType: !Ref key
              ImageId: !Ref value
          - !Sub Instance${i}
    ```
### Zipping
* You will be able to “zip” lists with `Fn::For`. You can iterate two or more lists with same length to aggregate iterator values.

# Limitations
#### Replicate multiple Resource objects in a single Fn::For usage
* To replicate a Resource/Output object, a custom Logical Id interpolated with an index has to be provided under `LogicalId` positional parameter. This means a Fn::For loop can be used to replicate only one Resource at a time. To replicate multiple Resources, Fn::For has to be defined for each Resource.
#### A Collection to Iterate on has to have known values
* All the values in a list have to be *known* values, otherwise it will throw an error message that Fn::For received a value which cannot be determined before deploying a stack. You can’t refer to an attribute of a Resource which hasn’t been deployed yet. Value must be known before CFN performs any remote resource interactions.
* For an example, below would not work
    ```
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
              - Fn::For:
                 - ["i", "x"]
                 - !Ref SecurityGroup
                 - !Ref x
    ```
* This is not working example.

#### You can’t reference NoEcho Parameter
* NoEcho Parameter value cannot be used as an argument to Fn::For.
  ```
  Parameters:
     SomeList:
       Type: CommaDelimitedList
       NoEcho: True
  Resources:
    ResourceReferringBucket:
       Type: AWS::SNS::Topic
       Properties:
          Subscription:
            - Fn::For:
              - ["i", "x"]
              - !Ref SomeList
              - Endpoint: !Ref x
  ```
* The value used in Fn::For is used to replicate a resource and will eventually be disclosed on Console, which is why NoEcho values are not allowed.

# FAQ
## Will this support a general loop like `for( idx: 0; idx < 10; idx++)`?
* No, iterating up to certain position in list is not supported.

# Appendix
## Not selected syntax candidate

* We separate this `Fn::For` function into two for two distinct usages.
    * Usage in Resources / Outputs section to replicate Resource and Output objects (Fn::ForObject)
    * Usage in Property of Resource to define a list type property of Resource (Fn::For)
        * Pros:
            * We can remove an optional parameter
            * Function behaves consistently regardless of an optional parameter and where the function is declared in template just as how other native intrinsic functions are modeled.
        * Cons:
            * We offer two distinct functions that provides very similar looping functionality simply due to key collision limitation. A user can perceive it as not intuitive.
            * Naming of functions is not straightforward
            * `Fn::ForObject` has to be declared underneath Logical Id
