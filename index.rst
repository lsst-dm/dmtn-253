##############################################
Science Platform authentication for IDAC users
##############################################

.. abstract::

   International Data Access Centers (IDACs) are required to enforce the Rubin data rights policy for any Rubin data that is served to their users, and for any Rubin Data Facilities services they might invoke on behalf of their users.
   IDACs are not required to deploy instances of any Rubin software, and can choose how to implement enforcement of the Rubin data rights policies. 
   However, given the large number of people entitled to Rubin data and the difficulty of maintaining an accurate roll across identity domains, IDACs may find it helpful to delegate authentication and authorization of their Rubin data users to the Rubin Science Platform, which (by requirement and implementation) is capable of applying these controls for its public science deployment.

   This tech note describes a strategy for satisfying this use case by extending the RSP authentication and authorization service to allow IDACs to use OpenID Connect to "phone home" to the Rubin USDAC for data rights verification.
   `OpenID Connect`_ is a widely-supported industry standard for authentication and identity.

.. _OpenID Connect: https://openid.net/connect/

.. note::

   This is part of a tech note series on identity management for the Rubin Science Platform.
   The primary documents are :dmtn:`234`, which describes the high-level design; :dmtn:`224`, which describes the implementation; and :sqr:`069`, which provides a history and analysis of the decisions underlying the design and implementation.
   See the `references section of DMTN-224 <https://dmtn-224.lsst.io/#references>`__ for a complete list of related documents.

.. _problem:

Problem statement
=================

One of the responsibilities of the United States Data Access Center (USDAC) is to provide an identity management system that tracks all registered Rubin data rights holders, vets new requests for access, and uses that system to authorize access to the instance of the Rubin Science Platform running at he USDAC.
The design for this identity management system is described in :dmtn:`234`.

IDACs may provide additional resources to Rubin data rights holders.
They may or may not be running variations of the Rubin Science Platform, and could be using an entirely separate software platform.
IDACs should not be required to maintain a separate database of Rubin data rights holders or validate whether a given user has data rights.
This work will be done by the USDAC, and it serves no purpose to duplicate that effort.

As part of providing additional resources or running local services, IDACs may need to request some data from the USDAC, United States Data Facility (USDF), or another Rubin Data Facility.
To maintain the restrictions on data rights, and to protect against excessive use, those requests should be authenticated.
To allow rate limiting and usage tracking to be as accurate as possible, that authentication should identify both the user (the person holding Rubin data rights on behalf of whom the request was made) and the IDAC making the request on behalf of the user.

Given the authentication and authorization model described in :dmtn:`234`, this means the IDAC needs some mechanism to obtain authentication and possibly access tokens for the user.
The authentication token will provide the IDAC with information about the user's identity and data rights.
The access tokens, if provided, will allow the IDAC to make requests on behalf of the user to a specific Data Access Center or Data Facility.

User authentication
===================

Background
----------

The USDAC must verify data rights for any attempted access.
It does this by maintaining an identity management system with a record of all users with verified data rights, including a mechanism for adding new users and for removing users when they no longer have data rights.
The component that enforces this verification and interacts with the identity management system is named Gafaelfawr_.

.. _Gafaelfawr: https://gafaelfawr.lsst.io/

Gafaelfawr can also act as an `OpenID Connect`_ provider.
This is a widely implemented standard for delegating authentication to a third party, and will be used by IDACs that wish to use the USDAC to authenticate users.

OpenID Connect uses the following terms in very specific ways:

claim
    An individual attribute in a JWT (see :rfc:`7519`), such as an ID token.

client
    The application that wants to use an OpenID Connect server to authenticate a user.
    In the context of this tech note, this is an IDAC.
    The client is *not* the user being authenticated; it is another service that is delegating authentication to the OpenID Connect server.
    The client is more formally called the Relying Party (RP).

scope
    A requested scope of access to the user's data.
    This is unfortunately the same term that is used for the permissions granted to a Gafaelfawr token used internally by the Rubin Science Platform (see :dmtn:`235`).
    OpenID scopes are unrelated to Gafaelfawr token scopes and use a different set of valid values.

server
    The OpenID Connect server that authenticates the user.
    In the case of Gafaelfawr, this is normally done by delegating the authentication to yet another OpenID Connect server, although IDACs do not need to be aware of this.
    The server is more formally called the OpenID Provider (OP).

user
    The person or application being authenticated.
    In the case of IDAC authentication, this is always a human user, generally using a web browser.

Registering an IDAC
-------------------

Broadly speaking, there are two ways registration with an OpenID Connect server can work: dynamic registration of any client that wants to use it, or pre-registration that requires a client to be registered before it can authenticate users.
Dynamic registration requires asking the authenticating user for their permission to release information about them to the client.

