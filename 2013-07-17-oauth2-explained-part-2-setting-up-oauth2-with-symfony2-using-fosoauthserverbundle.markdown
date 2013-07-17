---
layout: post
title: "OAuth2 Explained: Part 2 - Setting up OAuth2 with Symfony2 using FOSOAuthServerBundle"
date: 2013-07-17 10:35
comments: true
categories: [OAuth, Symfony2]
---

- [Part 1 - Principles and Terminology](http://blog.tankist.de/blog/2013/07/16/oauth2-explained-part-1-principles-and-terminology/)
- __[Part 2 - Setting up OAuth2 with Symfony2 using FOSOAuthServerBundle](http://blog.tankist.de/blog/2013/07/17/oauth2-explained-part-2-setting-up-oauth2-with-symfony2-using-fosoauthserverbundle/)__
- Part 3 - Using OAuth2 with your bare hands
- Part 4 - Implementing OAuth2 Client with Symfony2 
- Part 5 - Implmenting custom Grant Type

## Prerequesites

Let's assume you already have a project running on Symfony2 with Doctrine2, and you would like to enable some OAuth2 provider functionality on it. In case you still don't have a running symfony installation, please go through [Symfony Book: Installation](http://symfony.com/doc/current/book/installation.html) instructions and get a fresh copy of a Symfony2.

Also your project already, most probably, should has a User Entity, if not you can create something like this one.

<!-- more -->
``` php
  <?php
	
	// src/Acme/DemoBundle/Entity/User.php
	
	namespace Acme\DemoBundle\Entity;
	
	use Symfony\Component\Security\Core\User\UserInterface;
	use Doctrine\ORM\Mapping as ORM;
	
	/**
	 * Acme\UserBundle\Entity\User
	 *
	 * @ORM\Table(name="acme_users")
	 * @ORM\Entity(repositoryClass="Acme\DemoBundle\Repository\UserRepository")
	 */
	class User implements UserInterface, \Serializable
	{
	    /**
	     * @ORM\Column(type="integer")
	     * @ORM\Id
	     * @ORM\GeneratedValue(strategy="AUTO")
	     */
	    private $id;
	
	    /**
	     * @ORM\Column(type="string", length=25, unique=true)
	     */
	    private $username;
	
	    /**
	     * @ORM\Column(type="string", length=25, unique=true)
	     */
	    private $email;
	
	    /**
	     * @ORM\Column(type="string", length=32)
	     */
	    private $salt;
	
	    /**
	     * @ORM\Column(type="string", length=40)
	     */
	    private $password;
	
	    /**
	     * @ORM\Column(name="is_active", type="boolean")
	     */
	    private $isActive;
	
	    public function __construct()
	    {
	        $this->isActive = true;
	        $this->salt = md5(uniqid(null, true));
	    }
	
	    public function getId(){
	        return $this->id;
	    }
	
	    /**
	     * @inheritDoc
	     */
	    public function getUsername()
	    {
	        return $this->username;
	    }
	
	    /**
	     * @inheritDoc
	     */
	    public function setUsername($username)
	    {
	        $this->username = $username;
	        $this->email = $username;
	    }
	
	    /**
	     * @inheritDoc
	     */
	    public function getSalt()
	    {
	        return $this->salt;
	    }
	
	    public function setSalt($salt)
	    {
	        $this->salt = $salt;
	    }
	
	    /**
	     * @inheritDoc
	     */
	    public function getPassword()
	    {
	        return $this->password;
	    }
	
	    public function setPassword($password)
	    {
	        $this->password = $password;
	    }
	
	    /**
	     * @inheritDoc
	     */
	    public function getRoles()
	    {
	        return array('ROLE_USER');
	    }
	
	    /**
	     * @inheritDoc
	     */
	    public function eraseCredentials()
	    {
	    }
	
	    /**
	     * @see \Serializable::serialize()
	     */
	    public function serialize()
	    {
	        return serialize(
	            array(
	                $this->id,
	            )
	        );
	    }
	
	    /**
	     * @see \Serializable::unserialize()
	     */
	    public function unserialize($serialized)
	    {
	        list (
	            $this->id,
	            ) = unserialize($serialized);
	    }
	}
```

We are going to need a user provider as well.

``` php
	<?php
		// src/Acme/DemoBundle/Provider/UserProvider.php

		namespace Acme\DemoBundle\Provider;
		
		use Symfony\Component\Security\Core\User\UserInterface;
		use Symfony\Component\Security\Core\User\UserProviderInterface;
		use Symfony\Component\Security\Core\Exception\UsernameNotFoundException;
		use Symfony\Component\Security\Core\Exception\UnsupportedUserException;
		use Doctrine\Common\Persistence\ObjectRepository;
		use Doctrine\ORM\NoResultException;
		
		class UserProvider implements UserProviderInterface
		{
		    protected $userRepository;
		
		    public function __construct(ObjectRepository $userRepository){
		        $this->userRepository = $userRepository;
		    }
		
		    public function loadUserByUsername($username)
		    {
		        $q = $this->userRepository
		            ->createQueryBuilder('u')
		            ->where('u.username = :username OR u.email = :email')
		            ->setParameter('username', $username)
		            ->setParameter('email', $username)
		            ->getQuery();
		
		        try {
		            $user = $q->getSingleResult();
		        } catch (NoResultException $e) {
		            $message = sprintf(
		                'Unable to find an active admin AcmeDemoBundle:User object identified by "%s".',
		                $username
		            );
		            throw new UsernameNotFoundException($message, 0, $e);
		        }
		
		        return $user;
		    }
		
		    public function refreshUser(UserInterface $user)
		    {
		        $class = get_class($user);
		        if (!$this->supportsClass($class)) {
		            throw new UnsupportedUserException(
		                sprintf(
		                    'Instances of "%s" are not supported.',
		                    $class
		                )
		            );
		        }
		
		        return $this->userRepository->find($user->getId());
		    }
		
		    public function supportsClass($class)
		    {
		        return $this->userRepository->getClassName() === $class
		        || is_subclass_of($class, $this->userRepository->getClassName());
		    }
		}
```

Now register the user manager, repository and provider in the Dependency Injection Container

``` xml
    <parameters>
        <parameter key="platform.entity.user.class">Acme\DemoBundle\Entity\User</parameter>
        <parameter key="platform.user.provider.class">Acme\DemoBundle\Provider\UserProvider</parameter>
    </parameters>

    <services>
        <service id="platform.user.manager" class="Doctrine\ORM\EntityManager"
                 factory-service="doctrine" factory-method="getManagerForClass">
            <argument>%platform.entity.user.class%</argument>
        </service>

        <service id="platform.user.repository"
                 class="Acme\DemoBundle\Repository\UserRepository"
                 factory-service="platform.user.manager" factory-method="getRepository">
            <argument>%platform.entity.user.class%</argument>
        </service>

        <service id="platform.user.provider" class="%platform.user.provider.class%">
            <argument type="service" id="platform.user.repository" />
        </service>
    </services>
```


Ok, now we are good to go with setting up OAuthServerBundle on our platform.


## Installing FOSOAuthServerBundle

To install FOSOauthServerBundle, execute the following in your command line:

``` sh
	 php composer.phar require friendsofsymfony/oauth-server-bundle dev-master
```

This will include the package to the composer.json and install it.

Now you need to enable this bundle in the Kernel:
``` php
	<?php
	// app/AppKernel.php
	
	public function registerBundles()
	{
	    $bundles = array(
	        // ...
	        new FOS\OAuthServerBundle\FOSOAuthServerBundle(),
	    );
	} 
```
Now we need to create three four additional entities. 

### Client Entity

``` php
	// src/Acme/DemoBundle/Entity/Client.php
	
	namespace Acme\DemoBundle\Entity;
	
	use FOS\OAuthServerBundle\Entity\Client as BaseClient;
	use Doctrine\ORM\Mapping as ORM;
	
	/**
	 * @ORM\Entity
	 */
	class Client extends BaseClient
	{
	    /**
	     * @ORM\Id
	     * @ORM\Column(type="integer")
	     * @ORM\GeneratedValue(strategy="AUTO")
	     */
	    protected $id;
	
	    public function __construct()
	    {
	        parent::__construct();
	    }
	}
```

### AccessToken Entity

``` php
	<?php
	// src/Acme/DemoBundle/Entity/AccessToken.php
	
	namespace Acme\DemoBundle\Entity;
	
	use FOS\OAuthServerBundle\Entity\AccessToken as BaseAccessToken;
	use Doctrine\ORM\Mapping as ORM;
	
	/**
	 * @ORM\Entity
	 */
	class AccessToken extends BaseAccessToken
	{
	    /**
	     * @ORM\Id
	     * @ORM\Column(type="integer")
	     * @ORM\GeneratedValue(strategy="AUTO")
	     */
	    protected $id;
	
	    /**
	     * @ORM\ManyToOne(targetEntity="Client")
	     * @ORM\JoinColumn(nullable=false)
	     */
	    protected $client;
	
	    /**
	     * @ORM\ManyToOne(targetEntity="User")
	     */
	    protected $user;
	}
```

### RefreshToken Entity

``` php
	<?php
	// src/Acme/DemoBundle/Entity/RefreshToken.php
	
	namespace Acme\DemoBundle\Entity;
	
	use FOS\OAuthServerBundle\Entity\RefreshToken as BaseRefreshToken;
	use Doctrine\ORM\Mapping as ORM;
	
	/**
	 * @ORM\Entity
	 */
	class RefreshToken extends BaseRefreshToken
	{
	    /**
	     * @ORM\Id
	     * @ORM\Column(type="integer")
	     * @ORM\GeneratedValue(strategy="AUTO")
	     */
	    protected $id;
	
	    /**
	     * @ORM\ManyToOne(targetEntity="Client")
	     * @ORM\JoinColumn(nullable=false)
	     */
	    protected $client;
	
	    /**
	     * @ORM\ManyToOne(targetEntity="User")
	     */
	    protected $user;
	}
```
### AuthCode Entity
``` php
	<?php
	// src/Acme/DemoBundle/Entity/AuthCode.php
	
	namespace Acme\DemoBundle\Entity;
	
	use FOS\OAuthServerBundle\Entity\AuthCode as BaseAuthCode;
	use Doctrine\ORM\Mapping as ORM;
	
	/**
	 * @ORM\Entity
	 */
	class AuthCode extends BaseAuthCode
	{
	    /**
	     * @ORM\Id
	     * @ORM\Column(type="integer")
	     * @ORM\GeneratedValue(strategy="AUTO")
	     */
	    protected $id;
	
	    /**
	     * @ORM\ManyToOne(targetEntity="Client")
	     * @ORM\JoinColumn(nullable=false)
	     */
	    protected $client;
	
	    /**
	     * @ORM\ManyToOne(targetEntity="User")
	     */
	    protected $user;
	}
```
Please pay attention to the user entity namespace, since your User entity might be in other bundle, make sure that namespaces pointing to User entity are correct. 

Ok, entities are created, now it's time to create a separate login page for users coming from OAuth direction. I prefer to use separate login forms to handle this, because usually inside the project you have different redirection policies, and in case of OAuth you strictly need to redirect back to referrer. But of course feel free to reuse your already existing login form inside the project, just make sure it redirects you to the right place then. 

Here is the controller responsible for the login form
``` php
	<?php

	# src/Acme/DemoBundle/Controller/SecurityController.php
	
	namespace Acme\DemoBundle\Controller;
	
	use Symfony\Bundle\FrameworkBundle\Controller\Controller;
	use Symfony\Component\HttpFoundation\Request;
	use Symfony\Component\Security\Core\SecurityContext;
	
	class SecurityController extends Controller
	{
	    public function loginAction(Request $request)
	    {
	        $session = $request->getSession();
	
	        if ($request->attributes->has(SecurityContext::AUTHENTICATION_ERROR)) {
	            $error = $request->attributes->get(SecurityContext::AUTHENTICATION_ERROR);
	        } elseif (null !== $session && $session->has(SecurityContext::AUTHENTICATION_ERROR)) {
	            $error = $session->get(SecurityContext::AUTHENTICATION_ERROR);
	            $session->remove(SecurityContext::AUTHENTICATION_ERROR);
	        } else {
	            $error = '';
	        }
	
	        if ($error) {
	            $error = $error->getMessage(
	            ); // WARNING! Symfony source code identifies this line as a potential security threat.
	        }
	
	        $lastUsername = (null === $session) ? '' : $session->get(SecurityContext::LAST_USERNAME);
	
	        return $this->render(
	            'AcmeDemoBundle:Security:login.html.twig',
	            array(
	                'last_username' => $lastUsername,
	                'error' => $error,
	            )
	        );
	    }
	
	    public function loginCheckAction(Request $request)
	    {
	
	    }
	}
```
and here is the corresponding minimal template
``` php
 	{# src/Acme/DemoBundle/Resources/views/Security/login.html.twig #}
	<div class="form">
	    <form id="login" class="vertical" action="{{ path('acme_oauth_server_auth_login_check') }}" method="post">
	        <div class="form_title">
	            OAuth Authorization
	        </div>
	        {% if(error) %}
	        <div class='form_error'>{{ error }}</div>
	        {% endif %}
	        <div class="form_item">
	            <div class="form_label"><label for="username">Username</label>:</div>
	            <div class="form_widget"><input type="text" id="username" name="_username" /></div>
	        </div>
	        <div class="form_item">
	            <div class="form_label"><label for="password">Password</label>:</div>
	            <div class="form_widget"><input type="password" id="password" name="_password" /></div>
	        </div>
	        <div class="form_button">
	            <input type="submit" id="_submit" name="_submit" value="Log In" />
	        </div>
	    </form>
	</div>
```

### Configuration


Following goes into __security.yml__
``` yaml
	security:
	    encoders:
	        Acme\DemoBundle\Entity\User:
	            algorithm:        sha1
	            encode_as_base64: false
	            iterations:       1
	
	    role_hierarchy:
	        ROLE_ADMIN:       ROLE_USER
	        ROLE_SUPER_ADMIN: ROLE_ADMIN
	
	    providers:
	        user_provider:
	            id: platform.user.provider
	
	
	    firewalls:
	        dev:
	            pattern:  ^/(_(profiler|wdt)|css|images|js)/
	            security: false
	
	        login:
	            pattern:  ^/demo/secured/login$
	            security: false
	
	
	        oauth_token:
	            pattern:    ^/oauth/v2/token
	            security:   false
	
	        secured_area:
	            pattern:    ^/demo/secured/
	            form_login:
	                provider: user_provider
	                check_path: _security_check
	                login_path: _demo_login
	            logout:
	                path:   _demo_logout
	                target: _demo
	            #anonymous: ~
	            #http_basic:
	            #    realm: "Secured Demo Area"
	
	        oauth_authorize:
	            pattern:    ^/oauth/v2/auth
	            form_login:
	                provider: user_provider
	                check_path: _security_check
	                login_path: _demo_login
	            anonymous: true
	
	        api:
	            pattern:    ^/api
	            fos_oauth:  true
	            stateless:  true
	
	    access_control:
	        # You can omit this if /api can be accessed both authenticated and anonymously
	        - { path: ^/api, roles: [ IS_AUTHENTICATED_FULLY ] }
	        - { path: ^/demo/secured/hello/admin/, roles: ROLE_ADMIN }
	        #- { path: ^/login, roles: IS_AUTHENTICATED_ANONYMOUSLY, requires_channel: https }
``` 

following goes to __routing.yml__
``` yaml

	# app/config/routing.yml
	fos_oauth_server_token:
	    resource: "@FOSOAuthServerBundle/Resources/config/routing/token.xml"
	
	fos_oauth_server_authorize:
	    resource: "@FOSOAuthServerBundle/Resources/config/routing/authorize.xml"
	
	acme_oauth_server_auth_login:
	    pattern:  /oauth/v2/auth_login
	    defaults: { _controller: AcmeDemoBundle:Security:login }
	
	acme_oauth_server_auth_login_check:
	    pattern:  /oauth/v2/auth_login_check
	    defaults: { _controller: AcmeDemoBundle:Security:loginCheck }
```

and finally __config.yml__
``` yaml
	# app/config/config.yml
	fos_oauth_server:
	    db_driver: orm
	    client_class:        Acme\DemoBundle\Entity\Client
	    access_token_class:  Acme\DemoBundle\Entity\AccessToken
	    refresh_token_class: Acme\DemoBundle\Entity\RefreshToken
	    auth_code_class:     Acme\DemoBundle\Entity\AuthCode
	    service:
	        user_provider: platform.user.provider
	        options:
	            supported_scopes: user
```

Now all configurations done, time to update database structure
``` sh
	php app/console doctrine:schema:update --force
```
### Client Creation

first thing you need to do is give the platform ability to easily create clients for OAuth protected communication, and a protected API call.

Create a command file
``` php
	<?php
	# src/Acmen/DemoBundle/Command/CreateClientCommand.php	
	namespace Acme\DemoBundle\Command;
	
	use Symfony\Bundle\FrameworkBundle\Command\ContainerAwareCommand;
	use Symfony\Component\Console\Input\InputArgument;
	use Symfony\Component\Console\Input\InputOption;
	use Symfony\Component\Console\Input\InputInterface;
	use Symfony\Component\Console\Output\OutputInterface;
	use Acme\OAuthServerBundle\Document\Client;
	
	class CreateClientCommand extends ContainerAwareCommand
	{
	    protected function configure()
	    {
	        $this
	            ->setName('acme:oauth-server:client:create')
	            ->setDescription('Creates a new client')
	            ->addOption(
	                'redirect-uri',
	                null,
	                InputOption::VALUE_REQUIRED | InputOption::VALUE_IS_ARRAY,
	                'Sets redirect uri for client. Use this option multiple times to set multiple redirect URIs.',
	                null
	            )
	            ->addOption(
	                'grant-type',
	                null,
	                InputOption::VALUE_REQUIRED | InputOption::VALUE_IS_ARRAY,
	                'Sets allowed grant type for client. Use this option multiple times to set multiple grant types..',
	                null
	            )
	            ->setHelp(
	                <<<EOT
	                    The <info>%command.name%</info>command creates a new client.
	
	<info>php %command.full_name% [--redirect-uri=...] [--grant-type=...] name</info>
	
	EOT
	            );
	    }
	
	    protected function execute(InputInterface $input, OutputInterface $output)
	    {
	        $clientManager = $this->getContainer()->get('fos_oauth_server.client_manager.default');
	        $client = $clientManager->createClient();
	        $client->setRedirectUris($input->getOption('redirect-uri'));
	        $client->setAllowedGrantTypes($input->getOption('grant-type'));
	        $clientManager->updateClient($client);
	        $output->writeln(
	            sprintf(
	                'Added a new client with public id <info>%s</info>.',
	                $client->getPublicId()
	            )
	        );
	    }
	}
```

In order to test it, please execute
``` sh
	 php app/console acme:oauth-server:client:create --redirect-uri="http://clinet.local/" --grant-type="authorization_code" --grant-type="password" --grant-type="refresh-token" --grant-type="token" --grant-type="client_credentials"
```
this will register an OAuth client on the platform side, and will Output something like this (of course, you will see slightly different output)
```
	Added a new client with public id 5_ebg354gknv48kc88o8oogwokckco0o40sc000cowc8soosw0k, secret 5ub5upfxih0k8g44w00ogwc4swog4088o8444sssos8k888o8g
```
Keep those public_id and secret somewhere private, since that's the credentials for the client application to access your backend using oauth.

### Check if it works

Execute the following request in your browser 
```
	http://portal.local/app_dev.php/oauth/v2/token?client_id=5_ebg354gknv48kc88o8oogwokckco0o40sc000cowc8soosw0k&client_secret=5ub5upfxih0k8g44w00ogwc4swog4088o8444sssos8k888o8g&grant_type=client_credentials
```
If you see response like this one, the we did everything correctly, otherwise, leave a comment, or contact me over twitter, let's figure this out together ;)
``` json
	{"access_token":"YTk0YTVjZDY0YWI2ZmE0NjRiODQ4OWIyNjZkNjZlMTdiZGZlNmI3MDNjZGQwYTZkMDNiMjliNDg3NWYwZWI0MQ","expires_in":3600,"token_type":"bearer","scope":"user","refresh_token":"ZDU1MDY1OTc4NGNlNzQ5NWFiYTEzZTE1OGY5MWNjMmViYTBiNmRjOTNlY2ExNzAxNWRmZTM1NjI3ZDkwNDdjNQ"}
```
