# TestDataFactory Patterns

## Overview

TestDataFactory is a centralized utility class for creating test records with sensible defaults. It ensures consistent test data across all test classes and reduces duplication.

See [assets/test-data-factory.cls](../assets/test-data-factory.cls) for the full template.

## API Convention

Every object has two methods:

| Method | Returns | DML |
|--------|---------|-----|
| `createObjects(count)` | `List<SObject>` (not inserted) | None |
| `createAndInsertObjects(count)` | `List<SObject>` (inserted) | Yes |

This gives callers flexibility to modify records before insert when needed.

```apex
// Create without insert (modify fields first)
List<Account> accounts = TestDataFactory.createAccounts(251);
accounts[0].Industry = 'Healthcare';
insert accounts;

// Create and insert in one call
List<Account> accounts = TestDataFactory.createAndInsertAccounts(251);
```

## Field Override Pattern

Allow callers to override default values:

```apex
public static Account createAccount(Map<String, Object> fieldOverrides) {
    Account acc = new Account(
        Name = 'Test Account',
        Industry = 'Technology'
    );

    for (String fieldName : fieldOverrides.keySet()) {
        acc.put(fieldName, fieldOverrides.get(fieldName));
    }

    return acc;
}

// Usage:
Account acc = TestDataFactory.createAccount(new Map<String, Object>{
    'Name' => 'Custom Name',
    'Industry' => 'Healthcare'
});
insert acc;
```

## Handling Required Fields and Validation Rules

```apex
public static Account createAccountWithRequiredFields() {
    return new Account(
        Name = 'Test Account',
        External_Id__c = 'EXT-' + String.valueOf(DateTime.now().getTime()),
        Phone = '555-123-4567',
        Website = 'https://example.com'
    );
}
```

## Record Type Support

```apex
public static Account createAccountByRecordType(String recordTypeName) {
    Id recordTypeId = Schema.SObjectType.Account
        .getRecordTypeInfosByDeveloperName()
        .get(recordTypeName)
        .getRecordTypeId();

    return new Account(
        Name = 'Test Account',
        RecordTypeId = recordTypeId
    );
}
```

## Best Practices

1. **Separate create from insert** -- use `create` + `createAndInsert` method pairs for flexibility
2. **Use unique identifiers** -- include index or timestamp in Name/Email fields to avoid duplicates
3. **Set all required fields** -- include all fields required by validation rules
4. **Return the created records** -- enables chaining and further manipulation
5. **Create bulk methods first** -- single record methods should call bulk methods with count=1
