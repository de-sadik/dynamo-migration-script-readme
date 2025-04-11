# Detailed Report on AWS DynamoDB Operations Requiring Wait Times

## 1. Introduction
Amazon DynamoDB is a foundational AWS service, offering a scalable and performant NoSQL database. Many of its management operations involve asynchronous processes that require waiting for changes to propagate. Premature actions can lead to errors or inconsistencies.

This report covers key operations that require wait times, the reasons behind these waits, and how to manage them using the AWS SDK.

---

## 2. Identifying DynamoDB Operations Requiring Wait Times

Several operations involve asynchronous updates, especially when changing table structure or Global Secondary Indexes (GSIs):

- **CreateTable**: Status is `CREATING` → wait until `ACTIVE`.
- **UpdateTable**: Status is `UPDATING` → wait until `ACTIVE`.
- **DeleteTable**: Status is `DELETING` → wait until it's removed.
- **UpdateGlobalSecondaryIndex**: Status is `UPDATING` → wait until `ACTIVE`.
- **DeleteGlobalSecondaryIndex**: Status is `DELETING` → wait until removed.

---

## 3. Deep Dive into the Reasons for Waiting

Reasons include:

- **Resource Provisioning**: Allocating distributed resources takes time.
- **Data Replication and Indexing**: Changes must replicate across availability zones.
- **Metadata Updates**: System-wide updates to table/index configurations.

---

## 4. The Significance of Waiting After `CreateTable`

If actions are attempted before a table becomes `ACTIVE`, errors such as `ResourceNotFoundException` may occur.

**Best Practice**: Use AWS SDK waiters like:
- `waitFor('tableExists', ...)` in SDK v2
- `waitUntilTableExists(...)` in SDK v3

---

## 5. Leveraging the AWS SDK for Handling Wait Times

### 5.1 Introduction to `waitFor` (v2) and `waitUntilTableExists` (v3)

- **v2**: `AWS.DynamoDB.waitFor('tableExists' | 'tableNotExists')`
- **v3**: `waitUntilTableExists(...)` (modular function from `@aws-sdk/client-dynamodb`)

### 5.2 TypeScript Example (SDK v2)

```ts
import AWS from 'aws-sdk';

AWS.config.update({
  region: 'ap-south-1',
  accessKeyId: 'YOUR_ACCESS_KEY_ID',
  secretAccessKey: 'YOUR_SECRET_ACCESS_KEY'
});

const dynamodb = new AWS.DynamoDB({ apiVersion: '2012-08-10' });
const tableName = 'MyNewTable';

const params: AWS.DynamoDB.CreateTableInput = {
  TableName: tableName,
  KeySchema: [/* your schema */],
  AttributeDefinitions: [/* your attributes */],
  ProvisionedThroughput: {
    ReadCapacityUnits: 5,
    WriteCapacityUnits: 5
  }
};

async function createAndWaitForTable() {
  try {
    console.log(`Creating table ${tableName}...`);
    const response = await dynamodb.createTable(params).promise();
    console.log('Table created:', response.TableDescription?.TableArn);

    console.log(`Waiting for table ${tableName} to become active...`);
    await dynamodb.waitFor('tableExists', { TableName: tableName }).promise();
    console.log(`Table ${tableName} is now active.`);

    const describe = await dynamodb.describeTable({ TableName: tableName }).promise();
    console.log('Table description:', describe.Table?.TableStatus);
  } catch (error) {
    console.error('Error creating or waiting for table:', error);
  }
}

createAndWaitForTable();
```

---

### 5.3 Configuration Options for Waiters

**v2 (waitFor)**:
- Default: 20s interval, 25 attempts (~8.3 mins)
- Custom: `$waiter: { delay: 5, maxAttempts: 10 }`

```js
dynamodb.waitFor('tableExists', {
  TableName: 'YourTableName',
  $waiter: { delay: 5, maxAttempts: 10 }
}, callback);
```

**v3 (waitUntilTableExists)**:

```ts
import { DynamoDBClient, waitUntilTableExists } from "@aws-sdk/client-dynamodb";

const client = new DynamoDBClient({ region: 'your-region' });
await waitUntilTableExists({ client, maxWaitTime: 60, minDelay: 2, maxDelay: 5 }, {
  TableName: 'YourTableName'
});
```

