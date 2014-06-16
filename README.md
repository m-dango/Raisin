# NAME

Raisin - REST-like API web micro-framework for Perl.

# SYNOPSIS

    use Raisin::API;
    use Types::Standard qw(Int Str);

    my %USERS = (
        1 => {
            name => 'Darth Wader',
            password => 'deathstar',
            email => 'darth@deathstar.com',
        },
        2 => {
            name => 'Luke Skywalker',
            password => 'qwerty',
            email => 'l.skywalker@jedi.com',
        },
    );

    namespace user => sub {
        params [
            #required/optional => [name, type, default, regex]
            optional => ['start', Int, 0],
            optional => ['count', Int, 10],
        ],
        get => sub {
            my $params = shift;

            my @users
                = map { { id => $_, %{ $USERS{$_} } } }
                  sort { $a <=> $b } keys %USERS;

            my $max_count = scalar(@users) - 1;
            my $start = $params->{start} > $max_count ? $max_count : $params->{start};
            my $count = $params->{count} > $max_count ? $max_count : $params->{count};

            my @slice = @users[$start .. $count];
            { data => \@slice }
        };

        get 'all' => sub {
            my @users
                = map { { id => $_, %{ $USERS{$_} } } }
                  sort { $a <=> $b } keys %USERS;
            { data => \@users }
        };

        params [
            required => ['name', Str],
            required => ['password', Str],
            optional => ['email', Str, undef, qr/.+\@.+/],
        ],
        post => sub {
            my $params = shift;

            my $id = max(keys %USERS) + 1;
            $USERS{$id} = $params;

            { success => 1 }
        };

        route_param id => Int,
        sub {
            get sub {
                my $params = shift;
                $USERS{ $params->{id} };
            };
        };
    };

    run;

# DESCRIPTION

