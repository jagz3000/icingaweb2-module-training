# Icinga Web 2 expand

## Write your own Icinga Web Module

Welcome! Nice that you are here to write your first own Icinga Web Module. Icinga Web makes getting started as easy as possible. In the next few hours, we will discover how pleasurable the whole thing is with a series of practical examples.

## Should I really? Why?

Absolutely, why not? It's wonderfully simple, and Icinga is 100% free open source software with a great community. Icinga Web 2 is a stable, easy-to-understand and future-proof platform. So exactly what you want to build your own projects.

## Only for monitoring?

Not at all! Sure, monitoring is where Icinga Web comes from. There it has its strengths, there it is at home. Since monitoring systems communicate with all sorts of systems in and outside of our own data center anyway, we found it obvious to do so in the same way in the frontend as well.

Icinga Web wants to be a modular framework that wants to make integration of third-party software as easy as possible. At the same time, we want to make it easy for third parties to use the logic of Icinga as conveniently as possible in their own projects.

Whether it is about the mere linking of third-party systems, the connection of a CMDB or the visualization of complex systems as a supplement to conventional check-plugins - the imagination knows no bounds.

## But I'm not a PHP/JavaScript/HTML5 hacker

No problem. Of course, it does not hurt to have profound knowledge of web development. Icinga Web allows you to write your own modules without profound PHP/HTML/CSS knowledge.

# Preparation

We use a debian base installation to demonstrate how little dependency Icinga Web 2 has. Although there are packages for all major distributions, we will be working directly with the GIT source tree in the better learning experience.

So that our notebooks wake up from the weekend sleep, we give them a small task:

    apt-get update

    # A few useful tools for the workshop:
    apt-get install git vim wget

    # Dependencies for Icinga Web 2:
    apt-get install php5-cli php5-mysql php5-gd php5-intl php5-ldap

    # Current source code from the Icinga Web 2 Master:
    cd /usr/local
    git clone https://git.icinga.org/icingaweb2.git

In the meantime, we will dedicate ourselves to the introduction!

## Structure of the training

* Establishment of Icinga Web 2
* Creation of a separate module
  * Own CLI-Commands
  * Working with parameters
  * Colors and other gimmicks
* Extension of the web frontends
  * Own pictures
  * Own stylesheets
  * Extension of the menu
  * Provision of dashboards
* Working with data
  * Provide data
  * Pack code in libraries
  * Working with parameters
  * Tricks for working comfortably
* Configuration
* Translations
* Integration in third party software
* Free Lab


## Icinga Web 2 architecture

During the development of Icinga Web 2, three priorities were emphasized:

* Simplicity.
* Speed
* Reliability


Although we have dedicated ourselves to the DevOps movement, our target group with Icinga Web 2 is clearly the operator, the admin. Therefore, we try to have as few dependencies as possible on external components. We'll give up on one or the other hip feature, but then it will be less broken if the Hippster in the next version again want to do everything differently. The best example is this workshop: now one year old, written long before the first stable release of Icinga Web 2 - and yet almost all exercises still work without any changes to the code.

The web interface is designed to hang easily on the wall for weeks and even months on the same screen. We want to be able to rely on what we see there corresponds to the current state of our environment. If there are problems, these are visualized, even if they are in the application itself. If the problem is resolved, everything must continue as usual. And without anyone having to plug in a keyboard and intervene manually.


## Used libraries

* Zend Framework 1.x
* jQuery 1.11 and 2.1
* Smaller PHP-Libraries
  * HTMLPurifier
  * DOMPdf
  * lessphp
  * Parsedown
* * Smaller JS-libraries
  * jquery-...

## Anatomy of an Icinga Web 2 Module

Icinga Web 2 follows the paradigm "convention before configuration". After the experience with Icinga Web 1, we came to the conclusion that one of the best tools for XML processing is on each disk: `/ bin / rm`. Those who stick to a few simple conventions will save a lot of configuration work. Basically, in Icinga Web you only have to configure paths for special cases. It is usually enough to simply save a file in the right place.

