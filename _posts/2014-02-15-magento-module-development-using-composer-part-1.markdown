---
title:  Magento module development using composer part 1
date:   2014-02-15
excerpt: In this lesson we'll set some ground work and prepare our environment to build the module.
redirect_from: "/blog/magento-module-development-using-composer-part-1/"
---

[Part 2 available here][part-2].

In the first part of this series of posts we’re going to set some ground work out and prepare our environment ready to create our module. We’ll be covering the following:

* [What we’ll be building](#what-well-be-building)
* [Our Approach](#our-approach)
* [Installing Composer](#installing-composer)
* [Installing N98 Magerun](#installing-n98-magerun)
* [Installing Magento](#installing-magento)
* [Creating our module repository](#creating-our-module-repository)
* [Pulling our repository into Magento](#pulling-our-repository-into-magento)

## What We’ll Be Building

To demonstrate how to use composer I thought it would be fun to extend the Magento API. It is something I had to do in a recent project, and it gives us plenty of scope to expose Magento data that might not be available as standard. Ever wanted to pull your tax rates from your Magento store? Just extend the API and you’re done.

## Our Approach

Developing a module using composer requires us to separate things out more than usual. This is of-course a good thing. Being able to work on a module away from the Magento filesystem makes it easier to work on. It not only de-clutters the filesystem, but personally it allows me to concentrate on the module as the single entity that it should be.

For the sake of this series we will be using the following directory structure

```
├── api-extensions
└── dummy-store
    └── magento
```

Our module is going to be created and tracked in our ```api-extensions``` folder. Our magento store which will be pulling in our module, will be stored and tracked inside the ```dummy-store``` folder. Notice that we actually have a sub-folder for our magento installation. This allows us to keep the ```composer.json``` file which (will contain a list of our modules), outside of our main installation folder. It also means our modules will be in a vendor folder outside of our main installation. If that is a little confusing right now, don’t worry it will all make sense once we’re done.

## Installing Composer

If you’ve never heard or used composer before then I recommend you head over [to the official composer site][composer] and take a look over the documentation. There are thousands of posts out there that explain what it is and how it works, so I’ll skip over that part. Needless to say, we’ll make sure you have it installed before continuing on. I prefer to install it globally, but install to suit your own needs.

```
$ curl -sS https://getcomposer.org/installer | php
$ mv composer.phar /usr/local/bin/composer
```

If all went well, you should now be able to check the current version of composer that you’re running. Close your terminal, start up again, then run the following:

```
$ composer -V
```

## Installing N98 Magerun

Before we start working on our module we’ll install the excellent [n98-magerun][n98-magerun] tool. For those of you that haven’t used it before, it is a development tool-chain designed to speed up working with Magento. It will save you no end of time, so lets install it globally so you can run it inside any of your Magento installations.

```
$ wget https://raw.github.com/netz98/n98-magerun/master/n98-magerun.phar
$ chmod +x ./n98-magerun.phar
$ sudo mv ./n98-magerun.phar /usr/local/bin/
```

Now, close your terminal and fire it back up before running the command

```
$ n98-magenrun.phar
```

You should see a large list of available commands. Take a minute to glance over them before moving on.

## Installing Magento

Ok we’re now going to install Magento so that we have something to test our new module on. So we need to go off to magento.com, sign-in, download Magento, extract the file, run the install etc etc right? **NOPE**! Lets use N98-Magerun to do it for us. Our only pre-requisite is that we have a blank database ready to install to. If you don’t already, then do that now. Navigate to your ```dummy-store``` folder and run the following:

```
$ n98-magenrun.phar install
```

n98-magerun will now ask you a series of questions, which version of Magento you would like to install, database credentials, if you’d like sample data installed etc. Go through the process and you should eventually have Magento fully installed onto the domain you specified. For sanities sake, open up the browser and give it a check to make sure everything is ok. Assuming they are, we’re ready to create our module repository.

## Creating our module repository

We’ll begin by defining our modules ```composer.json``` file. This is just a basic file that describes the module to composer so that it knows what to do with it once it is called. There is actually a lot more to these files than that, but for our sake that is all we are concerned about. So jump into the ```api-extensions``` directory and create a file in there called ```composer.json```

```
├── api-extensions
│   └── composer.json
└── dummy-store
    └── magento
```

Add the following content:

```json
{
    "name":"alanablett/api-extensions",
    "type":"magento-module",
    "require":{
        "magento-hackathon/magento-composer-installer":"*"
    },
    "extra":{
        "map":[
            ["app/etc/modules/Alanablett_Apiextensions.xml", "app/etc/modules/Alanablett_Apiextensions.xml"],
            ["app/code/local/Alanablett/Apiextensions/", "app/code/local/Alanablett/Apiextensions/"]
        ]
    }
}
```

Lets take a look at each of these definitions in turn.

* **name**: This is the name that we will reference our module by. Our dummy-store will use this name in its own composer.json file. For development we will be pulling this module in from the local file-system, but eventually it could come from an online repository.
* **type**: This tells our required module (magento-composer-installer) that we need it to take care of installing this module. Without this type being set, magento-composer-installer will not do anything with it.
* **require**: This is a standard composer require statement. It lets us depend on the magento-composer-installer, which in turn will actually perform the installation of this module’s files.
* **extra**: In order to let magento-composer-installer know what to do with our files, we need to give it a mapping to work with. This can be in the form of a package.xml file, a modman file, or as we have done here directly in the modules composer.json file. I recommend reading the article [How to make Magento Extensions work with Composer][mage-with-composer] by the excellent [Vinai Kopp][vinai] if you want to know more about the alternatives.

Now that you have something inside your ```api-extensions``` directory, go ahead and initialise a new repository. The add and commit the ```composer.json file```.

```
$ git init
Initialized empty Git repository in /home/alan/Sites/tutorial/api-extensions/.git/
$ git add .
$ git commit -m "Added composer.json file"
[master (root-commit) 8aaae65] Added composer json file
 1 file changed, 13 insertions(+)
 create mode 100644 composer.json
```

Ok now we have a repository, but we have no code. Let’s quickly scaffold the module ready to pull into our dummy-store. N98-magerun includes a command to do this for us, but here is where we hit a little snag. Since we’re not inside a Magento installation directory when trying to run the command, N98-magerun will throw us an error. According to the documentation we should be able to provide a different directory for our magento installation folder, but I’ve not got this to work. So we’re going to get our hands dirty and create it all ourselves.

Lets create the module xml file which will let magento know about our module. Assuming you’re still inside the ```api-extensions``` directory:

```
$ mkdir -p app/etc/modules
```

Now create the file ```api-extensions/app/etc/modules/Alanablett_Apiextensions.xml```. Inside that file define our module like this:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<config>
    <modules>
        <Alanablett_Apiextensions>
            <active>true</active>
            <codePool>local</codePool>
            <depends>
                <Mage_Api />
            </depends>
        </Alanablett_Apiextensions>
    </modules>
</config>
```

Now Magento knows where to find our module, lets add some code to define it. Assuming you’re back inside the ```api-extensions``` directory:

```
$ mkdir -p app/code/community/Alanablett/Apiextensions/etc
```

Now add you configuration file in ```app/code/local/Alanablett/Apiextensions/etc/config.xml```

```xml
<?xml version="1.0" encoding="UTF-8"?>
<config>
    <modules>
        <Alanablett_Apiextensions>
            <version>0.0.1</version>
        </Alanablett_Apiextensions>
    </modules>
</config>
```

We’ve now got enough to get us going. Add the new files to git, and commit them.

```
$ git add .
$ git commit -m "Added module config files"
[master 5f26c14] Added module config files
 2 files changed, 20 insertions(+)
 create mode 100644 app/code/local/Alanablett/Apiextensions/etc/config.xml
 create mode 100644 app/etc/modules/Alanablett_Apiextensions.xml
```

## Pulling our repository into Magento

So far we have our Magento install in one folder, and our new module in another. Its time to bring the two together. Ordinarily we would add modules into our stores composer.json file, and they would be pulled from a repository somewhere online. However, in this case we want to pull from a repository that exists on the local file system. We would also like to be able to directly work on the module whilst it in actually installed in our Magento shop. Lets add a reference to our module in our dummy store composer file. Assuming you are inside the ```dummy-store``` directory, add the following to ```composer.json```:

```json
{
    "require": {
        "magento-hackathon/magento-composer-installer":"*",
        "alanablett/api-extensions":"dev-master"
    },
    "repositories": [
        {
            "type": "vcs",
            "url": "https://github.com/magento-hackathon/magento-composer-installer"
        },
        {
            "type": "vcs",
            "url": "/home/alan/Sites/tutorial/api-extensions"
        }
    ],
    "extra":{
        "magento-root-dir": "magento/"
    }
}
```

Again lets take a look at what some of this means:

* **require**: This is telling our store which modules we would like to pull in. In this case we want the ever useful ```magento-composer-installer``` as well as our own module ```api-extensions```. Remember, that is the name that we gave it in its own composer file so that is the name we must use here.
* **repositories**: This tells composer where to look for our repositories. There are a number of different locations that composer will look for them, here we are specifying two. ```https://github.com/magento-hackathon/magento-composer-installer``` will be used to find ```magento-composer-installer```. For own our repository, we’re pointing to our local file system. You will need to change this to reflect the path on your own system.
* **extra**: Here we define where our magento installation folder is. This extra piece of information is used by ```magento-composer-installer``` when mapping our modules into our filesystem.

With that added, we can install our dummy-store dependencies with the following command.

```
$ composer install
```

Give that a moment to run. Once it’s finished we can confirm that our module has installed in a couple of different ways. You can check that the file ```magento/app/etc/modules/Alanablett_Apiextensions.xml``` exists (this will by default actually be a symbolic link to the vendor directory not a real file). Or we can user N98-magerun to show us a list of installed modules:

```
$ cd magento
$ n98-magerun.phar sys:modules:list
.
.
.
| local     | Alanablett_Apiextensions | 0.0.1      | active   |
.
.
.
```

**Success!**

Take a look around at the new filesystem. We have a vendor directory that contains our module, and we have our mapped files inside the magento folder which symbolicly link back to the vendor directory. That means we can work from within that vendor directory just on our particular module if we wanted to, or inside the magento folder if you prefer. Then we can push those changes back to our local repository or to a version hosted online if we change our remote repositories.

```
├── api-extensions
│   ├── app
│   └── composer.json
└── dummy-store
    ├── composer.json
    ├── composer.lock
    ├── magento
    └── vendor
```

## Thats A Wrap

So there we have it. We’ve downloaded and utilised some awesome tools. We’ve installed Magento from the command line. We’ve created our own module away from our magento core. Finally we’ve pulled our module into our example Magento shop with the help of magento-composer-installer.

In the next lesson we’re going to get into the finer details of our module. We’ll be extending the core API with our own method to fetch out the tax rules set in our store. Nothing too exciting, but the principle will allow you to expose anything available in Magento.

[Part 2 available here][part-2].

[composer]: https://getcomposer.org/
[n98-magerun]: http://magerun.net/
[mage-with-composer]: http://magebase.com/magento-tutorials/how-to-make-magento-extensions-work-with-composer/
[vinai]: https://twitter.com/VinaiKopp
[part-2]: {% post_url 2014-02-23-magento-module-development-using-composer-part-2 %}