Raisin is a REST-like API web micro-framework for Perl.
It's designed to run on Plack, providing a simple DSL to easily develop RESTful APIs.
It was inspired by [Grape](https://github.com/intridea/grape).

# KEYWORDS

## namespace

Adds a route to application.

    namespace user => sub { ... };

## route\_param

Define a route parameter as a namespace `route_param`.

    route_param id => Int, sub { ... };

## params, del, get, patch, post, put

It is are shortcuts to `route` restricted to the corresponding HTTP method.

Each method could consists of max three parameters:

- params - optional only if didn't starts from params keyword, required otherwise;
- path - optional;
- subroutine - required;

Where only `subroutine` is required.

    get sub { 'GET' };

    del 'all' => sub { 'OK' };

    params [
        required => ['id', Int],
        optional => ['key', Str],
    ],
    get => sub { 'GET' };

    params [
        required => ['id', Int],
        optional => ['name', Str],
    ],
    put => 'all' => sub {
        'PUT'
    };

## req

An alias for `$self->req`, this provides quick access to the
[Raisin::Request](https://metacpan.org/pod/Raisin::Request) object for the current route.

Use `req` to get access to the request headers, params, etc.

    use DDP;
    p req->headers;
    p req->params;

    say req->header('X-Header');

See also [Plack::Request](https://metacpan.org/pod/Plack::Request).

## res

An alias for `$self->res`, this provides quick access to the
[Raisin::Response](https://metacpan.org/pod/Raisin::Response) object for the current route.

Use `res` to set up response parameters.

    res->status(403);
    res->headers(['X-Application' => 'Raisin Application']);

See also [Plack::Response](https://metacpan.org/pod/Plack::Response).

## param

An alias for `$self->params` that gets the GET and POST parameters.
When used with no arguments, it will return an array with the names of all http
parameters. Otherwise, it will return the value of the requested http parameter.

Returns [Hash::MultiValue](https://metacpan.org/pod/Hash::MultiValue) object.

    say param('key'); # -> value
    say param(); # -> { key => 'value' }

## session

An alias for `$self->session` that returns (optional) psgix.session hash.
When it exists, you can retrieve and store per-session data from and to this hash.

    # store param
    session->{hello} = 'World!';

    # read param
    say session->{name};

## api\_version

Set an API version header.

    api_version 1.23;

## api\_format

Loads a plugin from `Raisin::Plugin::Format` namespace.

Already exists [Raisin::Plugin::Format::JSON](https://metacpan.org/pod/Raisin::Plugin::Format::JSON) and [Raisin::Plugin::Format::YAML](https://metacpan.org/pod/Raisin::Plugin::Format::YAML).

    api_format 'json';

## plugin

Loads a Raisin module. The module options may be specified after the module name.
Compatible with [Kelp](https://metacpan.org/pod/Kelp) modules.

    plugin 'Logger', params => [outputs => [['Screen', min_level => 'debug']]];

## middleware

Loads middleware to your application.

    middleware '+Plack::Middleware::Session' => { store => 'File' };
    middleware '+Plack::Middleware::ContentLength';
    middleware 'Runtime'; # will be loaded Plack::Middleware::Runtime

## mount

Mount multiple API implementations inside another one.  These don't have to be
different versions, but may be components of the same API.

In `RaisinApp.pm`:

    package RaisinApp;

    use Raisin::API;

    api_format 'json';

    mount 'RaisinApp::User';
    mount 'RaisinApp::Host';

    1;

## run, new

Creates and returns a PSGI ready subroutine, and makes the app ready for `Plack`.

# PARAMETERS

Request parameters are available through the params hash object. This includes
GET, POST and PUT parameters, along with any named parameters you specify in
your route strings.

Parameters are automatically populated from the request body on POST and PUT
for form input, `JSON` and `YAML` content-types.

In the case of conflict between either of:

- route string parameters
- GET, POST and PUT parameters
- the contents of the request body on POST and PUT

route string parameters will have precedence.

Query string and body parameters will be merged (see ["parameters" in Plack::Request](https://metacpan.org/pod/Plack::Request#parameters))

## Validation and coercion

You can define validations and coercion options for your parameters using a params block.

Parameters can be `required` and `optional`. `optional` parameters can have a
default value.

    get params => [
        required => ['name', Str],
        optional => ['number', Int, 10],
    ],
    sub {
        my $params = shift;
        "$params->{number}: $params->{name}";
    };

Positional arguments:

- name
- type
- default value
- regex

Optional parameters can have a default value.

## Types

Raisin supports Moo(se)-compatible type constraint
so you can use any of the [Moose](https://metacpan.org/pod/Moose), [Moo](https://metacpan.org/pod/Moo) or [Type::Tiny](https://metacpan.org/pod/Type::Tiny) type constraints.

By default [Raisin](https://metacpan.org/pod/Raisin) depends on [Type::Tiny](https://metacpan.org/pod/Type::Tiny) and it's [Types::Standard](https://metacpan.org/pod/Types::Standard)
type contraint library.

You can create your own types as well.
See [Type::Tiny::Manual](https://metacpan.org/pod/Type::Tiny::Manual) and [Moose::Manual::Types](https://metacpan.org/pod/Moose::Manual::Types).

# HOOKS

This blocks can be executed before or after every API call, using
`before`, `after`, `before_validation` and `after_validation`.

Before and after callbacks execute in the following order:

- before
- before\_validation
- after\_validation
- after

The block applies to every API call

    before sub {
        my $self = shift;
        say $self->req->method . "\t" . $self->req->path;
    };

    after_validation sub {
        my $self = shift;
        say $self->res->body;
    };

Steps 3 and 4 only happen if validation succeeds.

# API FORMATS

By default, Raisin supports `YAML`, `JSON`, and `TEXT` content-types.
The default format is `TEXT`.

Response format can be determined by Accept header or route extension.

Serialization takes place automatically. For example, you do not have to call
`encode_json` in each `JSON` API implementation.

Your API can declare which types to support by using `api_format`.

    api_format 'json';

Custom formatters for existing and additional types can be defined with a
[Raisin::Plugin::Format](https://metacpan.org/pod/Raisin::Plugin::Format).

- JSON

    Call `JSON::encode_json` and `JSON::decode_json`.

- YAML

    Call `YAML::Dump` and `YAML::Load`.

- TEXT

    Call `Data::Dumper->Dump` if output data is not a string.

The order for choosing the format is the following.

- Use the route extension.
- Use the value of the `Accept` header.
- Use the `api_format` if specified.
- Fallback to `TEXT`.

# LOGGING

Raisin has a built-in logger and support for `Log::Dispatch`.
You can enable it by:

    plugin 'Logger', outputs => [['Screen', min_level => 'debug']];

Or use [Raisin::Logger](https://metacpan.org/pod/Raisin::Logger) with a `fallback` option:

    plugin 'Logger', fallback => 1;

Exports `log` subroutine.

    log(debug => 'Debug!');
    log(warn => 'Warn!');
    log(error => 'Error!');

See [Raisin::Plugin::Logger](https://metacpan.org/pod/Raisin::Plugin::Logger).

# API DOCUMENTATION

## Raisin script

You can see application routes with the following command:

    $ raisin --routes examples/simple/routes.pl
      GET     /user
      GET     /user/all
      POST    /user
      GET     /user/{id}
      PUT     /user/{id}
      GET     /user/{id}/bump
      PUT     /user/{id}/bump
      GET     /failed

Verbose output with route parameters:

    $ raisin --routes --params examples/simple/routes.pl
      GET     /user
        optional: `start', type: Integer, default: 0
        optional: `count', type: Integer, default: 10

      GET     /user/all

      POST    /user
        required: `name', type: String
        required: `password', type: String
        optional: `email', type: String

      GET     /user/{id}
        required: `id', type: Integer

      PUT     /user/{id}
        optional: `password', type: String
        optional: `email', type: String
        required: `id', type: Integer

      GET     /user/{id}/bump
        required: `id', type: Integer

      PUT     /user/{id}/bump
        required: `id', type: Integer

      GET     /failed

      GET     /params

## Swagger

[Swagger](https://github.com/wordnik/swagger-core) compatible API documentations.

    plugin 'APIDocs';

Documentation will be available on `http://<url>/api-docs` URL.
So you can use this URL in Swagger UI.

For more see [Raisin::Plugin::APIDocs](https://metacpan.org/pod/Raisin::Plugin::APIDocs).

# MIDDLEWARE

You can easily add any [Plack](https://metacpan.org/pod/Plack) middleware to your application using
`middleware` keyword. See ["middleware" in Raisin](https://metacpan.org/pod/Raisin#middleware).

# PLUGINS

Raisin can be extended using custom _modules_. Each new module must be a subclass
of the `Raisin::Plugin` namespace. Modules' job is to initialize and register new
methods into the web application class.

For more see ["plugin" in Raisin](https://metacpan.org/pod/Raisin#plugin) and [Raisin::Plugin](https://metacpan.org/pod/Raisin::Plugin).

# TESTING

See [Plack::Test](https://metacpan.org/pod/Plack::Test), [Test::More](https://metacpan.org/pod/Test::More) and etc.

    my $app = Plack::Util::load_psgi("$Bin/../script/raisinapp.pl");

    test_psgi $app, sub {
        my $cb  = shift;
        my $res = $cb->(GET '/user');

        subtest 'GET /user' => sub {
            if (!is $res->code, 200) {
                diag $res->content;
                BAIL_OUT 'FAILED!';
            }
            my $got = Load($res->content);
            isdeeply $got, $expected, 'Data!';
        };
    };

# DEPLOYING

Deploying a Raisin application is done the same way any other Plack
application is deployed:

    > plackup -E deployment -s Starman app.psgi

## Kelp

    use Plack::Builder;
    use RaisinApp;
    use KelpApp;

    builder {
        mount '/' => KelpApp->new->run;
        mount '/api/rest' => RaisinApp->new;
    };

## Dancer

    use Plack::Builder;
    use Dancer ':syntax';
    use Dancer::Handler;
    use RaisinApp;

    my $dancer = sub {
        setting appdir => '/home/dotcloud/current';
        load_app "My::App";
        Dancer::App->set_running_app("My::App");
        my $env = shift;
        Dancer::Handler->init_request_headers($env);
        my $req = Dancer::Request->new(env => $env);
        Dancer->dance($req);
    };

    builder {
        mount "/" => $dancer;
        mount '/api/rest' => RaisinApp->new;
    };

## Mojolicious::Lite

    use Plack::Builder;
    use RaisinApp;

    builder {
        mount '/' => builder {
            enable 'Deflater';
            require 'my_mojolicious-lite_app.pl';
        };

        mount '/api/rest' => RaisinApp->new;
    };

Also see [Plack::Builder](https://metacpan.org/pod/Plack::Builder), [Plack::App::URLMap](https://metacpan.org/pod/Plack::App::URLMap).

# EXAMPLES

See examples.

# GITHUB

[https://github.com/khrt/Raisin](https://github.com/khrt/Raisin)

# AUTHOR

Artur Khabibullin - rtkh <at> cpan.org

# ACKNOWLEDGEMENTS

This module was inspired both by Grape and [Kelp](https://metacpan.org/pod/Kelp),
which was inspired by [Dancer](https://metacpan.org/pod/Dancer), which in its turn was inspired by Sinatra.

# LICENSE

This module and all the modules in this package are governed by the same license
as Perl itself.
