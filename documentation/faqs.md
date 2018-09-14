# Frequently Asked Questions

#### Q: What happens if ArgentSea isn’t flexible enough to do what I need it to do?

A: ArgentSea is a simple layer over ADO.Net. You can easily amend its functionality by simply coding your unique requirements directly in ADO.Net (as you would have done without the framework).

For example, if you are using the Mapper and the model attributes add a database parameter that you don’t want; just remove the parameter after calling the mapper! Or set the parameters yourself.

Setting parameters, invoking queries, and collecting results are all separate processes, so you can skip any one that has unique requirements, and still use any of the others. You can, for example, create parameters using the Mapper and invoke a query using your own Command object.

#### Q: Can I combine the PostgreSQL and SQL Server implementations?

Nope. A review of the effort involved in enabling both platforms in the same project indicates that the effort to support this exceeds the potential value. If you disagree with this conclusion, sending a note describing the value in your implementation.

(The first obstacle is that Mapper does not know which provider will eventually execute the parameters it creates, so if it encountered multiple attributes referencing different providers, it would not know which to use; the other problem is that we would have to separately the function delegates from each provider. Both concerns might be surmountable, but they would add considerable complexity).

#### Q: Why is an approach that exclusively uses stored procedures better?

A: Stored procedures offer performance, manageability, and security benefits.

Stored procedures do not require the database engine to parse your SQL string, so this can have a performance benefit. In most cases, the benefit is very small, however SQL
Server’s compiled procedures offer a potentially dramatic improvement.

If your application only has EXECUTE permission to stored procedures, then it becomes possible for DBA to comprehensively determine which tables/views/columns are access by the
application. Knowing this allows the database to be refactored and improved much less concern about unintended consequences.

With dynamic SQL, DBAs must resort to traces or logs to see what activity is being performed, which makes troubleshooting much harder. When a bad SQL plan is uncovered, the fix can be even harder. Stored procedures, on the other hand, allow the DBA to hint, rewrite, and optimize as necessary.

Finally, stored procedures allow DBAs (or data access SMEs) to review and approve 
data access code changes, also to ensure that indexes exist to support the new queries.

#### Q: How can I make sure that my data is secure?

A: Start by hiring an knowledgeable DBA. ArgentSea helps in a few ways:

* ArgentSea’s configuration design helps protect against unsafe storage of login passwords within connection strings.
* Because it uses stored procedures, users cannot run arbitrary SQL statements. This reduces the opportunity for mistaken SQL statements that corrupt data and SQL injection attacks.
* Because you can run your application with only EXECUTE permissions, no user would have access  to operations that are not explicitly enabled by a procedure.
