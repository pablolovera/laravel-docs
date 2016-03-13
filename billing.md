# Faturas

- [Introdução](#introduction)
- [Assinaturas](#subscriptions)
    - [Criadno Assinaturas](#creating-subscriptions)
    - [Verificando Status da Assinatura](#checking-subscription-status)
    - [Alterando Entre Planos de Assinaturas](#changing-plans)
    - [Quantidade da Assinatura](#subscription-quantity)
    - [Impostos da Assinatura](#subscription-taxes)
    - [Cancelando Assinatura](#cancelling-subscriptions)
    - [Retornando a Assinante](#resuming-subscriptions)
- [Manipulando Stripe Webhooks](#handling-stripe-webhooks)
    - [Falha na Assinatura](#handling-failed-subscriptions)
    - [Outros Webhooks](#handling-other-webhooks)
- [Pagamento Único](#single-charges)
- [Faturas](#invoices)
    - [Gerando Faturas em PDF](#generating-invoice-pdfs)

<a name="introduction"></a>
## Introdução

O Caixa é um pacote adicional que ofereçe serviços de assinaturas e pagamentos através de uma interface de integração com [Stripe's](https://stripe.com). Com ele é facilmente possível efetuar cobranças de assinaturas, cupons, alterar o métodos da assinatura para assinatura quantitativa, cancelamento com carência e até mesmo gerar faturas em PDF.

<a name="configuration"></a>
### Configuração

#### Composer

Primeiarmente é necessário adicionar o pacote no arquivo `composer.json` e executar o comando `composer update`

    "laravel/cashier": "~6.0"

#### Service Provider

Depois da instalação do pacote, adicionar o [service provider](/docs/{{version}}/providers) `Laravel\Cashier\CashierServiceProvider` no arquivo de configuração `app.php`.

#### Migration

Antes de usar o pacote Cashier, é necessário preparar o banco de dados, adicionando algumas colunas na tabela `users` e criar a tabela `subscriptions` para armazenar as assinaturas dos clientes:

    Schema::table('users', function ($table) {
        $table->string('stripe_id')->nullable();
        $table->string('card_brand')->nullable();
        $table->string('card_last_four')->nullable();
    });

    Schema::create('subscriptions', function ($table) {
        $table->increments('id');
        $table->integer('user_id');
        $table->string('name');
        $table->string('stripe_id');
        $table->string('stripe_plan');
        $table->integer('quantity');
        $table->timestamp('trial_ends_at')->nullable();
        $table->timestamp('ends_at')->nullable();
        $table->timestamps();
    });

Depois de criar as migrations, basta executar o comando `migrate`.

#### Configuração do Model

Adicionar o trait `Billable` no model User:

    use Laravel\Cashier\Billable;

    class User extends Authenticatable
    {
        use Billable;
    }

#### Chave Stripe

Inserir a sua chave do Stripe no arquivo de configuração `services.php`:

    'stripe' => [
        'model'  => App\User::class,
        'secret' => env('STRIPE_API_SECRET'),
    ],

<a name="subscriptions"></a>
## Assinaturas

<a name="creating-subscriptions"></a>
### Criando Assinaturas

Para criar uma a assinatura, primeiro crie uma instância do model onde foi inserido o trait Billable, geralmente é uma instância de `App\User`. Feito isso, é possível usar o método `newSubscription` para criar uma assinatura:

    $user = User::find(1);

    $user->newSubscription('main', 'monthly')->create($creditCardToken);

O primeiro parâmetro passado para o método `newSubscription` é o nome da assinatura. Se sua aplicação possui somente uma assinatura, você pode nomear de `main` ou `primary`. O segundo parâmetro é o plano de assinatura que o usuário irá aderir. Este valor deve ser o identificador do plano no Stripe.

O método `create` irá criar automaticamente uma assinatura no Stripe, bem como atualizar o banco de dados com o ID do cliente e outras informações referentes ao pagamento. Se o seu plano estiver configurado como trial no Stripe, a data final do trial será automaticamente configurada no registro do usuário.

#### Detalhes do Usuário

É possível especificar maiores detalhes sobre seus clientes, para isso, basta passar estes dados como um segundo parâmetro do método `create`:

    $user->newSubscription('main', 'monthly')->create($creditCardToken, [
        'email' => $email, 'description' => 'Our First Customer'
    ]);

Para saber quais são os campos suportados, [documentation on customer creation](https://stripe.com/docs/api#create_customer).

#### Cupons

Para aplicar cupons quando cria uma assinatura, basta usar o método `withCoupon`:

    $user->newSubscription('main', 'monthly')
         ->withCoupon('code')
         ->create($creditCardToken);

<a name="checking-subscription-status"></a>
### Verificando Status da Assinatura

Uma vez que o usuário possúi uma assinatura de sua aplicação, você pode verificar o status da assinatura deste usuário usando alguns métodos. O método `subscribed` retorna `true` quando a assinatura do usuário estiver ativa, mesmo estando no período de avaliação:

    if ($user->subscribed('main')) {
        //
    }

Com o método `subscribed` é possível ser usado para filtrar o acesso as rotas e controladores, aplicando-o no [route middleware](/docs/{{version}}/middleware):

    public function handle($request, Closure $next)
    {
        if ($request->user() && ! $request->user()->subscribed('main')) {
            // This user is not a paying customer...
            return redirect('billing');
        }

        return $next($request);
    }

Você pode verificar se o usuário ainda está no período de avaliação, usando o método `onTrial`. Assim você poderá exibir um aviso sobre isso:

    if ($user->subscription('main')->onTrial()) {
        //
    }

O método `onPlan` pode ser usando para verificar em qual plano o usuário está inscrito, baseado em sua Stripe ID:

    if ($user->onPlan('monthly')) {
        //
    }

#### Status de Assinatura Cancelada

Para verificar se o usuário já foi assinante ativo, mas cancelou a sua inscrição, você pode usar o método `cancelled`:

    if ($user->subscription('main')->cancelled()) {
        //
    }

You may also determine if a user has cancelled their subscription, but are still on their "grace period" until the subscription fully expires. For example, if a user cancels a subscription on March 5th that was originally scheduled to expire on March 10th, the user is on their "grace period" until March 10th. Note that the `subscribed` method still returns `true` during this time.

    if ($user->subscription('main')->onGracePeriod()) {
        //
    }

<a name="changing-plans"></a>
### Alterando Entre Planos de Assinaturas

Depois que o usuário estiver inscrido em sua aplicação, caso este usuário deseje alterar seu plano de assinatura para o plano `premium`, por exemplo, pode ser usado o método `swap`:

    $user = App\User::find(1);

    $user->subscription('main')->swap('stripe-plan-id');

Caso o usuário esteja no período de avaliação ou existir uma "quanditdade" de sua assinatura, ambos serão mantidos. Você poderá emitir uma fatura para este usuário ao alterar os planos, para isso, use o método `invoice`:

    $user->subscription('main')->swap('stripe-plan-id');

    $user->invoice();

<a name="subscription-quantity"></a>
### Quantidade da Assinatura

Caso queira usar "quantidade" para a assinatura, quando, por exemplo, sua aplicação faz uma cobranaça mensal por usuário de $10. Você pode alterar as quantidades usando os métodos `incrementQuantity` e `decrementQuantity`:

    $user = User::find(1);

    $user->subscription('main')->incrementQuantity();

    // Add five to the subscription's current quantity...
    $user->subscription('main')->incrementQuantity(5);

    $user->subscription('main')->decrementQuantity();

    // Subtract five to the subscription's current quantity...
    $user->subscription('main')->decrementQuantity(5);

Também, você pode usar uma quantidade específica, usando o método `updateQuantity`:

    $user->subscription('main')->updateQuantity(10);

Para mais informações sobre quantidades de assinatura, consulte [Stripe documentation](https://stripe.com/docs/guides/subscriptions#setting-quantities).

<a name="subscription-taxes"></a>
### Impostos da Assinatura

Com o Cashier é fácil informar o percentual de tributos `tax_percent` a ser enviado  ao Stripe. Para especificar o percentual de impostos que um usuário paga em uma assinatura, basta implementar o método `taxPercentage`, e retornar um valor entre 0 e 100 com no máximo 2 casas decimais.

    public function taxPercentage() {
        return 20;
    }

Isso lhe permite aplicar taxas tributárias baseado em model-by-model, que pode ser útil quando os usuário estão localizados me outros países.

<a name="cancelling-subscriptions"></a>
### Cancelando Assinatura

Para cancelar uma assinatura, basta usar o método `cancel` na assinatura do usuário:

    $user->subscription('main')->cancel();

Quando uma assinatura é cancelada o Cashier, automáticamente grava a data de cancelamento na coluna `ends_at` na tabela do banco de dados. Esta coluna é usada pelo método `subscribed` para saber quando retornar `false`. Por exemplo, se o usuário cancelar a sua assinatura no dia 1º de Março, e a assinatura não está configurada para ser encerrada antes do dia 5 de Março, o método `subscribed` continuará retornando `true` até o dia 5 de Março.

Você pode verificar se um usuário cancelou a assinatura enquanto estava no período de avaliação usando o método `onGracePeriod`:

    if ($user->subscription('main')->onGracePeriod()) {
        //
    }

<a name="resuming-subscriptions"></a>
### Retornando a Assinante

Se o usuário cancelou sua assinatura, e ele desejar retornar a ser assinante, basta usar o método `resume`. O usuário **must** ainda estará em seu período de avaliação:

    $user->subscription('main')->resume();

Se um usuário cancelar a sua assinatura e, em seguida, retornar a ser assinante antes de ter expirado a assinatura, apenas irá ocorrer uma reativação da assinatura e a cobrança ocorrerá normalmente como se nada estivesse ocorrido.

<a name="handling-stripe-webhooks"></a>
## Manipulando Stripe Webhooks

<a name="handling-failed-subscriptions"></a>
### Falha na Assinatura

E se o cartão de crédito do cliente expirar? Não se preocupe, o Cashier possui um controlador Webhook que pode efetuar o cancelamento da assinatura do cliente para você, Basta criar a rota para o controlador:

    Route::post(
        'stripe/webhook',
        '\Laravel\Cashier\Http\Controllers\WebhookController@handleWebhook'
    );

Isso mesmo! As falhas de pagamentos serao capturadas pelo controlador. O controlador irá cancelar a assinatura do usuário quando o Stripe considerar o pagamento não efetuado, normalmente, após 3 tentativas de cobrança. Não esqueça de configurar o webhook URI nas configurações em seu painel de controle do Stripe.

O Stripe webhooks deve ser ignorado em [CSRF verification](/docs/{{version}}/routing#csrf-protection), certifique-se de adicionar a URI na lista de exceções no middleware `VerifyCsrfToken`:

    protected $except = [
        'stripe/*',
    ];

<a name="handling-other-webhooks"></a>
### Outros Webhooks

Se você deseja controlar eventos adicionais, basta extender Webhook controller. Seu método deve corresponder a conveção do Cashier, o nome do método que você quer manipular deve conter o prefixo `handle` e em "camel case" seguidos do nome do médoto em Stripe webhook. Por exemplo, se você desejar manipular o webhook `invoice.payment_succeeded` você precisará adicionar o método `handleInvoicePaymentSucceeded` no seu controlador.

    <?php

    namespace App\Http\Controllers;

    use Laravel\Cashier\Http\Controllers\WebhookController as BaseController;

    class WebhookController extends BaseController
    {
        /**
         * Handle a stripe webhook.
         *
         * @param  array  $payload
         * @return Response
         */
        public function handleInvoicePaymentSucceeded($payload)
        {
            // Handle The Event
        }
    }

<a name="single-charges"></a>
## Pagamento Único

Você pode possibilitar ao usuário um pagamento único de sua assinatura, para isso, use o método `charge` em uma instância do model que possui o trait billable. O método `charge` aceita qualquer valor que você desejar, sendo ele no **menor denominador da moeda utilizada em sua aplicação. O exemplo a seguir irá cobrar 100 centavos, ou $ 1,00 do cartão de crédito do usuário:

    $user->charge(100);

O método `charge` aceita um array como segundo parâmetro, possibilidando outras opções ao efetuar uma única cobrança pelo Stripe:

    $user->charge(100, [
        'source' => $token,
        'receipt_email' => $user->email,
    ]);

O método `charge` retornará `false` caso a cobrança for negada:

    if ( ! $user->charge(100)) {
        // The charge was denied...
    }

Se o pagamento for bem sucedido, o valor total será retornado.

<a name="invoices"></a>
## Faturas

É possível lisar as faturas do usuário através do método `invoices`:

    $invoices = $user->invoices();

Na lista das faturas do usuário, você user métodos auxiliares para exibir informações relevantes a cada fatura. Por exemplo, você pode listar as faturas em uma tabela, permitindo que o usuário faça o download de cada fatura:

    <table>
        @foreach ($invoices as $invoice)
            <tr>
                <td>{{ $invoice->date()->toFormattedDateString() }}</td>
                <td>{{ $invoice->total() }}</td>
                <td><a href="/user/invoice/{{ $invoice->id }}">Download</a></td>
            </tr>
        @endforeach
    </table>

<a name="generating-invoice-pdfs"></a>
#### Gerando Faturas em PDF

Antes de gerar faturas em PDF, é perciso instalar a biblioteca do PHP `dompdf`:

    composer require dompdf/dompdf

Em uma rota ou controlador, use o método `downloadInvoice` para gerar o PDF de uma fatura e disponibilizar para download. Este método gerará automáticamente uma resposta HTTP enviada ao navegador para efetuar o download:

    Route::get('user/invoice/{invoice}', function ($invoiceId) {
        return Auth::user()->downloadInvoice($invoiceId, [
            'vendor'  => 'Your Company',
            'product' => 'Your Product',
        ]);
    });