---

### 5.4 Error Handling with Waiters

- Always wrap waiters in `try...catch`.
- Handle timeouts, check error details (`error.code`, `error.message`).
- Consider retry logic if operation fails due to transient issues.

---

## 6. General Best Practices

- ✅ Use waiters for all state-changing operations.
- ⏳ Set appropriate timeouts.
- 📡 Design for asynchronous behavior (e.g., DynamoDB Streams).

---

## 7. Conclusion

Waiting for the right resource state in DynamoDB is essential for:
- Avoiding runtime errors
- Ensuring consistency
- Enhancing application resilience

AWS SDK provides utilities to simplify these wait-handling tasks. Proper timeout configurations, error handling, and thoughtful design patterns are critical for building reliable DynamoDB applications.

---

## Tables

### Table 1: DynamoDB Operations Requiring Wait Times

| Operation                     | Description                         | Status      | Wait Condition             | SDK Method(s)                            |
|------------------------------|-------------------------------------|-------------|----------------------------|------------------------------------------|
| CreateTable                  | Creates a new table                 | CREATING    | Until `ACTIVE`             | `waitFor('tableExists')`, `waitUntil...` |
| UpdateTable                  | Modifies table configuration        | UPDATING    | Until `ACTIVE`             | `waitFor('tableExists')`, `waitUntil...` |
| DeleteTable                  | Deletes a table                     | DELETING    | Until non-existent         | `waitFor('tableNotExists')`              |
| UpdateGlobalSecondaryIndex   | Updates GSI throughput              | UPDATING    | Until `ACTIVE`             | Custom polling                           |
| DeleteGlobalSecondaryIndex   | Deletes GSI                         | DELETING    | Until non-existent         | Custom polling                           |

---

### Table 2: AWS SDK Waiter Configuration

| SDK Version | Wait Function           | Timeout Config               | Retry Config                     |
|-------------|--------------------------|------------------------------|----------------------------------|
| v2          | `waitFor`               | ~500s default (20s × 25)     | Linear backoff (20s)             |
| v2          | `waitFor` + `$waiter`   | Custom delay via `$waiter`   | Linear backoff (customizable)    |
| v3          | `waitUntilTableExists`  | `maxWaitTime` parameter      | Exponential backoff w/ jitter    |
| v3          | `waitUntilTableExists`  | Service-defined default      | Exponential backoff (default)    |

----

###

