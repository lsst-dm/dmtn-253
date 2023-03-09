:tocdepth: 1

Abstract
========

International Data Access Centers (IDACs) are required to enforce the Rubin data rights policy both for any Rubin data that is served to their users, and for any Rubin Data Facilities services they might invoke on behalf of their users.
IDACs are not required to deploy instances of any Rubin software, and can choose how to implement enforcement of the Rubin data rights policies. 
However, given the large number of people entitled to Rubin data and the challenges in maintaining an accurate roll of authorised users across identity domains, IDACs might find it helpful to delegate authentication and authorisation of their Rubin data users to the  Rubin Science Platform which (by requirement and implementation) has the capability of applying these controls for its public science deployment.
This tech note describes a strategy for satisfying this use case by extending the RSP authentication and authorisation service to allow IDACs to use OpenID Connect (a leading industry standard  for identity interoperability with wide implementation support) to "phone home" to the Rubin USDAC for data rights verification. 

.. note::

   This is part of a tech note series on identity management for the Rubin Science Platform.
   The primary documents are :dmtn:`234`, which describes the high-level design; :dmtn:`224`, which describes the implementation; and :sqr:`069`, which provides a history and analysis of the decisions underlying the design and implementation.
   See the `references section of DMTN-224 <https://dmtn-224.lsst.io/#references>`__ for a complete list of related documents.

.. _problem:

Problem statement
=================

One of the responsibilities of the USDAC is to provide an identity management system that tracks all registered Rubin data rights holders, vets new requests for access, and uses that system to authorize access to the instance of the Rubin Science Platform running at he USDAC.
The design for this identity management system is described in :dmtn:`234`.

IDACs may provide additional resources to Rubin data rights holders.
They may or may not be running variations of the Rubin Science Platform, and could be using an entirely separate software platform.
IDACs should not be required to maintain a separate database of Rubin data rights holders or validate whether a given user has data rights.
This work will be done by the USDAC, and it serves no purpose to duplicate that effort.

As part of providing additional resources or running local services, IDACs may need to request some data from the USDAC, United States Data Facility (USDF), or another Rubin Data Facility.
To maintain the restrictions on data rights, and to protect against excessive use, those requests should be authenticated.
To allow rate limiting and usage tracking to be as accurate as possible, that authentication should identify both the user (the person holding Rubin data rights on behalf of whom the request was made) and the IDAC making the request on behalf of the user.

Given the authentication and authorization model described in :dmtn:`234`, this means the IDAC needs some mechanism to obtain an authentication token on behalf of the user, which it can then use for subsequent requests.
This token will then be used to authenticate requests to a specific Data Access Center or Data Facilty.

Proposed solution
=================

Verifying data rights
---------------------

The USDAC must verify data rights for any attempted access.
It does this by maintaining an identity management system with a record of all users with verified data rights, including a mechanism for adding new users and for removing users when they no longer have data rights.
The component that enforces this verification and interacts with the identity management system is named Gafaelfawr_.

.. _Gafaelfawr: https://gafaelfawr.lsst.io/

Gafaelfawr can also act as an `OpenID Connect`_ provider.
This support was originally added to provide authentication to third-party software running inside the Rubin Science Platform that uses OpenID Connect for authentication.
However, OpenID Connect services need not be restricted to applications within the same Science Platform instance.

.. _OpenID Connect: https://openid.net/connect/

In this proposed design, an IDAC that wanted to verify whether a user had Rubin data rights would do so by initiating an OpenID Connect authentication, using the USDAC as the authentication provider.
The user would be sent to the USDAC to authenticate using whatever mechanisms they have registered with its identity provider.
After successful authentication, the user would be returned to the IDAC with authentication credentials.
Using those authentication credentials, the IDAC could verify the user's data rights and obtain additional metadata about the user, such as full name, email address, and group membership.
(See :dmtn:`225` for a list of the metadata the USDAC will maintain.)

.. warning::

   The mere ability to authenticate does not guarantee that a user has data rights.
   The IDAC will need to retrieve metadata for the user and check that user's scopes and group membership to determine their current access rights.
   See :ref:`metadata` for additional details that will need to be designed.

Obtaining a delegated token
---------------------------

The authentication system described in :dmtn:`234` has a mechanism for obtaining delegated tokens on behalf of a user.
The purpose of these tokens is to allow some service accessed by the user to make further requests on behalf of that user to other services.
(For example, the Portal Aspect may need to make TAP queries, and those queries should be done as the user so that appropriate access restrictions can be applied.)

