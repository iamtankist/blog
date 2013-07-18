---
layout: post
title: "OAuth2 Explained: Part 3 - Using OAuth2 with your bare hands"
date: 2013-07-18 14:35
comments: true
categories: [OAuth, Symfony2]
published: true
---

- [Part 1 - Principles and Terminology](http://blog.tankist.de/blog/2013/07/16/oauth2-explained-part-1-principles-and-terminology/)
- [Part 2 - Setting up OAuth2 with Symfony2 using FOSOAuthServerBundle](http://blog.tankist.de/blog/2013/07/17/oauth2-explained-part-2-setting-up-oauth2-with-symfony2-using-fosoauthserverbundle/)
- [Part 3 - Using OAuth2 with your bare hands](http://blog.tankist.de/blog/2013/07/18/2013-07-18-oauth2-explained-part-3-using-oauth2-with-your-bare-hands)
- Part 4 - Implementing custom Grant Type
- Part 5 - Implementing OAuth2 Client with Symfony2 


## Preparations

We'll need a controller that mimics a dummy API behavior for us.
<!-- more -->
``` php
	<?php

	namespace Acme\DemoBundle\Controller;

	use Symfony\Bundle\FrameworkBundle\Controller\Controller;
	use Symfony\Component\HttpFoundation\JsonResponse;

	class ApiController extends Controller
	{
	    public function articlesAction()
	    {
	        $articles = array('article1', 'article2', 'article3');
	        return new JsonResponse($articles);
	    }

	    public function userAction()
	    {
	        $user = $this->container->get('security.context')->getToken()->getUser();
	        if($user) {
	            return new JsonResponse(array(
	                'id' => $user->getId(),
	                'username' => $user->getUsername()
	            ));
	        }

	        return new JsonResponse(array(
	            'message' => 'User is not identified'
	        ));

	    }
	}
```

## Let's Fail!

Please keep in mind, that the point is to obtain access_token. 

Why? Request

	PROVIDER_HOST/api/articles

What? 

	{"error":"access_denied","error_description":"OAuth2 authentication required"}

Right. That's why. Our API is protected, we don't let everybody in. Let's try to do this again once we have the access_token at the end of the chapter.

## Registering OAuth Client

First we need to create a Client and allow all possible grant types for it. Here's how to do that. Navigate to the project root of the provider (the one we created in [previous part](http://blog.tankist.de/blog/2013/07/17/oauth2-explained-part-2-setting-up-oauth2-with-symfony2-using-fosoauthserverbundle/)) and then execute the following command. Client host represents a URL, where your client application is deployed. You will get redirected here if everything went as planned.

	php app/console acme:oauth-server:client:create --redirect-uri="CLIENT_HOST" --grant-type="authorization_code" --grant-type="password" --grant-type="refresh-token" --grant-type="token" --grant-type="client_credentials"


command should respond with something like 

	Added a new client with public id 2_1y1zqhh7ws5c8kok8g8w88kkokos0wwswwwowos4o48s48s88w, secret 16eqpwofy5dwo4wggk4s40s80sgcs4gc0cwgwsc8k8w0k8sks4


Please keep those values somewhere in an easy accessible place, we will need those during this tutorial. 
To keep URLs short and readable I'll refer to those as CLIENT\_ID and CLIENT\_SECRET in the future.

Let's start from the more complicated grant types and move on to the simpler ones. 

## Authorization Code

That's the most commonly used one, recommended to authorize end customers. A good example is the Facebook Login for websites. Here's how it works. 

Request this url in the browser:

	PROVIDER_HOST/oauth/v2/auth?client_id=CLIENT_ID&response_type=code&redirect_uri=CLIENT_HOST
	
note: redirect_uri should be identical to the one provided on client creation, otherwise you will get a corresponding error message. 

The page you are requesting will offer you a login, then authorization of the client permissions, once you confirm everything it will redirect you back to the url you provided in redirect_url. In our case, redirect will look like

	CLIENT_HOST/?code=Yjk2MWU5YjVhODBiN2I0ZDRkYmQ1OGM0NGY4MmUyOGM2NDQ2MmY2ZDg2YjUxYjRiMzAwZTY2MDQxZmUzODg2YQ

I'll refer to this long code parameter as CODE in the future. This code is stored on the Provider side, and once you request for the token, it can uniquely identify the client which made request and the user. 

It's time to request the token

	PROVIDER_HOST/oauth/v2/token?client_id=CLIENT_ID&client_secret=CLIENT_SECRET&grant_type=authorization_code&redirect_uri=http%3A%2F%2Fclinet.local%2F&code=CODE

Most probably this request will fail. That's because CODE expires rather quickly. Fear not, just request first URL, repeat the process, prepare the second url in the text editor of your choice, copy in the code rather quickly, and you will get the desired result. 

It's a JSON which contains access_token and looks like this

	{"access_token":"NjlmNDNiZTU4ZDY3ZGFlYTI5MGEzNDcxZWVmZDU4Y2E1NGJmZTJlMjNjNzc2M2E0MmZlZTk2ZjliMWE0MDQyNw","expires_in":3600,"token_type":"bearer","scope":null,"refresh_token":"ZGU2NzlhOTQ2MmRlY2YyYjAyMjBkYmJmMmJhMDllNTgyNmJkNmQxOWZlNGQ4NzczY2RiMThlNmRhMjBiYjFjNg"}

this suggests that access_token expires in 3600 seconds, and to refresh it you have the refresh token. We will discuss how to handle that later on this chapter.

## Implicit Grant

It's similar to Authorization Code grant, it's just a bit simpler. You just need to make only one request, and you will get the access_token as a part of redirect URL, there's no need for second response. That's for the situations where you trust the user and the client, but you still want the user to identify himself in the browser.

	PROVIDER_HOST/oauth/v2/auth?client_id=2_1y1zqhh7ws5c8kok8g8w88kkokos0wwswwwowos4o48s48s88w&redirect_uri=http%3A%2F%2Fclinet.local%2F&response_type=token

then you will get redirected to 

	CLIENT_HOST/#access_token=YWZhZWQ5NjQxOTI2ODJmZWE4YjJiYmExZTIxZmE5OWUxOWZjZjgwZDFlZWMwMjkyZDQwZWU1NWI4YWIzODllNQ&expires_in=3600&token_type=bearer&refresh_token=YzQ1YjRhODk2YzJiYTZmMzNiNjI5ZjI2MDI3ZmMwMDg3MjkxMDdhYmE5YjBlYzRlZmM2M2Q0NTM3ZjFmZDZiYQ
	
## Password flow

Let's say you have no luxury of redirecting user to some website, then handle redirect call, all you have is just an application which is able to send HTTP requests. And you still want to somehow authenticate user on the server side, and all you have is username and password. 

Request: 

	PROVIDER_HOST/oauth/v2/token?client_id=CLIENT_ID&client_secret=CLIENT_SECRET&grant_type=password&username=USERNAME&password=PASSWORD

Response:

	{"access_token":"MjY1MWRhYTAyZDZlOTEyN2EzNTg4MGMwMTcyYjczY2Y0MWI3NzZjODc1OGM2NDdjODgxZjY3YzEyMDdhZjU0Yg","expires_in":3600,"token_type":"bearer","scope":null,"refresh_token":"MDNmNzBmNWQ2NzdhYWVmYjE2NjI3ZjAyZTM4Y2Q1NDRiNDY1YjUyZGE1ZDk0ODZjYmU0MDM0NTQxNjhiZmU3ZA"}

## Client Credentials

This one is the most simplistic flow of them all. You just need to provide CLIENT\_ID and CLIENT\_SECRET. 

	PROVIDER_HOST/oauth/v2/token?client_id=CLIENT_ID&client_secret=CLIENT_SECRET&grant_type=client_credentials

Response will be

	{"access_token":"YTk0YTVjZDY0YWI2ZmE0NjRiODQ4OWIyNjZkNjZlMTdiZGZlNmI3MDNjZGQwYTZkMDNiMjliNDg3NWYwZWI0MQ","expires_in":3600,"token_type":"bearer","scope":"user","refresh_token":"ZDU1MDY1OTc4NGNlNzQ5NWFiYTEzZTE1OGY5MWNjMmViYTBiNmRjOTNlY2ExNzAxNWRmZTM1NjI3ZDkwNDdjNQ"}

## Refresh flow

Before I mentioned that access_tokens have a lifetime of one hour, after which they will expire. With every access_token you were provided a refresh_token. You can exchange refresh token and get a new pair of access_token and refresh_token

	PROVIDER_HOST/oauth/v2/token?client_id=CLIENT_ID&client_secret=CLIENT_SECRET&grant_type=refresh_token&refresh_token=REFRESH_TOKEN

response

	{"access_token":NEW_ACCESS_TOKEN,"expires_in":3600,"token_type":"bearer","scope":"user","refresh_token":"NEW_REFRESH_TOKEN"}

## Let's not fail or using the access\_token

Remember our failed attempt to request an API at the beginning of the article? Let's try again.

Request

	PROVIDER_HOST/api/articles?access_token=ACCESS_TOKEN

Response
	
	["article1","article2","article3"]

Seems like a proper response, does it?

Let's try the other request, which is supposed to maintain user session with the access_token.

Request

	PROVIDER_HOST/api/user?access_token=ACCESS_TOKEN

If you obtained your access_token through Authorization Code, Implicit Grant or Password, you should see a JSON representation of the user object.

	{"id":1,"username":"user1"}

If that was the Client Credentials, ACCESS_TOKEN doesn't contain any user information association with it, therefore there can't be any user information retrieved, as the response suggests

	{"message":"User is not identified"}

In the next part we will define a custom Grant type which allows us to retrieve access_token based on API Key.
