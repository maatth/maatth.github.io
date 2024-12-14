---
sidebar_position: 1
---
# PostgreSQL isolation transaction

This article illustrates some behaviours of the various transaction isolation settings described in the official PostgreSQL documentation: [https://www.postgresql.org/docs/12/transaction-iso.html](https://www.postgresql.org/docs/12/transaction-iso.html). It is recommended to read it first.



## Reminders
This section contains quick reminders from the PostgreSQL documentation.

A transaction begins with the BEGIN command and is validated by the COMMIT command. Each SQL query is automatically encapsulated in a transaction by PostgreSQL.

The SQL standard describes 4 problems (phenomena) to avoid when executing a transaction:
- **dirty read**: occur when one transaction reads data written by another, uncommitted, transaction. The danger with dirty reads is that the other transaction might never commit, leaving the original transaction with "dirty" data.
- **nonrepeatable read**: A transaction re-reads data (during the same transaction) it has previously read and finds that data has been modified by another transaction (that committed since the initial read)
- **phantom read**: A transaction re-executes a query returning a set of rows that satisfy a search condition and finds that the set of rows satisfying the condition has changed due to another recently-committed transaction
- **serialization anomaly**: The result of successfully committing a group of transactions is inconsistent with all possible orderings of running those transactions one at a time

The SQL standard describes 4 transaction isolation modes that will be used to progressively prevent its phenomena (in order of tolerance to the phenomena):
1. Read Uncommitted
2. Read committed
3. Repeatable Read
4. Serializable



## Read Uncommitted

It is the weakest isolation level because it can read the data which are acquired exclusive lock to the resources by the other transactions (unlike "read committed") So, it might help to avoid locks and deadlock problems for the data reading operations. It can be used to debug inside a transaction. In PostgreSQL, this isolation mode behaves like "Read Committed".



## Read Committed

This is the default transaction isolation.

In this mode, each request sees all already validated data (before the beginning of the transaction). In the following example, we simulate the concurrent access of two transactions by the use of two consoles in which the commands are executed. So the SELECT query in console 2 (<font color="#FF4500">(1)</font>  below) sees 123 (and not 124) because the UPDATE in console 1 has not yet been committed in the other transactions (because COMMIT is executed after SELECT):

![1](postgresql-isolation-asset/1.png)

The two consoles in which the commands are executed simulate the concurrent access of two transactions.

Thus, in the same transaction, if a second SELECT is executed after the validation of the transaction from console 1, it will see 124. See <font color="#FF4500">(2)</font> below. This
behaviour is called **"nonrepeatable read"** and is one of the 4 phenomena that we try to avoid.

![2](postgresql-isolation-asset/2.png)

This SELECT (<font color="#FF4500">(3)</font> below) can see the result of previous query that are in the same transaction (even though they are not committed). Example, the SELECT (<font color="#FF4500">(3)</font> below) will return 151.

![3](postgresql-isolation-asset/3.png)

For the UPDATE, DELETE commands, this behaviour is a little bit different since if the line they update is also modified by another transaction, these commands will wait for the end of this other transaction (either the cancellation or the validation), then use its result. In the example below, the UPDATE of console 2 <font color="#FF4500">(4)</font> will see 124 (result of the transaction on the left after the commit) and will return 125.

![4](postgresql-isolation-asset/4.png)

The same behaviour as UPDATE and DELETE can be obtained with SELECT by adding the FOR UPDATE or FOR SHARE clause. For example:

`SELECT balance FROM account WHERE account_num = 9 FOR UDPATE;` 

will wait for other transactions that concern this row to finish.

When UPDATE is executed (after the wait), its WHERE clause is re-evaluated to make sure it's the same row that it is modifying. If the row no longer matches the WHERE clause, the UPDATE is canceled.

This re-evaluation then cancellation can also cause unexpected behaviour: the phenomenon of **"phantom read"**. For example the following DELETE will not delete any line:

![5](postgresql-isolation-asset/5.png)

Because, after the transaction in console 1 updates all the lines, the DELETE of console 2 (<font color="#FF4500">(5)</font> above) will:
- evaluate its WHERE clause (which only concerns the row of id 2),
- then wait for the first transaction to finish because it modifies this line too.
- reevaluate the WHERE clause only for row of id 2
- the balance of this line no longer checks the WHERE clause so the DELETE is canceled.



## Repeatable Read

Unlike the **"Read Committed"** mode, this mode will only see the data validated before the start of the transaction. Thus, the phenomenon of **"nonrepeatable read"** becomes impossible and the SELECT <font color="#FF4500">(6)</font> below returns the same result as the previous SELECT.

![6](postgresql-isolation-asset/6.png)

The behaviour of UPDATE, DELETE, SELECT FOR UPDATE / SHARE will change: instead of waiting for the result of the concurrent transaction (see <font color="#FF4500">(4)</font>), the transaction will be rolled back with the message:
*"ERROR: could not serialize access due to concurrent update" (see <font color="#FF4500">(7)</font> below)*

![7](postgresql-isolation-asset/7.png)

So it will be necessary to implement retry mechanisms when this error occurs.

:::tip To summarise

The differences with "Read Committed" mode are:
- advantage: avoids the **"non repeatable read"** and **"phantom read"** phenomena.
- drawback: requires the implementation of retry.

:::


## Serializable

This mode has the same behaviour as "Repeatable Read", but it will also check that the database modifications are identical even if the execution order of the concurrent transactions is changed.

In the following example:
- the transaction in console 1 sums the values of class 1, then inserts the result with a class 2 <font color="#FF4500">(8)</font>
- the transaction in console 2 sums the values of class 2, then inserts the result with a class 1

![8](postgresql-isolation-asset/8.png)

In this case, PostgreSQL will check if the result is the same by first executing the transaction from console 2 before the transaction from console 1.

![9](postgresql-isolation-asset/9.png)

Here, the result of the table is not the same after the inversion <font color="#FF4500">(9)</font>, so the transaction of console 1 will be committed, but the one of console 2 will be rolled back.


:::tip To summarise

The differences with "Repeatable Read" mode are:
- advantage: avoids the "serialization anomaly" phenomenon.
- disadvantage: is more expensive in computation than the two other modes (except if one abuses the explicit locks FOR UPDATE / SHARE).

::: 


TLDR
imaginez qu'il y a deux transactions (dont un commence avant l'autre). La 1ere fait un update sur une ligne et l'autre lit cette ligne juste après (ou fait aussi un update). Qu'est ce qui se passe ? La valeur lue est-elle celle avant le début de la première transaction ou après celle-ci ?
Et bien, ca dépend de ce vous choisissez comme mode d'isolation entre transaction. Un peu comme pour définir un degré d'indépendance entre ces deux transactions.
Pour nous aider à choisir, le standard SQL a défini 4 pb que l'on voudra (ou pas) éviter : 
dirty read : (lecture sans se soucier des autres transactions)
nonrepeatable read : une transaction lit deux fois une ligne mais le second résultat diffère par rapport à la première lecture.
phantom read : pareil mais entre temps, la ligne a carrément disparue !
serialization anomaly : le resultat dépend de l'ordre d'execution des transactions :(

Dans sa grande bonté, à chaque pb est associé un mode d'interaction entre transaction appelé "mode d'isolation". C'est-à-dire que chaque mode va empecher un nb croissant de ces pb.


Read committed: 
T1 update et T2 select : le select lit uniquement ce qui est validé (ne voit pas le update) (sauf si FOR UPDATE)
T1 update et T2 update (ou delete) : T2 attend la fin de T1 pour recalculer le where. Si ca retourne la meme ligne, T2 continue, sinon l'update est ignoré. (si T1 roll back, alors on prend la valeur d'avant T1)
donc regarde la valeur avant T1 (pour les select) et après T2 (pour update et delete), dans tt les cas c'est validé, donc plus de dirty read
mais il reste non repeatable read phenomene car la valeur lue avant peut-etre modifiée par T1

repetable read : pareil qu'au dessus (attente que T1 finisse) pour le select, mais error si T2 update, delete ou select for update 

serializable : vérifie le meme résultat si ordre des transaction change (très consommateur de ressources)
