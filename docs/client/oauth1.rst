OAuth 1 Client
==============

.. meta::
    :description: An OAuth 1 protocol implementation for requests.Session and
        aiohttp.ClientSession, powered by Authlib.

.. module:: authlib.client


OAuth1Session for requests
--------------------------

The :class:`OAuth1Session` in Authlib is a subclass of ``requests.Session``.
It shares the same API with ``requests.Session`` and extends it with OAuth 1
protocol. This section is a guide on how to obtain an access token in OAuth 1
flow.

.. note::
    If you are using Flask or Django, you may have interests in
    :ref:`flask_client` and :ref:`django_client`.

If you are not familiar with OAuth 1.0, it is better to
:ref:`understand_oauth1` now.

There are three steps in OAuth 1 to obtain an access token. Initialize
the session for reuse::

    >>> from authlib.client import OAuth1Session
    >>> client_id = 'Your Twitter client key'
    >>> client_secret = 'Your Twitter client secret'
    >>> session = OAuth1Session(client_id, client_secret)

.. _fetch_request_token:

Fetch Temporary Credential
~~~~~~~~~~~~~~~~~~~~~~~~~~

The first step is to fetch temporary credential, which will be used to generate
authorization URL::

    >>> request_token_url = 'https://api.twitter.com/oauth/request_token'
    >>> request_token = session.fetch_request_token(request_token_url)
    >>> print(request_token)
    {'oauth_token': 'gA..H', 'oauth_token_secret': 'lp..X', 'oauth_callback_confirmed': 'true'}

Save this temporary credential for later use (if required).

You can assign a ``redirect_uri`` before fetching the request token, if
you want to redirect back to another URL other than the one you registered::

    >>> session.redirect_uri = 'https://your-domain.org/auth'
    >>> session.fetch_request_token(request_token_url)

Redirect to Authorization Endpoint
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The second step is to generate the authorization URL::

    >>> authenticate_url = 'https://api.twitter.com/oauth/authenticate'
    >>> session.create_authorization_url(authenticate_url, request_token['oauth_token'])
    'https://api.twitter.com/oauth/authenticate?oauth_token=gA..H'

Actually, the second parameter ``request_token`` can be omitted, since session
is re-used::

    >>> session.create_authorization_url(authenticate_url)

Now visit the authorization url that :meth:`OAuth1Session.create_authorization_url`
generated, and grant the authorization.

.. _fetch_oauth1_access_token:

Fetch Access Token
~~~~~~~~~~~~~~~~~~

When the authorization is granted, you will be redirected back to your
registered callback URI. For instance::

    https://example.com/twitter?oauth_token=gA..H&oauth_verifier=fcg..1Dq

If you assigned ``redirect_uri`` in :ref:`fetch_oauth1_access_token`, the
authorize response would be something like::

    https://your-domain.org/auth?oauth_token=gA..H&oauth_verifier=fcg..1Dq

Now fetch the access token with this response::

    >>> resp_url = 'https://example.com/twitter?oauth_token=gA..H&oauth_verifier=fcg..1Dq'
    >>> session.parse_authorization_response(resp_url)
    >>> access_token_url = 'https://api.twitter.com/oauth/access_token'
    >>> token = session.fetch_access_token(access_token_url)
    >>> print(token)
    {
        'oauth_token': '12345-st..E',
        'oauth_token_secret': 'o67..X',
        'user_id': '12345',
        'screen_name': 'lepture',
        'x_auth_expires': '0'
    }
    >>> save_access_token(token)

Save this token to access protected resources.

The above flow is not always what we will use in a real project. When we are
redirected to authorization endpoint, our session is over. In this case, when
the authorization server send us back to our server, we need to create another
session::

    >>> # restore your saved request token, which is a dict
    >>> request_token = restore_request_token()
    >>> oauth_token = request_token['oauth_token']
    >>> oauth_token_secret = request_token['oauth_token_secret']
    >>> session = OAuth1Session(
    ...     client_id, client_secret,
    ...     token=oauth_token,
    ...     token_secret=oauth_token_secret)
    >>> # there is no need for `parse_authorization_response` if you can get `verifier`
    >>> verifier = request.args.get('verifier')
    >>> access_token_url = 'https://api.twitter.com/oauth/access_token'
    >>> token = session.fetch_access_token(access_token_url, verifier)

Access Protected Resources
~~~~~~~~~~~~~~~~~~~~~~~~~~

Now you can access the protected resources. If you re-use the session, you
don't need to do anything::

    >>> account_url = 'https://api.twitter.com/1.1/account/verify_credentials.json'
    >>> resp = session.get(account_url)
    <Response [200]>
    >>> resp.json()
    {...}

The above is not the real flow, just like what we did in
:ref:`fetch_oauth1_access_token`, we need to create another session ourselves::

    >>> access_token = restore_access_token_from_database()
    >>> oauth_token = access_token['oauth_token']
    >>> oauth_token_secret = access_token['oauth_token_secret']
    >>> session = OAuth1Session(
    ...     client_id, client_secret,
    ...     token=oauth_token,
    ...     token_secret=oauth_token_secret)
    >>> account_url = 'https://api.twitter.com/1.1/account/verify_credentials.json'
    >>> resp = session.get(account_url)

