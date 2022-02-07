<img alt="CryptoPayApi" src="images/crypto.png"/>

# klev-o/crypto-pay-api

Simple and convenient implementation Crypto Pay API with php version ^7.4 support. Based on the [Official Crypto Pay Api](https://telegra.ph/Crypto-Pay-API-11-25)

[![License](https://img.shields.io/github/license/klev-o/crypto-pay-api)](https://github.com/klev-o/crypto-pay-api/blob/main/LICENSE)
![Packagist Downloads](https://img.shields.io/packagist/dt/klev-o/crypto-pay-api)
![GitHub release (latest by date including pre-releases)](https://img.shields.io/github/v/release/klev-o/crypto-pay-api?include_prereleases)
![Scrutinizer code quality (GitHub/Bitbucket)](https://img.shields.io/scrutinizer/quality/g/klev-o/crypto-pay-api)
![GitHub last commit](https://img.shields.io/github/last-commit/klev-o/crypto-pay-api)

## Installation

Run this command in your command line:
```php
composer require klev-o/crypto-pay-api
```

## Usage

### Common

First, you need to create an object of the CryptoPay class, passing the API key of your application to the constructor:


```php
<?php

require_once '../vendor/autoload.php';

$api = new \Klev\CryptoPayApi\CryptoPay('YOUR_APP_TOKEN');
```

> To create an app and get an api token, open [@CryptoBot](http://t.me/CryptoBot?start=pay) or [@CryptoTestnetBot](http://t.me/CryptoTestnetBot?start=pay) (for testnet), send /pay command, then go to ‘Create App’.

You can pass the second parameter $isTestnet to the CryproPay constructor to make all requests go through the testnet. This is useful for any experiments)

```php
<?php

require_once '../vendor/autoload.php';

//the second parameter true - activates the testnet
$api = new \Klev\CryptoPayApi\CryptoPay('YOUR_APP_TOKEN', true);
```
Then you can call all possible methods. To check that everything is working correctly, you can call the `$api->getMe()` method, which will return basic information about your application.

```php
<?php

require_once '../vendor/autoload.php';

//the second parameter true - activates the testnet
$api = new \Klev\CryptoPayApi\CryptoPay('YOUR_APP_TOKEN', true);

$result = $api->getMe();

print_r($result)
```

If everything works well, then you will see something like this

```php
//Display if everything is working well
Array
(
    [app_id] => 12345 //your id will be different
    [name] => Some App //your name will be different
    [payment_processing_bot_username] => CryptoBot
)
```


For more complex methods, value objects are used, for example, this is how you can create invoices:

```php
<?php

use Klev\CryptoPayApi\Methods\CreateInvoice;
use Klev\CryptoPayApi\CryptoPay;
use Klev\CryptoPayApi\Enums\PaidBtnName;

require_once '../vendor/autoload.php';

$api = new CryptoPay('YOUR_APP_TOKEN');

$data = new CreateInvoice('TON', '0.53');
$data->allow_anonymous = false;
$data->paid_btn_name = PaidBtnName::OPEN_CHANNEL;

$createdInvoice = $api->createInvoice($data)
```

As you can see above, required parameters are passed through the constructor, while optional parameters can be set directly. Also, for values that expect "one of", special enums are created, located at the namespace `Klev\CryptoPayApi\Enums`.

By the same principle, objects are created for the methods `getInvoice()` and `Transfer()`

***

More detailed examples can be found in the [demo directory](https://github.com/klev-o/crypto-pay-api/tree/main/demo)

All available methods can be viewed at the `$api` object, or refer to the [official documentation](https://telegra.ph/Crypto-Pay-API-11-25)

### Webhooks

Using a webhook, you can receive updates for your application

> Please note that the interceptor must first be activated in your application. This can be done in the following way: open [@CryptoBot](http://t.me/CryptoBot?start=pay) or [@CryptoTestnetBot](http://t.me/CryptoTestnetBot?start=pay) (for testnet), send /pay command, then go to ‘My Apps’, choose your app, open ‘Webhooks’ and click ‘Enable Webhooks’. Read more [here](https://telegra.ph/Crypto-Pay-API-11-25#Webhooks)

To receive updates, you need to use the `$api->getWebhookUpdates()` method and subscribe to listen for the necessary events. There is currently only one event (or update type) available - invoice_paid. It also has an enum - `PaidType::INVOICE_PAID`:

```php
<?php

use Klev\CryptoPayApi\Methods\CreateInvoice;
use Klev\CryptoPayApi\CryptoPay;
use Klev\CryptoPayApi\Enums\PaidType;
use Monolog\Handler\StreamHandler;
use Monolog\Logger;

require_once '../vendor/autoload.php';

//The logger does not have to be created, it is used only for an example
$log = new Logger('App');
$log->pushHandler(new StreamHandler('../var/logs/app.log'));

$api = new CryptoPay('YOUR_APP_TOKEN');

//subscribe to an event
$api->on(PaidType::INVOICE_PAID, static function(Update $update) use ($log) {
    //do something with the data
    $log->info('webhook data: ', (array)$update);
});

$api->getWebhookUpdates();
```

You can register multiple handlers for the same event.

###Advanced

If you are going to use api in some more or less large project, in a firework, etc., then it is possible to register a dependency injection container in the api object, and through it you can do any things that you may need. Let's take an example:

> Let's say we have installed `composer require php-di/php-di`, although you can use any other psr-compatible

```php
<?php

use Klev\CryptoPayApi\CryptoPay;
use Klev\CryptoPayApi\Enums\PaidType;
use Monolog\Handler\StreamHandler;
use Monolog\Logger;
use Psr\Log\LoggerInterface;

require_once '../vendor/autoload.php';

$api = new CryptoPay('YOUR_APP_TOKEN');

//Use Container builder
$builder = new DI\ContainerBuilder();

$builder->addDefinitions([
    //specify the rules on how to create an object
    LoggerInterface::class => function(\DI\Container $c) {
        $log = new Logger('App');
        $log->pushHandler(new StreamHandler('../var/logs/app.log'));
        return $log;
    },
    //specify the rules on how to create an object
    InvoicePaidListener::class => function(\DI\Container $c) use ($log) {
        return new InvoicePaidListener($c->get(LoggerInterface::class));
    }
]);
$container = $builder->build();

//register container in api
$api->setContainer($container);

//Instead of using an anonymous function, we can now use a custom class, into which,
//if necessary, we can pull everything we need (working with the database, sending by mail, etc.)
class InvoicePaidListener
{
    private Logger $log;
    
    public function __construct(Logger $log)
    {
        $this->log = $log;
    }
    public function __invoke(Update $update)
    {
        $this->log->info(PaidType::INVOICE_PAID . 'event from class: ', (array)$update);
    }

}

//Now the event subscription looks more concise
$api->on(PaidType::INVOICE_PAID, InvoicePaidListener::class);

$api->getWebhookUpdates();
```

You can use your dependency injection container to pipe all the necessary functionality from your code into handlers

## Troubleshooting

Please, if you find any errors or not exactly - report this [problem page](https://github.com/klev-o/crypto-pay-api/issues)