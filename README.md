# aws-notification-cloudformation
汎用的なaws sns の Pub/Subを使った通知を実現するための設定

このテンプレートでは、次のリソースをプロビジョニングします：

1. **Amazon S3バケット**: Lambda関数のコードを格納するためのS3バケット。
2. **AWS Lambda関数**: S3バケットからコードを読み込み、実行するLambda関数。
3. **Amazon SNSトピック**: Lambda関数をトリガーするためのSNSトピック。

CloudFormationテンプレートの例は以下のとおりです：

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Resources:
  MyS3Bucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: my-lambda-code-bucket-unique-name

  MySNSTopic:
    Type: AWS::SNS::Topic
    Properties:
      TopicName: my-sns-topic-for-lambda-trigger

  MyLambdaFunction:
    Type: AWS::Lambda::Function
    Properties:
      Handler: index.handler
      Role: !GetAtt LambdaExecutionRole.Arn
      FunctionName: my-lambda-function
      Code:
        S3Bucket: !Ref MyS3Bucket
        S3Key: lambda-code.zip
      Runtime: python3.8

  LambdaExecutionRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: lambda.amazonaws.com
            Action: sts:AssumeRole
      Policies:
        - PolicyName: LambdaS3ReadAccess
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action:
                  - s3:GetObject
                Resource: !Sub 'arn:aws:s3:::${MyS3Bucket}/*'
        - PolicyName: LambdaLogging
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action:
                  - logs:CreateLogGroup
                  - logs:CreateLogStream
                  - logs:PutLogEvents
                Resource: arn:aws:logs:*:*:*

  MySNSTopicSubscription:
    Type: AWS::SNS::Subscription
    Properties:
      Protocol: 'lambda'
      TopicArn: !Ref MySNSTopic
      Endpoint: !GetAtt MyLambdaFunction.Arn

  LambdaPermissionToInvokeFromSNS:
    Type: AWS::Lambda::Permission
    Properties:
      FunctionName: !GetAtt MyLambdaFunction.Arn
      Action: 'lambda:InvokeFunction'
      Principal: 'sns.amazonaws.com'
      SourceArn: !Ref MySNSTopic
```

このテンプレートは、指定された要件を満たすリソースを作成します。テンプレートに含まれる主要なポイントは以下の通りです：

- S3バケット(`MyS3Bucket`)を作成し、Lambda関数のコードを格納します。バケット名はユニークである必要があります。
- Lambda関数(`MyLambdaFunction`)を作成し、実行に必要なIAMロール(`LambdaExecutionRole`)と、S3バケットからコードを読み込むためのポリシーを設定します。
- SNSトピック(`MySNSTopic`)を作成し、Lambda関数をトリガーするために使用します。Lambda関数をSNSトピックにサブスクライブ(`MySNSTopicSubscription`)し、SNSからのイベントを受け取るための権限(`LambdaPermissionToInvokeFromSNS`)をLambda関数に付与します。

このテンプレートを使用する前に、`my-lambda-code-bucket-unique-name`と`s3Key`（ここでは`lambda-code.zip`）を実際のバケット名とLambda関数のコードファイル名に置き換える必要があります。また、Lambda関数の`Runtime`と`Handler`も、使用するプログラミング言語とハンドラー情報に応じて調整してください。
