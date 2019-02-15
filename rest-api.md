---
layout: documentation
doctitle: RESTful-API
---

### Introducation
We provide a simple, asynchronous RESTful-API. Authorization is done via OAuth2 (Password Credentials/Resource Owner Flow) using a Keycloak authorization server instance. Keycloak can be set up as described in this excellent [tutorial](
https://piotrminkowski.wordpress.com/2017/09/15/building-secure-apis-with-vert-x-and-oauth2/).
All you have to change is setting the client-id to "sirix" and put the client secret into our [configuration file]( https://raw.githubusercontent.com/sirixdb/sirix/master/bundles/sirix-rest-api/src/main/resources/sirix-conf.json). Change the value of "client.secret" to whatever Keycloak set up (can be found on the credetials tab of your account). Regarding Keycloak the direct access grant on the settings tab must be enabled. Our user-roles are "create" to allow creating databases/resources, "view" to allow to query database resources, "modify" to modify a database resource and "delete" to allow deletion thereof. Furthermore, a `key.pem` and a `cert.pem` file are needed. These two files have to be in your user home directory in a directory called "sirix-data", where Sirix stores the databases. For demo purposes they can be copied from our [resources directory](https://github.com/sirixdb/sirix/tree/master/bundles/sirix-rest-api/src/main/resources).

To created a fat-JAR. Download our ZIP-file for instance, then

1. `cd bundles/sirix-rest-api`
2. `mvn clean package -DskipTests`

And a fat-JAR with all required dependencies should have been created in your target folder.

Once also Keycloak is set up we can start the server via:

`java -jar -Duser.home=/opt/intrexx sirix-rest-api-*-SNAPSHOT-fat.jar -conf sirix-conf.json -cp opt/intrexx/*`

If you like to change your user home directory to `/opt/sirix` for instance.

The fat-JAR in the future will be downloadable from the [maven repository](https://oss.sonatype.org/content/repositories/snapshots/io/sirix/sirix-rest-api/0.8.9-SNAPSHOT/).

After Keycloak and our server are up and running, we can write a simple HTTP-Client. We first have to obtain a token from the `/login` endpoint with a given "username/password" JSON-Object. Using an asynchronous HTTP-Client (from Vert.x) in Kotlin, it looks like this:

```kotlin
val server = "https://localhost:9443"

val credentials = json {
  obj("username" to "testUser",
      "password" to "testPass")
}

val response = client.postAbs("$server/login").sendJsonAwait(credentials)

if (200 == response.statusCode()) {
  val user = response.bodyAsJsonObject()
  val accessToken = user.getString("access_token")
}
```

This access token must then be sent in the Authorization HTTP-Header for each subsequent request. Storing a first resource would look like (simple HTTP PUT-Request):

```kotlin
val xml = """
    <xml>
      foo
      <bar/>
    </xml>
""".trimIndent()

var httpResponse = client.putAbs("$server/database/resource1").putHeader(HttpHeaders.AUTHORIZATION.toString(), "Bearer $accessToken").putHeader(HttpHeaders.CONTENT_TYPE.toString(), "application/xml").putHeader(HttpHeaders.ACCEPT.toString(), "application/xml").sendBufferAwait(Buffer.buffer(xml))
  
if (200 == response.statusCode()) {
  println("Stored document.")
} else {
  println("Something went wrong ${response.message}")
}
```
First, an empty database with the name `database` with some metadata is created, second the XML-fragment is stored with the name `resource1`. The PUT HTTP-Request is idempotent. Another PUT-Request with the same URL endpoint would just delete the former database and resource and create the database/resource again. Note that every request now has to contain an `HTTP-Header` which content type it sends and which resource-type it expects (`Content-Type: application/xml` and `Accept: application/xml`) for instance. This is needed as we now support the storage of and retrieval of XML or JSON-data. The following sections show the API for usage with out binary and in-memory XML representation.

The HTTP-Response should be 200 and the HTTP-body yields:

```xml
<rest:sequence xmlns:rest="https://sirix.io/rest">
  <rest:item>
    <xml rest:id="1">
      foo
      <bar rest:id="3"/>
    </xml>
  </rest:item>
</rest:sequence>
```

We are serializing the generated IDs from our storage system for element-nodes.

Via a `GET HTTP-Request` to `https://localhost:9443/database/resource1` we are also able to retrieve the stored resource again.

However, this is not really interesting so far. We can update the resource via a `POST-Request`. Assuming we retrieved the access token as before, we can simply do a POST-Request and use the information we gathered before about the node-IDs:

```kotlin
val xml = """
    <test>
      yikes
      <bar/>
    </test>
""".trimIndent()

val url = "$server/database/resource1?nodeId=3&insert=asFirstChild"

val httpResponse = client.postAbs(url).putHeader(HttpHeaders.AUTHORIZATION
                         .toString(), "Bearer $accessToken").putHeader(HttpHeaders.CONTENT_TYPE.toString(), "application/xml").putHeader(HttpHeaders.ACCEPT.toString(), "application/xml").sendBufferAwait(Buffer.buffer(xml))
```

The interesting part is the URL, we are using as the endpoint. We simply say, select the node with the ID 3, then insert the given XML-fragment as the first child. This yields the following serialized XML-document:

```xml
<rest:sequence xmlns:rest="https://sirix.io/rest">
  <rest:item>
    <xml rest:id="1">
      foo
      <bar rest:id="3">
        <test rest:id="4">
          yikes
          <bar rest:id="6"/>
        </test>
      </bar>
    </xml>
  </rest:item>
</rest:sequence>
```
The interesting part is that every PUT- as well as POST-request does an implicit `commit` of the underlying transaction. Thus, we are now able send the first GET-request for retrieving the contents of the whole resource again for instance through specifying an simple XPath-query, to select the root-node in all revisions `GET https://localhost:9443/database/resource1?query=/xml/all-time::*` and get the following XPath-result:

```xml
<rest:sequence xmlns:rest="https://sirix.io/rest">
  <rest:item rest:revision="1" rest:revisionTimestamp="2018-12-20T18:44:39.464Z">
    <xml rest:id="1">
      foo
      <bar rest:id="3"/>
    </xml>
  </rest:item>
  <rest:item rest:revision="2" rest:revisionTimestamp="2018-12-20T18:44:39.518Z">
    <xml rest:id="1">
      foo
      <bar rest:id="3">
        <xml rest:id="4">
          foo
          <bar rest:id="6"/>
        </xml>
      </bar>
    </xml>
  </rest:item>
</rest:sequence>
```

In general we support several additional temporal XPath axis:

```xquery
future::
future-or-self::
past::
past-or-self::
previous::
previous-or-self::
next::
next-or-self::
first::
last::
all-time::
```

The same can be achieved through specifying a range of revisions to serialize (start- and end-revision parameters) in the GET-request:

```GET https://localhost:9443/database/resource1?start-revision=1&end-revision=2```

or via timestamps:

```GET https://localhost:9443/database/resource1?start-revision-timestamp=2018-12-20T18:00:00&end-revision-timestamp=2018-12-20T19:00:00```

We for sure are also able to delete the resource or any subtree thereof by an updating XQuery expression (which is not very RESTful) or with a simple `DELETE` HTTP-request:

```kotlin
val url = "$server/database/resource1?nodeId=3"

val httpResponse = client.deleteAbs(url).putHeader(HttpHeaders.AUTHORIZATION
                         .toString(), "Bearer $accessToken").putHeader(HttpHeaders.ACCEPT.toString(), "application/xml").sendAwait()

if (200 == httpResponse.statusCode()) {
  ...
}
```

This deletes the node with ID 3 and in our case as it's an element node the whole subtree. For sure it's committed as revision 3 and as such all old revisions still can be queried for the whole subtree (or in the first revision it's only the element with the name "bar" without any subtree).

If we want to get a diff, currently in the form of an XQuery Update Statement (but we could serialize them in any format), simply call the XQuery function `sdb:diff`:

`sdb:diff($coll as xs:string, $res as xs:string, $rev1 as xs:int, $rev2 as xs:int) as xs:string`

For instance via a GET-request like this for the database/resource we created above, we could make this request:

`GET https://localhost:9443/?query=sdb%3Adiff%28%27database%27%2C%27resource1%27%2C1%2C2%29`

Note that the query-String has to be URL-encoded, thus it's decoded

`sdb:diff('database','resource1',1,2)`

The output for the diff in our example is this XQuery-Update statement wrapped in an enclosing sequence-element:

```xml
<rest:sequence xmlns:rest="https://sirix.io/rest">
  let $doc := sdb:doc('database','resource1', 1)
  return (
    insert nodes <xml>foo<bar/></xml> as first into sdb:select-node($doc, 3)
  )
</rest:sequence>
```

This means the `resource1` from `database` is opened in the first revision. Then the subtree `<xml>foo<bar/></xml>` is appended to the node with the stable node-ID 3 as a first child.

https://github.com/sirixdb/sirix/wiki/RESTful-API gives an overview about the API.
