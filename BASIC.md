# Getting Started with RELL: A Developer's Guide
RELL is a blockchain-oriented programming language designed for the Chromia platform. It combines SQL-like syntax with modern programming concepts, making it familiar for developers coming from both database and traditional programming backgrounds.

## Basic Structure

In RELL, your code starts with a module declaration. The basic building blocks are entities (similar to database tables), operations (similar to transactions), and queries.
kotlinCopymodule;

```rell
// Entities are like database tables
entity user {
    key pubkey: byte_array;  // Primary key
    name: text;
    balance: integer = 0;    // Default value
}
```

## Entities and Data Modeling

Entities in RELL support various data types and relationships:

```rell
// Basic entity with a single key
entity company {
    key name: text;
    city: text;
}

// Entity representing a many-to-many relationship
entity employment {
    key user: user;         // Reference to user entity
    key company: company;   // Reference to company entity
}
```

## Operations (Transactions)

Operations are functions that modify the blockchain state. They often include authorization checks:

```rell
operation register_user(pubkey: byte_array, name: text) {
    // Verify transaction signer
    require(op_context.is_signer(pubkey), "Transaction needs to be signed by provided pubkey");

    // Create new user record
    create user (
        pubkey,
        name,
    );
}
```

## Queries
Queries in RELL use a SQL-like syntax but with modern programming conveniences:

```rell
// Query with filtering and relationships
query get_london_chromia_users() {
    return (user, company, employment) @* {
        company.name == "Chromia",
        company.city == "London",
        employment.user == user,
        employment.company == company
    };
}

// Query with sorting, pagination, and limiting
query get_latest_logs(n_records: integer = 10) {
    return llm_log @* {} (
        @sort_desc @omit .created_at,  // Sort by created_at descending
        .chat_id,
        .model,
        .user_question,
        .assistant_reply
    ) limit n_records;
}
```

## Advanced Features
### Function Helpers

```rell
function is_owner() {
    require(op_context.is_signer(owner.address), "Only owner can call this operation");
}
```

### Mutable Objects

```rell
object owner {
    mutable address: pubkey = x"03EE16E..."; // Mutable state
}
```

### Structured Data Transfer

```rell
struct open_ai_logs_dto {
    address: byte_array;
    uuid: text;
    chat_id: text;
    // ... other fields
}
```

# Best Practices

Use @omit with @sort_desc for efficient sorting
Implement authorization checks using require() and op_context.is_signer()
Use nullable types with ? for optional values
Structure complex queries with DTOs for clean data transfer
Leverage indexes for better query performance

This introduction covers the fundamentals of RELL, but there's much more to explore. The language is particularly well-suited for blockchain applications requiring complex data relationships and secure transaction handling.
