  Weak validation
n: The two resource representations are semantically equivalent, e.g. some of the content differences are not important from the business logic perspective e.g. current date displayed on the page might not be important for updating the entire resource for it.
The syntax for weak validation:

ETag: W/"<etag_value>" 
Note that this directive is entirely used for the server side logic and has no importance to the client browser.

  Strong Validation: 
The two resource representations are byte-for-byte identical. This is the default one and no special directive is used for it.

ETag, If-Match, and If-None-Match Headers
The ETag header provides a way to tag resources. This can prevent clients from overriding each other while also making it possible to reduce unnecessary calls.

Consider the following example:

Example 1. A POJO with a version number
link:../../../spring-data-rest-webmvc/src/test/java/org/springframework/data/rest/webmvc/support/ETagUnitTests.java[]
The @Version annotation (the JPA one in case you’re using Spring Data JPA, the Spring Data org.springframework.data.annotation.Version one for all other modules) flags this field as a version marker.

The POJO in the preceding example, when served up as a REST resource by Spring Data REST, has an ETag header with the value of the version field.

We can conditionally PUT, PATCH, or DELETE that resource if we supply a If-Match header such as the following:

curl -v -X PATCH -H 'If-Match: <value of previous ETag>' ...
Only if the resource’s current ETag state matches the If-Match header is the operation carried out. This safeguard prevents clients from stomping on each other. Two different clients can fetch the resource and have an identical ETag. If one client updates the resource, it gets a new ETag in the response. But the first client still has the old header. If that client attempts an update with the If-Match header, the update fails because they no longer match. Instead, that client receives an HTTP 412 Precondition Failed message to be sent back. The client can then catch up however is necessary.

Warning
The term, “version,” may carry different semantics with different data stores and even different semantics within your application. Spring Data REST effectively delegates to the data store’s metamodel to discern if a field is versioned and, if so, only allows the listed updates if ETag elements match.
The If-None-Match header provides an alternative. Instead of conditional updates, If-None-Match allows conditional queries. Consider the following example:

curl -v -H 'If-None-Match: <value of previous etag>' ...
The preceding command (by default) executes a GET. Spring Data REST checks for If-None-Match headers while doing a GET. If the header matches the ETag, it concludes that nothing has changed and, instead of sending a copy of the resource, sends back an HTTP 304 Not Modified status code. Semantically, it reads “If this supplied header value does not match the server-side version, send the whole resource. Otherwise, do not send anything.”

Note
This POJO is from an ETag-based unit test, so it does not have @Entity (JPA) or @Document (MongoDB) annotations, as expected in application code. It focuses solely on how a field with @Version results in an ETag header.
If-Modified-Since header
The If-Modified-Since header provides a way to check whether a resource has been updated since the last request, which lets applications avoid resending the same data. Consider the following example:

Example 2. The last modification date captured in a domain type
link:../../../spring-data-rest-tests/spring-data-rest-tests-mongodb/src/main/java/org/springframework/data/rest/tests/mongodb/Receipt.java[]
Spring Data Commons’s @LastModifiedDate annotation allows capturing this information in multiple formats (JodaTime’s DateTime, legacy Java Date and Calendar, JDK8 date/time types, and long/Long).

With the date field in the preceding example, Spring Data REST returns a Last-Modified header similar to the following:

Last-Modified: Wed, 24 Jun 2015 20:28:15 GMT
This value can be captured and used for subsequent queries to avoid fetching the same data twice when it has not been updated, as the following example shows:

curl -H "If-Modified-Since: Wed, 24 Jun 2015 20:28:15 GMT" ...
With the preceding command, you are asking that a resource be fetched only if it has changed since the specified time. If so, you get a revised Last-Modified header with which to update the client. If not, you receive an HTTP 304 Not Modified status code.

The header is perfectly formatted to send back for a future query.

Warning
Do not mix and match header value with different queries. Results could be disastrous. Use the header values ONLY when you request the exact same URI and parameters.
Architecting a More Efficient Front End
ETag elements, combined with the If-Match and If-None-Match headers, let you build a front end that is more friendly to consumers' data plans and mobile battery lives. To do so:

Identify the entities that need locking and add a version attribute.

HTML5 nicely supports data-* attributes, so store the version in the DOM (somewhere such as an data-etag attribute).

Identify the entries that would benefit from tracking the most recent updates. When fetching these resources, store the Last-Modified value in the DOM (data-last-modified perhaps).

When fetching resources, also embed self URIs in your DOM nodes (perhaps data-uri or data-self) so that it is easy to go back to the resource.

Adjust PUT/PATCH/DELETE operations to use If-Match and also handle HTTP 412 Precondition Failed status codes.

Adjust GET operations to use If-None-Match and If-Modified-Since and handle HTTP 304 Not Modified status codes.

By embedding ETag elements and Last-Modified values in your DOM (or perhaps elsewhere for a native mobile app), you can then reduce the consumption of data and battery power by not retrieving the same thing over and over. You can also avoid colliding with other clients and, instead, be alerted when you need to reconcile differences.
