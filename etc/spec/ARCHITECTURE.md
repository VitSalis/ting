# Ting architecture

The ting architecture is shown in the following diagram:

![Ting architecture](http://i.imgur.com/AdTsVNY.jpg)

The application is split into two parts: The server-side and the client-side.
The client-side portion is in Javascript and runs on the user's machine. The
server-side portion runs on the ting servers.

# Client-side architecture
The client-side architecture is built using
[react](http://facebook.github.io/react/).

There are several react components on the client-side. The Ting component is of
significance, because it is responsible for networking with the server-side of
the ting application. The Ting component uses web sockets and AJAX to communicate with
the real-time portion of the server and AJAX to communicate with the RESTful API
portion of the server.

The rest of the components are agnostic in regards to networking, and are able
to communicate with the server only through the Ting component.

# Server-side architecture
The server-side architecture is split into two portions: The real-time service
and the RESTful API service.

The real-time service is written in node.js and provides a web-socket end-point
that the client directly talks to. The real-time protocol allows exchanging data
such as currently online users and messages being currently exchanged.

The RESTful API service is written in Python using the Django framework. It
offers an HTTPS end-point that is used to manage users, history, and channels.

The client also talks to the RESTful API directly for non-real-time operations.
Such operations include the retrieval of message history for a given channel,
or profile information about a particular user.

The real-time service also communicates internally with the RESTful API
service. This is so that real-time data can be persisted over time. This
communication happens using a RESTful HTTPS protocol, where the real-time
service hits the URLs of the RESTful service. As an example of one such
operation, consider the persistence of chat messages.

The communication between the real-time service and the RESTful API is
privileged: The privileges of the real-time service are elevated so that
it can provide authoritative data to the RESTful API. Authentication for
this is performed by providing a shared secret in every request.

### Typing

The protocol is backwards-compatible with clients that do not support typing. These
clients can ignore real-time messages related to typing and simply send their completed
messages using the HTTP end-points without informing the server during typing.

## Real-time API

The real-time API deals with maintaining the state of who is online and performs
message exchange. Also it acts as a proxy for the RESTful API. The real-time
API decides which operations will be handled by it and which will be forwarded to
the Django RESTful API.
Communication happens using both HTTP and web sockets. Web socket
communication is implemented using [socket.io](http://socket.io/).

The real-time server allows the following web-socket messages to be sent:

* `login`: Indicates that a user is logging in on the server by providing the
  user information. Requires a `username` and a `ting_auth` parameter. The real-time
  service then checks that the authentication tokens are valid with the RESTful
  API. If the authentication fails, the real-time service closes the connection.
  The `ting_auth` parameter provided by the client must be gathered by the client
  by requesting a new session from the RESTful API. This message must be sent before
  any other messages.

The real-time server can publish using web-sockets the following messages:

* `login-response`: Indicates if a user logged in on the server. It includes
  two parameters. The `success` parameter which is a boolean that indicates 
  if the log in procedure succeded. The second parameter is a string called `error` 
  in case the log in failed and describes the error that occured, otherwise it
  is called `people` and is an array of strings which are usernames that are 
  online the time that the user logged in. The possible error responses are
  `empty`, `length`, `chars`, `taken`.

* `join`: Indicates a user is now online. Includes one parameter, the
  `username`. This message is also sent back to the user who
  has attempted to login if it was successful. All users online
  receive the join message for every user that goes online.

* `part`: Indicates a user is now offline. Includes the `username`.
  This message is sent to all online users when another user's connection
  is dropped or they go offline.

* `message`: Indicates a user has messaged in a channel you are in or in a private
  window. Includes four parameters, `username`, `type`, `target`, and `text`,
  as per above. If `type` is set to `user`, then `target` must be your username.

* `update-typing-messages`: Indicates that the messages that are being typed are updated.
  It includes one parameter, `messages_typing`, which is a dictionary of messages.
  The dictionary of messages contains a key with its messageid for each message, whose
  value is a dictionary that describes an individual message.
  That dictionary that describes the message contains four attributes:

  1. `text`: A string which is the text of the message.
  2. `username`: A string which is the username of the user that's typing the message.
  3. `typing`: A boolean that indicates if the message is currently being typed
     or not. If this is set to false, this means that the message is now
     persistent.
  4. `datetime_start`: A unix epoch in ms which indicates the datetime the
     message started being typed.

The real-time server exposes an HTTP endpoint at `https://ting.gr/api/v1` with the following
URLs:

* A PATCH operation on `/message`: Sends an update on the message with an id equal to <messageid>.
  The message can be sent from the user to a channel or to another user directly.
  Takes five parameters:

  1. `type`: A string which is either `channel` or `user`, indicating whether
     the recipient of the message is a channel or a user.
  2. `target`: The name of the channel or the user we wish to send the message
     to.
  3. `text`: The text of the message.
  4. `typing`: A boolean that indicates if the message is currently
     being typed or not.
  5. `messageid`: An integer which is the unique id of the message being sent.

     * There is no response for this operation.

* A POST operation on `/message`: Indicates that a user started typing a new message. Expects
  the following parameters:

  1. `type`: A string which is either `channel` or `user`.
  2. `target`: The name of the channel or the user we wish to send the message
     to.
  3. `text`: The text of the message typed so far.

    * The response contains the `messageid`, which is the id of the
      new message that is currently being typed.

* Operations on `/messages/` and `/channels/` are forwarded to the RESTful API
  with the appropriate parameters..

## RESTful API

The RESTful API deals with four resources: Messages, Channels, Users, Sessions. The
responses are always given in JSON. As such, we make no use of Django templates,
only models and views. The URLs of the RESTful API live under the
`https://ting.gr/restapi/v1` URL.

### Messages
The Messages resource is used to store and retrieve chat messages. It is
accessible through the `/messages` URL.

There are four operations:

1. A GET operation on `/messages/<type>/<target>`. This retrieves the chat
   messages recently exchanged on a channel or private. `type` is a string
   which is either `channel` or `user` and `target` is the name of the channel
   or the username.

   If `type` is `user`, then the private messages returned by this request are
   between the user making the request and the target.

   They are returned as a JSON array of messages. By default, the number of
   messages returned is limited to 100. The GET variable `lim` can be used to
   alter the limit. The messages are ordered from newest to oldest. Each
   message is represented as a dict with six keys:

   * `id`: The id of the message in the database.
   * `text`: The text of the chat message.
   * `username`: The username of the person who wrote the message. If the
     `type` was set to `user`, then this must always be equal to `target`.
   * `datetime_start`: The time the message started being typed, in UTC epoch milliseconds.
   * `datetime_sent`: The time the message was sent, in UTC epoch milliseconds.
   * `typing`: Indicates whether the message is currently being typed.
      Takes a boolean value.

2. A POST operation on `/messages/<type>/<target>`. This is a **privileged
   operation** that persists a message on a given channel or private. The POST
   body contains a dictionary with four keys, `text`, `username`,
   `datetime_start` and `typing`, with the semantics above.

3. A PATCH operation on `/messages/<id>`. This is a **privileged
   operation** that updates a message with a given id. The PATCH body
   contains a dictionary with three keys `text`, `datetime_sent`
   and `typing`, with the semantics above, and with the values to update.

4. A DELETE operation on `/messages/<id>`. This is a **privileged
   operation** that deletes a given message.

### Channels
The Channels resource is used to create and retrieve channel information.
It is accessible through the `/channels` URL.

There are two operations:

1. A GET operation on `/channels/<channel_name>`. This retrieves information
   about a given channel. If the channel does not exist, it causes a 404 error
   code. Otherwise, a JSON dict with a description of the channel is returned.
   It contains only a single key, `name`, with its value set to the channel
   name.

2. A POST operation on `/channels`. This creates a new channel with the given
   name. The POST body contains a dictionary with one key, "name", which
   contains the name of the channel.

###Users
The Users resource is used to update user data like the password and birthday.
It is accessible through the `/users` URL.

There is 1 operation:

1. A PATCH operation on `/users/<username>`. This updates the user's data with the given ones.
The PATCH body contains a dictionary with up to 4 keys. If a key isn't contained in 
the dict during the request then it doesn't get updated:

 * `email`: A string containing the user's e-mail address. The e-mail address is validated. The
   email can be empty, in case the user wants to delete it.
 * `password`: A string chosen from the user.
 * `birthday`: A past valid date in the form `YYYY-MM-DD` after 1900. 
 * `gender`: Its value is going to be chosen from 'male', 'female' and '-' with '-' being 
   the default.

In case the request contains invalid data, a 422 status code is returned.

The operation succeeds only if the <username> is the same as the username of the authenticated user.
In case these usernames differ or there isn't any authenticated user, it returns a 403 
status code.

###Session
The Session resource is used to create and delete a session for logging in and out. 
It is accessible through the `/sessions` URL. 

There are 3 operations:

####Logging in

This operation is performed by the client to the RESTful API server for logging in.

A POST request on `/sessions`. The body of the request contains a dictionary:

 * `username`: A mandatory string. In case it isn't reserved, it gets validated as 
   mentioned in the [spec](https://github.com/dionyziz/ting/blob/master/etc/spec/SPECIFICATION.md).

 * `password`: An optional string in case the username is reserved. 

The server checks if the username is reserved. 

 * If the username is reserved:
 
     - If the password of the already created user is set: 
        
        - If there isn't any password on the request body, the server
          returns a 403 status code with the error `password_required`.

        - If the password value of the request isn't correct, the server
          returns a 403 status code with the error `wrong_password`.
        
        - If the password value of the request is correct, the server returns
          a 200 status code.

     - If the password of the already created user is not set, the server returns
       a 403 status code with the error `username_reserved`. 
     
 * If the username is not reserved:

     - If the password is set in the request, the server returns a 404 status code.

     - If the password is not set in the request, the server returns a 200 status code. 

In case the session is created successfully, a cookie named `ting_auth` will be sent 
from the server for future authentication.
    
At this point we should mention that if the user isn't created before the `/sessions` POST
request, it gets created internally. 

####Logging out

This operation is performed by the client to the RESTful API server for logging out.

A DELETE request on `/sessions`.

During this request the server checks if this user has set a password. If not, then along 
with the session, the user gets deleted, too.

####Checking authentication

This operation is performed by the real-time service to the RESTful API server for verifying
that a given client cookie is correct.

A GET request on `/sessions/<username>`. This request must be performed using the `ting_auth`
cookie that the user provided. If the cookie is valid for the given user, a 204 (No Content) response
is returned. Otherwise, a 403 (Forbidden) response is returned.

This is a privileged operation.