Since we only want to allow known clients to use Rubin for authentication, will only release data about users to known services, and want the authentication flow to be uniform for any Rubin-affiliated Data Access Center, we will require pre-registration of all clients.

Concretely, this means that we only support the `Authorization Code Flow`_ and only `confidential clients <https://datatracker.ietf.org/doc/html/rfc6749#section-2.1>`__ are supported.

.. _Authorization Code Flow: https://openid.net/specs/openid-connect-core-1_0.html#CodeFlowAuth

Any IDAC that wants to delegate authentication and get data rights information from the USDAC must agree in advance with the USDAC on the following information:

``client_id``
    A unique identifier for this IDAC.

``client_secret``
    A randomly-generated key that will be used by the IDAC to authenticate to the USDAC during the Token Request step of an OpenID Connect 1.0 authentication.

``redirect_uri``
    The URL to which the user will be redirected following a successful authentication for this IDAC.
    This must exactly match (apart from query parameters) the URL provided by the IDAC in the ``redirect_uri`` request parameter when it initiates authentication and the ``redirect_uri`` POST parameter in the Token Request.

Authenticating a user
---------------------

To authenticate a user, an IDAC will follow the normal OpenID Connect 1.0 `Authorization Code Flow`_.
The standard URL ``/.well-known/openid-configuration`` at the USDAC can be used to get the relevant URLs and other OpenID Connect configuration.
See the `OpenID Connect Discovery 1.0`_ standard for details about what that URL will return.

.. _OpenID Connect Discovery 1.0: https://openid.net/specs/openid-connect-discovery-1_0.html

The USDAC OpenID Connect server will support the following scopes:

``openid``
    Required, per the OpenID Connect specification.
    The ``sub`` claim will always be set to the user's username at the USDAC.
    IDACs do not have to use the same username for the user, but doing so may be convenient and less confusing.

``profile``
    Adds the ``preferred_username`` claim, with the same value as ``sub``, and, if this information is available, the ``name`` claim.
    By design, we do not support attempting to break the name into components such as given name or family name.
    This transformation is only meaningful in certain cultures.

``email``
    Adds the ``email`` claim if the user's email address is known.

``rubin``
    Adds the ``data_rights`` claim with a space-separated list of data releases the user has access to, if there are any.

The expiration time of the ID token will be inherited from the expiration time of the underlying user credentials used for authentication.
It will therefore be capped at the maximum session token lifetime for a user of the USDAC and may be much shorter if the user has not authenticated recently.

.. note::

   The OpenID Connect ``max_age`` parameter is not currently supported, but support could be added if there is a need for IDACs to ensure that the user authentication is not too stale (and thus the expiration of the ID token is not too close).

Verifying data rights
---------------------

To verify a user has data rights, the IDAC **must** request the ``rubin`` scope during authentication and inspect the ``data_rights`` claim of the resulting ID token to see if it contains the relevant data release.
Alternately, the IDAC can present the access token obtained via the Token Request to the Userinfo Endpoint, which will return the same information in the ``data_rights`` key.

.. warning::

   The mere ability to authenticate does not guarantee that a user has data rights.
   Users with no data rights or with access only to old data releases will still be able to successfully authenticate.
   An IDAC **must** verify user data rights before granting access to a restricted data set.

Delegated tokens
================

In the initial implementation, delegated access by IDACs to USDAC resources will not be allowed.
The provided access token will only have access to the OpenID Connect Userinfo endpoint.

This section describes how delegated access tokens could be implemented in the future, should there be a need.

Delegated token format
----------------------

The OpenID Connect flow provides up to three tokens to the client: the ID token, which must be a JWT (:rfc:`7519`) and which is discussed above; the access token, which must at least have access to the Userinfo endpoint; and an optional refresh token.
This proposed implementation does not support refresh tokens.

In the OpenID Connect standard, the access token need not be a JWT and may be an opaque token.
In this implementation, it will indeed be an opaque token rather than a JWT for all of the `reasons discussed in SQR-069 <https://sqr-069.lsst.io/#token-format>`__.

OpenID Connect access tokens will have a new internal Gafaelfawr token type, ``openid``, which is similar to the existing ``internal`` token type but indicates that the token was returned as an OpenID Connect access token.
These tokens will have a new database field, ``client``, which holds the client ID of the OpenID Connect client to which they were issued.
This field will be returned by the Gafaelfawr token-info endpoint.
To support rate limiting, this field will likely also need to be added to the data stored in Redis.

Delegated tokens of this type will by default have no Gafaelfawr scopes, which means that they can only be used to access user and token information endpoints.
If delegated tokens should have access to some services, there are two possible ways we could implement that:

#. Create additional non-standard OpenID Connect scopes that request access token scopes.

#. Configure OpenID Connect clients, during registration, with a set of delegated access scopes, and add those token scopes to every access token created for that OpenID Connect client.

