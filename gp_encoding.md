# gpdb encoding issues
## client_encoding && server_encoding 

client_encoding is a GUC, you can change it in a session.

server_encoding can not be changed after the database has been inited. It is decided by the current system locale and –E option in the initdb command.
You can check the following doc to know which encodings gpdb supports.
[gpdb character sets](https://gpdb.docs.pivotal.io/43320/ref_guide/character_sets.html)

Gpinisystem may have a bug, setting locale option doesn’t work, we can change the server_encoding with the following steps:
- Locale: set the system locale

- Modify demo-cluster.sh as below: 

	`ENCODING=LATIN1

	LOCALE_SETTING=en_NZ.ISO8859-1`

### How about when client_encoding and server_encoding are different

- Insert data into database:

	In PostgresMain function, it will call the below function to transform the insert data from client encoding to server encoding.

	`pstring = pg_client_to_server(pbuf.data, plength);`

- Query data from database

	In printtup.c file, `pg_server_to_client` function is called to transform the tuple from server encoding to client encoding.

### psql’s encoding and client_encoding
psql’s encoding is decided by the locale on the client machine. Client_encoding is just a config used by database to recognize the client encoding. 

If you use the psql and set the client_encoding which is different from the psql, strange things will happen. For example:

psql encoding is UTF8 , client encoding is gbk and server_endcoding is UTF8

![encoding1](/fanfuxiaoran.github.io/images/encoding1)

When the client_encoding is 'UTF8', Why the output is not correct?

Let’s first check how does the "浣犲ソ” come from? The follwing python code can help
	`
	str = "你好";
	str_utf8 = str.encode("UTF-8")
	str_gbk = str.encode("GBK")

	print(str)
	print("utf 编码", str_utf8)
	print("gbk 编码", str_gbk)
	print("utf2gpk：", str_utf8.decode('GBK','strict'))
	`
The out put is
	`
	你好
	utf 编码 b’\xe4\xbd\xa0\xe5\xa5\xbd’
	gbk 编码 b’\xc4\xe3\xba\xc3'
	utf2gpk： 浣犲ソ
	`
"你好"'s UTF8 encoding is the same as the "浣犲ソ" gbk encoding.

![encoding2](/fanfuxiaoran.github.io/images/encoding2)

Actually, the binary in the test table is not "你好"'s UTF-8 encoding, but is "浣犲ソ"'s UTF-8 encoding. 

How does that happen?

psql : "你好" , actually is utf8 encoding, but the client encoding is set to 'GBK'. So the input is recognized as "浣犲ソ" and stored in the test table. 

## External table encoding 

External table has an option called "ENCODING", you can set the correct encoding there. And its deafult value is client_encoding., and when you loading data by the external table into the database, it will transform the data from "ENCODING" to server_encoding.

where does the transform happen？

The encoding transform is done in formatter. In the csv and text formatter, the function "pg_any_to_server" is called ( in CopyReadLine function,copy.c)

## Jsonb cannot handle the unicode above 0x7f when server encoding is not utf-8
`

server_encoding

-----------------

LATIN1



select jsonb '{ "a":  "\ud83d\ude04\ud83d\udc36" }' ->> 'a';

ERROR:  unsupported Unicode escape sequence

LINE 1: select jsonb '{ "a":  "\ud83d\ude04\ud83d\udc36" }' ->> 'a';

DETAIL:  Unicode escape values cannot be used for code point values above 007F when the server encoding is not UTF8.

CONTEXT:  JSON data, line 1: { "a":...
`