The case of the IDAC making subsequent requests on behalf of the user to the USDAC is similar, except that the requests would originate from outside the Science Platform.

OpenID Connect (via OAuth 2.0, see :rfc:`6749`) has a mechanism to return an access token in addition to the required ID token.
That access token is intended for precisely this purpose: making subsequent requests on behalf of the user.

Unlike the ID token, which is required to be a JWT (see :rfc:`7519`), the access token can be any OAuth 2.0 bearer token.
Gafaelfawr can therefore return one of its normal bearer tokens to use for subsequent requests, and associate the identity of the IDAC (which is provided to Gafaelfawr as part of the OpenID Connect authentication flow) with that token.
Subsequent internal tokens can be generated from that token following the normal token usage pattern described in :dmtn:`234`.

Gafaelfawr's rate limiting support (see :sqr:`073`) should be enhanced to allow setting rate limits on an entire IDAC as well as on individual users, allowing rejection of requests from an IDAC on behalf of a user without affecting that user's other accesses.

See :ref:`idac-tokens` for a few implementation questions about this approach.

Implementation details
======================

.. _metadata:

User metadata
-------------

Currently, the Gafaelfawr OpenID Connect provider is very simple and does not provide all of the metadata an IDAC would need.
Specifically, it does not include either scopes or group membership, and therefore doesn't provide the necessary information to determine whether the user has data rights.

Possible approaches to communicating this information to an IDAC include:

- Put the user's scopes (the same ones used internally by the USDAC) into the issued identity token.
  The IDAC can then retrieve the scopes from the identity token and look for a scope that indicates that the user has data rights.
  The drawback of this approach is that user scopes are more granular than "has data rights" or "does not have data rights" (see :dmtn:`235`), so there would need to be clear documentation for what IDACs should look for.
  Also, the Science Platform scopes will, by design, only indicate whether the user has access to any Data Release (not necessarily the current one).
  More granular information is only available in group membership.

- Put the user's USDAC groups into the issued identity token.
  This is cleaner in that there will be groups specifically for data access rights (and separated by Data Release when that is relevant).
  However, there is no standard JWT field for group membership, and this would also expose a lot of other group details that is likely not of interest to IDACs and could change at any time.

- Determine, at the USDAC Gafaelfawr side, whether the user has data rights (and to which Data Releases if applicable) and synthesize a token claim that says this specifically.
  This too would be a non-standard claim specific for this purpose.
  The drawback of this approach is that it is awkward to put this type of configuration at the Gafaelfawr layer, since it normally only cares about group memberships and scopes derived from those group memberships.
  The advantage is that this would clearly communicate precisely the information of interest to the IDAC.

When implementing this proposal, we will need to choose an approach and document that in the instructions for IDACs.

.. _idac-tokens:

Access tokens for IDACs
-----------------------

We have to decide what form the access token returned to the IDAC in the OpenID Connect token response should take.
There are a few possibilities:

- Provide a JWT token that's usable in the same places a normal Gafaelfawr opaque token is used.
  While this is what OpenID Connect flows normally do, it's not required by the standard and many of the reasons why we `chose not to use JWTs <https://sqr-069.lsst.io/#token-format>`__ still apply.

- Provide a service token, with the service set to some identifier for the IDAC.
  If we take this approach, we should reserve some naming convention for IDAC identities, such as any service that begins with ``idac-``.
  This doesn't require any new infrastructure, changes to the data model, or new token types, but it does mix internal delegated tokens used inside the Science Platform with tokens returned by OpenID Connect to entities outside the Science Platform.
  It's arguable whether those concepts are distinct enough to warrant a separate token type.

- Add a new token type with a new piece of associated metadata that identifies the IDAC to which the token was delegated.
  This has the advantage of unambiguously identifying this token as one delegated outside the Science Platform to an IDAC, but it adds additional complexity that may not be necessary.
  It's not obvious what to call these tokens without using Rubin-specific terminology, which may be a sign that this is not a generalizable authentication concept and therefore shouldn't be represented at the protocol level like this.

Currently, Gafaelfawr does not use refresh tokens, in part because the tokens are all validated by the same service that issues the tokens, so there is no need to worry about validation by a service that does not realize the token has been invalidated.
This will remain true for IDAC access tokens as long as the JWT approach is not chosen.
However, we should still revisit the decision not to use refresh tokens to ensure nothing about the security model warrants them.

It's not immediately obvious how long of a lifetime IDAC access tokens should have.
This should be configurable so that we can change our minds.
