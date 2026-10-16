---
description: Create or update contracts in the system. This command allows you to manage contracts by providing the necessary details such as contract ID, title, description, and other relevant information. You can use this command to add new contracts or modify existing ones.
---

# Go Contracts Command

This command allows you to create or update contracts in the system based on enum specifications.

## What This Command Does

1. **Required Parameters**: Accepts contract name and search in specification
2. **Verify And Compare**: Сhecks whether a description of the contract exists and whether it exists in the specification
3. **Create Or Update**: If the contract does not exist, it creates a new one; if it exists, it updates the existing contract with the provided details.
4. **Refactor**: Improve while keeping tests green
5. **Check Coverage**: Ensure 80%+ coverage
6. **Update Readme**: Update readme

## When To Use

Use `/go-contracts` when:
- Need to update contracts module
- After writing or modifying contracts code