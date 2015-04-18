SQLWIN is a RDD for Harbour that works as a translator, that takes the DBFs syntax and convert it into SQL sentences.

Basically you connect to a Database engine and start issuing SQL sentences.

The advantage over ADO (we also created ADORDD) is that ADO is not needed at all. You can directly connect to a Database client DLL and avoid any intermediate layer, so it could work on any operating system (no ADO required)

The real advantage of SQLWIN is that we don't need to modify our existing applications source code. The idea is to replace the RDD and the RDD logic will do the "miracle" to use SQL