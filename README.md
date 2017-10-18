# NAME

LibreCat::Auth::SSO - LibreCat role for Single Sign On (SSO) authentication

# SYNOPSIS

    package MySSOAuth;

    use Moo;
    use Catmandu::Util qw(:is);

    with "LibreCat::Auth::SSO";

    sub to_app {

        my $self = shift;

        sub {

            my $env = shift;
            my $request = Plack::Request->new($env);
            my $session = Plack::Session->new($env);

            #did this app already authenticate you?
            #implementation of LibreCat::Auth::SSO should write hash to session key,
            #configured by "session_key"
            my $auth_sso = $self->get_auth_sso($session);

            #already authenticated: what are you doing here?
            if( is_hash_ref($auth_sso) ){

                return [ 302, [ Location => $self->uri_for($self->authorization_path) ], [] ];

            }

            #not authenticated: do your internal work
            #..

            #everything ok: set auth_sso
            $self->set_auth_sso(
                $session,
                {
                    package => __PACKAGE__,
                    package_id => $self->id,
                    response => "Long response from external SSO application"
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
            uri_base => "http://localhost:5001"

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

# CONFIG

- session\_key

    When authentication succeeds, the implementation saves the response
    from the SSO application in this session key.

    The response should look like this:

        {
            package => "<package-name>",
            package_id => "<package-id>",
            response => "Long response from external SSO application like CAS"
        }

    This is usefull for two reasons:

        * this application can distinguish between authenticated and not authenticated users

        * the authorization application can pick up the saved response from the session

- authorization\_path

    (internal) path of the authorization route. This path will be prepended by "uri\_base" to
    create the full url.

    When authentication succeeds, this application should redirect you here

- uri\_for( path )

    method that prepends your path with "uri\_base".

- id

    identifier of the authentication module. Defaults to the package name.
    This is handy when using multiple SSO instances, and you need to known
    exactly which package authenticated the user.

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
        response => "Long response from external SSO application like CAS"
    }

# AUTHOR

Nicolas Franck, `<nicolas.franck at ugent.be>`

# LICENSE AND COPYRIGHT

This program is free software; you can redistribute it and/or modify it
under the terms of either: the GNU General Public License as published
by the Free Software Foundation; or the Artistic License.

See [http://dev.perl.org/licenses/](http://dev.perl.org/licenses/) for more information.

# SEE ALSO

[LibreCat::Auth::SSO::CAS](https://metacpan.org/pod/LibreCat::Auth::SSO::CAS),
[LibreCat::Auth::SSO::ORCID](https://metacpan.org/pod/LibreCat::Auth::SSO::ORCID)