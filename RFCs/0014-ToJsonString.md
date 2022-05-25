# New Intrinsic Function to Convert Template Block to JSON String

* **Original Author(s):**: @mluk-aws
* **Tracking Issue**: [Tracking Issue](https://github.com/aws-cloudformation/cfn-language-discussion/issues/14)

# Summary

We will support an intrinsic function called `Fn::ToJsonString` that enables developers to convert a template block into an escaped JSON string, which can be used as input values to string-type properties of CloudFormation resources.

# Motivation

A CloudFormation user may want to use JSON strings as input to a resource property. For example, `AWS::CloudWatch::Dashboard` requires a JSON string for the `DashboardBody` attribute. A user may even want to use a JSON string as input to attributes where only a general string is required, such as in `SecretString` of `AWS::SecretsManager::Secret`.

Today, this can only be accomplished through the use of external tools to convert object bodies to JSON strings, or use workarounds such as YAML multiline syntax (which has limitations such as the content only being parsed as generic text and not YAML by IDEs). Having a feature to automate this transformation can provide various benefits, such as improving development workflow, improving code readability, or being able to utilize JSON/YAML validation and syntax highlighting in text editors.

Open Github request regarding this: https://github.com/aws-cloudformation/cloudformation-coverage-roadmap/issues/78

# Examples

Here is an example of a resource containing a string-type property using a JSON formatted string.
```json
"MyDashboard": {
    "Type": "AWS::CloudWatch::Dashboard",
    "Properties": {
        "DashboardBody": "{\"start\":\"-PT6H\",\"periodOverride\":\"inherit\",\"widgets\":[{\"type\":\"text\",\"x\":0,\"y\":7,\"width\":3,\"height\":3,\"properties\":{\"markdown\":\"Hello world\"}}]}"
    }
}
```

With the new `Fn::ToJsonString` intrinsic function we can simplify to the following:

## JSON
```json
"MyDashboard": {
    "Type": "AWS::CloudWatch::Dashboard",
    "Properties": {
        "DashboardBody": {
            "Fn::ToJsonString": {
                "start": "-PT6H",
                "periodOverride": "inherit",
                "widgets": [
                    {  
                        "type": "text",
                        "x": 0,
                        "y": 7,
                        "width": 3,
                        "height": 3,
                        "properties": { "markdown": "Hello world" }
                    }
                ]
            }
        }
    }
}
```

## YAML
```yaml
MyDashboard:
    Type: AWS::CloudWatch::Dashboard
    Properties:
        DashboardBody:
            Fn::ToJsonString:
                start: "-PT6H"
                periodOverride: inherit
                widgets:
                    - type: text
                    x: 0
                    y: 7
                    width: 3
                    height: 3
                    properties:
                        markdown: Hello world
```

### Short Form Syntax
```yaml
MyDashboard:
    Type: AWS::CloudWatch::Dashboard
    Properties:
        DashboardBody:
            !ToJsonString:
                start: "-PT6H"
                periodOverride: inherit
                widgets:
                    - type: text
                    x: 0
                    y: 7
                    width: 3
                    height: 3
                    properties:
                        markdown: Hello world
```

# Details

`Fn::ToJsonString` is an intrinsic function that takes in a template block as input and converts it into an escaped JSON string.

* It will be restricted to only be used as the value of string-type resource properties.
* Intrinsic functions (e.g. `Fn::If`, `Ref`) or pseudo parameters (e.g. `AWS::NoValue`) can be used within the input template block. The input template block will be processed by the intrinsic functions and pseudo parameters before it is converted to a string (i.e. the resolved value of the intrinsic function is converted to the output string, and not the intrinsic function itself).
* Conversion will always retain the same order of key-value pairs such that the converted strings of the same input template block are guaranteed to not change.
* The following pseudo parameters will be supported:
    * `AWS::AccountId`, `AWS::Region`, `AWS::Partition`, `AWS::NoValue`

# Limitation

* YAML is a superset of JSON, meaning there are features supported in YAML but not in JSON. Most notable is YAML's inline comment feature which does not have an equivalent in JSON. If `Fn::ToJsonString` is used on a YAML template block that contains comments, the comments will be automatically stripped out when converted to JSON. This can potentially be a confusing developer experience if the developer is not aware of these limitations and have different expectations.
* Unsupported pseudo parameters:
    * `AWS::StackId`, `AWS::StackName`, `AWS::NotificationArns`, `AWS::URLSuffix`

# FAQ
* Will the CloudFormation Linter (cfn-lint) support validations regarding Fn::ToJsonString?
    * Yes. It needs to validate that it’s only used for string-type resource properties, and that the input template block is valid.