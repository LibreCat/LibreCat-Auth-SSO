# NAME

Plack::Auth::SSO - role for Single Sign On (SSO) authentication

# STATUS

[![Build Status](https://travis-ci.org/LibreCat/Plack-Auth-SSO.svg?branch=master)](https://travis-ci.org/LibreCat/Plack-Auth-SSO)
[![Coverage](https://coveralls.io/repos/LibreCat/Plack-Auth-SSO/badge.png?branch=master)](https://coveralls.io/r/LibreCat/Plack-Auth-SSO)
[![CPANTS kwalitee](http://cpants.cpanauthors.org/dist/Plack-Auth-SSO.png)](http://cpants.cpanauthors.org/dist/Plack-Auth-SSO)

# IMPLEMENTATIONS

\* SSO for Central Authentication System (CAS): [Plack::Auth::SSO::CAS](https://metacpan.org/pod/Plack::Auth::SSO::CAS)

\* SSO for ORCID: [Plack::Auth::SSO::ORCID](https://metacpan.org/pod/Plack::Auth::SSO::ORCID)

\* SSO for Shibboleth: [Plack::Auth::SSO::Shibboleth](https://metacpan.org/pod/Plack::Auth::SSO::Shibboleth)

# SYNOPSIS

    package MySSOAuth;

    use Moo;
    use Data::Util qw(:check);

    with "Plack::Auth::SSO";

    sub to_app {

        my $self = shift;

        sub {

            my $env = shift;
            my $request = Plack::Request->new($env);
            my $session = Plack::Session->new($env);

            #did this app already authenticate you?
            #implementation of Plack::Auth::SSO should write hash to session key,
            #configured by "session_key"
            my $auth_sso = $self->get_auth_sso($session);

            #already authenticated: what are you doing here?
            if( is_hash_ref($auth_sso) ){

                return [ 302, [ Location => $self->uri_for($self->authorization_path) ], [] ];

            }

            #not authenticated: do your internal work
            #..

            #authentication done in external application code, but here something went wrong..
            unless ( $ok ) {

                #error is set in auth_sso_error..
                $self->set_auth_sso_error(
                    $session,
                    {
                        package => __PACKAGE__,
                        package_id => $self->id,
                        type => "connection_failed",
                        content => ""
                    }
                );

                #user is redirected to error_path
                return [ 302, [ Location => $self->uri_for($self->error_path) ], [] ];

            }

            #everything ok: set auth_sso
            $self->set_auth_sso(
                $session,
                {
                    package => __PACKAGE__,
                    package_id => $self->id,
                    response => {
                        content => "Long response from external SSO application",
                        content_type => "text/xml"
                    },
                    uid => "<uid>",
                    info => {
                        attr1 => "attr1",
                        attr2 => "attr2"
                    },
                    extra => {
                        field1 => "field1"
                    }
                }
            );

            #redirect to other application for authorization:
            return [ 302, [ Location => $self->uri_for($self->authorization_path) ], [] ];

        };
    }

    1;


    #in your app.psgi

    builder {

        mount "/auth/myssoauth" => MySSOAuth->new(

            session_key => "auth_sso",
            authorization_path => "/auth/myssoauth/callback",
            uri_base => "http://localhost:5001",
            error_path => "/auth/error"

        )->to_app;

        mount "/auth/myssoauth/callback" => sub {

            my $env = shift;
            my $session = Plack::Session->new($env);
            my $auth_sso = $session->get("auth_sso");

            #not authenticated yet
            unless($auth_sso){

                return [ 403, ["Content-Type" => "text/html"], ["forbidden"] ];

            }

            #process auth_sso (white list, roles ..)

            [ 200, ["Content-Type" => "text/html"], ["logged in!"] ];

        };

        mount "/auth/error" => sub {

            my $env = shift;
            my $session = Plack::Session->new($env);
            my $auth_sso_error = $session->get("auth_sso_error");

            unless ( $auth_sso_error ) {

                return [ 302, [ Location => $self->uri_for( "/" ) ], [] ];

            }

            [ 200, [ "Content-Type" => "text/plain" ], [
                $auth_sso_error->{content}
            ]];

        };

    };

# DESCRIPTION

This is a Moo::Role for all Single Sign On Authentication packages. It requires
`to_app` method, that returns a valid Plack application

An implementation is expected is to do all communication with the external
SSO application (e.g. CAS). When it succeeds, it should save the response
from the external service in the session, and redirect to the authorization
url (see below).

The authorization route must pick up the response from the session,
and log the user in.

This package requires you to use Plack Sessions.

# CONFIG

- session\_key

    When authentication succeeds, the implementation saves the response
    from the SSO application in this session key, together with extra information.

    The response should look like this:

        {
            package => "<package-name>",
            package_id => "<package-id>",
            response => {
                content => "Long response from external SSO application like CAS",
                content_type => "<mime-type>"
            },
            uid => "<uid-in-external-app>",
            info => {
                attr1 => "attr1",
                attr2 => "attr2"
            },
            extra => {
                field1 => "field1"
            }
        }

    This is usefull for several reasons:

        * the authorization application can distinguish between authenticated and not authenticated users

        * it can pick up the saved response from the session

        * it can lookup a user in an internal database, matching on the provided "uid" from the external service.

        * the key "package" tells which package authenticated the user; so the application can do an appropriate lookup based on this information.

        * the key "package_id" defaults to the package name, but is configurable. This is usefull when you have several external services of the same type,
          and your application wants to distinguish between them.

        * the original response is stored as text, along with the content type.

        * other attributes stored in the hash reference "info". It is up to the implementing package whether it should only used attributes as pushed during
          the authentication step (like in CAS), or do an extra lookup.

        * "extra" should be used to store request information.
            e.g. "ORCID" gives a "token".
            e.g. "Shibboleth" supplies the "Shib-Identity-Provider".

- authorization\_path

    (internal) path of the authorization route. This path will be prepended by "uri\_base" to
    create the full url.

    When authentication succeeds, this application should redirect you here

- error\_path

    (internal) path of the error route. This path will be prepended by "uri\_base" to
    create the full url.

    When authentication fails, this application should redirect you here

    If not set, it has the same value as the authorizaton\_path. In that case make sure that you also

    check for auth\_sso\_error in your authorization route.

    The implementor should expect this in the session key "auth\_sso\_error" ( "\_error" is appended to the configured session\_key ):

        {
            package => "Plack::Auth::SSO::TYPE",
            package_id => "Plack::Auth::SSO::TYPE",
            type => "my-error-type",
            content => "Something went terribly wrong!"
        }

    Error types should be documented by the implementor.

- uri\_for( path )

    method that prepends your path with "uri\_base".

- id

    identifier of the authentication module. Defaults to the package name.
    This is handy when using multiple SSO instances, and you need to known
    exactly which package authenticated the user.

    This is stored in "auth\_sso" as "package\_id".

- uri\_base

    base url of the Plack application

# METHODS

## to\_app

returns a Plack application

This must be implemented by subclasses

## get\_auth\_sso($plack\_session)

get saved SSO response from your session

## set\_auth\_sso($plack\_session,$hash)

save SSO response to your session

$hash should be a hash ref, and look like this:

    {
        package => __PACKAGE__,
        package_id => __PACKAGE__ ,
        response => {
            content => "Long response from external SSO application like CAS",
            content_type => "<mime-type>",
        },
        uid => "<uid>",
        info => {},
        extra => {}
    }

## get\_auth\_sso\_error($plack\_session)

get saved SSO error response from your session

## set\_auth\_sso\_error($plack\_session,$hash)

save SSO error response to your session

$hash should be a hash ref, and look like this:

    {
        package => __PACKAGE__,
        package_id => __PACKAGE__ ,
        type => "my-type",
        content => "my-content"
    }

## generate\_csrf\_token()

Generate unique CSRF token. Store this token in your session, and supply it as parameter
to the redirect uri.

## set\_csrf\_token($session,$token)

Save csrf token to the session

The token is saved in key session\_key + "\_csrf"

## get\_csrf\_token($session)

Retrieve csrf token from the session

## csrf\_token\_valid($session,$token)

Compare supplied token with stored token

## cleanup($session)

removes additional session keys like auth\_sso\_error and auth\_sso\_csrf
before redirecting to the authorization path.

# EXAMPLES

See examples/app1:

    #copy example config to required location
    $ cp examples/catmandu.yml.example examples/catmandu.yml

    #edit config
    $ vim examples/catmandu.yml

    #start plack application
    plackup examples/app1.pl

# AUTHOR

Nicolas Franck, `<nicolas.franck at ugent.be>`

# LICENSE AND COPYRIGHT

This program is free software; you can redistribute it and/or modify it
under the terms of either: the GNU General Public License as published
by the Free Software Foundation; or the Artistic License.

See [http://dev.perl.org/licenses/](http://dev.perl.org/licenses/) for more information.

# SEE ALSO

[Plack::Auth::SSO::CAS](https://metacpan.org/pod/Plack::Auth::SSO::CAS),
[Plack::Auth::SSO::ORCID](https://metacpan.org/pod/Plack::Auth::SSO::ORCID)
[Plack::Auth::SSO::Shibboleth](https://metacpan.org/pod/Plack::Auth::SSO::Shibboleth)