We will defer deciding which approach to implement until we have a use case for delegated access.

Obtaining and using a delegated token
-------------------------------------

The delegated token will be returned as the access token from the Token Response as part of the OpenID Connect authentication flow.

This token is a bearer authentication token (see :rfc:`6750`).
It must be used by putting it in the ``Authorization`` header of a request with a credential type of ``bearer``.

Gafaelfawr's rate limiting support (see :sqr:`073`) should be enhanced to allow setting rate limits on an entire IDAC as well as on individual users, allowing rejection of requests from an IDAC on behalf of a user without affecting that user's other accesses.
This will require adding the ``client`` field to the data stored in Redis so that it can be used for rate limiting decisions.

Design discussion
=================

.. _metadata:

User metadata
-------------

We considered the following approaches for communicating data rights information to IDACs:

- Put the user's scopes (the same ones used internally by the USDAC) into the issued identity token.
  The IDAC can then retrieve the scopes from the identity token and look for a scope that indicates that the user has data rights.
  The drawback of this approach is that scopes only convey whether the user has access to any data release (including historical ones), not which data releases they have access to or whether they have access to the most recent data release.
  More granular information is only available in group membership.
  This option is therefore eliminated if IDACs have a requirement to determine whether a user has access to the current or a specific data release.

- Put the user's USDAC groups into the issued identity token.
  Group membership will provide granular information about which data releases the user has access to.
  However, there is no standard JWT field for group membership, and this would also expose a lot of other group details that is likely not of interest to IDACs and could change at any time.

- Determine, at the USDAC Gafaelfawr side, whether the user has data rights (and to which Data Releases if applicable) and synthesize a token claim that says this specifically.
  This too would be a non-standard claim specific for this purpose.
  The drawback of this approach is that it is awkward to put this type of configuration at the Gafaelfawr layer, since it normally only cares about group memberships and scopes derived from those group memberships.
  The advantage is that this would clearly communicate precisely the information of interest to the IDAC.

We picked the third option, despite the awkwardness and Rubin-specific scopes, since it provided the clearest communication of the necessary information without including extraneous details that could be used incorrectly.

Access token format
-------------------

We considered the following options for the format of the access token:

- Provide a JWT token that's usable in the same places a normal Gafaelfawr opaque token is used.
  While this is what OpenID Connect flows normally do, it's not required by the standard and many of the reasons why we `chose not to use JWTs <https://sqr-069.lsst.io/#token-format>`__ still apply.

- Provide a service token, with the service set to some identifier for the IDAC.
  If we take this approach, we should reserve some naming convention for IDAC identities, such as any service that begins with ``idac-``.
  This doesn't require any new infrastructure, changes to the data model, or new token types, but it does mix internal delegated tokens used inside the Science Platform with tokens returned by OpenID Connect to entities outside the Science Platform.
  It's arguable whether those concepts are distinct enough to warrant a separate token type.

- Add a new token type with a new piece of associated metadata that identifies the IDAC to which the token was delegated.
  This has the advantage of unambiguously identifying this token as one delegated outside the Science Platform to an IDAC, but it adds additional complexity that may not be necessary.
  It's not obvious what to call these tokens without using Rubin-specific terminology, which may be a sign that this is not a generalizable authentication concept and therefore shouldn't be represented at the protocol level like this.

We picked the third option and decided to call them ``openid`` tokens, which seems like a reasonable generalization.
When implemented, this will require Gafaelfawr database transitions, but the additional clarity seems worth the effort.

Future work
===========

OpenID protocol
---------------

Currently, the OpenID Connect server implemented by Gafaelfawr is very simple and not fully standards-compliant.
It implements only the minimum functionality required for this design to work, and may not work properly with every OpenID Connect client implementation.

Ideally, it should be enhanced over time to become a more full-featured implementation.
Missing features of particular interest include:

- Support the ``max_age`` parameter to the Authentication Request and reauthenticate the user if their authentication is too stale.
  This would provide IDACs with some amount of control over the lifetime of the returned ID and access tokens, and the ability to force reauthentication for privileged operations.

- Support the ``display``, ``prompt``, and ``id_token_hint`` parameters to the Authentication Request, which would allow probing of authentication state without forcing authentication.

- Support the ``email_verified`` claim.
  Gafaelfawr can be configured to know whether the email metadata for users has been verified.

- Support ``POST`` to the login and userinfo endpoints for better standards compliance.

Refresh tokens
--------------

Currently, Gafaelfawr does not use refresh tokens, in part because the tokens are all validated by the same service that issues the tokens, so there is no need to worry about validation by a service that does not realize the token has been invalidated.
We should revisit the decision not to use refresh tokens to ensure nothing about the security model warrants them.