Please note, there are duplicated steps in the documentation, read carefully
and ignore the duplicated explains.

Using OAuth1Auth
~~~~~~~~~~~~~~~~

It is also possible to access protected resources with ``OAuth1Auth`` object.
Create an instance of OAuth1Auth with an access token::

    auth = OAuth1Auth(
        client_id='..',
        client_secret=client_secret='..',
        token='oauth_token value',
        token_secret='oauth_token_secret value',
        ...
    )

Pass this ``auth`` to ``requests` to access protected resources::

    import requests

    url = 'https://api.twitter.com/1.1/account/verify_credentials.json'
    resp = requests.get(url, auth=auth)


AsyncOAuth1Client for aiohttp
-----------------------------

.. versionadded:: v0.11
    This is an experimental feature.

The ``AsyncOAuth1Client`` is located in ``authlib.client.aiohttp``. Authlib doesn't
embed ``aiohttp`` as a dependency, you need to install it yourself.

Here is an example on how you can initialize an instance of ``AsyncOAuth1Client``
for ``aiohttp``::

    import asyncio
    from aiohttp import ClientSession
    from authlib.client.aiohttp import AsyncOAuth1Client, OAuthRequest

    REQUEST_TOKEN_URL = 'https://api.twitter.com/oauth/request_token'

    async def main():
        # OAuthRequest is required to handle auth
        async with ClientSession(request_class=OAuthRequest) as session:
            client = AsyncOAuth1Client(session, 'client_id', 'client_secret', ...)
            token = await client.fetch_request_token(REQUEST_TOKEN_URL)
            print(token)

    loop = asyncio.get_event_loop()
    loop.run_until_complete(main())

The API is similar with ``OAuth1Session`` above. Using the ``client`` for the
three steps authorization:

Fetch Temporary Credential
~~~~~~~~~~~~~~~~~~~~~~~~~~

The first step is to fetch temporary credential, which will be used to generate
authorization URL::

    request_token_url = 'https://api.twitter.com/oauth/request_token'
    request_token = await client.fetch_request_token(request_token_url)
    print(request_token)
    {'oauth_token': 'gA..H', 'oauth_token_secret': 'lp..X', 'oauth_callback_confirmed': 'true'}

Save this temporary credential for later use (if required).

Redirect to Authorization Endpoint
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The second step is to generate the authorization URL::

    authenticate_url = 'https://api.twitter.com/oauth/authenticate'
    url = client.create_authorization_url(authenticate_url, request_token['oauth_token'])
    print(url)
    'https://api.twitter.com/oauth/authenticate?oauth_token=gA..H'

Actually, the second parameter ``request_token`` can be omitted, since session
is re-used::

    url = client.create_authorization_url(authenticate_url)
    print(url)
    'https://api.twitter.com/oauth/authenticate?oauth_token=gA..H'

Fetch Access Token
~~~~~~~~~~~~~~~~~~

When the authorization is granted, you will be redirected back to your
registered callback URI. For instance::

    https://example.com/twitter?oauth_token=gA..H&oauth_verifier=fcg..1Dq

If you assigned ``redirect_uri`` in :ref:`fetch_oauth1_access_token`, the
authorize response would be something like::

    https://your-domain.org/auth?oauth_token=gA..H&oauth_verifier=fcg..1Dq

In the production flow, you may need to create a new instance of
``AsyncOAuth1Client``, it is the same as above. You need to use the previous
request token to exchange an access token::

    # twitter redirected back to your website
    resp_url = 'https://example.com/twitter?oauth_token=gA..H&oauth_verifier=fcg..1Dq'

    # you may use the ``oauth_token`` in resp_url to
    # get back your previous request token
    request_token = {'oauth_token': 'gA..H', 'oauth_token_secret': '...'}

    # assign request token to client
    client.token = request_token

    # resolve the ``oauth_verifier`` from resp_url
    oauth_verifier = get_oauth_verifier_value(resp_url)

    access_token_url = 'https://api.twitter.com/oauth/access_token'
    token = await client.fetch_access_token(access_token_url, oauth_verifier)

You can save the ``token`` to access protected resources later.


Access Protected Resources
~~~~~~~~~~~~~~~~~~~~~~~~~~

Now you can access the protected resources. Usually, you will need to create
an instance of ``AsyncOAuth1Client``::

    # get back the access token if you have saved it in some place
    access_token = {'oauth_token': '...', 'oauth_secret': '...'}

    # assign it to client
    client.token = access_token

    account_url = 'https://api.twitter.com/1.1/account/verify_credentials.json'
    async with client.get(account_url) as resp:
        data = await resp.json()

Notice, it is also possible to create the client instance with access token at
the initialization::

    client = AsyncOAuth1Client(
        session, 'client_id', 'client_secret',
        token='...', token_secret='...',
        ...
    )

.. warning::
    This is still an experimental feature in Authlib. Use it with caution.