An extensive, adult module could have the following structure:

    .
    └── training                Basic directory of the module
        ├── application
        │   ├── clicommands     CLI commands
        │   ├── controllers     Web Controller
        │   ├── forms           Forms
        │   ├── locale          Translations
        │   └── views
        │       ├── helpers     View Helper
        │       └── scripts     View Scripts
        ├── configuration.php   Deploy menu, dashlets, permissions
        ├── doc                 Documentation
        ├── library
        │   └── Training        Library-Code, Module-namespace
        ├── module.info         Metadata about the module
        ├── public
        │   ├── css             Own CSS-Code
        │   ├── img             Own pictures
        │   └── js              Own JavaScript
        ├── run.php             Registration of hooks and more
        └── test
            └── php             PHP Unit-Tests

We will work on this one step by step during this training and fill it with life.

## Prepare Source Tree

To get started we first need Icinga Web 2. This can be checked out of the GIT Source Tree and used directly on the spot. If you subsequently set `DocumentRoot` of a correspondingly configured web server in the public directory, you can already start. For testing purposes, it's even easier:

    cd /usr/local
    # If not done yet:
    git clone https://git.icinga.org/icingaweb2.git
    ./icingaweb2/bin/icingacli web server

Finished. To use the installation wizard, a token is required for security reasons. This is the web interface. This ensures that there is never a time between installation and setup when an attacker could take over an environment. For Packager this point is completely optional, the same applies to those who roll out Icinga Web with a CM tool like Puppet: if there is a configuration on the system, you never get to see the Wizard.

  http://localhost

## Full installation with web server

So far we have been running fun with Icinga Web 2 without an external web server. This would be performant enough even for most productive environments, yet most of us feel comfortable with a "real" web server. So if you have not already done so, stop the PHP process and clean up first:

```sh
rm -rf /tmp/FileCache_icingaweb/ /var/lib/php5/sess_*
```

These files, probably created by root, would otherwise only cause us problems. Then we install our web server:

```
apt-get install libapache2-mod-php5
./icingaweb2/bin/icingacli setup config webserver apache \
  > /etc/apache2/conf.d/icingaweb2.conf
service apache2 restart
```

You see, Icinga Web 2 can generate its own configuration for Apache (2.x, also compatible with 2.4). This is true not only for Apache, but also for Nginx.

## The configuration directory

If not configured differently, Icinga Web 2 looks for its configuration in `/etc/icingaweb`. This can be overridden at any time with the environment variable ICINGAWEB_CONFIGDIR. Also in the web server we can use this:

    SetEnv ICINGAWEB_CONFIGDIR /another/directory

## Manage multiple module paths

Especially those who always work with the latest version or want to switch between GIT branches safely, usually do not want to have to change files in their working copy. Therefore, it is recommended to use multiple module paths in parallel from the beginning. This can be done in the system settings or in the configuration under `/etc/icingaweb/config.ini`:

    [global]
    module_path= "/usr/local/icingaweb-modules:/usr/local/icingaweb2/modules"


## Set up Icinga CLI

When installing from packages you do not have to worry about anything, for our GIT working copy we create a symbolic link for your convenience:

    ln -s /usr/local/icingaweb2/bin/icingacli /usr/local/bin/

## Installation from packages

