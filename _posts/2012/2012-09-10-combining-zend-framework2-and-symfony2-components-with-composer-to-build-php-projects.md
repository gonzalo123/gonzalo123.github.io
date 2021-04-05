---
layout: post
title: "Combining Zend Framework2 and Symfony2 components with Composer to build PHP projects"
date: "2012-09-10"
tags: 
  - "php"
  - "symfony2"
  - "technology"
  - "zend-framework"
---

Zend Framework 2 is finally [stable](http://framework.zend.com/). I must admit that I'm not a big fan of ZF (or even Symfony2) as a full stack framework. I normally prefer to use micro frameworks, but those two frameworks (ZF2 and SF2) are great as component libraries. Today we are going to build a simple console application (using [symfony/console](http://symfony.com/doc/2.0/cookbook/console/console_command.html) component) to list the database tables (using [zendframework/zend-db](http://framework.zend.com/manual/2.0/en/index.html#zend-db)'s [Metadata](http://framework.zend.com/manual/2.0/en/modules/zend.db.metadata.html)). Let's start.

First we need our composer.json file. We can find our Symfony components at [Packaist](http://packagist.org/packages/symfony/console) (that's means we don't need to do anything special in the composer.json file), but Zend Framework2 has its own repository. No problem, it's properly described in the [documentation](http://framework.zend.com/downloads/composer): \

```json
{
    "repositories": [
        {
            "type": "composer",
            "url": "http://packages.zendframework.com/"
        }
    ],
    "require": {
        "symfony/console":"dev-master",
        "zendframework/zend-db":"2.0.*"
    },
    "autoload":{
        "psr-0":{
            "":"lib/"
        }
    }
}
```

Now we run _composer install_ (as always) and we already have our components in vendor _folder_ and the autoloader will properly include the files on demand. So we can work in our console application without any problem.

```php
<?php
namespace GonzaloDb;
 
// file: lib/GonzaloDb/SchemeCommand.php
 
use Symfony\Component\Console\Command\Command;
use Symfony\Component\Console\Input\InputArgument;
use Symfony\Component\Console\Input\InputOption;
use Symfony\Component\Console\Input\InputInterface;
use Symfony\Component\Console\Output\OutputInterface;
 
use Zend\Db\Sql\Sql;
use Zend\Db\Adapter\Adapter;
use Zend\Db\Metadata\Metadata;
 
class SchemeCommand extends Command
{
    protected function configure()
    {
        $command = $this->setName('GonzaloDb:listTables')->setDescription('list all tables');
        $command->addArgument('host', InputArgument::REQUIRED, 'DB Host');
        $command->addArgument('port', InputArgument::REQUIRED, 'DB Port');
        $command->addArgument('database', InputArgument::REQUIRED, 'DB name');
        $command->addArgument('username', InputArgument::REQUIRED, 'Username');
        $command->addArgument('password', InputArgument::REQUIRED, 'Password');
 
        $command->addOption('listFields', NULL, InputOption::VALUE_NONE, 'list table Fields');
    }
 
    protected function execute(InputInterface $input, OutputInterface $output)
    {
        $adapterParameter = array(
            'driver'   => 'PDO_Pgsql',
            'host'     => $input->getArgument('host'),
            'port'     => $input->getArgument('port'),
            'database' => $input->getArgument('database'),
            'username' => $input->getArgument('username'),
            'password' => $input->getArgument('password')
        );
 
        $adapter = new Adapter($adapterParameter);
 
        $metadata   = new Metadata($adapter);
        $tableNames = $metadata->getTableNames();
        foreach ($tableNames as $tableName) {
            $output->writeln("Table: <info>$tableName</info>");
            if ($input->getOption('listFields')) {
                $table = $metadata->getTable($tableName);
 
                foreach ($table->getColumns() as $column) {
                    $output->writeln('  <comment>' . $column->getName() . '</comment> -> ' . $column->getDataType());
                }
            }
        }
    }
}
```

We also can use unit tests within console applications:

```php
<?php
 
// file: tests/SchemeCommandTest.php
/*
CREATE TABLE users(
  userid character varying(50),
  name character varying(100),
  email character varying(50),
    CONSTRAINT users_pkey PRIMARY KEY (userid)
);
 */
use Symfony\Component\Console\Application;
use Symfony\Component\Console\Tester\CommandTester;
use GonzaloDb\SchemeCommand;
 
class SchemeCommandTest extends \PHPUnit_Framework_TestCase
{
    private $command;
 
    public function setUp()
    {
        $application = new Application();
        $application->add(new SchemeCommand());
 
        $this->command = $application->find('GonzaloDb:listTables');
    }
 
    public function testListTables()
    {
        $commandTester = new CommandTester($this->command);
        $commandTester->execute(
            array(
                 'command'  => $this->command->getName(),
                 'host'     => '127.0.0.1',
                 'port'     => 5432,
                 'database' => 'mydb',
                 'username' => 'username',
                 'password' => 'password',
            )
        );
        $this->assertRegExp('/Table: users/', $commandTester->getDisplay());
        $this->assertNotRegExp('/name -> character varying/', $commandTester->getDisplay());
    }
 
    public function testListTablesAndFields()
    {
        $commandTester = new CommandTester($this->command);
        $commandTester->execute(
            array(
                 'command'      => $this->command->getName(),
                 'host'         => '127.0.0.1',
                 'port'         => 5432,
                 'database'     => 'mydb',
                 'username'     => 'username',
                 'password'     => 'password',
                 '--listFields' => TRUE
            )
        );
        $this->assertRegExp('/Table: users/', $commandTester->getDisplay());
        $this->assertRegExp('/name -> character varying/', $commandTester->getDisplay());
    }
}
```

And that's all. Zend Framework2 increases or toolbox as developers and with the power of Composer we can start building applications very fast. I like it :)
