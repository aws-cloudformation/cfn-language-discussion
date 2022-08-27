# Fn::FindInMap enhancements (default value and intrinsic functions)

* **Original Author(s):**: @minggon
* **Tracking Issue**: [Tracking Issue 43](https://github.com/aws-cloudformation/cfn-language-discussion/issues/43) [Tracking Issue 73](https://github.com/aws-cloudformation/cfn-language-discussion/issues/73) [Tracking Issue 79](https://github.com/aws-cloudformation/cfn-language-discussion/issues/79)

# Summary

`Fn::FindInMap` provides flexibility to customers looking to generalize templates across different attributes that can be specified with parameters. After declaring appropriate key-value mappings in the `Mappings` section, users can supply a `MapName`, `TopLevelKey`, and `SecondLevelKey` to `Fn::FindInMap` to get the corresponding value from the `Mappings` section in the template. 

We will add support for using other intrinsic functions, including `Fn::Select`, `Fn::Join`, `Fn::Sub`, and `Fn::If` within `Fn::FindInMap`. We will also add an optional default value for `Fn::FindInMap`, which will be the output of the `Fn::FindInMap` if the mapping is not present. The first three parameters of `Fn::FindInMap` will remain the same as before, while the optional fourth parameter `DefaultValue` will be of type `String` or `List`.

# Motivation

## Use of other intrinsic functions:

CloudFormation users may want to use intrinsic functions such as `Fn::Select`, `Fn::Join`, `Fn::Sub`, and `Fn::If` within `Fn::FindInMap`. While CloudFormation users can currently nest `Fn::Ref` and `Fn::FindInMap` in `Fn::FindInMap`, more function support has been requested to give CloudFormation users more flexibility when using `Fn::FindInMap`.

* Reference:
    * https://github.com/aws-cloudformation/cfn-language-discussion/issues/73
    * https://github.com/aws-cloudformation/cfn-language-discussion/issues/79

## Default Value:

CloudFormation users may want to reuse their templates across many different use cases, such as different regions. Mappings and `Fn::FindInMap` allow users to switch between variables via parameters, but if a Mapping cannot be found using the provided `MapName`, `TopLevelKey`, and `SecondLevelKey`, an error will be thrown. Users must specify all possible permutations of values in a Mapping, which can result in a significant amount of duplicate code. 

An example is as follows. 

### YAML
```yaml
Mappings:
  DNS:
    us:
      dns: mypage.us.com
      ttl: '600'
    ca:
      dns: mypage.us.com
      ttl: '600'
    mx:
      dns: mypage.default.com
      ttl: '300'
    ar:
      dns: mypage.default.com
      ttl: '300'
    br:
      dns: mypage.default.com
      ttl: '300'

!FindInMap [DNS, !Ref country, dns]
```

### JSON
```json
{
    "Mappings" : {
        "DNS": {
            "us" : {"dns" : "mypage.us.com", "ttl" : "600"},
            "ca" : {"dns" : "mypage.us.com", "ttl" : "600"},
            "mx" : {"dns" : "mypage.default.com", "ttl" : "300"},
            "ar" : {"dns" : "mypage.default.com", "ttl" : "300"},
            "br" : {"dns" : "mypage.default.com", "ttl" : "300"}
        }
    }
}

{ "Fn::FindInMap" : [ "DNS", { "Ref" : "country" }, "dns" ]}
```

The desired behavior would be something like this:

```yaml
!If
- !MappingPresent [Mapping, 'foo', 'bar']
- !FindInMap [Mapping, 'foo', 'bar']
- 'MyDefaultValue'
```

The above example could then be simplified as follows.

### YAML
```yaml
Mappings:
  DNS:
    us:
      dns: mypage.us.com
      ttl: '600'
    ca:
      dns: mypage.us.com
      ttl: '600'

!FindInMap [DNS, !Ref country, dns, mypage.default.com]
```

### JSON
```json
{
    "Mappings" : {
        "DNS": {
            "us" : {"dns" : "mypage.us.com", "ttl" : "600"},
            "ca" : {"dns" : "mypage.us.com", "ttl" : "600"}
        }
    }
}

{ "Fn::FindInMap" : [ "DNS", { "Ref" : "country" }, "dns", "mypage.default.com"]}
```

A current workaround can be found here (https://stackoverflow.com/questions/35904204/default-value-on-mappings-aws-cloudformation).

This RFC seeks to create an easier built-in way for customers to specify a default value if a Mapping cannot be found. 

* Reference:
    * https://github.com/aws-cloudformation/cfn-language-discussion/issues/43
    * https://stackoverflow.com/questions/35904204/default-value-on-mappings-aws-cloudformation

# Examples

## Use of other intrinsic functions:

Using Fn::Select in Fn::FindInMap. 

### YAML
```yaml
Parameters:

  AsymmetricRSAKeyUsage:
    Type: String
    AllowedValues:
      - ENCRYPT_DECRYPT
      - SIGN_VERIFY
    Default: ENCRYPT_DECRYPT

  KeySpec:
    Type: String
    AllowedValues:
      - ECC_NIST_P256
      - ECC_NIST_P384
      - ECC_NIST_P521
      - ECC_SECG_P256K1
      - HMAC_224
      - HMAC_256
      - HMAC_384
      - HMAC_512
      - RSA_2048
      - RSA_3072
      - RSA_4096
      - SYMMETRIC_DEFAULT
    Default: SYMMETRIC_DEFAULT

Conditions:
  IsKeyAsymmetricRSA: !Equals ['RSA', !Select [0, !Split [_, !Ref KeySpec]]]

Mappings:
  KeyPrefix:
    ECC:
      usage: SIGN_VERIFY
    HMAC:
      usage: GENERATE_VERIFY_MAC
    SYMMETRIC:
      usage: ENCRYPT_DECRYPT

Resources:
  Key:
    Type: AWS::KMS::Key
    Properties:
      KeyUsage: !If
        - IsKeyAsymmetricRSA
        - !Ref AsymmetricRSAKeyUsage
        - !FindInMap [KeyPrefix, !Select [0, !Split [_, !Ref KeySpec]], usage]
```

### JSON
```json
{
    "Parameters": {
        "AsymmetricRSAKeyUsage": {
            "Type": "String",
            "AllowedValues": [
                "ENCRYPT_DECRYPT",
                "SIGN_VERIFY"
            ],
            "Default": "ENCRYPT_DECRYPT"
        },
        "KeySpec": {
            "Type": "String",
            "AllowedValues": [
                "ECC_NIST_P256",
                "ECC_NIST_P384",
                "ECC_NIST_P521",
                "ECC_SECG_P256K1",
                "HMAC_224",
                "HMAC_256",
                "HMAC_384",
                "HMAC_512",
                "RSA_2048",
                "RSA_3072",
                "RSA_4096",
                "SYMMETRIC_DEFAULT"
            ],
            "Default": "SYMMETRIC_DEFAULT"
        }
    },
    "Conditions": {
        "IsKeyAsymmetricRSA": {
            "Fn::Equals": [ "RSA", { "Fn::Select": [ 0, { "Fn::Split": [ "_", { "Ref": "KeySpec" } ] } ] } ]
        }
    },
    "Mappings": {
        "KeyPrefix": {
            "ECC": {
                "usage": "SIGN_VERIFY"
            },
            "HMAC": {
                "usage": "GENERATE_VERIFY_MAC"
            },
            "SYMMETRIC": {
                "usage": "ENCRYPT_DECRYPT"
            }
        }
    },
    "Resources": {
        "Key": {
            "Type": "AWS::KMS::Key",
            "Properties": {
                "KeyUsage": {
                    "Fn::If": [
                        "IsKeyAsymmetricRSA",
                        { "Ref": "AsymmetricRSAKeyUsage" },
                        { "Fn::FindInMap": [ "KeyPrefix", { "Fn::Select": [ 0, { "Fn::Split": [ "_", { "Ref": "KeySpec" } ] } ] }, "usage" ] }
                    ]
                }
            }
        }
    }
}
```

Using Fn::Select, Fn::Sub, and Fn::Join in Fn::FindInMap.

### YAML
```yaml
Parameters:
  SecondKey:
    Type: String
    Default: HVM64
Mappings:
  RegionMap:
    us-east-1:
      HVM64: ami-0ff8a91507f77f867
      HVMG2: ami-0a584ac55a7631c0c
    us-west-1:
      HVM64: ami-0bdb828fd58c52235
      HVMG2: ami-066ee5fd4a9ef77f1
    eu-west-1:
      HVM64: ami-047bb4163c506cd98
      HVMG2: ami-0a7c483d527806435
    ap-southeast-1:
      HVM64: ami-08569b978cc4dfa10
      HVMG2: ami-0be9df32ae9f92309
    ap-northeast-1:
      HVM64: ami-06cd52961ce9f0d85
      HVMG2: ami-053cdd503598e4a9d
Resources:
  myEC2Instance:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: !FindInMap
        - !Select
          - 1
          - - other
            - RegionMap
        - !Join
          - '-'
          - - us
            - east
            - '1'
        - !Sub '${SecondKey}'
      InstanceType: t2.small
```

### JSON
```json
{
    "Parameters": {
        "SecondKey": {
            "Type": "String",
            "Default": "HVM64"
        }
    },
    "Mappings" : {
      "RegionMap" : {
        "us-east-1" : {
          "HVM64" : "ami-0ff8a91507f77f867", "HVMG2" : "ami-0a584ac55a7631c0c"
        },
        "us-west-1" : {
          "HVM64" : "ami-0bdb828fd58c52235", "HVMG2" : "ami-066ee5fd4a9ef77f1"
        },
        "eu-west-1" : {
          "HVM64" : "ami-047bb4163c506cd98", "HVMG2" : "ami-0a7c483d527806435"
        },
        "ap-southeast-1" : {
          "HVM64" : "ami-08569b978cc4dfa10", "HVMG2" : "ami-0be9df32ae9f92309"
        },
        "ap-northeast-1" : {
          "HVM64" : "ami-06cd52961ce9f0d85", "HVMG2" : "ami-053cdd503598e4a9d"
        }
      }
    },
    "Resources" : {
      "myEC2Instance" : {
        "Type" : "AWS::EC2::Instance",
        "Properties" : {
          "ImageId" : {
            "Fn::FindInMap" : [
                { "Fn::Select": [
                    1,
                    [
                        "other",
                        "RegionMap"
                    ]
                ]},
                { "Fn::Join": [
                    "-", 
                    [
                      "us",
                      "east",
                      "1"
                    ]
                  ]
                },
                { "Fn::Sub": "${SecondKey}" }
            ]
          },
          "InstanceType" : "t2.small"
        }
      }
    }
  }
```

Nesting Fn::If in Fn::FindInMap

### YAML
```yaml
Conditions:
  conditionName: !Equals
    - 1
    - 1
Parameters:
  SecondKey:
    Type: String
    Default: HVM64
Mappings:
  RegionMap:
    us-east-1:
      HVM64: ami-0ff8a91507f77f867
      HVMG2: ami-0a584ac55a7631c0c
    us-west-1:
      HVM64: ami-0bdb828fd58c52235
      HVMG2: ami-066ee5fd4a9ef77f1
    eu-west-1:
      HVM64: ami-047bb4163c506cd98
      HVMG2: ami-0a7c483d527806435
    ap-southeast-1:
      HVM64: ami-08569b978cc4dfa10
      HVMG2: ami-0be9df32ae9f92309
    ap-northeast-1:
      HVM64: ami-06cd52961ce9f0d85
      HVMG2: ami-053cdd503598e4a9d
Resources:
  myEC2Instance:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: !FindInMap
        - RegionMap
        - !If
          - conditionName
          - us-east-1
          - us-east-2
        - !Ref 'SecondKey'
      InstanceType: t2.small
```

### JSON
```json
{
    "Conditions": {
        "conditionName": {
            "Fn::Equals": [
                1,
                1
            ]
        }
    },
    "Parameters": {
        "SecondKey": {
            "Type": "String",
            "Default": "HVM64"
        }
    },
    "Mappings": {
        "RegionMap": {
            "us-east-1": {
                "HVM64": "ami-0ff8a91507f77f867",
                "HVMG2": "ami-0a584ac55a7631c0c"
            },
            "us-west-1": {
                "HVM64": "ami-0bdb828fd58c52235",
                "HVMG2": "ami-066ee5fd4a9ef77f1"
            },
            "eu-west-1": {
                "HVM64": "ami-047bb4163c506cd98",
                "HVMG2": "ami-0a7c483d527806435"
            },
            "ap-southeast-1": {
                "HVM64": "ami-08569b978cc4dfa10",
                "HVMG2": "ami-0be9df32ae9f92309"
            },
            "ap-northeast-1": {
                "HVM64": "ami-06cd52961ce9f0d85",
                "HVMG2": "ami-053cdd503598e4a9d"
            }
        }
    },
    "Resources": {
        "myEC2Instance": {
            "Type": "AWS::EC2::Instance",
            "Properties": {
                "ImageId": {
                    "Fn::FindInMap": [
                        "RegionMap",
                        { "Fn::If": [ "conditionName", "us-east-1", "us-east-2" ] },
                        { "Ref": "SecondKey" }
                    ]
                },
                "InstanceType": "t2.small"
            }
        }
    }
}
```

## Default Value

Using Fn::FindInMap with the DefaultValue parameter

### YAML
```yaml
Mappings:
  RegionMap:
    us-east-1:
      HVM64: ami-0ff8a91507f77f867
      HVMG2: ami-0a584ac55a7631c0c
    us-west-1:
      HVM64: ami-0bdb828fd58c52235
      HVMG2: ami-066ee5fd4a9ef77f1
    eu-west-1:
      HVM64: ami-047bb4163c506cd98
      HVMG2: ami-0a7c483d527806435
    ap-southeast-1:
      HVM64: ami-08569b978cc4dfa10
      HVMG2: ami-0be9df32ae9f92309
    ap-northeast-1:
      HVM64: ami-06cd52961ce9f0d85
      HVMG2: ami-053cdd503598e4a9d
Resources:
  myEC2Instance:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: !FindInMap
        - RegionMap
        - us-east-1
        - not-found
        - ami-0ff8a91507f77f867
      InstanceType: t2.small
```

### JSON
```json
{
  "Mappings": {
    "RegionMap": {
      "us-east-1": {
        "HVM64": "ami-0ff8a91507f77f867",
        "HVMG2": "ami-0a584ac55a7631c0c"
      },
      "us-west-1": {
        "HVM64": "ami-0bdb828fd58c52235",
        "HVMG2": "ami-066ee5fd4a9ef77f1"
      },
      "eu-west-1": {
        "HVM64": "ami-047bb4163c506cd98",
        "HVMG2": "ami-0a7c483d527806435"
      },
      "ap-southeast-1": {
        "HVM64": "ami-08569b978cc4dfa10",
        "HVMG2": "ami-0be9df32ae9f92309"
      },
      "ap-northeast-1": {
        "HVM64": "ami-06cd52961ce9f0d85",
        "HVMG2": "ami-053cdd503598e4a9d"
      }
    }
  },
  "Resources": {
    "myEC2Instance": {
      "Type": "AWS::EC2::Instance",
      "Properties": {
        "ImageId": {
          "Fn::FindInMap": [
            "RegionMap",
            "us-east-1",
            "not-found",
            "ami-0ff8a91507f77f867"
          ]
        },
        "InstanceType": "t2.small"
      }
    }
  }
}
```

# Limitations

Users will not be able to nest `Fn::Split` within `Fn::FindInMap` to specify all parameters of `Fn::FindInMap`. Three or four parameters are required in `Fn::FindInMap`.
The following usage of `Fn::Split` within `Fn::FindInMap` is invalid. 

### YAML
```yaml
!FindInMap [ !Split [ ".", "RegionMap.us-east-1.HVM64" ] ]
```

### JSON
```json
"Fn::FindInMap": [ { "Fn::Split": [ ".", "RegionMap.us-east-1.HVM64" ] } ]
```

If users use a reference to a resource property as a key in `Fn::FindInMap`, the `DefaultValue` will be returned, provided that the `DefaultValue` is supplied and does not reference a resource property.

# Details

`Fn::FindInMap` (https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference-findinmap.html) is an intrinsic function that takes a `String` representing the `MapName`, a String representing the `TopLevelKey`, a String representing the `SecondLevelKey`, and optionally, a `String` or `List` representing the `DefaultValue`. 

The `DefaultValue` could be 

* a hardcoded `String` in the template
* a hardcoded `List` in the template
* `Ref` to a parameter of type `List` or `String` 
* A `Fn::FindInMap` 
* An intrinsic function returning a `List` or `String`

If the mapping can be found, it returns the value of the mapping as before, while the `DefaultValue` is returned if the mapping does not exist. 

Within `Fn::FindInMap`, the following intrinsic functions with parameters of hardcoded values or of `Ref` to template parameters will be supported:

* `Fn::FindInMap`
* `Fn::Join`
* `Fn::Select`
* `Fn::Sub`
* `Fn::If`
* `Fn::Split` (only for the `DefaultValue` parameter)

Backwards compatibility is maintained by making the `DefaultValue` optional. CloudFormation users will not be required to use other intrinsic functions in `Fn::FindInMap`, they can simply use Strings as they did before. 

Appendix

* https://github.com/aws-cloudformation/cfn-language-discussion/issues/73
* https://github.com/aws-cloudformation/cfn-language-discussion/issues/79
* https://github.com/aws-cloudformation/cfn-language-discussion/issues/43
* https://stackoverflow.com/questions/35904204/default-value-on-mappings-aws-cloudformation

