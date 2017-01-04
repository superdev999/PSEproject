# Magento Project Mess Detector
Author: superdev999
```
n98-magerun.phar | grep mpmd
mpmd
 mpmd:codepooloverrides                  Find all code pool overrides
 mpmd:corehacks                          Find all core hacks
 mpmd:dependencycheck                    Find dependencies
 mpmd:dependencycheck:configured         Returns the list of modules that are CONFIGURED in these module's config xml.
 mpmd:dependencycheck:graph:class        Creates a class graph
 mpmd:dependencycheck:graph:configured   Creates a graph of all configured depencencies.
 mpmd:dependencycheck:graph:module       Creates a module graph
 mpmd:dependencycheck:verify             Checks if found dependencies match a given module's xml file

```

## Table of Contents

* [Installation](#installation)
* **Compare files**
	* [`mpmd:corehacks`](#command-mpmdcorehacks)
	* [`mpmd:codepooloverrides`](#command-mpmdcodepooloverrides)
* **[Dependency Checker](#dependency-checker)**
	* [How does it work?](#how-does-it-work)
	* [Why?](#why)
	* [Parsers](#parsers)
	* [Handlers](#handlers)
	* [Specifying sources](#specifying-sources)
	* Commands: Tables
		* [`mpmd:dependencychecker`](#command-mpmddependencychecker)
		* [`mpmd:dependencychecker:verify`](#command-mpmddependencycheckerverify)
		* [`mpmd:dependencychecker:configured`](#command-mpmddependencycheckerconfigured)
	* Commands: Graphs
		* [`mpmd:dependencychecker:graph:module`](#command-mpmddependencycheckergraphmodule)
		* [`mpmd:dependencychecker:graph:class`](#command-mpmddependencycheckergraphclass)
		* [`mpmd:dependencychecker:graph:configured`](#command-mpmddependencycheckergraphconfigured)
	* [How to run the unit tests](#how-to-run-the-unit-tests)
	* [Interesting Graphviz commands](#interesting-graphviz-commands)


## Installation

There are a few options. You can check out the different options in the [MageRun docs](http://magerun.net/introducting-the-new-n98-magerun-module-system/).

Here's the easiest:

0. Install n98-magerun if you haven't already. Find the instructions on the [n98-magerun wiki](https://github.com/netz98/n98-magerun/wiki/Installation-and-Update).

1. Create `~/.n98-magerun/modules/` if it doesn't already exist. (or `/usr/local/share/n98-magerun/modules` or put your modules inside your Magento instance in `lib/n98-magerun/modules` if you prefer that)
```
mkdir -p ~/.n98-magerun/modules/
```
2. Clone the mpmd repository in there. 
```
git clone git@github.com:AOEpeople/mpmd.git ~/.n98-magerun/modules/mpmd
```
3. It should be installed. To verify that it was installed correctly, check if the new commands show up in the command list:
```
n98-magerun.phar | grep mpmd
```

## Commands

### Command `mpmd:corehacks`
```
Usage:
 mpmd:corehacks [--format[="..."]] pathToVanillaCore [htmlReportOutputPath] [skipDirectories]

Arguments:
 pathToVanillaCore     Path to Vanilla Core used for comparison
 htmlReportOutputPath  Path to where the HTML report will be written
 skipDirectories       ':'-separated list of directories that will not be considered (defaults to '.svn:.git')
```

This command requires a vanilla version of Magento (same version and edition! Run `n98-magerun.phar sys:info` for more details) to be present somewhere in the filesystem.
It will then traverse all project files and compare them with the original files. 
This command will also be able to tell the difference between whitespace or code comments changes and real code changes.
It will generate a HTML report that also includes the diffs.

```
$ cd /var/www/magento/htdocs
$ n98-magerun.phar mpmd:corehacks /path/to/vanilla/magento /path/to/report.html
Comparing project files in 'var/www/magento/htdocs' to vanilla Magento code in '/path/to/vanilla/magento'...
+----------------------+-------+
| Type                 | Count |
+----------------------+-------+
| differentFileContent | 2     |
| identicalFiles       | 16049 |
| fileMissingInB       | 1     |
| sameFileButComments  | 0     |
+----------------------+-------+
Generating detailed HTML Report
```

Report preview:

![Image](/docs/img/corehacks.jpg)

### Command: `mpmd:codepooloverrides`
```
Usage:
 mpmd:codepooloverrides [--format[="..."]] [htmlReportOutputPath] [skipDirectories]

Arguments:
 htmlReportOutputPath  Path to where the HTML report will be written
 skipDirectories       ':'-separated list of directories that will not be considered (defaults to '.svn:.git')
```

This command will compare all code pools with each other and detect files that are overriding each other.
It will show identical files (What's the point of these? But yes, seen projects where this happened), copied files with changes in comments and whitespace only,
and real changes. Of course with diff...
 
Report preview:

![Image](/docs/img/codepooloverride.jpg)




## Dependency Checker

![](/docs/img/graph_intro.png)

The dependency checker parses one or more files or directories and detects PHP classes that are being "used" there. 

### How does it work?

The dependency checker is a "semi" static code analysis tool. That means it does the job without actually executing any of the PHP code you're pointing it to, but it does need that module to be installed correctly and it will invoke the Magento framework to resolve classpaths (`catalog/product` -> `Mage_Catalog_Model_Product`)

### Why?

While tools like [pdepend](http://pdepend.org/) exist those tools don't know anything about Magento in general, Magento's special classpaths and where they are being used. Also sometimes the numbers generated by pdepend are a little overwhelming and after all what are you going to do knowing that you're module has an avarage cyclomatic complexity of x? 

The **mpmd:dependencychecker** will 
- help you to detect other Magento modules that the module you're currently looking at depends on.
- check you module's configuration and let's you know if all actual dependencies are declared correctly and will also show if the module is declaring dependencies that this tool didn't detect (Note: this tool isn't perfect, so please double check before removing any dependencies)
- show you the relations between modules and individual classes
- produce pretty graphs that you can render with [Graphviz](http://www.graphviz.org/)
- help you detect code that requires some refactoring and will help you create cleaner - less dependent - modules in the first place. 

#### How to add your own parser

Add a new parser via n98-magerun's YAML configuration

```
commands:
  Mpmd\Magento\DependencyCheckCommand:
    parsers:
      - Mpmd\DependencyChecker\Parser\Tokenizer
      - Mpmd\DependencyChecker\Parser\Xpath
      - (... add your parser here ...)
```

All parsers need to implement [`Mpmd\DependencyChecker\Parser\ParserInterface`](src/Mpmd/DependencyChecker/Parser/ParserInterface.php). Also checkout the [`AbstractParser`](src/Mpmd/DependencyChecker/Parser/AbstractParser.php) that implements that interface and might be a good starting point. 

#### How to add your own handler

Add a new handler via [n98-magerun's YAML configuration](https://github.com/netz98/n98-magerun/wiki/Config) (also checkout [n98-magerun's documentation for custom commands](https://github.com/netz98/n98-magerun/wiki/Add-custom-commands))

```
commands:
  Mpmd\Magento\DependencyCheckCommand:
	Mpmd\DependencyChecker\Parser\Tokenizer:
      handlers:
        - Mpmd\DependencyChecker\Parser\Tokenizer\Handler\WhitespaceString
        - Mpmd\DependencyChecker\Parser\Tokenizer\Handler\Interfaces
        - (... add your tokenizer handler here ...)
    <Parser>:
      handlers:
        - (... add your <parser> handler here ...)    
```

All handlers need to implement [`Mpmd\DependencyChecker\HandlerInterface`](src/Mpmd/DependencyChecker/HandlerInterface.php). Also checkout the [`AbstractHandler`](src/Mpmd/DependencyChecker/AbstractHandler.php) and the inheriting abstract handlers for the [tokenizer parser](src/Mpmd/DependencyChecker/Parser/Tokenizer/Handler/AbstractHandler.php) and the [xpath parser](src/Mpmd/DependencyChecker/Parser/Xpath/Handler/AbstractHandler.php) that implement that interface and might be a good starting point. 
