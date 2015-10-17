PHP Speaks HTTP?
================

## PHP was built for the Web.

- In the 90s.

## But does it reflect how we use HTTP today?

- APIs
- Quality Assurance (testing!)
- Non-Form Payloads (XML! JSON!)

## HTTP Primer

### Request

```http
{REQUEST_METHOD} {REQUEST_URI} HTTP/{PROTOCOL_VERSION}
Header: value
Another-Header: value

Message body
```

### Response

```http
HTTP/{PROTOCOL_VERSION} {STATUS_CODE} {REASON_PHRASE}
Header: value
Another-Header: value

Message body
```

### URI

```
<scheme>://<host>(:<port>)?/<path>?<query string arguments>
```

### Query string arguments

```
name=value&name2=value2&etc
```

## PHP's Role

- Accept an incoming Request
- Create and return a Response

### Request Considerations

- We MAY need to act on the request method
- We MAY need to act on the request URI, including _query string arguments_
- We MAY need to act on one or more headers
- We MAY need to parse the request message body
- *How do we interact with a request?*

### Response Considerations

- We MAY need to set the status code
- We MAY need to specify one or more headers
- We MAY need to provide message body content
- *How do we create the response?*

## The Past: PHP v3 - 4.0.6

### Request Considerations

#### Retrieving the protocol version and request method

```php
$version = $HTTP_SERVER_VARS['PROTOCOL_VERSION']
$method  = $HTTP_SERVER_VARS['REQUEST_METHOD']
```

#### URL

```php
$url = $HTTP_SERVER_VARS['REQUEST_URI'];
// except when it isn't...
```

#### URL sources

- `$HTTP_SERVER_VARS['SCHEME']` or `$HTTP_SERVER_VARS['HTTP_X_FORWARDED_PROTO']`?
- `$HTTP_SERVER_VARS['HOST']` or `$HTTP_SERVER_VARS['SERVER_NAME']` or `$HTTP_SERVER_VARS['SERVER_ADDR']`?
- `$HTTP_SERVER_VARS['REQUEST_URI']` or `$HTTP_SERVER_VARS['UNENCODED_URL']` or `$HTTP_SERVER_VARS['HTTP_X_ORIGINAL_URLSERVER_ADDR']` or `$HTTP_SERVER_VARS['ORIG_PATH_INFO']`?

Lots of different pieces to consider: depending on the server you use, or if you're behind a proxy, the scheme and protocol may be more accurately obtained from other metadata, as will the host and port. Similarly, the URI reported in REQUEST_URI may not be accurate based on the server, whether or not rewriting occurred, etc -- and because of this, you may want to pull the query string arguments from $_SERVER instead of the request URI.

#### And...

- `$HTTP_SERVER_VARS` is only available in the global scope.

#### Getting headers

```php
// Most headers, e.g. Accept:
$accept = $HTTP_SERVER_VARS['HTTP_ACCEPT'];

// Some headers, e.g. Content-Type:
$contentType = $HTTP_SERVER_VARS['CONTENT_TYPE'];
```

In other words, an inconsistent mess, and totally unintuitive. But we didn't care, because we rarely needed to access them.

#### Slightly easier

If you used Apache:

```php
$headers = apache_request_headers();

// or its alias

$headers = getallheaders();

$contentType = $headers['Content-Type'];
```

#### But...

- Apache-only (until 4.3.3, when NSAPI support was added)
- Some headers are not returned (`If-*`)

Which means that while it's present, it's not a good cross-platform solution.

#### Input: register_globals

```php
// Assume /foo?duck=goose
echo $duck; // "goose"

// Also works for POST: assume cat=dog
echo $cat; // "dog"

// And cookies: assume bird=bat
echo $bird; // "bat"
```

#### But what if...?

```php
// What if /foo?duck=goose
// AND POST duck=bluebird
// AND cookie duck=eagle ?
echo $duck; // ?
```

#### Introducing variables_order

```dosini
variables_order = "EGPCS" ; Get -> Post -> Cookie
```

#### So...

```php
// What if /foo?duck=goose
// AND POST duck=bluebird
// AND cookie duck=eagle ?
echo $duck; // "eagle"
```

