---
title: "Mocking AWS Services Using Moto"
description: "This post explores the inner workings of WebSockets by building a simple WebSocket server in Java using raw sockets."
publishDate: "22 May 2022"
updatedDate: "22 Mar 2022"
---
<hr />

Recently, I was creating a AWS Lambda service written in Python, And my lambda was doing calls to AWS DynamoDB.
I was following the best practice that AWS [recommended](https://docs.aws.amazon.com/lambda/latest/dg/best-practices.html) that we should Initialize SDK clients and database connections outside of the function handler. I faced some problems to mock these AWS connections which are created outside handler function.

Below is sample lambda code which is querying to DynamoDB and fetching some data. We will write unit test for this code.

```python
import boto3

dynamo_db_connection = boto3.resource('dynamodb', region_name='us-east-1')
anime_table_name = os.environ['anime_table_name']

def lambda_handler(event, context):
    print("Incoming event ", event)
    anime_name = str(event["anime_name"])

    response = dynamodb_connection.query(
        TableName=anime_table_name,
        KeyConditionExpression='anime_name = :anime_name',
        ExpressionAttributeValues={
            ':anime_name': {'S': anime_name}
        }
    )

    anime_info = response['Items'][0]
    print(anime_info)

    return {"main_character": anime_info["main_character"]["S"]}
```

## Challenges
While writing unit-test I faced challenge to provide mocks for this boto3 connections. As this connections gets executed
at time of initialization of lambda so, I need to have this mocks ready and injected before our lambda file gets executed.

## Monkey Patching Concept
We should write a testable code, by testable code means we should use dependency injections concept in our code. If our code follows DI concept then while writing unit test we can easily provide mock connection objects and it’s mock behavior to our code so it’s easy to write unit test.

But in Python we have a another functionality called monkey patching and many unit-test frameworks of python uses this functionality.

I have used moto framework which is used monkey patching concept to mocks our boto3 API calls.

<hr />
Below frameworks you need to install.

boto3 = 1.23.3, moto = 3.1.9

<hr />

## Let’s start to write a unit test for our lambda code

1. We will create a class to test our lambda_handler method and this class will inherit the unittest.TestCase class of unittest framework.
2. We will also inherit the setUp and tearDown methods from unittest testcase, these methods basically invoke before and after each time when our each test case runs.

let’s see how our class looks now.
```python
import unittest

class LambdaFunctionTest(unittest.TestCase):
      def setUp(self) -> None:
          pass
      def tearDown(self) -> None:
          pass
```

3. Now we will create mock environment for our test.
```python
import unittest

import boto3
import os
from moto import mock_dynamodb
from unittest.mock import patch


@mock_dynamodb  # Decorator applied to whole class. This will provides mock for each dynamodb calls which creates inside this class.
@patch.dict(os.environ, {
    "AWS_REGION": "us-east-1",
    "anime_table_name": 'anime_info'})
class LambdaFunctionTest(unittest.TestCase):

    def setUp(self) -> None:
        print("Setup invoked for testing suite")
        self.dynamodb = boto3.resource('dynamodb', 'us-east-1')
        anime_table_name = 'anime_info'
        self.dynamodb.create_table(TableName=anime_table_name,
                                   KeySchema=anime_table_data['KeySchema'],
                                   AttributeDefinitions=anime_table_data['AttributeDefinitions'],
                                   ProvisionedThroughput=anime_table_data['ProvisionedThroughput']
                                   )

        self.registration_status_table = self.dynamodb.Table(anime_table_name)

    def tearDown(self) -> None:
        print("Tearing down resources")
        self.registration_status_table.delete()


anime_table_data = {
    'KeySchema': [{
        'AttributeName': 'anime_name',
        'KeyType': 'HASH'
        }
    ],
    'AttributeDefinitions': [
        {
            'AttributeName': 'anime_name',
            'AttributeType': 'S'
        }
    ],
    'ProvisionedThroughput': {
        'ReadCapacityUnits': 1,
        'WriteCapacityUnits': 1
    }
}

user_table_data = {
    'KeySchema': [{
        'AttributeName': 'user_id',
        'KeyType': 'HASH'
    },
        {
            'AttributeName': 'registration_id',
            'KeyType': 'RANGE'
        }
    ],
    'AttributeDefinitions': [
        {
            'AttributeName': 'user_id',
            'AttributeType': 'S'
        },
        {
            'AttributeName': 'registration_id',
            'AttributeType': 'S'
        }
    ],
    'ProvisionedThroughput': {
        'ReadCapacityUnits': 1,
        'WriteCapacityUnits': 1
    }
}
```
Let’s understand what we have done in the above code.

- We have used @mock_dynamodb decorator of moto framework in our class. This decorator mocks all boto3 API calls which will invoke inside this class.

- @patch.dict decorator we have used to mock our environment variables. It’s part of unittest framework.

- In setUp() method we have created DynamoDB table using boto3. As earlier I said we have created this tables under moto mocked environment. Hence, all boto3 calls are mocked here. And we got mocked table created.

4. Now we will write a test case function to test one success scenario of lambda.
```python
import unittest

import boto3
import os
from moto import mock_dynamodb
from unittest.mock import patch


@mock_dynamodb  # Decorator applied to whole class for providing mock for each dynamodb calls
@patch.dict(os.environ, {
    "AWS_REGION": "us-east-1",
    "anime_table_name": 'anime_info'})
class LambdaFunctionTest(unittest.TestCase):

    def setUp(self) -> None:
        print("Setup invoked for testing suite")
        self.dynamodb = boto3.resource('dynamodb', 'us-east-1')
        anime_table_name = 'anime_info'
        self.dynamodb.create_table(TableName=anime_table_name,
                                   KeySchema=anime_table_data['KeySchema'],
                                   AttributeDefinitions=anime_table_data['AttributeDefinitions'],
                                   ProvisionedThroughput=anime_table_data['ProvisionedThroughput']
                                   )

        self.registration_status_table = self.dynamodb.Table(anime_table_name)

    def test_get_pending_request_success(self):
        registration_status_data_list = [
            {"anime_name": "naruto", "main_character": "naruto"},
            {"anime_name": "attack on titan", "main_character": "eren"},
            {"anime_name": "demon slayer", "main_character": "tanjiro"}
        ]
        for data in registration_status_data_list:
            self.registration_status_table.put_item(Item=data)

        print("test case started")
        """
        Importing lambda_function here because we have some AWS resources are eager loading type(Defined outside of the
        function). If we import this file at top, those eager loaded AWS resources will start executing and they will
        try to call real AWS components. So we are importing file after setting up mocks.
        """
        import lambda_function
        response = lambda_function.lambda_handler({"anime_name": "naruto"}, None)
        print(response)
        expected_response = {'main_character': 'naruto'}
        self.assertEqual(response, expected_response)

    def tearDown(self) -> None:
        print("Tearing down resources")
        self.registration_status_table.delete()


anime_table_data = {
    'KeySchema': [{
        'AttributeName': 'anime_name',
        'KeyType': 'HASH'
        }
    ],
    'AttributeDefinitions': [
        {
            'AttributeName': 'anime_name',
            'AttributeType': 'S'
        }
    ],
    'ProvisionedThroughput': {
        'ReadCapacityUnits': 1,
        'WriteCapacityUnits': 1
    }
}

user_table_data = {
    'KeySchema': [{
        'AttributeName': 'user_id',
        'KeyType': 'HASH'
    },
        {
            'AttributeName': 'registration_id',
            'KeyType': 'RANGE'
        }
    ],
    'AttributeDefinitions': [
        {
            'AttributeName': 'user_id',
            'AttributeType': 'S'
        },
        {
            'AttributeName': 'registration_id',
            'AttributeType': 'S'
        }
    ],
    'ProvisionedThroughput': {
        'ReadCapacityUnits': 1,
        'WriteCapacityUnits': 1
    }
}
```
Let’s see what we have done in test_get_pending_request_success() method.

- First we have ingested some data in the table, that we created in setUp() method.

- Then, we have imported our lambda_function file.

<div class="note-text">
This is the important point shows how we achieved to provide mock behavior for the connections that are created outside
of the handler. As we are importing lambda function inside our test method, we are invoking lambda inside our moto mock
environment, hence these boto3 API connections will also create under the mock environment.
</div>

- At last, you can see we have called our lambda_function handler by passing the inputs, the dynamodb query present in
lambda_function handler is also part of moto mock environment, hence it will return the mock results from the earlier
created mock tables and mock data that we have ingested.

5. Last we have inherited tearDown() method, which we used to delete all mocked resources.

<hr />

So this is how I did unit testing for my AWS lambda written in Python. Due to moto framework it’s really easy to mock
AWS connections.

<hr />

You can get full source code from this [github](https://github.com/sats17/mocky-lambda) URL.
