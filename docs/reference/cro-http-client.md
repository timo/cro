# Cro::HTTP::Client

The `Cro::HTTP::Client` class provides a flexible HTTP and HTTPS client
implementation, scaling from simple to more complex use cases. It can be
consumed in two ways:

* By making calls on the type object (`Cro::HTTP::Client.get($url)`). This
  is good for one-off requests, but does not provide connection re-use when
  making multiple requests to the same server (such as by using keep-alive).
* By making an instance of `Cro::HTTP::Client`. By default, this enables
  re-use of a pool of connections. It may also be configured with a default
  base URL, default authorization data to pass along, and even middleware to
  insert into the request/response processing pipeline. An instance of
  `Cro::HTTP::Client` may be used concurrently.

## Making basic requests

The `get`, `post`, `put`, `delete`, and `head` methods may be called on either
the type object or an instance of `Cro::HTTP::Client`. They will all return a
`Supply`. Since by default only a single HTTP response will be produced, it is
possible to `await` it:

    my $resp = await Cro::HTTP::Client.get('https://www.perl6.org/');

The response will be provided as a `Cro::HTTP::Response` object. It will be
produced as soon as the request headers are available; the body may not yet
have been received. By default, errors (4xx and 5xx status codes) will result
in an exception that does the `X::Cro::HTTP::Error` role, which has a
`response` property containing the `Cro::HTTP::Response` object.

    my $resp = await Cro::HTTP::Client.delete($product-url);
    CATCH {
        when X::Cro::HTTP::Error {
            if .response.status == 404 {
                say "Product not found!";
            }
            else {
                say "Unexpected error: $_";
            }
        }
    }

The actual exception type will be either `X::Cro::HTTP::Error::Client` for
4xx errors, and `X::Cro::HTTP::Error::Server` for 5xx errors (which is useful
when setting up retries that should distinguish server errors from client
errors).

## Adding extra request headers

One or more headers can be set for a request by passing an array to the
`headers` named argument. It may contain either `Pair` objects, instances
of `Cro::HTTP::Header`, or a mix of the two.

    my $resp = await Cro::HTTP::Client.get: 'example.com',
        headers => [
            referer => 'http://anotherexample.com',
            Cro::HTTP::Header.new(
                name => 'User-agent',
                value => 'Cro'
            )
        ];

If the headers should be added to all requests, they can be set by default at
construction time:

    my $client = Cro::HTTP::Client.new:
        headers => [
            User-agent => 'Cro'
        ];

## Setting the request body

To give the request a body, pass the `body` named argument. The `content-type`
named argument should typically be passed too, to indicate the type of the
body. For example, a request with a JSON body can be sent as:

    my %panda = name => 'Bao Bao', eats => 'bamboo';
    my $resp = await Cro::HTTP::Client.post: 'we.love.pand.as/pandas',
        content-type => 'application/json',
        body => %panda;

If writing a client for a JSON API, it may become tedious to set the content
type on every request. In this case, it can be set when constructing an
instance of the client, and used by default (note that it will only be used
if a body is set):

    # Configure with JSON content type.
    my $client = Cro::HTTP::Client.new: content-type => 'application/json';

    # And later get it added by default.
    my %panda = name => 'Bao Bao', eats => 'bamboo';
    my $resp = await $client.post: 'we.love.pand.as/pandas', body => %panda;

The `Cro::HTTP::Client` class uses a `Cro::HTTP::BodySerializer` in order to
serialize request bodies for sending. Besides JSON, there are body parsers
encoding and sending a `Str`:

    my $resp = await Cro::HTTP::Client.post: 'we.love.pand.as/facts',
        content-type => 'text/plain; charset=UTF-8',
        body => "99% of a Panda's diet consists of bamboo";

A `Blob`:

    my $resp = await Cro::HTTP::Client.put: 'we.love.pand.as/images/baobao.jpg',
        content-type => 'image/jpeg',
        body => slurp('baobao.jpg', :bin);

Form data formatted according to 'application/x-www-form-urlencoded` (this is
the default in a web browser):

    my $resp = await Cro::HTTP::Client.post: 'we.love.pand.as/pandas',
        content-type => 'application/x-www-form-urlencoded',
        # Can use a Hash; an Array of Pair allows multiple values per name
        body => [
            name => 'Bao Bao',
            eats => 'bamboo'
        ];

Or form data formatted according to `multipart/form-data` (this is used in web
browsers for forms that contain file uploads):

    my $resp = await Cro::HTTP::Client.post: 'we.love.pand.as/pandas',
        content-type => 'multipart/form-data',
        body => [
            # Simple pairs for simple form values
            name => 'Bao Bao',
            eats => 'bamboo',
            # For file uploads, make a part object
            Cro::HTTP::Body::MultiPartFormData::Part.new(
                headers => [Cro::HTTP::Header.new(
                    name => 'Content-type',
                    value => 'image/jpeg'
                )],
                name => 'photo',
                filename => 'baobao.jpg',
                body-blob => slurp('baobao.jpg', :bin)
            )
        ];

To replace the set of body serializers that a client will use, pass an array
of them when constructing an instance of `Cro::HTTP::Client` using the
`body-serializers` named argument:

    my $client = Cro::HTTP::Client.new:
        body-serializers => [
            Cro::HTTP::BodySerializer::JSON,
            My::BodySerializer::XML
        ];