#### But I wanted POST!

And thus we have `$HTTP_POST_VARS`:

```php
$duck = $HTTP_POST_VARS['duck'];
echo $duck; // "bluebird"
```

#### "Explicit" Access

- `$HTTP_GET_VARS`: Query string arguments
- `$HTTP_POST_VARS`: POST form-encoded data
- `$HTTP_COOKIE_VARS`: Cookies

#### What about alternate messages?

`$HTTP_RAW_POST_DATA` to the rescue!

```php
$data = parse($HTTP_RAW_POST_DATA); // parse somehow...
$profit($data);
```

#### But...

- Until 4.1, you had to enable `track_vars` for these to be visible!
- Only available in the global scope - not within functions/methods!
- `$HTTP_RAW_POST_DATA` will not be present for form-encoded data, unless `always_populate_raw_post_data` is enabled.

### Response Considerations

#### Status code/reason phrase

```php
header('HTTP/1.0 404 Not Found'); // Send a status line
header('Location: /foo'); // Implicit 302
header('Location: /foo', true, 301); // Status code as argument
```

Easy-peasy. In the second and third cases, PHP supplies the default IANA reason phrase.
That said... why is this via header()? Why not a dedicated function?!?!?!

#### Headers

```php
header('X-Foo: Bar');
header('X-Foo: Baz', false); // Emit an additional header value!
header('X-Foo: Bat');        // Replace any previous values for the header!
```

#### But...

`echo()`, `print()`, or emit anything to the output buffer, and no more headers can be sent!

#### Message bodies

```php
echo "Foo!";
```

#### HTML is so easy!

```php
<?php
$url  = 'http://http://www.phpconference.com.br/';
$text = 'PHP Conference Brasil';
?>
<a href="<?php echo $url ?>"><?php echo $text ?></a>
```

#### Mixing headers and body

- Aggregate body in variables
- Or get comfortable with PHP's `ob*()` API...

### Ouch!

## The Present: v4.1 - present (5.6.3)

### Request Considerations

#### Goodbye register_globals!

Well, it's still there, but you can disable it, and later it's disabled by default.

#### Superglobals

- Available in _any_ scope
- Prefixed with an underscore
- `$_SERVER`: Server configuration, HTTP metadata, etc.
- `$_GET`: Query string arguments
- `$_POST`: Deserialized form-encoded POST data
- `$_COOKIE`: Cookie values

#### BUT...

- `$_SERVER` is still in the same format.
- Testing + superglobals == recipe for disaster.
- Superglobals are _mutable_.

Essentially, while this was a nice move, it meant coupling between superglobals and userland code. In a way, it w as a step back, because now instead of passing in the POST and GET values into our applications, we fetch them -- which would be fine if they were immutable, but they are _not_.

#### $_REQUEST

- Merges GET, POST, and Cookie variables, according to `variables_order` or `request_order_string`.
- Just. Don't.

#### Message bodies

`php://input`

- Always available
- Read-only
- Uses streams: more performant, less memory intensive.

#### But...

- Read-once until 5.6

Why does that matter? Well, if you have multiple handlers that try and read it, that means you have to cache it between reads -- which adds to memory usage, and can be brittle if you forget.

### Response Considerations

#### Nothing changes

We're stuck with lack of a way to set the status line other than with `header()`, and fiddling with output buffering. Fortunately, the frameworks that start to sprout up in the later versions of PHP4 and in PHP5 help us abstract that away... mostly in completely different ways.

## Argh!

## How do other languages deal with this?

### Python: WSGI

environ:
- Encapsulates the request method, URI information, and headers.
- Body is an input stream.
start_response:
- Encapsulates status and headers.
- Body is a stream.


Notes:
- WSGI == Web Server Gateway Interface
- Most of environ resembles $_SERVER in PHP

### Ruby: Rack

Environment:
- Encapsulates the request method, URI information, and headers.
- Body is a stream.
Response:
- Encapsulates status and headers.
- Body is iterable.

### Node

