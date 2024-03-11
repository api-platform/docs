# Adding a OAuth2 Authentication using `FOSOAuthServerBundle`

> [OAuth](https://oauth.net/2/) is an open standard for authorization, commonly used as a way for Internet users to authorize websites or applications to access their information on other websites but without giving them the passwords. This mechanism is used by companies such as Google, Facebook, Microsoft and Twitter to permit the users to share information about their accounts with third party applications or websites.

[Wikipedia](https://en.wikipedia.org/wiki/OAuth)

API Platform allows to easily add a OAuth2-based authentication to your API using [FOSOAuthServerBundle](https://github.com/FriendsOfSymfony/FOSOAuthServerBundle).

API Platform is fully working with [FOSOAuthServerBundle](https://github.com/FriendsOfSymfony/FOSOAuthServerBundle).

This tutorial is based on [Getting Started With FOSOAuthServerBundle](https://github.com/FriendsOfSymfony/FOSOAuthServerBundle/blob/master/Resources/doc/index.md) and [Basic RESTful API with Symfony 2 + FOSRestBundle (JSON format only) + FOSUserBundle + FOSOauthServerBundle](https://gist.github.com/tjamps/11d617a4b318d65ca583)

## Install FOSOauthServerBundle

Install the bundle with composer:

```bash
composer require friendsofsymfony/oauth-server-bundle
```

Enable the bundle in the kernel:

```php
<?php
// app/AppKernel.php

public function registerBundles()
{
    $bundles = [
        // ...
        new FOS\OAuthServerBundle\FOSOAuthServerBundle(),
    ];
}
```

## Creating Entities to Store OAuth2 Data

Add all the following classes to your entities and run `php bin/console doctrine:schema:update --force`:

```php
<?php
// AppBundle/Entity/AccessToken.php

namespace AppBundle\Entity;

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
     * @ORM\ManyToOne(targetEntity="AppBundle\Entity\User")
     */
    protected $user;
}
```

```php
<?php
// AppBundle/Entity/AuthCode.php

namespace AppBundle\Entity;

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
     * @ORM\ManyToOne(targetEntity="AppBundle\Entity\User")
     */
    protected $user;
}

```

```php
<?php
// AppBundle/Entity/Client.php

namespace AppBundle\Entity;

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
        // your own logic
    }
}

```

```php
<?php
// AppBundle/Entity/RefreshToken.php

namespace AppBundle\Entity;

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
     * @ORM\ManyToOne(targetEntity="AppBundle\Entity\User")
     */
    protected $user;
}

```

`AppBundle/Entity/User.php`

Add the file from the [FOSUserBundle Integration](fosuser-bundle.md#creating-a-user-entity-with-serialization-groups).

## Security Configuration

```yaml
# To get started with security, check out the documentation:
# http://symfony.com/doc/current/book/security.html
security:
    encoders:
        FOS\UserBundle\Model\UserInterface: bcrypt

    providers:
        fos_user_bundle:
            id: fos_user.user_provider.username

    firewalls:
        # disables authentication for assets and the profiler, adapt it according to your needs
        dev:
            pattern: ^/(_(profiler|wdt)|css|images|js)/
            security: false

        oauth_token:
            pattern: ^/oauth/v2/token
            security: false

        oauth_authorize:
            pattern: ^/oauth/v2/auth
            # Add your favorite authentication process here

        api_docs:
            pattern: ^/(|docs.json|docs.jsonld|index.json)$
            form_login: false
            provider: fos_user_bundle
            http_basic:
                realm: "Please enter your login data."

        api:
            pattern: ^/.+
            fos_oauth: true
            stateless: true
            anonymous: true

    access_control:
        - { path: ^/.+, roles: [IS_AUTHENTICATED_FULLY] }
```

## Routing Configuration

```yaml
api:
    resource: "."
    type: "api_platform"

fos_oauth_server_token:
    resource: "@FOSOAuthServerBundle/Resources/config/routing/token.xml"

fos_oauth_server_authorize:
    resource: "@FOSOAuthServerBundle/Resources/config/routing/authorize.xml"
```

## Create clients

Add this class to `AppBundle/Command/CreateOAuthClientCommand.php` to generate oauth clients via console:

```php
<?php

namespace AppBundle\Command;

use AppBundle\Entity\Client;
use Symfony\Bundle\FrameworkBundle\Command\ContainerAwareCommand;
use Symfony\Component\Console\Input\InputArgument;
use Symfony\Component\Console\Input\InputInterface;
use Symfony\Component\Console\Output\OutputInterface;
use Symfony\Component\HttpFoundation\Request;

class CreateOAuthClientCommand extends ContainerAwareCommand
{
    protected function configure()
    {
        $this
            ->setName('oauth:client:create')
            ->setDescription('Create OAuth Client')
            ->addArgument(
                'grantType',
                InputArgument::REQUIRED,
                'Grant Type?'
            )
            ->addArgument(
                'redirectUri',
                InputArgument::OPTIONAL,
                'Redirect URI?'
            );
    }

    protected function execute(InputInterface $input, OutputInterface $output)
    {
        $container = $this->getContainer();
        $redirectUri = $input->getArgument('redirectUri');
        $grantType = $input->getArgument('grantType');

        $clientManager = $container->get('fos_oauth_server.client_manager.default');
        /** @var Client $client */
        $client = $clientManager->createClient();
        $client->setRedirectUris($redirectUri ? [$redirectUri] : []);
        $client->setAllowedGrantTypes([$grantType]);
        $clientManager->updateClient($client);

        $output->writeln(sprintf('<info>The client <comment>%s</comment> was created with <comment>%s</comment> as public id and <comment>%s</comment> as secret</info>',
            $client->getId(),
            $client->getPublicId(),
            $client->getSecret()
        ));
    }
}
```

Now you can generate two clients. One for our swagger api documentation and one for our application that wants to get data from our api.

```bash
# Application client
php bin/console oauth:client:create password

# Swagger api documentation client
php bin/console oauth:client:create client_credentials
```

## OAuth2 Configuration

Add the following code to your `app/config/config.yml` and replace the `clientId` and `clientSecret` with the data from the generated application client with the `client_credentials` grant type.

```yaml
# ...
fos_oauth_server:
    db_driver: orm # Drivers available: orm, mongodb, or propel
    client_class: 'AppBundle\Entity\Client'
    access_token_class: 'AppBundle\Entity\AccessToken'
    refresh_token_class: 'AppBundle\Entity\RefreshToken'
    auth_code_class: 'AppBundle\Entity\AuthCode'
    service:
        user_provider: fos_user.user_provider.username
        options:
            access_token_lifetime: 10800
            supported_scopes: user

api_platform:
    # ...
    oauth2:
        enabled: true
        clientId: "enter-swagger-api-documentation-client-id"
        clientSecret: "enter-swagger-api-documentation-client-secret"
```

That's all, now your OAuth2 authentication should work.
