# Mail

- [Introdução](#introduction)
- [Enviando Email](#sending-mail)
    - [Anexos](#attachments)
    - [Anexos Na View](#inline-attachments)
    - [Usando Fila De Email](#queueing-mail)
- [E-mail e Desenvolvimento Local](#mail-and-local-development)
- [Eventos](#events)

<a name="introduction"></a>
## Introdução

Laravel provê uma API simples e fácil desenvolvida sobre a biblioteca [SwiftMailer](http://swiftmailer.org). O Laravel disponibiliza drivers para SMTP, Mailgun, Mandrill, Amazon SES, função `mail` do PHP e `sendmail`, possibilitando que você rapidamente aprenda a enviar emails localmente ou em um serviço baseado na nuvem de sua escolha.

### Pré-requisitos de Drivers

Drivers baseados em API como o Mailgun e Mandrill são mais simples e rápidos que servidores SMTP. Todos os drivers de API requerem a biblioteca Guzzle HTTP instalada na sua aplicação. Você pode instalar o Guzzle no seu projeto adicionando a seguinte linha ao seu arquivo `composer.json`:

    "guzzlehttp/guzzle": "~5.3|~6.0"

#### Driver Mailgun

Para utilizar o driver Mailgun, primeiramente instale o Guzzle, e altere a opção `driver` no seu arquivo `config/mail.php` para `mailgun`. Após, verifique se o arquivo `config/services.php` contém as seguintes opções:

    'mailgun' => [
        'domain' => 'your-mailgun-domain',
        'secret' => 'your-mailgun-key',
    ],

#### Driver Mandrill

Para utilizar o driver Mandrill, instale o Guzzle e altere a opção `driver` do seu arquivo `config/mail.php` para `mandrill`. Após, verifique se o arquivo `config/services.php` contém as seguintes opções:

    'mandrill' => [
        'secret' => 'your-mandrill-key',
    ],

#### Driver SES

Para utilizar o driver Amazon SES, instale o Amazon AWS SDK para PHP. Você pode instalar essa biblioteca adicionando a seguinte linha no seu arquivo `composer.json` na seção `require`:

    "aws/aws-sdk-php": "~3.0"

Altere a opção `driver` do seu arquivo `config/mail.php` para `ses` e verifique se o arquivo `config/services.php` contém as seguintes opções:

    'ses' => [
        'key' => 'your-ses-key',
        'secret' => 'your-ses-secret',
        'region' => 'ses-region',  // e.g. us-east-1
    ],

<a name="sending-mail"></a>
## Enviando E-mail

O Laravel permite que você armazene suas mensagens de email em [views](/docs/{{version}}/views). Por exemplo, para organizar seus e-mails, você pode criar um diretório `emails` dentro da pasta `resources/views`:

Para enviar uma mensagem, use o método `send` do [facade](/docs/{{version}}/facades) `Mail`. O método `send` aceita três argumentos. O primeiro é o nome da [view](/docs/{{version}}/views) que contém a mensagem, o segundo um array de informações que você deseja passar para a view e por último uma `Closure` que recebe uma instância da mensagem, possibilitando que você customize os destinatários, assunto, entre outros aspectos de uma mensagem de e-mail:

    <?php

    namespace App\Http\Controllers;

    use Mail;
    use App\User;
    use Illuminate\Http\Request;
    use App\Http\Controllers\Controller;

    class UserController extends Controller
    {
        /**
         * Send an e-mail reminder to the user.
         *
         * @param  Request  $request
         * @param  int  $id
         * @return Response
         */
        public function sendEmailReminder(Request $request, $id)
        {
            $user = User::findOrFail($id);

            Mail::send('emails.reminder', ['user' => $user], function ($m) use ($user) {
                $m->from('hello@app.com', 'Your Application');

                $m->to($user->email, $user->name)->subject('Your Reminder!');
            });
        }
    }

Se nós passarmos um array `user` como no exemplo acima, nós podemos mostrar o nome e o email na view usando o seguinte código PHP:

    <?php echo $user->name; ?>

> **Nota:** A variável `$message` é automaticamente passada para a view do email, possibilitando [anexos](#attachments)  diretamente no corpo da mensagem. Tenha cuidado para não passar uma variável `message` para sua view.

#### Construindo O E-mail

Como falamos previamente, o terceiro argumento para o método `send` é uma `Closure` que possibilita customizar várias opções na mensagem do e-mail. Utilizando essa Closure você pode especificar vários atributos, como cópias, cópias ocultas, etc:

    Mail::send('emails.welcome', $data, function ($message) {
        $message->from('us@example.com', 'Laravel');

        $message->to('foo@example.com')->cc('bar@example.com');
    });

Abaixo você pode conferir uma lista dos métodos disponíveis utilizando a instância de `$message`:

    $message->from($address, $name = null);
    $message->sender($address, $name = null);
    $message->to($address, $name = null);
    $message->cc($address, $name = null);
    $message->bcc($address, $name = null);
    $message->replyTo($address, $name = null);
    $message->subject($subject);
    $message->priority($level);
    $message->attach($pathToFile, array $options = []);

    // Attach a file from a raw $data string...
    $message->attachData($data, $name, array $options = []);

    // Get the underlying SwiftMailer message instance...
    $message->getSwiftMessage();

> **Nota:** A instância de mensagem passada para a Closure em `Mail::send` estende a class SwiftMailer, possibilitando que você chame qualquer método desta classe para construir suas mensagens de e-mail.

#### Enviando E-mails Com Views De Texto Simples

Por padrão, a view informada no método `send` é tratada como conteúdo HTML. Entretanto, passando um array como primeiro argumento no método `send`, você pode especificar uma view com texto puro para ser enviada ao invés de uma view HTML:

    Mail::send(['html.view', 'text.view'], $data, $callback);

Se você deseja enviar um texto simples, sem view, passe a key `text` no array:

    Mail::send(['text' => 'view'], $data, $callback);

#### Enviando E-mails Como Texto Puro

Você pode utilizar o método `raw` para enviar um email com uma string de texto simples:

    Mail::raw('Text to e-mail', function ($message) {
        //
    });

<a name="attachments"></a>
### Anexos

Para adicionar anexos a um e-mail utilize o método `attach` do objeto `$message` passado na CLosure. O método `attach` aceita o caminho completo do arquivo como primeiro argumento:

    Mail::send('emails.welcome', $data, function ($message) {
        //

        $message->attach($pathToFile);
    });

Ao adicionar anexos você pode especificar o nome de exibição / ou o tipo MIME passando um `array` como segundo argumento do método `attach`:

    $message->attach($pathToFile, ['as' => $display, 'mime' => $mime]);

<a name="inline-attachments"></a>
### Anexos Na Mensagem

#### Incluindo Uma Imagem Na Sua View De E-mail

Incluir imagens nos seus e-mails é normalmente complicado, entretanto o Laravel provê uma forma simples e prática de realizar essa tarefa. Para incluir uma imagem utilize o método `embed` da variável `$message` dentro da view do seu e-mail. Lembre-se, o Laravel automaticamente torna a variável `$message` disponível para todas as suas views de e-mail:

    <body>
        Here is an image:

        <img src="<?php echo $message->embed($pathToFile); ?>">
    </body>

#### Incluindo Arquivos Brutos

Se você já possui uma string de de dados que deseja incluir na mensagem de e-mail, você pode utilizar o método `embedData` na variável `$message`:

    <body>
        Here is an image from raw data:

        <img src="<?php echo $message->embedData($data, $name); ?>">
    </body>

<a name="queueing-mail"></a>
### Utilizando Filas de E-mail

#### Colocando Uma Mensagem de E-mail Em Fila

Enviar mensagens de e-mail podem diminuir drasticamente o tempo de resposta da sua aplicação. Muitos desenvolvedores optam por colocar os e-mails numa fila para serem enviadas em background. O Laravel torna essa tarefa fácil utilizando a [API de Filas](/docs/{{version}}/queues) interna. Para colocar um e-mail em uma fila, utilize o método `queue` do facade `Mail`:

    Mail::queue('emails.welcome', $data, function ($message) {
        //
    });

Este método automaticamente criará uma nova tarefa na fila para enviar sua mensagem em background. Não esqueça de [configurar suas filas](/docs/{{version}}/queues) antes de usar este recurso.

#### Utilizando Filas de E-mail Com Atraso

Se você precisa 'atrasar' o envio de um e-mail em fila, utilize o método `later`. Simplesmente passe a quantia de segundos que o email deve ser atrasado como primeiro argumento do método:

    Mail::later(5, 'emails.welcome', $data, function ($message) {
        //
    });

#### Enviando Para Uma Fila Específica

Se você deseja especificar uma determinada fila para processar a mensagem, utilize os métodos `queueOn` e `laterOn`:

    Mail::queueOn('queue-name', 'emails.welcome', $data, function ($message) {
        //
    });

    Mail::laterOn('queue-name', 5, 'emails.welcome', $data, function ($message) {
        //
    });

<a name="mail-and-local-development"></a>
## E-mail E Desenvolvimento Local

Quando você está desenvolvendo uma aplicação que envia e-mails, provavelmente não quer enviar e-mails para endereços reais. O Laravel provê vários métodos para "desabilitar" o envio dos e-mails.

#### Log Driver

Uma solução é utilizar o driver de e-mail `log` durante o desenvolvimento. Este driver armazenará todas as mensagens de e-mail em seus arquivos de log para inspeção. Para maiores informações de como configurar sua aplicação por ambiente, confira a [documentação de configuração](/docs/{{version}}/installation#environment-configuration).

#### Único Destinatário

Outra solução fornecida pelo Laravel é configurar um endereço de e-mail para receber todos os e-mails enviados pelo framework. Desta forma, todos os e-mails gerados pela sua aplicação serão direcionados para esse destinatário ao invés dos destinatários especificados na mensagem. Isso pode ser feito através da opção `to`  no arquivo `config/mail.php`:

    'to' => [
        'address' => 'dev@domain.com',
        'name' => 'Dev Example'
    ],

#### Mailtrap

Por fim você também pode utilizar um serviço como o [Mailtrap](https://mailtrap.io) e o driver `smtp` para enviar suas mensagens de e-mail para um e-mail "fantasma" onde você pode visualizar as mensagens como um cliente real. Desta forma possível que você inspecione o resultado final de seus envios .

<a name="events"></a>
## Eventos

O Laravel dispara o evento `mailer.sending` antes de enviar mensagens de email. Lembre-se, este evento é disparado quando o e-mail é *enviado* e não quando é enviado a uma fila. Você pode registrar seu próprio listener no `EventServiceProvider`:

    /**
     * Register any other events for your application.
     *
     * @param  \Illuminate\Contracts\Events\Dispatcher  $events
     * @return void
     */
    public function boot(DispatcherContract $events)
    {
        parent::boot($events);

        $events->listen('mailer.sending', function ($message) {
            //
        });
    }
