# Fn::FindInMap enhancements (default value and additional intrinsic functions support)

* **Original Author(s):**: @minggon, @lejiati
* **Tracking Issue**: [#101](https://github.com/aws-cloudformation/cfn-language-discussion/issues/101)


## Summary

`Fn::FindInMap` provides flexibility to customers looking to generalize templates across different attributes that can be specified with parameters. After declaring appropriate key-value mappings in the `Mappings` section, users can supply a `MapName`, `TopLevelKey`, and `SecondLevelKey` to `Fn::FindInMap` to get the corresponding value from the `Mappings` section in the template. 

We will add support for using other intrinsic functions, including `Fn::Select`, `Fn::Join`, `Fn::Sub`, and `Fn::If` within `Fn::FindInMap`. We will also add an optional default value for `Fn::FindInMap`, which will be the output of the `Fn::FindInMap` if the mapping is not present. The first three parameters of `Fn::FindInMap` will remain the same as before, while the optional named parameter `DefaultValue` will be of type `String` or `List`.


## Motivation

### Use of other intrinsic functions

Currently CloudFormation users can use `Fn::Ref` and `Fn::FindInMap` in `Fn::FindInMap`, more function support has been requested to give CloudFormation users more flexibility when using `Fn::FindInMap`. Here are some customer requests:

* **[Enable use of additional intrinsic functions within Fn::FindInMap](https://github.com/aws-cloudformation/cfn-language-discussion/issues/73)**
* **[Fn::FindInMap support Fn conditions](https://github.com/aws-cloudformation/cfn-language-discussion/issues/79)**

This RFC seeks to enable the usage of other intrinsic functions within `Fn::FindInMap` function providing more flexibility in template authoring. 

### Default value

CloudFormation users may want to reuse their templates across many different use cases, such as different regions. Mappings and `Fn::FindInMap` allow users to switch between variables via parameters, but if a Mapping cannot be found using the provided `MapName`, `TopLevelKey`, and `SecondLevelKey`, an error will be thrown. Users must specify all possible permutations of values in a Mapping, which can result in a significant amount of duplicate code. 

Here is an example where DNS varies from country to country

```
AWSTemplateFormatVersion: 2010-09-09
Parameters:
  country:
    Type: String
Mappings:
  DNS:
    usa:
      dns: mypage.com
      ttl: '600'
    canada:
      dns: mypage.ca
      ttl: '600'
    norway:
      dns: mypage.com
      ttl: '300'
    germany:
      dns: mypage.de
      ttl: '300'
    iceland:
      dns: mypage.com
      ttl: '300'
    finland:
      dns: mypage.com
      ttl: '300'
Resources:
  DemoApiGateway:
    Type: 'AWS::ApiGatewayV2::Api'
    Properties:
      Name: Demo API Gateway
      ProtocolType: HTTP
  DemoApiStage:
    Type: 'AWS::ApiGatewayV2::Stage'
    Properties:
      ApiId: !Ref DemoApiGateway
      StageName: live
      Description: Live Stage
      AutoDeploy: true
  DemoApiDomainName:
    Type: 'AWS::ApiGatewayV2::DomainName'
    Properties:
      DomainName: !FindInMap 
        - DNS
        - !Ref country
        - dns
```

There are 2 deficiencies with this approach:

1. DNS mapping will keep growing as more countries are added.
2. Input parameter `country` has to match at least one entry, or the whole stack operation will fail.

There is a [work around](https://stackoverflow.com/questions/35904204/default-value-on-mappings-aws-cloudformation) to set default value for mapping but it is verbose and clumsy. This RFC seeks to create an easier built-in way for customers to specify a default value if a Mapping cannot be found.

## Details

### Use of other intrinsic functions in `Fn::FindInMap`

Within `Fn::FindInMap` the following intrinsic functions with parameters or hardcoded values or of `Ref` to template parameters will be supported. 

* `Fn::FindInMap`
* `Fn::Join`
* `Fn::Sub`
* `Fn::If`
* `Fn::Select`
* `Fn::Length`
* `Fn::ToJsonString`
* `Fn::Split` - Unless it’s used for the default value, `Fn::Split` has to be used in conjunction with intrinsic functions that produce a string, such as `Fn::Join` or `Fn::Select`.

### Default value in `Fn::FindInMap`

[Fn::FindInMap](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference-findinmap.html) is an intrinsic function that takes a `String` representing the `MapName`, a `String` representing the `TopLevelKey`, a `String` representing the `SecondLevelKey`, and with this new RFC, user can optionally supply a `String` or `List` representing the `DefaultValue`.

The `DefaultValue` could be

* A hardcoded `string` in the template
* A hardcoded `List` in the template
* `Ref` to a parameter of type `List` or `String`
* A `Fn::FindInMap`
* An intrinsic function returning a `List`, `String` or `AWS::NoValue`

If the mapping can be found, it returns the value of the mapping as before, while the `DefaultValue` is returned if the mapping does not exist. Backwards compatibility is maintained by making the `DefaultValue` optional. CloudFormation users will not be required to use other intrinsic functions in `Fn::FindInMap`, they can simply use Strings as they did before.

## Example

### Use of other intrinsic functions in `Fn::FindInMap`

Here is an example using `Fn::Select` and `Fn::Split` in `Fn::FindInMap` - [credit](https://github.com/aws-cloudformation/cfn-language-discussion/issues/73)

```
Transform: AWS::LanguageExtensions
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

### Default value

#### Example 1

Here is an example that simplifies the use case we discussed before

```
AWSTemplateFormatVersion: 2010-09-09
Transform: AWS::LanguageExtensions
Parameters:
  Country:
    Type: String
Mappings:
  DNS:
    usa:
      dns: mypage.com
      ttl: '600'
    canada:
      dns: mypage.ca
      ttl: '600'
    germany:
      dns: mypage.de
      ttl: '300'
Resources:
  DemoApiGateway:
    Type: 'AWS::ApiGatewayV2::Api'
    Properties:
      Name: Demo API Gateway
      ProtocolType: HTTP
  DemoApiStage:
    Type: 'AWS::ApiGatewayV2::Stage'
    Properties:
      ApiId: !Ref DemoApiGateway
      StageName: live
      Description: Live Stage
      AutoDeploy: true
  DemoApiDomainName:
    Type: 'AWS::ApiGatewayV2::DomainName'
    Properties:
      DomainName: !FindInMap 
        - DNS
        - !Ref country
        - dns
        - DefaultValue: !FindInMap 
            - DNS
            - usa
            - dns
```

With default value added, the size of Mapping section is significantly reduced. In addition, there is no need to update template when new countries that uses default DNS are added.

#### Example 2

Here is another example using default value to set EC2 instance type

```
AWSTemplateFormatVersion: 2010-09-09
Transform: AWS::LanguageExtensions
Mapping:
  InstanceConfiguration:
    us-east-1:
      Type: m5.large
    us-west-2:
      Type: m5.large
    eu-west-1:
      Type: m5.medium
Resources:
  Instance:
    Type: 'AWS::EC2::Instance'
    Properties:
      InstanceType: !FindInMap 
        - 'InstanceConfiguration'
        - !Ref 'AWS::Region'
        - 'Type'
        - DefaultValue: m5.small
# Use m5.small instance unless it's in us-east-1, us-west-2 or eu-west-1
```

## Limitation

There will be few limitations when using other intrinsic functions or default value in `Fn::FindInMap`:

1. User can not use `Fn::Split` to supply argument list (partial or whole) of `Fn::FindInMap`. However, please do note that it **CAN** be used in conjunction with other functions that produce a `string` such as `Fn::Select`. Here is an example:

```
# This is not allowed
!FindInMap !Split [".", "RegionMap.us-east-1.HVM64"] ]

# This is allowed
!FindInMap
  - !Select [0, !Split [ ".", "RegionMap.us-east-1.HVM64" ] ]
  - !Select [1, !Split [ ".", "RegionMap.us-east-1.HVM64" ] ]
  - !Select [2, !Split [ ".", "RegionMap.us-east-1.HVM64" ] ]
```

1. `Fn::FindInMap` enhancement is made available via [`AWS::LanguageExtensions` transform](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/transform-aws-languageextensions.html), hence `Fn::FindInMap` can only support operations whose return values are known during transform time. When unsupported intrinsic functions are used, an exception will be returned back to customers.
    1. Here are examples not supported:
        1. `Ref` to a resource.
        2. `Fn::GetAtt` to retrieve an attribute from a resource.
        3. `Fn::Sub` with nested `Fn::GetAtt` or `Ref` which points to a resource or resource attribute. 

```
Mapping:
  AttributeMap:
    mypage:
      ttl: '300'
    otherpage:
      ttl: '600'  
Resources:
  LoadBalancer:
    Type: AWS::ElasticLoadBalancing::LoadBalancer

# Customer will get an error back in either case.
!FindInMap 
- DNSMap
- !Select [0, !Split [".", !GetAtt LoadBalancer.DNSName]]
- ttl
- DefaultValue: 500

!FindInMap
- DNSMap
- NonExistPage
- ttl
- DefaultValue: !Select [0, !Split [".", !GetAtt LoadBalancer.DNSName ] ]
```

## Appendix

### 1. Customer Request:

1. https://github.com/aws-cloudformation/cfn-language-discussion/issues/43
2. https://github.com/aws-cloudformation/cfn-language-discussion/issues/73
3. https://github.com/aws-cloudformation/cfn-language-discussion/issues/79
4. https://stackoverflow.com/questions/35904204/default-value-on-mappings-aws-cloudformation
