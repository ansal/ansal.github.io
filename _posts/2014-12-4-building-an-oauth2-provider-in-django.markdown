---
layout: post
title:  "Building an OAuth 2.0 provider in Django"
date:   2014-12-4 10:30:00
---

I was working on a Django project recently that required OAuth 2.0 authentication so that others can authenticate their app and consume the API provided by the app. The OAuth reference guide - http://tools.ietf.org/html/rfc6749 was a little dense fore me. My understandings of OAuth mostly comes from two articles - [Zapier API Guide](https://zapier.com/learn/apis/) and [Mashape OAuth Bible](https://github.com/Mashape/mashape-oauth/blob/master/FLOWS.md). I hope you will find them useful too.

Django has excellent packages for providing OAuth capabilities. A quick search found me CaffieneHit’s [django-oauth2-provider](https://github.com/caffeinehit/django-oauth2-provider) which has excellent OAuth provisioning capabilities.

I created a small django app to demonstrate how I built the OAuth 2.0 provider in Django and django-oauth2-provider. You can clone the source [here](https://github.com/ansal/django_oauth_demo).

Setting up the project
-----------
Use [virtualenv](http://virtualenv.readthedocs.org/en/latest/) to create a new virtual environment for the project. I use [virtualenvwrapper](https://virtualenvwrapper.readthedocs.org/en/latest/) for managing my virtual environments. Creating and activating a new virtual environment using virtualenvwrapper is as easy as

    mkvirtualenv django-oauth-demo
 
 Once its done, install Django using pip. Please note that I am installing Django 1.6 here as the current version of django-oauth2-provider is not compatiblie with Django 1.7. See https://github.com/caffeinehit/django-oauth2-provider/issues/96 for more information. This is possible due to Django 1.7 dropping 'mimetype' parameter in HttpResponse and using 'content_type' instead. There are pull requests for this available, but its still not merged into the master branch yet.
 

    pip install Django==1.6
    
Then we need to create a new Django project. I've also created a new app named api which will contain the code for OAuth 2.0.  

    django-admin startproject django_oauth_demo
    cd django_oauth_demo/
    python manage.py startapp api

Install django-oauth2-provider to our environment using

    pip install django-oauth2-provider

Once django-oauth2-provider is installed, add it to the INSTALLED_APPS in settings.py to enable it. Also add our newly created app api to the list of installed apps.


    # django_oauth_demo/django_oauth_demo/settings.py
    INSTALLED_APPS = (
        ...
        'django.contrib.staticfiles',
        'provider',
        'provider.oauth2',
        'api',
    )

Now run migrations to syncdb all the necessary tables.

    python manage.py syncdb

Include provider.oauth2.urls into your urls.py file.

    urlpatterns = patterns('',
        ...
        url(r'^oauth2/', include('provider.oauth2.urls', namespace = 'oauth2')),
    )

Finally use the Django shell to create a new OAuth client. You can also use Django admin to create the client. You need to enable Django admin  though. From Django shell, first create a user object to use with the client. And then use that user object to create OAuth client itself.

    >>> from django.contrib.auth.models import User
    >>> from provider.oauth2.models import Client
    # Create the user object
    >>> user = User(username='apitester')
    >>> user.set_password('apitester')
    >>> user.save()
    # Create new client with the user object created above
    >>> client = Client(user=user, name="api tester", client_type=1, url="http://example.com")
    >>> client.save()
    >>> client.client_id
    u'43e2ece67d8e263b054d'
    >>> client.client_secret
    u'c496768bf6546424e8a74d66b5098efe32dfe6a5'

Testing the Provider
--------------------
Now we can launch the server and test the OAuth Provider. Use the client_id and client_secret from the above shell output.

    python manage.py runserver
    
I use curl to test the API. You can use any http client like [Postman](https://chrome.google.com/webstore/detail/postman-rest-client/fdmmgilgnpjigdojojpjoooidkmcomcm?hl=en) to test too. Using curl to test the API with client_id and client_secret from above step is given below.

    curl -d "client_id=ec6b98fe31fc946b494b&client_secret=45490d6427d6f72c3d0864a66cdc7a6243171a8b&grant_type=password&username=apitester&password=apitester&scope=write" http://localhost:8000/oauth2/access_token
    
And if everything is fine, you will get back your access token like this -

    {"access_token": "d4f4a90739e334e3c08495017745c6a8e4f8965f", "token_type": "Bearer", "expires_in": 2591999, "scope": "read write read+write"}
    
With client ids and secrets passing around, SSL encryption is a must have for your APIs.

Writing the API
------------------
Now we've got Oauth 2.0 working, lets move ahead to build a small API. Of-course you can use django packages like [Tastypie](https://django-tastypie.readthedocs.org/en/latest/) or [Django-Rest-Framework](http://www.django-rest-framework.org/) to build the API and django-oauth2-provider will easily merge into these packages to provide authentication. But here we are going to build a simple view that returns the username of the API client by reading its access token. The verify_access_token function inside api/views.py validates the access token. The code for verify_access looks like

        from django.utils import timezone
        from provider.oauth2.models import AccessToken
        
        def verify_access_token(key):
            try:
                token = AccessToken.objects.get(token=key)
        
                if token.expires < timezone.now():
                    raise OAuthError('AccessToken has expired.')
            except AccessToken.DoesNotExist, e:
                raise OAuthError("AccessToken not found at all.")
                
            return token
        
And finally is_authenticated function inside the views.py adds token and other information to request object once call to verify_access_token is successful.

The actual view is pretty straightforward. It just calls the is_authenticated function with its request object. If is_authenticated returns False, it sends a 403 response. Otherwise username of the client is returned. We use Django's csrf_exempt decorator to exempt this view from CSRF protection. The code for the view will look like

    from django.views.decorators.csrf import csrf_exempt
    
    @csrf_exempt
    def verify_token(request):
    
        if is_authenticated(request) == False:
            return HttpResponse(
                json.dumps({
                    'error': 'invalid_request'
                }),
                status = 403
            )
    
        return HttpResponse(
            json.dumps({
                'user': request.user.username
            })
        )

Hook this view into urls.py by adding this url to your project's urls.py.

    import api.views
    
    urlpatterns = patterns('',
        ...
        url(r'^api/verify', api.views.verify_token, name='verify' ),
        ...
    )
    
And test this view using client by issuing the below command.

    curl -d "token=64b4a073573aed46dcafa8172573f1b4219d5945" http://localhost:8000/api/verify

Note that the token is obtained from our previous request to oauth2/access_token. If everything is fine, you will get your username back.

    {"user": "apitester"}

And thats it! I hope this small example will help you get started with building OAuth2.0 providers in Django. The type of authentication we used here is called [OAuth 2 (two-legged)](https://github.com/Mashape/mashape-oauth/blob/master/FLOWS.md#oauth-2-two-legged). You can find many more types in Mashape's OAuth bible.