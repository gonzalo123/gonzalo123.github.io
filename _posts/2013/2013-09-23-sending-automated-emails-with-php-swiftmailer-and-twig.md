---
layout: post
title: "Sending automated emails with PHP, Swiftmailer and Twig"
date: "2013-09-23"
tags: 
  - "dependency-injection"
  - "php"
  - "phpunit"
  - "symfony"
  - "tdd"
  - "technology"
  - "twig"
  - "coding-dojo"
  - "gmail"
  - "mediator"
  - "swiftmailer"
  - "tdd"
---

I'm the one of hosts of a **Coding Dojo** in my city called **Katayunos**. [Katayunos](http://katayunos.com/) is the mix of the word Kata (coding kata) and "Desayuno" (breakfast in Spanish). A group of brave programmers meet together one Saturday morning and after having breakfast we pick one coding kata and we practise **TDD** and **pair programming**. It's something difficult to explain to non-geek people (why the hell we wake up early one Saturday morning to do this) but if you are reading this post probably it sounds good:).

My work as host is basically pick the place and encourage people to join to the **Coding Dojo**. One way of doing this (besides twitter buzz) is take my address book and send one bulk email to all of them inviting to join us. I don't like this kind of mails. They look like spam, so I prefer to send a personalized email. This email has a common part (the place location, the hour, the event description, ...) and the personalized part. I can do it manually, the list isn't so huge, but definitely that's not cool. Because of that I have done a little script to perform this operation. I can do a simple PHP script but we are speaking about announcing a event about **TDD**, [SOLID](http://en.wikipedia.org/wiki/SOLID_(object-oriented_design)) and things like that, so I must use the "right way". Let's start.

I manage my list of contacts within a spreadsheet. In this spreadsheet I have the name, the email and a one paragraph with the personalized part to each one of my contact. I can easily export this spreadsheet to a csv document like this:

```
Peter Parker, spiderman@gmail.com, "Lorem ipsum dolor sit amet, ..."
Clark Kent, superman@gmail.com, "consectetur adipisicing elit, ..."
Juan López Fernández, superlopez@gmail.com, "sed do eiusmod tempor incididunt .."
```

So first of all I need to parse this file.

```php
class Parser
{
    private $data;
 
    public function createFromCsvFile($path)
    {
        $handle = fopen($path, "r");
        while (($data = fgetcsv($handle)) !== false) {
            $this->data[] = [
                'name'  => trim($data[0]),
                'email' => trim($data[1]),
                'body'  => isset($data[2]) ? trim($data[2]) : null,
            ];
        }
    }
 
    public function getData()
    {
        return $this->data;
    }
}
```

