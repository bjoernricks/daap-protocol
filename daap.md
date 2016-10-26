# Digital Audio Access Protocol (DAAP) Protocol documentation

**Version 0.3**

The original version 0.2 can be found at http://tapjam.net/daap/

* [Introduction](#i-introduction)
* [DAAP Overview](#ii-daap-overview)
* [DAAP Packet Format](#iii-under-the-hood---the-daap-packet-format)
* [Client/Server conversation overview](#iv-clientserver-conversation-overview)
  * [server-info](#1-server-info)
  * [content-codes] (#2-content-codes)
  * [login] (#3-login-and-session-id)
  * [update] (#4-update-and-server-revision)
  * [databases] (#5-database-list)
  * [items] (#6-song-list)
  * [containers] (#7-playlist-list)
  * [playlist] (#8-playlist)
  * [song] (#9-stream-song)
* [Appendix A - Content Codes] (#appendix-a---content-codes)
* [Appendix B - Content Types] (#appendix-b---content-types)
* [Appendix C - Protocol Changes] (#appendix-c---protocol-changes)
* [Changes to this document] (#changes-to-this-document)

## I. Introduction


Apple introduced the Digital Audio Access Protocol (DAAP) with
iTunes 4, to allow iTunes to share its library with other machines also
running iTunes.  iTunes uses Bonjour <sup id="a1">[1](#f1)</sup>,
specifically the multi-cast dns and service discovery bits, to find other local
machines that happen to be running iTunes and also sharing their
libraries.

## II. DAAP Overview


DAAP, in a nutshell, is a fairly simple protocol.  Once iTunes has
found another instance, it will connect to, and download basic server
information about that instance.  It will then download a dictionary
of attributes that will be used at various other points in the
conversation.  iTunes then makes a series of DAAP requests to download
the list of songs from the server, as well as the various playlists of
those songs that the server has shared.  Finally, when iTunes wishes to
stream a song from the server, once again, a request is made to the
server to stream the selected track back to iTunes.  In addition,
there is a provision for allowing the client to pick up on when
changes are made to the server.

The iTunes implementation of DAAP imposes some restrictions.  Most
notably, it limits a user to 5 connections/sessions at any given
point.  Note, that a session doesn't necessarily have to be actively
streaming music to be counted to that five connections.  (The limit on
connections is actually 10, as a client may have up to two connections
to the server - one for the database/playlist updates, the other for
streaming).

Interestingly enough, the iTunes implementation of DAAP does not
restrict access to the local network.  This has lead to the rise of a
number of iTunes server repositories, including one that uses the DAAP
protocol to download the playlists of servers registered with it, and
uses that information to provide a search engine.


## III. Under the Hood - the DAAP packet format


At it's heart, DAAP is the latest in a long line of protocols that
uses HTTP as it's underlying transport mechanism.
It appears to prefer using the connection-oriented HTTP/1.1.
*Though there is also some support, it seems, for HTTP/1.0.  This document will
focus on the HTTP/1.1 implementation, and assume that the HTTP/1.0 implementation
is around for some strange legacy reason.*

(Note: the specifics of writing HTTP clients and servers are beyond
the scope of this document.  There are a number of excellent libraries
and resources out there for providing this functionality, however).

While the requests are normal URLs (documented further below), the
responses have the mime-type application/x-dmap-tagged.  This type
appears to be a flattened form of a plist or xml formatted file.  (In
fact, it would be fairly trivial to write an x-dmap-tagged to xml
converter, with just a little bit of foreknowledge on the specific
tags).

The basic format of DAAP (expanded on below) is:

|tag name      |size        | data       |
|:------------:|:----------:|:----------:|
| 4 byte ASCII | 4 byte int | size bytes |

where the length of the data is determined by the size.  The 4 byte
tag can be represented as either an integer, or as a four character
string.  (Of course, the characters in that string form the bytes of
the 4 byte integer, highest order byte first - e.g. the code 'minm'
has the numeric value 1835626093.  This jumping back and forth comes
in real handy in writing code to deal with daap, as it allows you to
use switch/jump tables to react to the various codes, with a minimal
amount of hash-table lookup/string conversion/etc...)

Very early on in the DAAP conversation, a 'map' of content codes,
their long name, and their types gets sent back.  This map is used to
provide the basic description for a number of fields used in various
points further down.  See appendix A for a full listing of codes used
in the iTunes conversation.

An example, simple dmap-tagged block, the response to a /login
request:

```
0000000  155 154 157 147 000 000 000 044 155 163 164 164 000 000 000 004
           m   l   o   g  \0  \0  \0   $   m   s   t   t  \0  \0  \0 004
0000020  000 000 000 310 155 154 151 144 000 000 000 004 000 000 037 336
          \0  \0  \0 310   m   l   i   d  \0  \0  \0 004  \0  \0 037 336
```

or:


  * [mlog](#user-content-mlog) (36 bytes)
    * [mstt](#user-content-mstt) - 200  (4 bytes)
    * [mlid](#user-content-mlid) - 8158 (4 bytes)


NOTE: For some reason, iTunes sometimes lies about the size of the
response that its sending back, claiming to be sending back more than
it actually ends up sending.  No reason for this has been found, and
iTunes behaves properly if sent a packet with the correct size, so it
seems to be a non issue (save for possibly sloppy coding on apple's
part?).  The login response may be padded for an extra field that
doesn't always get dropped in (but iTunes always counts it).

For container types (e.g. content codes with a type 12, or list), the
data can, itself contain more code/size/data blocks.  Lists will
contain multiple code/size/data blocks, one for each item in the list.

All sizes are in bytes and count only the following data block.


## IV. Client/Server conversation overview


Here's a quick diagram of the flow of requests/responses in a DAAP
session:

```
    client        server   Description
    ---/server-info---->   Request for server info
    <-------msrv--------   server info response

    ---/content-codes-->   request for content-codes
    <-------mccr--------   content-codes response

    ---/login---------->   login
    <-------mlog--------   login response (with sessionid)

    ---/update--------->   update request
    <-------mupd--------   update response (with server rev)

    ---/databases------>   database request
    <-------avdb--------   data base response (with dbid)

    ---/db/id/items---->   request songs
    <-------adbs-------    list of songs in database

    -/db/id/containers->   request playlists
    <-------aply--------   list of playlists in database

    -/db/id/c/id/items->   request playlist
    <-------apso--------   list of songs in playlist
    -/db/id/c/id/items->
    <-------apso--------
    -/db/id/c/id/items->
    <-------apso--------
    -/db/id/c/id/items->
    <-------apso--------

    -/db/id/items/x.mp3->  request mp3
    <--stream-mp3-file---  stream'd mp3
```

(For some reason, the iTunes server flags the content type as
x-dmap-tagged for the streamed mp3 - i don't know if this is laziness
on apple's part, or an intent to trip up web browsers trying to
download tracks).

The following sections will describe the requests and responses in
detail.  These sections will be expanded on ures are discovered and
documented.  (Note: daap://server can be interchanged with
http://server:3689 for purposes of coding/using command line utilities
such as curl/wget to poke at things)

### 1. Server Info

Request: daap://server/server-info (or http://server:3689/)

Response: msrv

Description: Provides basic negotiation info on what the server does
    and doesn't support and protocols.

Content: (See appendix A for detailed information on codes)
```
    msrv
      mstt - status
      apro - daap protocol
      msix - does the server support indexing?
      msex - does the server support extensions?
      msup - does the server support update?
      msal - does the server support auto-logout?
      mstm - timeout interval
      mslr - is login required?
      msqy - does the server support queries?
      minm - server name
      msrs - does the server support resolve?  (needs persistent ids)
      msbr - does the server support browsing?
      mspi - does the server support persistent ids?
      mpro - dmap protocol version
```

### 2. Content Codes

Request: daap://server/content-codes

Response: mccr

Description: Provides a dictionary of content codes, names, and size
    information.

Content: ( See Appendix A for detailed code information )
```
    mccr
      mstt      status
      mdcl      dictionary entry (one for each type)
        mcna    item name
        mcnm    item number
        mcty    item type
```

### 3. Login (and session id)

Request: daap://server/login

Response: mlog

Description: Provides session information to use for the rest of the
    session.  If requiresLogin is set in the server-info, then
    from this point on, a basic http authentication header needs
    to be included with the request.  It is basically a header of the
    form:
        Authentication: base64encodedusernamepassword
    where the base64 encoded password is the string:
        username:password
    base64 encoded (obviously replacing username and password with
    the appropriate values).  Username appears to be meaningless.
    A shame - i like the idea of being able to do user level
    permissions.

Content:
```
    mlog
      mstt  status
      mlid  session id (used in remaining requests <sid> below)
```

### 4. Update (and server revision)

Request: daap://server/update?session-id=**&lt;sid&gt;**

Response: mupd

Description: There seem to be two forms update requests.  The first is
    the update when logging in, to get the initial data.  The
    second (not yet documented) is to catch updates/changes to
    the server database.  Note the addition of the session-id
    from the login.

Conent:
```
    mupd
      musr  the server revision (<rid> below)
      mstt
```

### 5. Database list

Request: daap://server/databases?session-id=**&lt;sid&gt;**&revision-id=**&lt;rid&gt;**

Response: avdb

Description: This provides a list of databases served up by the
    server.  At present, it appears that only one db per server
    is the norm, but it may be possible to have multiple servers.

Content:
```
    avdb
      mstt - status
      muty - update type - always 0 for now
      mtco - total number of matching records
      mrco - total number of records returned
      mlcl - listing of records
        mlit - single record
          miid - database id (<dbid> in subsequent requests)
          mper - database persistent id
          minm - database name
          mimc - number of items (songs) in the database
          mctc - number of containers (playlists) in the database
```

### 6. Song list

Request: daap://server/databases/**&lt;dbid&gt;**/items?type=music&meta=**&lt;list of fields&gt;**&session-id=**&lt;sid&gt;**&revision-id=**&lt;rid&gt;**

Response: apso

Description: This is the list of songs in the database.  Every song.
    Note given the format of the records and the request, I believe
    that it's possible to have other types as well.  The standard
    list of fields is (e.g. the value for meta):

```
        dmap.itemid,dmap.itemname,dmap.itemkind,dmap.persistentid,
        daap.songalbum,daap.songartist,daap.songbitrate,
        daap.songbeatsperminute,daap.songcomment,daap.songcompilation,
        daap.songcomposer,daap.songdateadded,daap.songdatemodified,
        daap.songdisccount,daap.songdiscnumber,daap.songdisabled,
        daap.songeqpreset,daap.songformat,daap.songgenre,
        daap.songdescription,daap.songrelativevolume,
        daap.songsamplerate,daap.songsize,daap.songstarttime,
        daap.songstoptime,daap.songtime,daap.songtrackcount,
        daap.songtracknumber,daap.songuserrating,daap.songyear,
        daap.songdatakind,daap.songdataurl,com.apple.itunes.norm-volume
```

    Note that in the response, it is very important that itemtype(mikd)
    be returned first for each record.  It looks like the order of the
    remaining fields can be arbitrary, but it is vital that itemid
    be first.  It also appears to be permissable to leave out fields
    that are request.  I'd imagine there is some subset of fields
    that is required, however.

Response:
```
    apso
      mstt - status
      muty - update type (0)
      mtco - matched items
      mrco - items in message
      mlcl - list of items
        mlit - single item (one per song)
          mikd - item kind (2 for music)
          miid - item id
          minm - item name (e.g. the trackname)
          ...  - the remaining fields for the track
```

### 7. Playlist list

Request: daap://server/databases/**&lt;dbid&gt;**/containers?meta=**&lt;containermeta&gt;**&session-id=**&lt;sid&gt;**&revision-id=**&lt;rid&gt;**

Response: aply

Description: provides a list of the playlists stored on the server, as
    well as some basic information about them, depending on the
    metadata that the client asks for.  So far, iTunes seems to ask
    for:
```
        dmap.itemid,dmap.itemname,dma.persistentid,
        com.apple.itunes.smart-playlist
```

Content:
```
    aply
      mstt - status
      muty - update type(0)
      mtco - matched items
      mrco - items in response
      mlcl
        mlit - playlist entry (one per playlist)
          miid - the playlist's itemid (plid below)
          mper - the playlist's persistent id
          minm - the playlist's name
          mimc - the number of items in the playlist
          aeSP (optional) - is it a smart playlist
```


### 8. Playlist

Request: daap://server/databases/**&lt;dbid&gt;**/containers/**&lt;plid&gt;**/items?type=music&meta=**&lt;playlistmeta&gt;**&session-id=**&lt;sid&gt;**&revision-id=**&lt;rid&gt;**

Response: apso

Description: provides a list of items in the given playlist, providing
    the requested meta information for each item.  The most commonly
    requested items seem to be:

```
      dmap.itemkind,dmap.itemid,dmap.containeritemid
```

Content:
```
    apso
      mstt - status
      muty - update type(0)
      mtco = matched items
      mrco - returned items
      mlcl - list of playlist entries
        mlit - single playlist entry (one per song)
          mikd - item kind (2 for music)
          miid - itemid of the song
          containeritemid - id in the container
```

### 9. Stream song

Request: daap://server/databases/**&lt;dbid&gt;**/items/**&lt;itemid&gt;**.mp3?session-id=**&lt;sid&gt;**

Response: streamed mp3

Description: Request a song from the server.  Note that it is possible
    to use the 'range' http header to specify the range of a song,
    e.g. the start and end bytes for the track.  It's important for
    a server to support this, in case itunes users decide to jump
    around.

Content: the raw mp3 stream


## Appendix A - Content Codes

This is a list of the content codes used by iTunes and DAAP server
implementations.

|Code                 |  Type  |  Name                           | Description                               |
|--------------------:|:------:|:-------------------------------:|:------------------------------------------|
|<b id="mdcl">mdcl</b>|  list  | dmap.dictionary                 | a dictionary entry |
|<b id="mstt">mstt</b>|  int   | dmap.status                     | the response status code these appear to be http status codes, e.g. 200 |
|<b id="miid">miid</b>|  int   | dmap.itemid                     | an item's id |
|<b id="minm">minm</b>| string | dmap.itemname                   | an items name |
|<b id="mikd">mikd</b>|  byte  | dmap.itemkind                   | the kind of item.  So far, only '2' has been seen, an audio file? |
|<b id="mper">mper</b>|  long  | dmap.persistentid               | a persistend id |
|<b id="mcon">mcon</b>|  list  | dmap.container                  | an arbitrary container |
|<b id="mcti">mcti</b>|  int   |dmap.containeritemid             | the id of an item in its container |
|<b id="mpco">mpco</b>|  int   | dmap.parentcontainerid          | |
|<b id="msts">msts</b>| string | dmap.statusstring               | |
|<b id="mimc">mimc</b>|  int   | dmap.itemcount                  | number of items in a container |
|<b id="mrco">mrco</b>|  int   | dmap.returnedcount              | number of items returned in a request |
|<b id="mtco">mtco</b>|  int   | dmap.specifiedtotalcount        | number of items in response to a request |
|<b id="mlcl">mlcl</b>|  list  | dmap.listing                    | a list |
|<b id="mlit">mlit</b>|  list  | dmap.listingitem                | a single item in said list |
|<b id="mbcl">mbcl</b>|  list  | dmap.bag                        | |
|<b id="mdcl">mdcl</b>|  list  | dmap.dictionary                 | |
||
|<b id="msrv">msrv</b>|  list  | dmap.serverinforesponse         | response to a /server-info |
|<b id="msau">msau</b>|  byte  | dmap.authenticationmethod       | (should be self explanitory) |
|<b id="mslr">mslr</b>|  byte  | dmap.loginrequired              |
|<b id="mpro">mpro</b>| version| dmap.protocolversion            |
|<b id="apro">apro</b>| version| daap.protocolversion            |
|<b id="msal">msal</b>|  byte  | dmap.supportsuatologout         |
|<b id="msup">msup</b>|  byte  | dmap.supportsupdate             |
|<b id="mspi">mspi</b>|  byte  | dmap.supportspersistentids      |
|<b id="msex">msex</b>|  byte  | dmap.supportsextensions         |
|<b id="msbr">msbr</b>|  byte  | dmap.supportsbrowse             |
|<b id="msqy">msqy</b>|  byte  | dmap.supportsquery              |
|<b id="msix">msix</b>|  byte  | dmap.supportsindex              |
|<b id="msrs">msrs</b>|  byte  | dmap.supportsresolve            |
|<b id="mstm">mstm</b>|  int   | dmap.timeoutinterval            |
|<b id="msdc">msdc</b>|  int   |dmap.databasescount              |
||
|<b id="mccr">mccr</b>|  list  | dmap.contentcodesresponse       | the response to the content-codes request |
|<b id="mcnm">mcnm</b>|  int   | dmap.contentcodesnumber         | the four letter code |
|<b id="mcna">mcna</b>| string | dmap.contentcodesname           | the full name of the code |
|<b id="mcty">mcty</b>|  short | dmap.contentcodestype           | the type of the code (see appendix b for type values) |
||
|<b id="mlog">mlog</b>|  list  | dmap.loginresponse              | response to a /login |
|<b id="mlid">mlid</b>|  int   | dmap.sessionid                  | the session id for the login session |
||
|<b id="mupd">mupd</b>|  list  | dmap.updateresponse             | response to a /update |
|<b id="msur">msur</b>|  int   | dmap.serverrevision             | revision to use for requests |
|<b id="muty">muty</b>|  byte  | dmap.updatetype                 | |
|<b id="mudl">mudl</b>|  list  | dmap.deletedidlisting           | used in updates? (document soon) |
||
|<b id="avdb">avdb</b>|  list  | daap.serverdatabases            | response to a /databases |
|<b id="abro">abro</b>|  list  | daap.databasebrowse             | |
|<b id="abal">abal</b>|  list  | daap.browsealbumlistung         | |
|<b id="abar">abar</b>|  list  | daap.browseartistlisting        | |
|<b id="abcp">abcp</b>|  list  | daap.browsecomposerlisting      | |
|<b id="abgn">abgn</b>|  list  | daap.browsegenrelisting         | |
||
|<b id="adbs">adbs</b>|  list  | daap.databasesongs              | response to a /databases/id/items |
|<b id="asal">asal</b>|  string| daap.songalbum                  | the song ones should be self exp. |
|<b id="asar">asar</b>|  string| daap.songartist                 | |
|<b id="asbt">asbt</b>|  short | daap.songsbeatsperminute        | |
|<b id="asbr">asbr</b>|  short | daap.songbitrate                | |
|<b id="ascm">ascm</b>|  string| daap.songcomment                | |
|<b id="asco">asco</b>|  byte  | daap.songcompilation            | |
|<b id="asda">asda</b>|  date  | daap.songdateadded              | |
|<b id="asdm">asdm</b>|  date  | daap.songdatemodified           | |
|<b id="asdc">asdc</b>|  short | daap.songdisccount              | |
|<b id="asdn">asdn</b>|  short | daap.songdiscnumber             | |
|<b id="asdb">asdb</b>|  byte  | daap.songdisabled               | |
|<b id="aseq">aseq</b>|  string| daap.songeqpreset               | |
|<b id="asfm">asfm</b>|  string| daap.songformat                 | |
|<b id="asgn">asgn</b>|  string| daap.songgenre                  | |
|<b id="asdt">asdt</b>|  string| daap.songdescription            | |
|<b id="asrv">asrv</b>|  byte  | daap.songrelativevolume         | |
|<b id="assr">assr</b>|  int   | daap.songsamplerate             | |
|<b id="assz">assz</b>|  int   | daap.songsize                   | |
|<b id="asst">asst</b>|  int   | daap.songstarttime              | (in milliseconds) |
|<b id="assp">assp</b>|  int   | daap.songstoptime               | (in milliseconds) |
|<b id="astm">astm</b>|  int   | daap.songtime                   | (in milliseconds) |
|<b id="astc">astc</b>|  short | daap.songtrackcount             | |
|<b id="astn">astn</b>|  short | daap.songtracknumber            | |
|<b id="asur">asur</b>|  byte  | daap.songuserrating             | |
|<b id="asyr">asyr</b>|  short | daap.songyear                   | |
|<b id="asdk">asdk</b>|  byte  | daap.songdatakind               | |
|<b id="asul">asul</b>|  string| daap.songdataurl                | |
||
|<b id="aply">aply</b>|  list  | daap.databaseplaylists          | response to /databases/id/containers |
|<b id="abpl">abpl</b>|  byte  | daap.baseplaylist               | |
||
|<b id="apso">apso</b>|  list  | daap.playlistsongs              | response to /databases/id/containers/id/items |
|<b id="prsv">prsv</b>|  list  | daap.resolve                    | |
|<b id="arif">arif</b>|  list  | daap.resolveinfo                | |
||
|<b id="aeNV">aeNV</b>|  int   | com.apple.itunes.norm-volume    | |
|<b id="aeSP">aeSP</b>|  byte  | com.apple.itunes.smart-playlist | |


## Appendix B - Content Types

(Note: the ones in parenthesis are assumed based on the arrangement of
types, however they have not yet been confirmed)

|id |type            | size     | description |
|--:|:---------------|---------:|:------------|
|1  |byte            |   1 byte |             |
|2  |(unsigned byte) |   1 byte |             |
|3  |short           |   2 byte |             |
|4  |(unsigned short)|   2 byte |             |
|5  |int             |   4 byte |             |
|6  |(unsigned int)  |   4 byte |             |
|7  |long            |   8 byte |             |
|8  |(unsigned long) |   8 byte |             |
|9  |string          | variable | Encoded as UTF-8 |
|10 |date            |   4 byte | 4 byte int as seconds since 1.1.1970 (UNIX timestamp) |
|11 |version         |   4 byte | represented as 4 single bytes, e.g. 0.1.0.0 or as two shorts, e.g. 1.0 |
|12 |list            | variable |             |

## Appendix C - Protocol Changes

With iTunes 4.0.1 apple introduced a handful of changes that
break compatability with iTunes 4.0.  The changes are simply:

 * the version number is bumped to 2.0, from 1.0
 * the sessionid needs to be an unsigned integer, it looks
   like the way session id's are created caused a signed
   sessiondid, e.g. -9184 as a signed int, the string sessionid
   needs to be unsigned, e.g. 4294958112 - I don't know if
   this goes for all numeric values, or just sessionid

## Changes to this document

 * In Version 0.3
   * Conversion into Markdown format
   * Added appendix C - Protocol Changes
   * Cleanup of chapters I, II and III

## Footnotes

<b id="f1">1</b> Apples implementation of the zeroconf standards
[Wikipedia](https://en.wikipedia.org/wiki/Bonjour_(software)). [↩](#a1)
