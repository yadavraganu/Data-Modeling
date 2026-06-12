Each and every dimension attribute has participated in a simple one-tomany relationship with every fact
Real-world complexity sometimes makes it
impossible to model a process in this manner. This chapter explores two such challenges:
•	 Multi-valued dimensions occur when a fact table row may need to refer to more
than one row in a dimension table.
•	 Multi-valued attributes occur when a dimension row needs to capture multiple
values for a single attribute.
In both cases, a star configuration simply will not support the true complexities of the
process.
The solution to both these challenges introduces a new kind of table, called a bridge.
