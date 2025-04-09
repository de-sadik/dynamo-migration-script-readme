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