http.IncomingMessage:
- Encapsulates request method, url, and headers.
- Body is a stream.
http.ServerResponse:
- Encapsulates status and headers.
- Body is a stream.

## If Ruby and Node can solve this problem, why can't PHP?

## The Future

### Framework Interop Group

People dedicated to making code work across projects, including:

- Autoloading
- Shared, independent coding standards
- Logging
- … and now, HTTP?

### PSR-7

HTTP Message Interfaces

### Client-side

- OutgoingRequestInterface
- IncomingResponseInterface

Not too worried or concerned about this. I want to talk about server-side development today.

### Server-side

- IncomingRequestInterface
- OutgoingResponseInterface

#### Request Metadata

```php
// Protocol
$request->getProtocolVersion();

// Method
$request->getMethod();

// Url
$request->getUrl();
```

#### Request Headers

```php
foreach ($request->getHeaders() as $header => $values) {
    printf("%s: %s\n", $header, implode(', ', $values);
}
```

#### Message Body

```php
$body = $request->getBody();

// Treat it as a string
echo $body;

// Or as a stream-like implementation
while (! $body->eof()) {
  echo $body->read(4096);
}
```

#### Request data

```php
$server = $request->getServerParams();
$cookie = $request->getCookieParams();
$query  = $request->getQueryParams();
$data   = $request->getBodyParams();
$files  = $request->getFileParams();
```

#### Response Metadata

```php
// Protocol
$response->setProtocolVersion('1.0');

// Status
$response->setStatus(404, 'Not Found');

header(sprintf(
    'HTTP/%s %d %s',
    $response->getProtocolVersion(),
    $response->getStatusCode(),
    $response->getReasonPhrase()
));
```

#### Response Headers

```php
$response->setHeader('X-Foo', 'Bar');
$response->addHeader('X-Foo', 'Baz'); // add another value!
$response->setHeader('X-Foo', 'Bat'); // overwrite!

foreach ($request->getHeaders() as $header => $values) {
    foreach ($values as $value) {
      header(sprintf(
          '%s: %s',
          $header,
          $value
      ), false);
    }
}
```

#### Response Body

```php
$body = $response->getBody();
$body->write('Foo! Bar!');
$body->write(' More!');

echo $body;
```

#### PSR-7 in a nutshell

Uniform access to HTTP messages

### An end to framework silos

Notes:
- We've all seen it: Symfony-specific bundles, ZF2-specific modules,
  Laravel-specific packages. These mean that people are solving the same
  problems over and over again, but wrapping them up as framework-specific
  solutions. We need to stop that.

#### Target PSR-7

More specifically, start thinking in terms of *middleware*:

```php
use Psr\Http\Message\IncomingRequestInterface as Request;
use Psr\Http\Message\OutgoingResponseInterface as Response;

$handler = function (Request $request, Response $response) {
    // Do something with the request
    // Update the response
};
```

Notes:
- If we all start writing code like this, we can share it broadly.  Developers
  can write and/or ship their framework-specific code that consumes it - by
  just passing on the request/response pairs they already have.

#### Make frameworks consume middleware

```php
use Phly\Contact\Middleware as Contact;
use Psr\Http\Message\IncomingRequestInterface as Request;
use Psr\Http\Message\OutgoingResponseInterface as Response;
use Zend\Stdlib\DispatchableInterface;

class ContactController extends DispatchableInterface
{
    private $contact;

    public function __construct(Contact $contact)
    {
        $this->contact = $contact;
    }

    public function dispatch(Request $request, Response $response)
    {
        return call_user_func($this->contact, $request, $response);
    }
}
```

Notes:
- Hypothetical at this point, but demonstrates the idea: the framework now consumes generic middleware. Theoretically, the framework could simply route to generic middleware!

### And looking farther

Developers are already working on a PHP extension.

There's a possibility we can get this in PHP 7!

## Resources

- https://github.com/php-fig/fig-standards/blob/master/proposed/http-message.md
- https://github.com/phly/http for an implementation
- https://github.com/phly/conduit for an example of something built on them

## The future is bright

## Thank you

Matthew Weier O'Phinney
https://mwop.net/ - https://apigility.org - http://framework.zend.com
@mwop