Easy. Now I want to send this parsed array by email. Because of that I will include [Swiftmailer](http://swiftmailer.org/) in my composer.json file.

My email will also be one template and one personalized part. We will use [Twig](http://twig.sensiolabs.org/) to manage the template.

```json
"require": {
        "swiftmailer/swiftmailer": "v5.0.2",
        "twig/twig": "v1.13.2",
}
```

Now we will create a class to wrap the needed code to send emails

```php
class Mailer
{
    private $swiftMailer;
    private $swiftMessage;
 
    function __construct(Swift_Mailer $swiftMailer, Swift_Message $swiftMessage)
    {
        $this->swiftMailer  = $swiftMailer;
        $this->swiftMessage = $swiftMessage;
    }
 
    public function sendMessage($to, $body)
    {
        $this->swiftMessage->setTo($to);
        $this->swiftMessage->setBody(strip_tags($body));
        $this->swiftMessage->addPart($body, 'text/html');
 
        return $this->swiftMailer->send($this->swiftMessage);
    }
}
```

Our Mailer class sends mails. Our Parser class parses one csv file. Now we need something to join those two classes: the Spammer class. Spammer class will take one parsed array and it will send one by one the mails using Mailer class.

```php
class Spammer
{
    private $twig;
    private $mailer;
 
    function __construct(Twig_Environment $twig, Mailer $mailer)
    {
        $this->twig       = $twig;
        $this->mailer     = $mailer;
    }
 
    public function sendEmails($data)
    {
        foreach ($data as $item) {
            $to = $item['email'];
            $this->mailer->sendMessage($to, $this->twig->render('mail.twig', $item));
        }
    }
}
```

Ok with this three classes I can easily send my emails. This script is a console script and we also want pretty console colours and this kind of stuff. [symfony/console](http://symfony.com/doc/current/components/console/introduction.html) to the rescue. But I've a problem now. I want to write one message when one mail is sent and another one when something wrong happens. If I want to do that I need to change my Spammer class. But my Spammer class does't know anything about my console Command. If I inject the console command into my Spammer class I will violate the [Demeter law](http://en.wikipedia.org/wiki/Law_of_Demeter), and that's a sin. What can we do? Easy: The [mediator pattern](http://en.wikipedia.org/wiki/Mediator_pattern). We can write one implementation of mediator pattern but we also can use [symfony/event-dispatcher](http://symfony.com/doc/current/components/event_dispatcher/introduction.html), a well done implementation of this pattern. We change our Spammer class to:

```php
use Symfony\Component\EventDispatcher\EventDispatcher;
 
class Spammer
{
    private $twig;
    private $mailer;
    private $dispatcher;
 
    function __construct(Twig_Environment $twig, Mailer $mailer, EventDispatcher $dispatcher)
    {
        $this->twig       = $twig;
        $this->mailer     = $mailer;
        $this->dispatcher = $dispatcher;
    }
 
    public function sendEmails($data)
    {
        foreach ($data as $item) {
            $to = $item['email'];
            try {
                $this->mailer->sendMessage($to, $this->twig->render('mail.twig', $item));
                $this->dispatcher->dispatch(MailEvent::EVENT_MAIL_SENT, new MailEvent\Sent($to));
            } catch (\Exception $e) {
                $this->dispatcher->dispatch(MailEvent::EVENT_SENT_ERROR, new MailEvent\Error($to, $e));
            }
        }
    }
}
```

Now can easily build of console command class:

```php
use Symfony\Component\Console\Command\Command;
use Symfony\Component\Console\Input\InputArgument;
use Symfony\Component\Console\Input\InputInterface;
use Symfony\Component\Console\Input\InputOption;
use Symfony\Component\Console\Output\OutputInterface;
use Symfony\Component\EventDispatcher\EventDispatcher;
 
class SpamCommand extends Command
{
    private $parser;
    private $dispatcher;
 
    protected function configure()
    {
        $this->setName('spam:run')
            ->setDescription('Send Emails');
    }
 
    protected function execute(InputInterface $input, OutputInterface $output)
    {
        $output->writeln("Sending mails ...");
        $this->dispatcher->addListener(MailEvent::EVENT_MAIL_SENT, function (MailEvent\Sent $event) use ($output) {
                $output->writeln("<info>Mail sent to</info>: <fg=black;bg=cyan>{$event->getTo()}</fg=black;bg=cyan>");
            }
        );
 
        $this->dispatcher->addListener(MailEvent::EVENT_SENT_ERROR, function (MailEvent\Error $event) use ($output) {
                $output->writeln("<error>Error sending mail to</error>: <fg=black;bg=cyan>{$event->getTo()}</fg=black;bg=cyan> Error: " . $event->getException()->getMessage());
            }
        );
 
        $this->spammer->sendEmails($this->parser->getData());
        $output->writeln("End");
    }
 
    public function setSpammer(Spammer $spammer)
    {
        $this->spammer = $spammer;
    }
 
    public function setParser(Parser $parser)
    {
        $this->parser = $parser;
    }
 
    public function setDispatcher(EventDispatcher $dispatcher)
    {
        $this->dispatcher = $dispatcher;
    }
}
```

With all this parts we can build our script. Our classes are decoupled. That's good but setting up the dependencies properly can be hard. Because of that we will use [symfony/dependency-injection](http://symfony.com/doc/current/components/dependency_injection/introduction.html). With symfony DIC we can set up our dependency tree within a yaml file:

Our main services.yml 

```yaml
imports:
  - resource: conf.yml
  - resource: mail.yml
  - resource: twig.yml
 
parameters:
  base.path: .
 
services:
  parser:
    class: Parser
    calls:
      - [createFromCsvFile, [%mail.list%]]
 
  mailer:
    class: Mailer
    arguments: [@swift.mailer, @swift.message]
 
  spam.command:
    class: SpamCommand
    calls:
      - [setParser, [@parser]]
      - [setDispatcher, [@dispatcher]]
      - [setSpammer, [@spammer]]
 
  spammer:
    class: Spammer
    arguments: [@twig, @mailer, @dispatcher]
 
  dispatcher:
    class: Symfony\Component\EventDispatcher\EventDispatcher
```

I like to separate the configuration files to reuse those files between projects and to make them more readable.

One for twig:

```yaml
parameters:
  twig.path: %base.path%/templates
  twig.conf:
    auto_reload: true
 
services:
  twigLoader:
    class: Twig_Loader_Filesystem
    arguments: [%twig.path%]
 
  twig:
    class: Twig_Environment
    arguments: [@twigLoader, %twig.conf%]
```

another one for swiftmailer:

```yaml
services:
  swift.message:
    class: Swift_Message
    calls:
      - [setSubject, [%mail.subject%]]
      - [setFrom, [%mail.from.mail%: %mail.from.name%]]
 
  swift.transport:
    class: Swift_SmtpTransport
    arguments: [%mail.smtp.host%, %mail.smtp.port%, %mail.smtp.encryption%]
    calls:
      - [setUsername, [%mail.smtp.username%]]
      - [setPassword, [%mail.smtp.password%]]
 
  swift.mailer:
    class: Swift_Mailer
    arguments: [@swift.transport]
```

and the last one for the configuration parameters:

```yaml
parameters:
  mail.do.not.send.mails: false
 
  mail.list: %base.path%/mailList.csv
  mail.subject: mail subject
  mail.from.name: My Name
  mail.from.mail: my_email@mail.com
 
  mail.smtp.username: my_smtp_username
  mail.smtp.password: my_smtp_password
  mail.smtp.host: smtp.gmail.com
  mail.smtp.port: 465
  mail.smtp.encryption: ssl
```

Now we can build our script.

```php
use Symfony\Component\Console\Application;
use Symfony\Component\DependencyInjection\ContainerBuilder;
use Symfony\Component\Config\FileLocator;
use Symfony\Component\DependencyInjection\Loader\YamlFileLoader;
 
$container = new ContainerBuilder();
$loader = new YamlFileLoader($container, new FileLocator(__DIR__ . '/conf'));
$loader->load('services.yml');
 
$container->setParameter('base.path', __DIR__);
 
$application = new Application();
$application->add($container->get('spam.command'));
$application->run();
```

And that's all. My colleagues of the next **Katayuno** will be invited in a "**SOLID**" way :). Source code is available in my [github](https://github.com/gonzalo123/spam) account.

> BTW: Do you want to organize one **Katayuno** in your city? It's very easy. Feel free to [contact me](http://gonzalo123.com/contact/) for further information.
