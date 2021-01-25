.. _client_api:

Client API
==========


Clixon client API offers a simple way to communicate with the built-in
XML datastore. This can be used to fetch or manipulate configuration
which is handled by Clixon from the host system.

For example there might be another deamon running on the host system
that want to get a certain configured value from the datastore, we can
then use the client API to achieve that.


Comparison between Clixon and ConfD
===================================

Cisco ConfD is another well known configuration manager which also
maintains the configuration in its buil-in datastore. Just like
Clixon, ConfD offers an API for configuration access.

Applications that are integrated with ConfD using ConfDs CDB API can
easly be converted to use Clixon instead. Below we have a minimal
ConfD example application which will connect to ConfD and get a value
which is stored under "/table/parameter" in the XML store:

::

   #include <unistd.h>
   #include "confd_lib.h"
   #include "confd_cdb.h"

   #define CONFD_IPC_ADDR "127.0.0.1"
   #define CONFD_IPC_PORT  11004

   int main(int argc, char **argv)
   {
       int s = 0;
       u_int32_t n = 0;
       struct sockaddr_in addr;

       addr.sin_addr.s_addr = inet_addr(CONFD_IPC_ADDR);
       addr.sin_family = AF_INET;
       addr.sin_port = htons(CONFD_IPC_PORT);
       
       confd_init("server", stderr, CONFD_DEBUG);

       // No need to create a socket in Clixon
       if ((s = socket(PF_INET, SOCK_STREAM, 0)) < 0){
	   confd_fatal("socket failed\n");
	   return -1;
       }

       // Clixon: clixon_client_init
       if (cdb_connect(s, CDB_READ_SOCKET, (struct sockaddr*)&addr, sizeof(struct sockaddr_in)) < 0) {
           cdb_close(s);
	   return -1;
       }

       // Clixon: clixon_client_connect
       if (cdb_start_session(s, CDB_RUNNING) != CONFD_OK){
           cdb_close(s);
	   return -1;
       }
       
       // This is where we get the actual value
       // Clixon: clixon_client_get_uint32
       if (cdb_get_u_int32(s, &n, "/table/parameter[0]/value") != CONFD_OK)
           return -1;

       printf("Response: %d\n", n);

       // Clixon: clixon_client_disconnect
       cdb_end_session(s);

       // Clixon: clixon_client_terminate
       cdb_close(s);

       return 0;
   }


To use Clixon instead, there are very few things we have to change and
most of the functions are nearly identical:

::

   #include <unistd.h>
   #include <stdio.h>
   #include <stdint.h>
   #include <syslog.h>

   #include <clixon/clixon_log.h>
   #include <clixon/clixon_client.h>

   int main(int argc, char **argv)
   {
       clixon_handle h = NULL;
       clixon_client_handle ch = NULL;

       uint32_t u = 0;
       
       if ((h = clixon_client_init("/usr/local/etc/example.xml")) == NULL)
           return -1;

       if ((ch = clixon_client_connect(h, CLIXON_CLIENT_NETCONF)) == NULL)
           return -1;

       // This is where we get the actual value
       if (clixon_client_get_uint32(ch, &u, "urn:example:clixon", "/table/parameter[name='a']/value") < 0)
           return -1;
   
       printf("Response: %u\n", u);
      
       clixon_client_disconnect(ch);
       clixon_client_terminate(h);
      
       return 0;
   }


One major advantage for Clixons client API is the possibility to use
XPATHs when dealing with data. If we look at the
"clixon_client_get_uint32" function above we can see that it is using an XPATH:

::
   
   "/table/parameter[name='a']/value"

This is not possible with ConfD, there we must iterate over the list
of parameters until we eventually find what we are looking for.
