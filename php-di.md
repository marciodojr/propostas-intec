# Injeção de dependências

Autor: Márcio Dias

## Tags

- Backend
- DI
- Pimple
- PHP-DI


## Introdução

Com a consolidação da [PSR-11](https://www.php-fig.org/psr/psr-11/), os frameworks passaram fornecer ou permitir o uso de containers de injeção de dependências como modo padrão de criação de objetos no fluxo `requisição -> resposta`. No Slim o container utilizado é o [Pimple](https://pimple.symfony.com/). O Pimple é um container de injeção de dependências bastante simples, seu uso no Slim ocorre no arquivo `config/dependencies.php`, ver [Figura 1](#fig1).

Ao utilizar o Pimple o desenvolvedor tem os seguintes benefícios:

- Simplificação e eliminação de repetição de código na criação de objetos
- Centralização da árvore de dependências da aplicação
- Redução de erros, já que o único responsável por manipular o container é o Slim

<sup id="fig1"></sup>
```php
<?php

// ...

$c = $app->getContainer();

// ----------------------------------------- Base

$c[PDO::class] = function ($c) {
    $db = $c['settings']['db'];
    return new PDO(
        'mysql:host='.$db['host'].';dbname='.$db['db_name'].';charset=' . $db['charset'],
        $db['db_user'],
        $db['db_pass'],
        [
            PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION,
            PDO::ATTR_PERSISTENT => false,
            PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC
        ]
    );
};

$c[Pheanstalk::class] = function ($c) {
    $settings = $c['settings']['pheanstalk'];
    return new Pheanstalk($settings['host'], $settings['port']);
};

$c['logger'] = function ($c) {
    $settings = $c->get('settings')['logger'];
    $logger = new Monolog\Logger($settings['name']);
    $logger->pushProcessor(new Monolog\Processor\UidProcessor());
    $logger->pushHandler(
        new Monolog\Handler\StreamHandler(
            $settings['path'],
            $settings['level']
        )
    );
    return $logger;
};

$c[Mailgun::class] = function ($c) {
    $apiKey = $c['settings']['mailgun']['api_key'];
    return new Mailgun($apiKey);
};

// ----------------------------------------- Entity

$c[UserEntity::class] = function ($c) {
    $db = $c[DbHandler::class];
    return new UserEntity($db);
};

$c[ConfirmationEntity::class] = function ($c) {
    $db = $c[DbHandler::class];
    return new ConfirmationEntity($db);
};

// continuação  do arquivo

```
<center><small> Figura 1. Exemplo de utilização do Pimple.</small></center>


No entanto, alguns pontos negativos são detectados:

1. O arquivo de dependências aumenta de tamanho rapidamente.
2. Caso a dependencia seja algo elementar, por exemplo, variável de ambiente ou valor escalar é necessário salvar nos arquivos `config/settings{.local,}.php`
2. A construção de classes e a interrupção para registro de dependências é um pouco desagradável
3. A alteração de dependências das classes e posterior atualização do registro de dependências também é desagradável
4. Ao mesclar o código de vários desenvolvedores existe a possibilidade de conflito

É importante mencionar que os pontos negativos, apesar de relevantes, são compensados pelos pontos positivos. Na tentativa de reduzir os pontos negativos é proposto a substituição do Pimple pelo [PHP-DI](http://php-di.org/). O PHP-DI, dentre outras coisas, permite resolução de dependências via *autowiring*, eliminando necessidade de declaração explícita pelo desenvolvedor.

A substituição deve produzir os seguintes resultados:

1. Eliminação do arquivo `config/dependencies.php` e redução de possibilidade de conflito (ainda poderá ocorrer conflitos no arquivo `config/settings.php`).
2. Eliminação da necessidade de registro (e atualização) de dependências em casos onde as dependências são objetos (o que representa a grande maioria dos casos).



## Exemplo

Faremos a conversão da definição de dependências do Pimple para o PHP-DI e discutiremos as diferenças a medida que o exemplo for construído. Ao final do exemplo, será feita a comparação de ambos.

### Instalação

**Pimple**

Como o Pimple já vem como padrão no Slim, seu uso é imediato. Para utilizá-lo basta instanciar a classe `Slim\App` e chamar o método `getContainer()` ([Figura 2](#fig2)). Com o Pimple dois tipos de arquivo são utilizados: `config/settings{.local,}.php` e `config/dependencies.php`.

<sup id="fig2"></sup>
```php
<?php
// public/index.php
// ...

$app = new \Slim\App([
    'settings' => $settings
]);
// ...
```
```php
<?php
// config/dependencies.php
// ...
$c = $app->getContainer();
```
<center><small>Figura 2. Configuração da aplicação com o Pimple.</small></center>

**PHP-DI**

Para utilizar o PHP-DI é necessário instalar o pacote `php-di/slim-bridge` via composer e por conveniência, é aconselhada a criação de uma nova classe `App` ([Figura 3](#fig3)). O arquivo `config/dependencies.php` deixa de ser necessário e o método `getContainer()`, apesar de ainda existir, não é mais utilizado. O único ponto de definição de dependências são os arquivos `config/settings{.local,}.php`

<sup id="fig3"></sup>
```php
<?php
// public/index.php
// ...

$app = new \IntecPhp\App($settings);
// ...
```
```php
<?php
// src/App.php

namespace IntecPhp;

use DI\Bridge\Slim\App as DIBridgeApp;
use DI\ContainerBuilder;

class App extends DIBridgeApp
{
    /**
     * PHP-DI container definition
     *
     * @var array
     */
    private $containerDefinition;

    public function __construct(array $containerDefinition)
    {
        $this->containerDefinition = $containerDefinition;
        parent::__construct();
    }

    /**
     * {@inheritDoc}
     */
    protected function configureContainer(ContainerBuilder $builder)
    {
        $builder->addDefinitions($this->containerDefinition);
    }
}
```
<center><small>Figura 3. Configuração da aplicação com o PHP-DI.</small></center>

### Tipos de dependências

Para auxiliar no entendimento do uso do PHP-DI e na forma como as dependencias são organizados no projeto, classificaremos as dependências da seguinte forma:

- Variáveis de ambiente. Ex: senha do banco de dados e flag para ativação do log
- Tipos primitivos (`int`, `array`, `string`, ...). Ex: Tempo de duração do cookie/sessão, nível de log e caminho para salvar arquivos
- Objetos Homogêneos. Objetos que possuem somento objetos como dependências.
- Objetos Heterogênios. Objetos que possuem objetos e outros tipos de dependencias.
- funções anônimas complexas. Caso onde a criação da dependência envolve algum tipo de configuração. Ex: Criação da instancia de conexão com com banco de dados.

A [Figura 4](#fig4) apresenta os tipos de dependências.


<sup id="fig4"></sup>
```php
<?php
// config/settings.php
return [
    'displayErrorDetails' => false,
// ...
```
<center><small>Figura 4a. Exemplo de dependência de tipo primitivo.</small></center>

```php
<?php
// config/settings.php
return [
    'db' => [
        'host' => getenv('DB_HOST'),
// ...
```
<center><small>Figura 4b. Exemplo de dependência de variável de ambiente.</small></center>

```php
<?php
// config/dependencies.php
//...
$c[Price::class] = function ($c) {
    $price = $c[PriceEntity::class];
    $pricePostBrasil = $c[PricePostGraduationBrasil::class];
    $pricePostAbroad = $c[PricePostGraduationAbroad::class];
    return new Price($price, $pricePostBrasil, $pricePostAbroad);
};
//...

```
<center><small>Figura 4c. Exemplo de dependência de objeto homogênio.</small></center>

```php
<?php
// config/dependencies.php
//...
// valor escalar como dependência
$c[Mailgun::class] = function ($c) {
    $apiKey = $c['settings']['mailgun']['api_key'];
    return new Mailgun($apiKey);
};

// objeto e valor escalar como dependência
$c['errorHandler'] = $c['phpErrorHandler'] = function ($c) {
    $errorDetails = $c->get('settings')['displayErrorDetails'];
    $logger = $c->get('logger');
    return new \IntecPhp\Handler\PhpError($errorDetails, $logger);
};
//...
```
<center><small>Figura 4d. Exemplo de dependência de objeto heterogênio.</small></center>

```php
<?php
// config/dependencies.php
//...
$c[PDO::class] = function ($c) {
    $db = $c['settings']['db'];
    return new PDO(
        'mysql:host=' . $db['host'] . ';dbname=' . $db['db_name'] . ';charset=' . $db['charset'],
        $db['db_user'],
        $db['db_pass'],
        [
            PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION,
            PDO::ATTR_PERSISTENT => false,
            PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC
        ]
    );
};

$c['logger'] = function ($c) {
    $settings = $c->get('settings')['logger'];
    $logger = new Monolog\Logger($settings['name']);
    $logger->pushProcessor(new Monolog\Processor\UidProcessor());
    $logger->pushHandler(
        new Monolog\Handler\StreamHandler(
            $settings['path'],
            $settings['level']
        )
    );
    return $logger;
};
//...
```
<center><small>Figura 4e. Exemplo de dependência de funções anônimas.</small></center>

### Objetivo

Através da utilização do PHP-DI é esperado que todas as definições de dependências do tipo objeto homogênio sejam eliminadas e as definições de objeto heterogênio sejam simplificadas. Isso é possível através do recurso de [autowiring](http://php-di.org/doc/autowiring.html).

O PHP-DI utiliza [*Reflection*](http://php-di.org/doc/getting-started.html#3-create-the-objects) e [*Annotations*](http://php-di.org/doc/annotations.html) para identificar dependências. A injeção de dependências via *Annotations* por padrão é desabilitada.

### Efeitos Colaterais

Ao integrar o PHP-DI na aplicação, os seguintes efeitos colaterais são observados:

1. O array retornado pelos arquivos `config/settings{.local,}.php` passa a ser construido de modo "plano" para otimizar a injeção de dependências ([Figura 5](#fig5)).

<sup id="fig5"></sup>
```php
<?php
// config/settings.php

return [
    'displayErrorDetails' => false, // slim
    'addContentLengthHeader' => false, // slim
    'logger' => [ // slim log
        'name' => 'AppLog',
        'path' => __DIR__ . '/../logs/app.log',
        'level' => \Monolog\Logger::DEBUG,
    ],
    'db' => [
        'host' => getenv('DB_HOST'),
        'db_name' => getenv('DB_NAME'),
        'db_user' => getenv('DB_USER'),
        'db_pass' => getenv('DB_PASS'),
        'db_port' => getenv('DB_PORT'),
        'charset' => 'utf8mb4'
    ],
    'jwt' => [
        'app_secret' => getenv('APP_SECRET'),
        'token_expires' => 1800000 // 5h
    ],
    //..
```

```php
<?php
// config/settings.php

use PDO;
use function DI\env;
use function DI\string;

return [
    'settings.displayErrorDetails' => false, // slim
    'settings.addContentLengthHeader' => false, // slim
    'logger.name' => 'AppLog',
    'logger.path' => __DIR__ . '/../logs/app.log',
    'logger.level' => \Monolog\Logger::DEBUG,
    'db.host' => env('DB_HOST'),
    'db.db_name' => env('DB_NAME'),
    'db.db_user' => env('DB_USER'),
    'db.db_pass' => env('DB_PASS'),
    'db.db_port' => env('DB_PORT'),
    'db.charset' => 'utf8mb4',
    'jwt.app_secret' => env('APP_SECRET'),
    'jwt.token_expires' => 1800000,
    'pdo.dsn' => string('mysql:host={db.host};port={db.db_port};dbname={db.db_name};charset={db.charset}'),
    'pdo.options' => [
        PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION,
        PDO::ATTR_PERSISTENT => false,
        PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC
    ],
    //..
```
<center><small>Figura 5. Planificação das configurações.</small></center>

2. Como as dependências do tipo objeto heterogênio e funções anônimas serão movidas para o arquivo `config/settings.php`, é esperado um aumento significativo em seu tamanho inicial e futuro, no entanto, o crescimento ao longo do tempo deve ser significativamente menor do que o crescimento atual do arquivo `config/dependencies` ([Figura 6](#fig6)).

<sup id="fig6"></sup>
```php
<?php
// config/dependencies.php
// forma atual
//...
$c[PDO::class] = function ($c) {
    $db = $c['settings']['db'];
    return new PDO(
        'mysql:host=' . $db['host'] . ';dbname=' . $db['db_name'] . ';charset=' . $db['charset'],
        $db['db_user'],
        $db['db_pass'],
        [
            PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION,
            PDO::ATTR_PERSISTENT => false,
            PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC
        ]
    );
};

$c['logger'] = function ($c) {
    $settings = $c->get('settings')['logger'];
    $logger = new Monolog\Logger($settings['name']);
    $logger->pushProcessor(new Monolog\Processor\UidProcessor());
    $logger->pushHandler(
        new Monolog\Handler\StreamHandler(
            $settings['path'],
            $settings['level']
        )
    );
    return $logger;
};
//...
```

```php
<?php
// config/settings.php
// nova forma
use PDO;
use function DI\env;
use function DI\get;
use function DI\factory;
use function DI\create;

return [
    // env e tipos primários
    // ...
    'logger.name' => 'AppLog',
    'logger.path' => __DIR__ . '/../logs/app.log',
    'logger.level' => \Monolog\Logger::DEBUG,
    'db.host' => env('DB_HOST'),
    'db.db_name' => env('DB_NAME'),
    'db.db_user' => env('DB_USER'),
    'db.db_pass' => env('DB_PASS'),
    'db.db_port' => env('DB_PORT'),
    'db.charset' => 'utf8mb4',
    'pdo.dsn' => string('mysql:host={db.host};port={db.db_port};dbname={db.db_name};charset={db.charset}'),
    'pdo.options' => [
        PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION,
        PDO::ATTR_PERSISTENT => false,
        PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC
    ],
    // objetos e funções anônimas
    // função anônima
    'logger' => factory(function ($c) {
        $logger = new \Monolog\Logger($c->get('logger.name'));
        $logger->pushProcessor(new \Monolog\Processor\UidProcessor());
        $logger->pushHandler(new \Monolog\Handler\StreamHandler(
            $c->get('logger.path'),
            $c->get('logger.level')
        ));
        return $logger;
    }),
    // objeto heterogênio
    PDO::class => create()
        ->constructor(
            get('pdo.dsn'),
            get('db.db_user'),
            get('db.db_pass'),
            get('pdo.options')
        )
```
<center><small>Figura 6. Exemplo de definição de objetos heterogênios e funções anônimas.</small></center>

3. A injeção de argumentos de rotas deve ser direta ao invés de utilizar um array de argumentos ([Figura 7](#fig7)).


<sup id="fig7"></sup>
```php
<?php
// config/routes.php
// rota: /virtual-domain/10 ---> $args['domainId'] possui o valor 10
$app->post('/virtual-domain/{domainId}', function ($request, $response, $args) {

});
```

```php
<?php
// config/routes.php
// rota: /virtual-domain/10 ---> $domainId possui o valor 10
$app->post('/virtual-domain/{domainId}', function ($request, $response, $domainId) {

});
```
<center><small>Figura 7. Injeção direta de argumentos de rota.</small></center>

4. Elimina-se a possibilidade de uso de dependências no estilo do Slim (invocação de métodos não estáticos através da notação *MyClass:method*). O uso deve ser via método `__invoke` apenas, conforme ([Figura 8](#fig8)).

<sup id="fig8"></sup>
```php
<?php
// config/routes.php
// o método login da classe Login será chamado
$app->post('/login', Login::class . ':login');
```

```php
<?php
// config/routes.php
// o método __invoke da classe Login será chamado
$app->post('/login', Login::class);
```
<center><small>Figura 8. Restrição de uso da notação *MyClass:method*.</small></center>

5. Elimina-se a possibilidade do uso de parâmetros sem tipo e com nome arbitrário na invocação de rotas (funções anônimas e __invoke em actions, middlewares e handlers). A [Figura 9](#fig9) ilustra as diferenças.

<sup id="fig9"></sup>
```php
<?php

// modo atual
public function __invoke($req, $resp)
{

}
```

```php
// nova forma: opção 1
// (nome igual ao nome da classe)
public function __invoke($request, $response)
{

}

// nova forma: opção 2
// especificação do tipo e utilização de nome arbitrário)
public function __invoke(Request $req, Response $resp)
{

}
```
<center><small>Figura 9. Restrição de nomes de parâmetros sem tipo definido.</small></center>

6. Permite o uso de parâmetros em ordem arbitrária, ver [Figura 10](#fig10).

<sup id="fig10"></sup>
```php
<?php

// modo atual: restringe a ordem
public function __invoke($req, $resp)
{

}
```

```php
// nova forma: ordem normal
public function __invoke(Request $req, Response $resp)
{

}
// nova forma: ordem trocada
public function __invoke(Response $resp, Request $req)
{

}
// nova forma: uso parcial
public function __invoke(Response $resp)
{

}

// nova forma: uso parcial com argumentos de rota /{name}
public function __invoke($name, Response $resp)
{

}
```
<center><small>Figura 10. Relaxamento da ordem de parâmetros.</small></center>

Para uma descrição completa da integração do PHP-DI e Slim consulte a [documentação](http://php-di.org/doc/frameworks/slim.html).

7. Nos raros casos onde é necessário manipular as dependências do container (por exemplo,testes), o uso deve ser via PSR-11 (uso dos métodos `get` e `set` ao invés de índice `[]`) conforme [Figura 11](#fig11).

<sup id="fig11"></sup>
```php
<?php
// forma atual

$c['key'] = function($c) {

}

$keyInstance = $c['key'];

```

```php
<?php
// forma atual

$c->set('key', function($c) {

});

$keyInstance = $c->get('key');

```
<center><small>Figura 11. Restrição de uso da notação de índice.</small></center>