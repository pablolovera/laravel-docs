# Service Providers

- [Introdução](#introduction)
- [Criando Um Service Provider](#writing-service-providers)
    - [O Método Register](#the-register-method)
    - [O Método Boot](#the-boot-method)
- [Registrando Providers](#registering-providers)
- [Deferred Providers](#deferred-providers)

<a name="introduction"></a>
## Introdução

Os Service providers são o centro de qualquer aplicação Laravel. A sua aplicação bem como em todos os serviços do núcleo do Laravel são iniciados via service providers.

Mas, o que quer dizer "iniciado"? Em geral, nós **registramos** as coisas, incluíndo bindings no container de serviços, escutas de eventos, middlewares e rotas. Os Service providers são a peça chave para configurar a sua aplicação.

Se você abrir o arquivo `config/app.php` incluso no Laravel, você poderá visualizar o array `providers`. Estas são todas as classe de service provider que serão carregadas pela sua aplicação. Entretanto, muitos desses são "deferred" providers, eles não são carregados em toda requisição, mas sim quando os serviços que eles provém são utilizados.

Vamos aprender agora como criar seus próprios service providers e registrá-los na sua aplicação Laravel.

<a name="writing-service-providers"></a>
## Criando Service Providers

Todos os service providers estendem a classe `Illuminate\Support\ServiceProvider`. Esta classe abstrata requer que você implemente um único método no seu provider: `register`. Dentro do método `register`  você pode  **criar bindings no [container de serviços](/docs/{{version}}/container)**. Você nunca deve tentar registrar qualquer escuta de evento, rotas, ou qualquer outra funcionalidade dentro do método `register`.

O CLI do Artisan cria um novo provider através do comando `make:provider`:

    php artisan make:provider RiakServiceProvider

<a name="the-register-method"></a>
### O Método Register

Como mencionado anteriormente, dentro do método `register` você pode criar binds no [container de serviços](/docs/{{version}}/container). Você nunca deve tentar registrar qualquer escuta de evento, rotas, ou qualquer outra funcionalidade dentro do método `register`. Caso contrário você pode acidentalmente utilizar um serviço fornecido por um service provider que não foi carregado corretamente.

Agora vamos dar uma olha em um service provider básico:

    <?php

    namespace App\Providers;

    use Riak\Connection;
    use Illuminate\Support\ServiceProvider;

    class RiakServiceProvider extends ServiceProvider
    {
        /**
         * Register bindings in the container.
         *
         * @return void
         */
        public function register()
        {
            $this->app->singleton('Riak\Contracts\Connection', function ($app) {
                return new Connection(config('riak'));
            });
        }
    }

Este service provider define um  método `register` e define uma implementação para `Riak\Contracts\Connection` no container de serviços. Se você ainda não entendeu como um service provider funciona, dê uma olhada [nesta documentação](/docs/{{version}}/container).

<a name="the-boot-method"></a>
### O Método Boot

O que você precisa para definir uma view composer no seu service provider? Simplesmente de um método `boot`. **Este método é chamado depois que todos os service providers tiverem sido registrados**, possibilitando que você tenha acesso a todos os serviços registrados pelo framework:

    <?php

    namespace App\Providers;

    use Illuminate\Support\ServiceProvider;

    class EventServiceProvider extends ServiceProvider
    {
        /**
         * Perform post-registration booting of services.
         *
         * @return void
         */
        public function boot()
        {
            view()->composer('view', function () {
                //
            });
        }

        /**
         * Register bindings in the container.
         *
         * @return void
         */
        public function register()
        {
            //
        }
    }

#### Injeção De Dependências No Método Boot

Nós podemos injetar dependências através de type-hint no método `boot`. O [container de serviços](/docs/{{version}}/container) irá injetar automaticamente qualquer dependência que você precise:

    use Illuminate\Contracts\Routing\ResponseFactory;

    public function boot(ResponseFactory $factory)
    {
        $factory->macro('caps', function ($value) {
            //
        });
    }

<a name="registering-providers"></a>
## Registrando Providers

Todos os service providers são registrados no arquivo `config/app.php`. Este arquivo contém um array `providers` onde você pode listar os nomes dos seus service providers. Por padrão, alguns providers do core do Laravel estão incluídos neste array. Estes providers iniciam o core dos componentes do Laravel, como o mail, filas, cache, entre outros.

Para registrar o seu provider, Simplesmente adicione-o ao array:

    'providers' => [
        // Other Service Providers

        App\Providers\AppServiceProvider::class,
    ],

<a name="deferred-providers"></a>
## Deferred Providers

Se o seu provider **somente** registrar bindings no [container de serviços](/docs/{{version}}/container), você pode escolher atrasar o seu registro até algum dos seus bindings serem realmente necessários. Atrasando o carregamento de um provider aumentará a performance da sua aplicação, pois ele não será carregado dos arquivos em cada requisição.

Para atrasar o carregamento de um provider, inicialize a propriedade `defer` como `true` e crie um método chamado `provides`. Este método retorna os bindings do service container que o provider registra:

    <?php

    namespace App\Providers;

    use Riak\Connection;
    use Illuminate\Support\ServiceProvider;

    class RiakServiceProvider extends ServiceProvider
    {
        /**
         * Indicates if loading of the provider is deferred.
         *
         * @var bool
         */
        protected $defer = true;

        /**
         * Register the service provider.
         *
         * @return void
         */
        public function register()
        {
            $this->app->singleton('Riak\Contracts\Connection', function ($app) {
                return new Connection($app['config']['riak']);
            });
        }

        /**
         * Get the services provided by the provider.
         *
         * @return array
         */
        public function provides()
        {
            return ['Riak\Contracts\Connection'];
        }

    }

O Laravel compila e salva uma lista de todos os serviços atrasados por service providers, juntamente com o nome da classe do service provider. Assim, quando você tentar resolver algum dos bindings destes serviços o Laravel carregará o service provider.
