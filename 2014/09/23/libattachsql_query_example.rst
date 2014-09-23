libAttachSQL Query Example
==========================

I was asked some questions on IRC last night about how the query example in libAttachSQL's code base works.  For those who missed previous posts, `libAttachSQL <http://libattachsql.org/>`_ is a lightweight, non-blocking, Apache 2.0 licensed C connector for MySQL servers which I am developing for HP's Advanced Technology Group.

In this blog post I'm going to break down `a basic query example <https://github.com/libattachsql/libattachsql/blob/master/examples/basic_query.c>`_ and explain what is happening at each step. It is possible that this syntax may change slightly by the time we hit GA but it will be similar to this.

.. code-block:: cpp

   #include <libattachsql-1.0/attachsql.h>
   #include <stddef.h>
   #include <stdio.h>
   #include <string.h>

Only one include is needed for the library itself, ``libattachsql-1.0/attachsql.h``. The others are used for other functions in the code.

.. code-block:: cpp

   int main(int argc, char *argv[])
   {
     attachsql_connect_t *con= NULL;
     attachsql_error_st *error= NULL;
     const char *query= "SELECT * FROM t1 WHERE name='fred'";
     attachsql_return_t ret= ATTACHSQL_RETURN_NONE;
     attachsql_query_row_st *row;
     uint16_t columns, current_column;

Here we have setup a few required variables:

* ``con`` is the connection object used throughout this example
* ``error`` is an error struct which is set when an error occurs
* ``query`` this is a constant containing the query we wish to execute
* ``ret`` is where the return code is store for the network functions
* ``row`` holds the row results
* ``columns`` and ``current_column`` are used for iterating though the result

.. code-block:: cpp

     con= attachsql_connect_create("localhost", 3306, "test", "test", "testdb", NULL);

This function creates the connection object.  A connection is not actually made at this point, we are just setting things up.

.. code-block:: cpp

     error= attachsql_query(con, strlen(query), query, 0, NULL);

This starts the process of connecting to the MySQL server and sending the query.  The last two parameters are for a way to escape data at the client end in a similar way to prepared statements.  I'll cover this in another blog post.

.. code-block:: cpp

     while ((ret != ATTACHSQL_RETURN_EOF) && (error == NULL))
     {
       ret= attachsql_connect_poll(con, &error);

This is where the magic happens.  Whilst we haven't hit a query EOF or an error we are repeatedly polling the connection to see if we have have a row ready in the network buffer.

Not only this but a non-blocking DNS lookup, connection and handshake is made here, so it is likely poll will be called several times.  There is an API call to make the connection explicit rather than implicit on the first query and it will need to call ``attachsql_connect_poll()`` in the same way.

.. code-block:: cpp

       if (ret != ATTACHSQL_RETURN_ROW_READY)
       {
         continue;
       }

If we don't have a row yet continue polling until we do.  In a real-world application you can have other tasks on your main thread going on here and/or poll many connections on a single thread.

.. code-block:: cpp

       row= attachsql_query_row_get(con, &error);

Process the row and return an array of pointers to the network read buffer for parts of the row.

.. code-block:: cpp

       columns= attachsql_query_column_count(con);

       for (current_column= 0; current_column < columns; current_column++)
       {
         printf("%.*s ", (int)row[current_column].length, row[current_column].data);
       }
       printf("\n");

Iterate through the columns in the row and print out the result.  Technically we only need to call ``attachsql_query_column_count()`` once when the first row is ready in the buffer.

.. code-block:: cpp

       attachsql_query_row_next(con);
     }

This tells the library that we are done with this row and are ready to retrieve the next, which brings us back to polling.

.. code-block:: cpp

     if (error != NULL)
     {
       printf("Error occurred: %s", error->msg);
       attachsql_error_free(error);
     }

If we have broken out of the loop due to an error, print that out and free the error object.

.. code-block:: cpp

     attachsql_query_close(con);
     attachsql_connect_destroy(con);
   }

Close the query and destroy the connection.  All done!



.. author:: default
.. categories:: MySQL, libAttachSQL
.. tags:: MySQL, libAttachSQL, HP, Advanced Technology Group
.. comments::