The Icinga project builds up-to-date snapshots for various operating systems every day, the package sources are available at [packages.icinga.org] (https://packages.icinga.org/). The current build status can be viewed at [build.icinga.org] (https://build.icinga.org/jenkins/view/Icinga%20Web%202/), and the latest development at any time at [git.icinga.org] (https://git.icinga.org/?p=icingaweb2.git) or [GitHub] (https://github.com/Icinga/icingaweb2/).

But for our training we use the git-repository directly. And I'm not reluctant to do that in production either. Checksums for everything, changed files never go undetected, version changes happen in a fraction of a second - which package management can offer that? In addition, this procedure shows nicely how simple Icinga Web 2 actually is. We did not have to change a single file in the source directory for installation. Automake, configure? What for?! The configuration is elsewhere, and WHERE it is is communicated to the runtime environment.

# Create your own module

## Where should I start?

Probably the most important question is usually what you really want to do with his module. In our training we will first experiment with the given possibilities and then implement a small practical example.

## How should I call my module?

Once you know what the module is about to do, the hardest task is often choosing a good name. Ideally, this will tell you what the module actually does. But the name should not be too complicated, because we will use it in PHP namespaces, directory names, and URLs.

Your own (company) name is often the first step in your own life. Our favorite module name for our first steps in training today is `training`.

## Create and activate a new module

    mkdir -p /usr/local/icingaweb-modules/training
    icingacli module list installed
    icingacli module enable training

Finished!

# Extend the Icinga CLI

The Icinga CLI was designed to provide as much as possible of what is available to application logic in Icinga Web 2 and its modules on the command line as well. The project wants to make the creation of cronjobs, plugins, useful tools and possibly small services as easy as possible.

## Configure Vim

We work with VIM in training and make some initial settings:

    echo 'syntax on
    set expandtab
    set tabstop=4
    set shiftwidth=4' > /root/.vimrc

## Own CLI commands

Structure of the CLI commands:

    icingacli <modul> <command> <action>


Creating a CLI command is very easy. In the directory `application/clicommands` a file will be created whose name corresponds to the desired command:

    cd /usr/local/icingaweb-modules/training
    mkdir -p application/clicommands
    vim application/clicommands/HelloCommand.php

Here, Hello corresponds to the desired command with a capital letter. The ending Command MUST always be set.

Example Command:

```php
<?php

namespace Icinga\Module\Training\Clicommands;

use Icinga\Cli\Command;

class HelloCommand extends Command
{
}
```

## Namespaces

* Namespaces help to delineate modules against each other
* Each module gets a namespace, which results from the module name:

```
Icinga\Module\<Modulname>
```

* The first letter MUST be capitalized
* For CLI commands, a dedicated namespace Clicommands is available

## Heredity

All CLI commands MUST inherit the Command class in the namespace `Icinga \ Cli`. This brings us a whole range of advantages, which we will discuss later. It is important that our class name corresponds to the name of the file. In our HelloCommand.php this would be class HelloCommand.

## Command-Actions

Each command can provide multiple actions. Any new public method that ends with action automatically becomes a CLI command action:

```php
<?php
class HelloCommand extends Command
{
    public function worldAction()
    {
        echo "Hello World!\n";
    }
}
```

## Task 1

We create a CLI Command with an action, which is operated as follows and generates the following output:

    icingacli training say hello


## Bash Autocompletion

The Icinga CLI provides autocomplete for all modules, commands and actions. If you install Icinga Web 2 per package, everything is already in the right place, for our test environment, we manually set:

## Bash completion

    apt-get install bash-completion
    cp etc/bash_completion.d/icingacli /etc/bash_completion.d/
    . /etc/bash_completion

If the input is ambiguous as in `icingacli mo`, then an appropriate help is displayed.


## Inline Documentation for CLI Commands

Commands and their actions can be easily documented via inline comments. The comment text is immediately available on the CLI as a help.

```php
<?php

/**
 * This is where we say hello
 *
 * The hello command allows us to be friendly to everyone
 * and his dog. That's how nice people behave!
 */
class HelloCommand extends Command
{
    /**
     * Use this to greet the world
     *
     * Greeting every single person would take some time,
     * so let's greet the whole world at once!
     */
    public function worldAction()
    {
        // ...
```

A few example combinations of how the help can be displayed:

    icingacli training
    icingacli training hello
    icingacli help training hello
    icingacli training hello world --help

The help command can be at the beginning or used at any point as a parameter with `--`.

## Task 2

Create and test documentation for a `something` action for the `say` command in the `training` module!


## Command line parameters

Of course, we can fully control, use and control command line parameters ourselves. Thanks to heredity, the corresponding instance of `Icinga\Cli\Params` is already available in` $ this-> params`. The object has a `get ()` method, to which we can give the desired parameter and optionally a default value. Without default value, we get `null` if the corresponding parameter is not given.

```php
<?php

// ...

    /**
     * Say hello as someone
     *
     * Usage: icingacli training hello from --from <someone>
     */
    public function fromAction()
    {
        $from = $this->params->get('from', 'Nowhere');
        echo "Hello from $from!\n";
    }
```

### Example call:

    icingacli training hello from --from Nürnberg
    icingacli training hello from --from "Netways Training"
    icingacli training hello from --help
    icingacli training hello from

## Standalone parameters

It is not absolutely necessary to assign an identifier to each parameter. If you want, you can simply string parameters together. Most conveniently, these are accessible via the `shift ()` method:

```php
<?php

// ...

/**
 * Say hello as someone
 *
 * Usage: icingacli training hello from <someone>
 */
public function fromAction()
{
    $from = $this->params->shift();
    echo "Hello from $from!\n";
}
```

### Example call

    icingacli training hello from Nuremberg

## Shiften is fun

The method `shift ()` behaves in the same way as you would expect from common programming languages. The first parameter of the list is returned and removed from the list. If you call `shift ()` several times in succession, all existing standalone parameters are returned until no longer available. With `unshift ()` you can undo such an action at any time.

A special case is `shift ()` with an identifier (key) as a parameter. So `shift ('to')` would not only return the value of the `--to` parameter, but also remove it from the Params object, regardless of its position. Again, it is possible to specify a default value:

```php
<?php
// ...
$person = $this->params->shift('from', 'Nobody');
```

Of course, this also works for standalone parameters. Since we have already used the first parameter of `shift ()` with the optional identifier (key), but still want to set something for the second (default value), we simply set the identifier to zero here:

```php
<?php
// ...
public function fromAction()
{
    $from = $this->params->shift(null, 'Nowhere');
    echo "Hello from $from!\n";
}
```

### Example call

    icingacli training hello from Nuremberg
    icingacli training hello from
    icingacli training hello from --help

## API Documentation

The Params class in the `Icinga\Cli` namespace documents other methods and their parameters. These are most conveniently accessible in the API documentation. This can be generated with phpDocumentor, in the near future there should also be a CLI command.

## Task 3

Extend the say command to support all of the following options:

    icingacli training say hello World
    icingacli training say hello --to World
    icingacli training say hello World --from "Icinga CLI"
    icingacli training say hello World "Icinga CLI"


## Exceptions

Icinga Web 2 wants to promote clean PHP code. This includes, among other things, that all Warnings generate errors. Errors are thrown for error handling. We can just try it:

```php
<?php
// ...
use Icinga\Exception\ProgrammingError;
// ...
/**
 * This action will always fail
 */
public function kaputAction()
{
    throw new ProgrammingError('No way');
}
```

### Call

    icingacli training hello kaput
    icingacli training hello kaput --trace

## Exit-Codes

As we can see, the CLI catches all exceptions and outputs pleasant readable error messages along with a colored reference to the error. The exit code in this case is always 1:

    echo $?

This allows reliable evaluation of failed jobs. Only the exit code 0 stands for successful execution. Of course, everyone is free to use additional exit codes. This is done in PHP using `exit ($ code)`. Example:

```php
<?php
echo "CRITICAL\n";
exit(2);
```

Alternatively, Icinga Web provides the `fail ()` function in the Command class. It is an abbreviation for a colored "ERROR", a status output and `exit (1)`:

```php
<?php
$this->fail('An error happened');
```

## To dye?

As we have just seen, the Icinga CLI can create colored output. The Screen class in the `Icinga\Cli` namespace provides useful help functions. We access it in our Command classes via `$ this-> screen`. This is how the output can be colored:

```php
<?php
echo $this->screen->colorize("Hello from $from!\n", 'lightblue');
```

As an optional third parameter, the `colorize ()` function can be given a background color. For the representation of the colors ANSI escape codes are used. If Icinga CLI detects that the output is NOT in a terminal/TTY, no colors will be output. This ensures that e.g. When redirecting the output to a file, no disturbing special characters appear.

> To recognize the terminal, PHP uses the POSIX extension. If this is not available, as a precaution, no ANSI codes are used.

Other useful features in the Screen class are:

* `clear()` to clear the screen (used by `--watch`)
* `underline()` to underline text
* `newlines($count = 1)` to output one or more newlines
* `strlen()` to determine the character width without ANSI codes
* `center($text)` to output text centered depending on the screen width
* `getRows()` and `getColumns()` where possible to determine the usable space
* `hasUtf8()` to query UTF 8 support of the terminal

Attention: Of course, it does not work to find out that someone is traveling in a UTF8 terminal with an ISO8859 Putty.

### Task

Our `hello` action in the `say` command should output the text in color and centered both horizontally and vertically. We use `--watch` to flash the output alternately in at least two colors.

# The own module in the web frontend

Icinga Web would not carry __"Web"__ in the name if its true qualities did not show up there as well. As we'll see shortly, __convention before configuration__ applies here as well. Of course, according to the classic __MVC-concept__, there are controllers with all available actions and matching view scripts for output and display.

We have deliberately omitted the separation library/model, and each additional layer increases the complexity. You could also look at the library code in many modules as a "model", but this should be clarified by the specialists. At least we would like to have as many chic modules as possible, ideally with a lot of reusable code, which then also benefits other modules.

## A first controller

Every `action` in a `controller` automatically becomes a `route` in our web frontend. This looks something like this:

    http(s)://<host>/icingaweb/<modul>/<controller>/<action>

If we want to create our "Hello World" again for our training module, we first create the basic directory for our controllers:

    mkdir -p training/application/controllers

Afterwards we put on our controller. As you suspect, this must be called HelloController.php and in the controller namespace of our module:

```php
<?php

namespace Icinga\Module\Training\Controllers;

use Icinga\Web\Controller;

class HelloController extends Controller
{
    public function worldAction()
    {
    }
}
```

When we call the url (training/application/controllers), we get an error message:


    Server error: script 'hello/world.phtml' not found in path
    (/usr/local/icingaweb-modules/training/application/views/scripts/)

Practically, she immediately tells us what we need to do next.

## Create a view script

The corresponding base directory is still missing. Since we create a view script per "action" in a dedicated file, there is one directory per "controller":

    mkdir -p training/application/views/scripts/hello
    
The view script is then just like the "action", so world.phtml:

```php
<h1>Hello World!</h1>
```

That's it, our new URL is now available. We could now use the full scope for our module and style it accordingly. But we can also use a few predefined elements. Two important classes are e.g. `controls` and `content` for header elements and the page content.

```php
<div class="controls">
<h1>Hello World!</h1>
</div>

<div class="content">
Here you go...
</div>
```

This automatically gives even spacing to the page margins and also gives the effect that when scrolling down the `controls` stop while the `content` scrolls. Of course, we will not notice that until we fill our module with more content.

## Menu entries

Menu entries in Icinga Web 2 can be personalized and / or specified by the administrator (*). Regardless, they can be provided by modules. This is a global configuration that can be made in the base directory of your own module in `configuration.php`:

```php
<?php

$this->menuSection('Training')
     ->add('Hello World')
     ->setUrl('training/hello/world');
```

### Icons for menu entries

So that our menu item looks better, we miss him on this occasion just one more Icon:

```php
<?php

$this->menuSection('Training')
     ->setIcon('thumbs-up')
     ->add('Hello World')
     ->setUrl('training/hello/world');
```

To find out which icons are available, we activate the `doc` module under `System`/`Module`. Then we find the icon list under `Documentation`/`Developer - Style`. These are icons that have been embedded in a font. This has the great advantage that much fewer requests have to be made via the line - the icons are simply "always there".

Alternatively, you can still use classic icons (.png etc) if you wish. This is especially useful if you want to use a special icon (for example, a company logo) for your module, which can not be integrated into the official Icinga Icon font:

```php
<?php

$this->menuSection('Training')->setIcon('img/icons/success.png');
```

## Add pictures

If you want to use your own images in your module, you can simply put them under `public/img`:

    mkdir -p public/img
    wget https://www.icinga.org/wp-content/uploads/2014/06/tgelf.jpg
    mv tgelf.jpg public/img/

Our pictures are immediately accessible via the web, the URL pattern is as follows:

    http(s)://<icingaweb>/img/<module>/<bild>

For our specific case so http://localhost/img/training/tgelf.jpg. It's also great to use in our view script. Instead of creating an img tag (which of course would be possible) we use one of the many practical view helpers:

```php
...
<div class="content">
<?= $this->img('img/training/tgelf.jpg', array('title' => 'Thomas Gelf')) ?> Here you go...
</div>
```

## Task

Create the URLs `training/hello/thomas` and `training/say/hello` and add an additional menu item. Also, look for our training module a nicer icon from the Internet and set it accordingly.

## Dashboards

Before we take care of serious issues we will of course still provide our useful URL as a default dashboard. This can also be done in the `configuration.php`:

```php
<?php
$this->dashboard('Training')->add('Hello', 'training/hello/world');
```

# We need data!

Of course, with our web routes working so well, we want to do something useful with them. An application can be so beautiful, without useful content, it quickly gets boring. In an MVC environment, the `controllers` usually use the `models` to get their data and feed the `view`.

## Fill our view with data

The controller provides access to our view in `$this->view`. In this way she can be refueled quite comfortably:

```php
<?php

public function worldAction()
{
    $this->view->application = 'Icinga Web 2';
    $this->view->moreData = array(
        'Work'   => 'done',
        'Result' => 'fantastic'
    );
}
```

We are now expanding our view script and presenting the submitted data accordingly:

```php
<h3>Some data...</h3>

This example is provided by <a href="http://www.netways.de">Netways</a> 
and based on <?= $this->application ?>.

<table>
<?php foreach ($this->moreData as $key => $val): ?>
    <tr><th><?= $key ?></th><td><?= $val ?></td></tr>
<?php endforeach ?>
</table> 
```

## Task

Under `training/list/files` the contents of our module directory should be listed in table form.
* Note: with `$this->Module()->getBaseDir()` we get our module directory
* More about opening directories at [http://en.php.net/opendir](http://de.php.net/opendir)

# But please with style!

Although this has not directly to do with our topic, but one thing stands out: our table is not very nice. Luckily we can easily put CSS in our module. We create a suitable directory, the name should be obvious:

    mkdir public/css

We then store our CSS statements there in the file `module.less`. Less is a CSS extension to all sorts of functions, more can be found under [...] (). Conventional CSS is definitely valid here. The nice thing about Icinga Web is that I do not have to worry about my CSS influencing other modules or Icinga Web itself: that's not the case.

So we can easily define the following, without making other tables "broken":

    table {
        width: 100%;
    }

    th {
        width: 20%;
        text-align: right;
        line-height: 2em;
        padding-right: 2em;
    }

When we watch the requests in our browser's developer tools, we see that Icinga Web loads css / icings.min.css as the only CSS file. We can also load css / icinga.css for convenient viewing of what Icinga Web has done with our CSS code:

    .icinga-module.module-training table {
      width: 100%;
    }
    .icinga-module.module-training th {
      width: 20%;
      text-align: right;
      line-height: 2em;
      padding-right: 2em;
    }

As we can see, prefixes ensure that our CSS only applies to those containers in which our module represents its contents.

## Useful CSS classes

Icinga Web 2 provides a set of CSS classes that make our job easier. So `common-table` is useful for the usual lists in tables, `name-value-table` For name / value pairs where on the left the identifier is represented as th and on the right the corresponding value in a td. Also useful is `table-row-selectable` - this changes the behavior of the table. The whole line is highlighted when you hover over it. If you click somewhere, the first link of the line comes to the train. In combination with `common-table`, the whole thing looks good without any additional work.

# Real data cleaned up

As we saw earlier, such a module becomes really interesting only with real data. What we did wrong, though, is that our controller gets the data itself. That is unattractive and would cause us problems at the latest if we also want to use this data on the CLI.

## Our own library

We create a new directory for our library in our module, following the scheme `library/<module name>`. In our case, then:

    mkdir -p library/Training

As already learned, we use the namespace `Icinga\Modules\<module name>` for our module. All of the namespaces underneath will search Icinga Web 2 automatically in the newly created directory. Exceptions are those seen earlier, such as `Clicommands` or `Controllers`.

A biblithek doing the task in the exercise might be in `File.php` and look like this:

```php
<?php

namespace Icinga\Module\Training;

use DirectoryIterator;

class Directory
{
    public static function listFiles($path)
    {
        $result = array();

        foreach (new DirectoryIterator($path) as $file) {
            if ($file->isDot()) continue;

            $result[] = (object) array(
                'name' => $file->getFilename(),
                'path' => $file->getPath(),
                'size' => $file->getSize(),
                'type' => $file->getType()
            );
        }
        return $result;
    }
}
```

Our controller can now easily retrieve the data from our small library:

```php
<?php

// ...
use Icinga\Module\Training\Directory;

class FileController extends Controller
{
    public function listAction()
    {
        $this->view->files = Directory::listFiles($this->Module()->getBaseDir());
    }
}
```

## Task

Put this or a comparable library in your module. Provide a view script, which can list the individual files to match. Importantly, use `$this->escape()` in the view script to escape data whose source is unsafe (for example, filenames).

# Parameter Handling

So far we have not given any parameters to our URLs. But that is easy. Just like on the command line, Icinga Web gives us simple access to Params. Access to it is as usual:

```php
<?php
$file = $this->params->get('file');
```

Also `shift()` and cohorts are of course available again.

## Task

Under `training/file/show?File=<filename>` additional information about the desired file should be displayed. Hard-working show owners, permissions, last change and mime-type - but it is also quite simply enough to re-file name and size "in beautiful".

## Related Links

In our file list we now want to link from each file to the corresponding detail area. In order to avoid problems with parameter escaping, we use a new helper, `qlink`:

```php
<td><?= $this->qlink(
    $file->name,
    'training/file/show',
    array('file' => $file->name)
) ?></td>
```

The first parameter here is the text to be displayed, the second is the link to be created, and third is the optional paramteter for this link. The fourth parameter could be an array with any other HTML attributes.

If we now click on a file in our list, we end up with the corresponding details. But that is also more convenient. Just try putting `data-base-target ="_next"` in the content-div:

    <div class="content" data-base-target="_next">

This is the first time that we have been managing the multi-column layout of Icinga Web for the first time without great effort!

# URL-Handling

Anyone who has observed how the browser behaves, may have noticed that not every click reloads the page. Icinga Web 2 intercepts all requests and sends them independently via XHR request. On the server side this is detected, and then only the respective HTML snippet is sent in response. This usually only corresponds to the output created by the corresponding view script.

Nevertheless, each link remains a link and can be e.g. open in a new tab. Here again it is recognized that this is not an XHR-request, the complete layout is delivered.

Usually, links always end up in the same container, but you can influence the behavior with `data-base-target`. The attribute closest to the element clicked wins. If you want to cancel `_next` for a part of the page, simply set `data-base-target="_self"`.

# Data handling made easy

Icinga Web offers a lot of nice tools. One thing we still want to examine, the so-called DataSources. We integrate the ArrayDatasource and add another function to our library code:

```php
<?php

use Icinga\Data\DataArray\ArrayDatasource;

// ...

    public static function selectFiles($path)
    {
        $ds = new ArrayDatasource(self::listFiles($path));
        return $ds->select();
    }
```

Then we also change our controller very easily:


```php
<?php
$query = Directory::selectFiles(
    $this->Module()->getBaseDir()
)->order('type')->order('name');

$this->view->files = $query->fetchAll();
```

## Task 1
Rebuild the list so that you can sort it up or down by mouse click.

## Additional task

```php
<?php

$editor = Widget::create('filterEditor')->handleRequest($this->getRequest());
$query->applyFilter($editor->getFilter());
```

## Autorefresh

As a monitoring interface, it goes without saying that Icinga Web provides a reliable and stable Autorefresh function. This can be conveniently controlled from the controllers:

```php
<?php

$this->setAutorefreshInterval(10);
```

## Task 2

Our file list should be updated automatically, the detail information also. Show the modification time of a file (`$file->getMtime()`) and use the `timeSince` helper to represent the time. Change a file on the hard drive and see what happens. How can that be explained?

# Configuration

Whoever develops a module would like to be able to configure it. Configuration for a module is stored under `/etc/icingaweb/modules/<modulename>`. What is found there in a `config.ini` is accessible in the controller as follows:

```php
<?php
$config = $this->Config();

/*
[section]
entry = "value"
*/
echo $config->get('section', 'entry');

// Returns "default" because "no entry" does not exist:
echo $config->get('section', 'no entry', 'default value');

// Reads from the special.ini instead of the config.ini:
$config = $this->Config('special');
```

## Task
The base path for the list controller of our training module should be configurable. If no path is configured, we continue to use our module directory.

# Translations
For a detailed description of the translation options, we open the documentation for the `translation` module. Here are the individual steps:

```php
<h1><?= $this->translate('My files') ?></h1>
```

    apt-get install gettext poedit
    icingacli module enable translation
    icingacli translation refresh module training de_DE
    # Translate with Poedit
    icingacli translation compile module training de_DE


# Using Icinga Web logic in third party software?

With Icinga Web 2, we do not just want to make the integration of third-party software as easy as possible. We also want it easy for others to use Icinga Web logic in their software.

This is basically the following call in any PHP file:

```php
<?php

require_once 'Icinga/Application/EmbeddedWeb.php';
Icinga\Application\EmbeddedWeb::start();
```

Finished! No authentication, no bootstrapping of the full web interface. But anything that exists on library code can be used.

## Task
Create an additional PHP file that embeds Icinga Web 2. Then use the directory handling library from your training module.

# Free Lab

You really arrived here? In a single day? Respect. The speaker now has guaranteed an exciting final exercise to finish the day with a practical example and many new tricks.

Have fun with Icinga Web 2 !!!

