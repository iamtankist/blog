---
layout: post
title: "OAuth2 Explained: Part 1 - Principles and Terminology"
date: 2013-07-16 15:35
comments: true
categories: [OAuth, Symfony2]
---

- [Part 1 - Principles and Terminology](http://blog.tankist.de/blog/2013/07/16/oauth2-explained-part-1-principles-and-terminology/)
- [Part 2 - Setting up OAuth2 with Symfony2 using FOSOAuthServerBundle](http://blog.tankist.de/blog/2013/07/17/oauth2-explained-part-2-setting-up-oauth2-with-symfony2-using-fosoauthserverbundle/)
- [Part 3 - Using OAuth2 with your bare hands](http://blog.tankist.de/blog/2013/07/18/oauth2-explained-part-3-using-oauth2-with-your-bare-hands/)
- Part 4 - Implementing custom Grant Type
- Part 5 - Implementing OAuth2 Client with Symfony2 


## Why OAuth and a bit of Terminology

Before we dive in into technical aspects of OAuth, let's get the OAuth terminology straight.

### Provider

So you have a functional platform, gathered some data and functionality and now it's time when you need to provide this data and functionality over an API to your users, mobile devices and other platforms. You will be an OAuth __Provider__, rather soon. 
<!-- more -->
### Access Token


Let's say you have an API to protect. You need to provide some functionality over it, including passing back and forth sensible data.

Let's assume, a user is able to log in using mobile application and request his current balance on his account. Simplest thing comes to your mind: 
> /user/15/balance

where 15 is user's id? Bad idea. Everyone who is able to use any kind of development console for the browser can change user id and get access to a sensible data. How can you solve this?

Right. The same way PHP Sessions work for example. You generate a token, something similar to PHPSESSID, you create it at the moment when user logs in, pass this token back and forth with each request, and that's how the server knows that this is the user associated with current session. There is no way one can manipulate session token in order to get data for specific user. And then when the user logs out you remove the token, or it just gets expired after awhile. 

Congratulations, you just invented the main entity of OAuth - __Access Token__.

In OAuth world obtaining Access Token means getting access to the application. Once you get it, you attach the access token to each of your requests, and Provider can uniquely identify who you are and what actions are you allowed to perform. Obtained Access Token? Game over, You Won!

### Clients and Scopes

What if, we need a special kind of service which is able to set user balance, and that's for each and one of them. One solution (and your operations team will really like it) you just give it another endpoint, and this endpoint is open only for white-listed IPs. If you googled for this article, most probably you already know at least 5 reasons why this solution stinks. 

A better solution would be: You are able, somehow, on the backend side, distinguish between different type of consumers for your applications. Some are more privileged, some are less. Some are allowed to request only their own balance, some clients are allowed to manipulate this balances for all possible users. So we need a mechanism to specify different __Clients__  and give them different privileges or in OAuth terminology - __Scopes__. You most probably don't need a Client for each User, you are fine with one Client for all Users, one Client for a MobileApp, one Client for an accounting application. Each client gets a pair of credentials. __Client ID__ and __Client Secret__. You need those in order to identify your client. 

### Grant Types

If you ever used Facebook Login for websites, you know how the most popular Grant Type works. You get redirected to Provider's website, if you are not yet logged in, the Provider offers you a login, then it asks for certain permissions, what can the Client do on your behalf, and then redirects you back. That's the __Authoritation_Code__ grant. 

But then we have our accounting application. There's no login associated with it, no web interface, it's just a dumb-as-hell cronjob which just wants to trigger some HTTP requests to update user balances on your side. Then we use __Client_Credentials__ grant type. It means in order to obtain access to the api, a client should only present his Client Id and Client Password.

Or an intermediate grant type - password. You provide a client id, client secret, username, password, and the Provider issues an access token, which is connected to this user, so when you get a request, you can identify which user made it.

So the __Grant Type__ represents the flow needed for the __Client__ to obtain __Access Token__. 

In OAuth2 there are five Grant Types that are defined by the standard, but you have an ability to implement a custom one for your needs. 

For example you need to identify user not by username/password, but you give them API keys, in order not to transmit password in clear text over API. Then you need to implement a grant type for that flow. How to do that in details will be described in the 5th part of this tutorial.