To instead retain the existing set of body serializers and add some new ones
(which will have higher precedence), use `add-body-serializers`:

    my $client = Cro::HTTP::Client.new:
        add-body-serializers => [ My::BodySerializer::XML ];

It is also possible to have the body come from a stream of bytes by passing a
`Supply` to `body-byte-stream`.

    my $resp = await Cro::HTTP::Client.post: 'example.com/incoming',
        content-type => 'application/octet-stream',
        body-byte-stream => $supply;

The `body` and `body-byte-stream` arguments cannot be used together; trying to
do so will result in a `X::Cro::HTTP::Client::BodyAlreadySet` exception.

## Getting the response body

The response body is always provided asynchronously, either by a `Promise` (if
requesting the enitre body) or a `Supply` (when the body is to be delivered as
it arrives).

The `body` method returns a `Promise` that will be kept when the body has
been received and parsed. There default set of body parsers are:

* JSON, which will be used when the `Content-type` header is either
  `application/json` or uses the `+json` suffix. `JSON::Fast` will be used to
  perform the parsing.
* String fallback, which is used when the `Content-type` type is `text`. A
  `Str` will be returned.
* Blob fallback, which is used in all other cases and returns a `Blob` with
  the body.

A `Cro::HTTP::Client` can be configured either with a replacement set of body
parsers by passing the `body-parsers` argument:

    my $client = Cro::HTTP::Client.new:
        body-parsers => [
            Cro::HTTP::BodyParser::JSON,
            My::BodyParser::XML
        ];

Or to add extra body parsers atop of the default set use `add-body-parsers`:

    my $client = Cro::HTTP::Client.new:
        add-body-parsers => [ My::BodyParser::XML ];

To get the entire response body as a `Blob`, use the `body-blob` method:

    my Blob $body = await $resp.body-blob();

To get the entire response body as a `Str`, use the `body-text` method:

    my Str $body = await $resp.body-text();

This method will look at the `Content-type` header to see if a `charset` is
specified, and decode the body using that. Otherwise, it will see if the body
starts with a [BOM](https://en.wikipedia.org/wiki/Byte_order_mark) and rely on
that. If it is not passed, the a heuristic will be used: if the body can be
decoded as `utf-8` then it will be deemed to be `utf-8`, and failing that it
will be decoded as `latin-1` (which can never fail as all bytes are valid).

## Cookies

By default, cookies in the response are ignored. However, constructing a
`Cro::HTTP::Client` with the `:cookie-jar` option (that is, passing `True`)
will create an instance of `Cro::HTTP::Client::CookieJar`. This will be used
to store all cookies set in responses. Relevant cookies will automatically be
included in follow-up requests.

    my $client = Cro::HTTP::Client.new(:cookie-jar);

Cookie relevance is determiend by considering host, path, and the `Secure`
extension. Cookies that have passed their expiration date for maximum age will
automatically be removed from the cookie jar.

It is also possible to pass in an instance of `Cro::HTTP::Client::CookieJar`,
which makes it possible to share one cookie jar amongst several instances of
the client (or to pass in a subclass that adds extra features).

    my $jar = Cro::HTTP::Client::CookieJar.new;
    my $client = Cro::HTTP::Client.new(cookie-jar => $jar);
    my $json-client = Cro::HTTP::Client.new:
        cookie-jar => $jar,
        content-type => 'application/json';

To include a particular set of cookies with a request, pass them in a hash
using the `cookies` named argument when making a reuqest:

    my $resp = await $client.get: 'http://somesite.com/',
        cookies => {
            session => $fake-session-id
        };

Cookies passed in this way will *override* any cookies from a cookie jar.

To get the cookies set by a response, use the `cookies` method on the
`Cro::HTTP::Response` object, which returns a `List` of `Cro::HTTP::Cookie`
objects.

## Following redirects

By default, `Cro::HTTP::Client` will follow HTTP redirect responses, with a
limit of 5 redirects being enforced in order to avoid circular redirects. If
there are more than 5 redirections, `X::Cro::HTTP::Client::TooManyRedirects'
will be thrown.

This behavior can be configured when constructing a new `Cro::HTTP::Client` or
on a per-request basis, with the per-request setting overriding the behavior
configured at construction time. In either case, it is done using the `follow`
named argument.

    :follow         # follow redirects (up to 5 times per request)
    :!follow        # never follow redirects
    :follow(2)      # follow redirects (up to 2 times per request)
    :follow(10)     # follow redirects (up to 10 times per request)

The 301, 307 and 308 redirects are treated identically at this time; no
caching of permanent redirects takes place. They retain the original request
method. 302 and 303 instead cause a `GET` request to be issued, regardless of
the original request method.

## Authentication

Both basic authentication and bearer authentication are supported directly by
`Cro::HTTP::Client`. These can be configured when instantiating the client, or
per request (which will override that configured on the instance).

For basic authentication, pass the `auth` option with a hash containing a
username and a password.

    auth => {
        username => $user,
        password => $password
    }

For bearer authentication, pass the `auth` option with a hash containing a
bearer:

    auth => { bearer => $jwt }

Failing to pass precisely either `username` and `password` *or* `bearer` will
result in an `X::Cro::Client::InvalidAuth` exception.

In both cases, the authentication information will be sent immediately with
the request. In order to only have it sent if the server responds to the
initial request with a 401 response, set the `if-asked` option to `True`.

    auth => {
        username => $user,
        password => $password,
        if-asked => True
    }