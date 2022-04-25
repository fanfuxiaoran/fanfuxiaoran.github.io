# The Internal of Pgbouncer
## Overviews of pools
![pgbouncer1](doc/images/pgbouncer1)

A pool is a set of connections to the same database for a given user. There may be many pools in a pgbouncer server. As the above picture shows, there are 3 different pools: db1/user1, db2/user2 and db2/user1.

How does a client connect to a database server through pgbouncer?
1. a client connects to pgbouncer server
2. pgbouncer finds a pool for a client by dbname and user name
3. pgbouncer finds a idle server connection for the client (if there is no available server connection, launches one), and link it to client
   client->link = server, server->link = client
4. when pgbouncer receives a query from client, pgbouncer forwards the query to the database server by copying query data from client socket into server socket.
5. when pgbouncer receives the query result from the database server, pgbouncer forwards the result to client by copying result data from server socket into client socket. 

### Client/Server sockets management

Client and server socket both have their own status. pgbouncer uses several status queues(list) to manage them. (we will discuss the status in part 2)

- client status queues:

	A client socket needs to link to a server socket to connect to the database server. If it has a server link, it will be in the active_client_list, otherwise, it will be in the waiting_client_list. If the client in the waiting_client_list, it can do nothing.

- server status queues

	If a server is linked to a client socket, it will be in the active_server_list, otherwise it will be in the idel_sever_list or tested_server_list or used_server_list. new_server_list is a special list for server sockets which are logging in into database server.

## Client and server connection
Client state diagram 
![pgbouncer2](doc/images/pgbouncer2)

- CL_LOGIN: client is doing login into pgbouncer. A client logins into to pgbouncer based on the auth_type.
- CL_ACTIVE: client has successfully login into pgbouncer
- CL_WAITING: client can not get a server connection after being CL_ACTIVE
- CL_WAITING_LOGIN：client can not get a server connection after being CL_LOGIN. (when client authenticates by auth_user and auth_query, the client needs a server connection to database server)
- CL_JUSTGREE：a client connection is disconnected

Server state diagram
![pgbouncer3](doc/images/pgbouncer3)

- SV_LOGIN: server is doing login into database server
- SV_IDLE: server connection has successfully logged into database server and the connection is not used by any client connection
- SV_ACTIVE: server connection is used by a client connection
- SV_USED: a server idle connection has being idle longer than server_check_delay
- SV_TESTD: a server idle connection is executing server_check_query to check is if the connection in SV_USED state is valid.
- SV_JUSTGREE：a server connection is disconnected.

## Authentication
**How does pgbouncer authenticate client connection?**

We usually run pgbouncer as: pgbouncer -d pgbouncer.ini. 

The following iterms are related to authentication:

	[pgbouncer]
	auth_type = md5
	auth_file = /etc/pgbouncer/userlist.txt
	auth_user = pgbouncer
	auth_query = SELECT p_user, p_password FROM

- auth_type: how to to authenticate users, it contains several methods: pam, hba, cert, md5,scram-sha-256,plain, trust, any.
- auth_file: the name of the file to load user names and passwords from.
- auth_user: if auth_user is set, then any user not specified in auth_file will be queried through the auth_query query from pg_shadow in the database, using auth_user. The password of auth_user will be taken from auth_file.
- auth_query: query to load user’s password from database.

**Where does pgbouncer get the correct username and password to authenticate a client?**

if auth type is md5、scram-sha-256 or plain, pgbouncer needs to check user name and password.
There are 2 ways for pgbouncer to get user name and password
![pgbouncer4](doc/images/pgbouncer4)

1. Auth_file

	Pgbouncer can load username and password from authfile

2. Database server

	If users and passwords change frequently, it would be annoying to have to change the user list all the time. In that case it is better to use an "authentication user" that can connect to the database and get the password from there. If a client user is not configured in auth_file, pgbouncer will use auth_user to query the user’s password from the database server.  

	Pam 、cert and the other authentication methods will not be discussed here.

**Where does pgbouncer get username and password to connect to database server?**

1. Auth_file
2. auth_query
3. database configuration

Let’t look at an example of a database configuration in pgbouncer.ini file
```
[databases]
; redirect bardb to bazdb on localhost
P0 = host=127.0.0.1 port=300 dbname=bazdb

; access to destination database will go with single user
forcedb1 = host=127.0.0.1 port=300 user=baz client_encoding=UNICODE datestyle=ISO
forcedb2 = host=127.0.0.1 port=300 user=baz password=foo client_encoding=UNICODE datestyle=ISO
```
- uesr

	If user is set, all connections to the destination database will be done with the specified user, meaning that there will be only one pool for this database.
Otherwise, PgBouncer logs into the destination database with the client user name, meaning that there will be one pool per user.

- password

	The length for password is limited to 160 characters maximum.

	If no password is specified here, the password from the auth_file or auth_query will be used.

In the above config, when pgbouncer connects to database p0, as it doesn’t contain any user name and password in the connection configuration, so we use the username and password  from the client connection (client connection password is from the auth_file or auth_query); when pgbouncer connects to database forcedb1, we use the user "baz" and  password from the auth_file  to connect to database; when pgbouncer connects to database forcedb2, we use user "baz" and password "foo" in the database configuration.


## Code structure
- loader.c: load parameters from pgbouncer.ini and userlist.txt
- pooler.c: handle pgbouncer listening sockets
- client.c: handle client connection
	    - client_proto: start to process the data readed from the client socket.
            - handle_client_startup: if client state is cl_login call this function to process socket data
            - handle_client_work: if client state is cl_active, call this function to process socket data
	    - start_auth_request: if client uses auth_query to get user password, call this function to execute the auth_query
	    - Handle_auth_response：called by server.c after geting the response of the auth_query(sent by start_auth_requset) . Parse the username and password from the response.
            - Send_client_authreq: send auth request to client to ask password from client. (refers to Postgres on the wire - A look at the PostgreSQL wire protocol)

- server.c
	    - Server_proto: start to process the data reader from the server socket
            - Handle_server_startup: when sever socket state is sv_login
            - Handle_server_work：when sever socket state is sv_active, sv_dle, sv_used, sv_tested
- sbuf.c: stream buffer. Can use it to copy data from client socket to server socket, or sever socket to client socket.
- objects.c: A lot of functions in it.
- Janitor.c:  Periodic called, it maintains the client connection and server connection, change thier states and put them into different queues.

References:
1. Postgres on the wire - A look at the PostgreSQL wire protocol
2. https://heap.io/blog/engineering/decrypting-pgbouncers-diagnostic-information
3. https://www.cybertec-postgresql.com/en/pgbouncer-authentication-made-easy/
