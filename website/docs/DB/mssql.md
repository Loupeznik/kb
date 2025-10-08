# MS SQL Server

## Enabling and disabling a constraint on a table

```sql
-- Disable
ALTER TABLE [dbo].[Customers] NOCHECK CONSTRAINT [FK_Customers_Address]

-- Perform operation requiring constraint to be disabled

-- Enable
ALTER TABLE [dbo].[Customer] WITH CHECK CHECK CONSTRAINT [FK_Customers_Address]
```