Creating and Configuring DynamoDB Tables with ... - CodeSignal, accessed April 4, 2025, https://codesignal.com/learn/courses/introduction-to-dynamodb-with-aws-sdk-for-python/lessons/creating-and-configuring-dynamodb-tables-with-aws-sdk-for-python
DynamoDB examples using SDK for JavaScript (v3) - AWS Documentation, accessed April 4, 2025, https://docs.aws.amazon.com/sdk-for-javascript/v3/developer-guide/javascript_dynamodb_code_examples.html
Use CreateTable with an AWS SDK or CLI - Amazon DynamoDB, accessed April 4, 2025, https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/example_dynamodb_CreateTable_section.html
How To Remove DynamoDB Local And Test With AWS Managed DynamoDB - bahr.dev, accessed April 4, 2025, https://bahr.dev/2021/04/09/replacing-local-with-managed-dynamodb/
[Solved] DynamoDB cannot do operations on a non-existent table, accessed April 4, 2025, https://dynobase.dev/dynamodb-errors/dynamodb-cannot-do-operations-on-a-non-existent-table/
Class: AWS.DynamoDB — AWS SDK for JavaScript - AWS Documentation, accessed April 4, 2025, https://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/DynamoDB.html
How to check DynamoDB Status while changing Throughput of WriteUnits and wait for it to complete? - Stack Overflow, accessed April 4, 2025, https://stackoverflow.com/questions/29251979/how-to-check-dynamodb-status-while-changing-throughput-of-writeunits-and-wait-fo
DynamoDB waitFor() with custom interval/max_attempts · Issue #881 · aws/aws-sdk-js, accessed April 4, 2025, https://github.com/aws/aws-sdk-js/issues/881
Class: AWS.DynamoDB — AWS SDK for JavaScript, accessed April 9, 2025, https://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/DynamoDB.html#waitFor-property
waitUntilTableExists Variable - AWS SDK for JavaScript v3, accessed April 4, 2025, https://docs.aws.amazon.com/AWSJavaScriptSDK/v3/latest/Package/-aws-sdk-client-dynamodb/Variable/waitUntilTableExists
Class: AWS.BedrockRuntime — AWS SDK for JavaScript - AWS Documentation, accessed April 4, 2025, https://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/BedrockRuntime.html
Waiters in modular AWS SDK for JavaScript | AWS Developer Tools Blog, accessed April 4, 2025, https://aws.amazon.com/blogs/developer/waiters-in-modular-aws-sdk-for-javascript/
@aws-sdk/client-dynamodb - npm, accessed April 4, 2025, https://www.npmjs.com/package/@aws-sdk/client-dynamodb
aws-sdk/client-dynamodb, accessed April 4, 2025, https://docs.aws.amazon.com/AWSJavaScriptSDK/v3/latest/Package/-aws-sdk-client-dynamodb
Auto Scaling examples using SDK for JavaScript (v3) - AWS Documentation - Amazon.com, accessed April 4, 2025, https://docs.aws.amazon.com/sdk-for-javascript/v3/developer-guide/javascript_auto-scaling_code_examples.html
Waiter doesn't error on Timeout · Issue #1917 · aws/aws-sdk-js-v3 - GitHub, accessed April 4, 2025, https://github.com/aws/aws-sdk-js-v3/issues/1917
accessed January 1, 1970, https://docs.aws.amazon.com/AWSJavaScriptSDK/v3/latest/client-dynamodb/function/waituntiltableexists/
accessed January 1, 1970, https://docs.aws.amazon.com/AWSJavaScriptSDK/v3/latest/client-dynamodb/classes/dynamodbclient/
Retry strategy in the AWS SDK for JavaScript v2, accessed April 4, 2025, https://docs.aws.amazon.com/sdk-for-javascript/v2/developer-guide/retry-strategy.html
AWS SDK Timeouts for JavaScript - jQN - Medium, accessed April 4, 2025, https://jqn.medium.com/aws-sdk-timeouts-for-javascript-967b86b100ee
amazon dynamodb - How do I set a timeout for AWS V3 Dynamo Clients - Stack Overflow, accessed April 4, 2025, https://stackoverflow.com/questions/66535671/how-do-i-set-a-timeout-for-aws-v3-dynamo-clients
Error handling with DynamoDB - Amazon DynamoDB, accessed April 4, 2025, https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Programming.Errors.html
Best Practices for ETL with S3, Lambda, and DynamoDB: Detailed Guide with Code Examples | by Mahtab Haider | Medium, accessed April 9, 2025, https://medium.com/@haider.mtech2011/best-practices-for-etl-with-s3-lambda-and-dynamodb-detailed-guide-with-code-examples-13a121cb49dc
Understanding Global Secondary Indexes in Amazon DynamoDB with the Rust SDK, accessed April 9, 2025, https://www.youtube.com/watch?v=O3dZU--wWRo
DynamoDB Secondary Indexes Archives - Jayendra's Cloud Certification Blog, accessed April 9, 2025, https://jayendrapatil.com/tag/dynamodb-secondary-indexes/
DynamoDB consistent reads for Global Secondary Index - Stack Overflow, accessed April 9, 2025, https://stackoverflow.com/questions/35414372/dynamodb-consistent-reads-for-global-secondary-index
Why is my dynamoDB queries taking so much time? - Stack Overflow, accessed April 9, 2025, https://stackoverflow.com/questions/76961827/why-is-my-dynamodb-queries-taking-so-much-time
DynamoDBClient - AWS SDK for JavaScript v3, accessed April 9, 2025, https://docs.aws.amazon.com/AWSJavaScriptSDK/v3/latest/client/dynamodb
40 DynamoDB Best Practices [Bite-sized Tips] - Dynobase, accessed April 9, 2025, https://dynobase.dev/dynamodb-best-practices/
