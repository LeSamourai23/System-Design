A transaction is a series of database operations that are considered to be a _"single unit of work"_. The operations in a transaction either all succeed, or they all fail. In this way, the notion of a transaction supports data integrity when part of a system fails. Not all databases choose to support ACID transactions, usually because they are prioritizing other optimizations that are hard or theoretically impossible to implement together.
_Usually, relational databases support ACID transactions, and non-relational databases don't (there are exceptions)._

## States
A transaction in a database can be in one of the following states:

![[transaction-states.webp]]

#### Active
In this state, the transaction is being executed. This is the initial state of every transaction.

#### Partially Committed
When a transaction executes its final operation, it is said to be in a partially committed state.

#### Committed
If a transaction executes all its operations successfully, it is said to be committed. All its effects are now permanently established on the database system.

#### Failed
The transaction is said to be in a failed state if any of the checks made by the database recovery system fails. A failed transaction can no longer proceed further.

#### Aborted
If any of the checks fail and the transaction has reached a failed state, then the recovery manager rolls back all its write operations on the database to bring the database back to its original state where it was prior to the execution of the transaction. Transactions in this state are aborted.

The database recovery module can select one of the two operations after a transaction aborts:
- Restart the transaction
- Kill the transaction

#### Terminated
If there isn't any roll-back or the transaction comes from the _committed state_, then the system is consistent and ready for a new transaction and the old transaction is terminated.