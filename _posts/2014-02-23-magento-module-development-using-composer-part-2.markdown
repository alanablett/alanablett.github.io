---
title:  Magento module development using composer part 2
date:   2014-02-23
excerpt: In this lesson we'll write our module code, then push it up to github ready to pull into our future magento builds.
---

[Part 1 available here][part-1].

Here’s what we’ll be covering this time:

* [Creating our config.xml configuration](#creating-our-configxml-configuration)
* [Creating our model and helper](#creating-our-model-and-helper)
* [Creating our api.xml configuration](#creating-our-apixml-configuration)
* [Testing our new module out](#testing-our-new-module-out)
* [Pushing our module to github](#pushing-our-module-to-github)
* [Conclusions](#conclusions)

## Recap

Last time we had created our module, pulled the module into our dummy-store and were able to see it appearing in our list of installed modules. You should have a directory structure something like this

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

**IMPORTANT NOTE**: For the rest of this article I will assume that you are working in the ```dummy-store/vendor/alanablett/api-extensions``` folder.

## Creating our config.xml configuration

Lets add some configuration to our module. We’re going to add a helper and model which will perform the actual logic of our module. Add the following to ```app/code/local/Alanablett/Apiextensions/etc/config.xml```:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<config>
    <modules>
        <Alanablett_Apiextensions>
            <version>0.0.1</version>
        </Alanablett_Apiextensions>
    </modules>
    <global>
        <helpers>
            <alanablett_apiextensions>
                <class>Alanablett_Apiextensions_Helper</class>
            </alanablett_apiextensions>
        </helpers>
        <models>
            <alanablett_apiextensions>
                <class>Alanablett_Apiextensions_Model</class>
            </alanablett_apiextensions>
        </models>
    </global>
</config>
```

Here we are simply defining the class prefixes for our models and helpers. Now lets actually implement the logic of our module.

## Creating our model and helper

The logic for this module is very simple. We will add a new method to the Api so that we can call ```getTaxRuleDataByName```, give it the tax rule to search for, and expect some data back from that call. Lets start by actually implementing the logic for this call. Create the following file ```app/code/local/Alanablett/Apiextensions/Model/Taxrules/Api.php```.

```php
<?php
 
class Alanablett_ApiExtensions_Model_Taxrules_Api {
     
    /**
     * Get tax rule data based on a tax rule name
     * @param  String $ruleName 
     * @return Array
     */
    public function getTaxRuleDataByName($ruleName)
    {
        $rulesCollection = Mage::getResourceModel('tax/calculation_rule_collection')
            ->addFieldToFilter('code', $ruleName);
         
        if ($rulesCollection->count() === 0) return array();
         
        return $this->getTaxRuleData($rulesCollection->getFirstItem());
    }
     
    /**
     * Gets the data of an individual tax rule
     * @param  Mage_Tax_Model_Calculation_Rule $taxRule 
     * @return Array 
     */
    private function getTaxRuleData($taxRule)
    {
        $taxRule = Mage::getModel('tax/calculation_rule')->load($taxRule->getId());
        return $this->getTaxRates($taxRule);
    }
     
    /**
     * Get the tax rates for the rule
     * @param  Mage_Tax_Model_Calculation_Rule $taxRule
     * @return Array
     */
    public function getTaxRates($taxRule)
    {
        $data = array();
         
        $rateIds = $taxRule->getRates();
        $ratesCollection = Mage::getResourceModel('tax/calculation_rate_collection')
            ->addFieldToFilter('tax_calculation_rate_id', $rateIds);
         
        foreach($ratesCollection as $rate)
        {
            $data[] = array(
                'code' => $rate->getCode(),
                'taxCountryId' => $rate->getTaxCountryId()
            );
        }
         
        return $data;
    }
}
```

This class will be the one used by our Api call. In particular we will be hooking up the ```getTaxRuleDataByName``` function in the ```api.xml``` configuration file in the next step.

Now lets create our helper file. We won’t actually be implementing anything ourselves here, it just extends the core abstract helper class. Create your helper as follows ```app/code/local/Alanablett/Apiextensions/Helper/Data.php```.

```php
<?php
 
class Alanablett_Apiextensions_Helper_Data extends Mage_Core_Helper_Abstract {}
```

## Creating our api.xml configuration

Our module ```api.xml``` configuration file will map the name that we want to call on the Api, to the function of our new model. Create the file in our modules etc folder at ```app/code/local/Alanablett/Apiextensions/etc/api.xml``` with the following content:

```xml
<config>
    <api>
        <resources>
            <alanablett_apiextensions_taxrules translate="title" module="alanablett_apiextensions">
                <title>Alanablett Tax Rule API Extensions</title>
                <model>alanablett_apiextensions/taxrules_api</model>
                <acl>alanablett_apiextensions/taxrules</acl>
                <methods>
                    <getTaxRuleDataByName translate="title" module="alanablett_apiextensions">
                        <title>Returns the tax rules of the website</title>
                        <acl>alanablett_apiextensions/taxrules/allaccess</acl>
                    </getTaxRuleDataByName>
                </methods>
            </alanablett_apiextensions_taxrules>
        </resources>
         
        <resources_alias>
            <taxrules>alanablett_apiextensions_taxrules</taxrules>
        </resources_alias>
         
        <acl>
            <resources>
                <alanablett_apiextensions translate="title" module="alanablett_apiextensions">
                    <title>Alanablett API Extensions</title>
                    <sort_order>100</sort_order>
                    <taxrules translate="title" module="alanablett_apiextensions">
                        <title>Alanablett Tax Rules</title>
                        <sort_order>100</sort_order>
                        <allaccess translate="title" module="alanablett_apiextensions">
                            <title>alanablett Tax Rules (all access)</title>
                            <sort_order>10</sort_order>
                        </allaccess>
                    </taxrules>
                </alanablett_apiextensions>
            </resources>
        </acl>
    </api>
</config>
```

There are a couple of interesting parts in this file. Inside the ```config > api > resources``` node we define the model and method name that we want to add as an endpoint. We then also add an alias in ```config > api > resources_alias``` called ```taxrules``` and map it to the full name of ```alanablett_apiextensions_taxrules```. This is just a convenience for when we call the Api. It means we can call ```taxrules.getTaxRuleDataByName``` instead of ```alanablett_apiextensions_taxrules.getTaxRuleDataByName```. The final piece of configuration ensures that we can add appropriate acl restrictions for web service users.

At this point we should now have a fully working module ready to try out. Before we do, we’ll need to create a web service user in the admin area and ensure they have access to our new resource.

Log in to the administration area and go to ```System > Web Services > SOAP/XML-RPC – Roles``` and create a new role. When doing so you should be able to see our new resource in the Role Resources tab. Make sure you give access to that resource for this group. Now go ahead and set up a new user ```System > Web Services > SOAP/XML-RPC – Users``` and assign them to the newly created group. Now we’re ready to give the api a call.

## Testing our new module out

To test our new Api endpoint create a simple stand alone php script with the following:

```php
<?php
 
$client = new SoapClient('http://local.domain/api/soap/?wsdl');
$session = $client->login('username', 'password');
 
$args = array('ruleName' => 'Retail Customer-Taxable Goods-Rate 1');
$taxRules = $client->call($session, 'taxrules.getTaxRuleDataByName', $args);
var_dump( $taxRules );
 
$client->endSession($session);
```

Make sure you swap out the url, username and password with those relevant to your own install. Provided you have a tax rule called ```Retail Customer-Taxable Goods-Rate 1``` (if you installed the dummy data with N98-magerun then you will), then you should receive back the codes and country ids for the queried tax name.

```
array (size=2)
  0 => 
    array (size=2)
      'code' => string 'US-CA-*-Rate 1' (length=14)
      'taxCountryId' => string 'US' (length=2)
  1 => 
    array (size=2)
      'code' => string 'US-NY-*-Rate 1' (length=14)
      'taxCountryId' => string 'US' (length=2)
```

Now that you have a fully working module, add the new files to your repo and commit with a suitable message.

```
$ git add .
$ git commit -m "Added model helper and api config"
```

## Pushing our module to github

Lets push our module up to github so we can re-use it in other projects. To do that we’ll need to do two things. Change the origin url of our module inside the vendor directory, and set the github url in our dummy-store composer.json file to use github instead of our local file system. Before we get started, make sure you have create a new repository on github ready to push to.

Navigate to ```dummy-store/vendor/alanablett/api-extensions``` and change the origin url of the repository to point to the newly created github repo. Then push everything up.

```
$ git remote set-url origin git@github.com:alanablett/api-extensions.git
$ git push origin --all
 
Counting objects: 33, done.
Delta compression using up to 2 threads.
Compressing objects: 100% (19/19), done.
Writing objects: 100% (33/33), 3.25 KiB, done.
Total 33 (delta 2), reused 0 (delta 0)
To git@github.com:alanablett/api-extensions.git
 * [new branch]      master -> master
```

Now modify your dummy-store composer.json file to use the new github repo rather than the local file system. In my case it looks like this:

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
            "url": "https://github.com/alanablett/api-extensions"
        }
    ],
    "extra":{
        "magento-root-dir": "magento/"
    }
}
```

Finally, to pull the repo in from github rather than your local file system do a composer update from the root of your dummy-store.

```
$ composer update
 
Loading composer repositories with package information
Updating dependencies (including require-dev)                                                
  - Updating alanablett/api-extensions dev-master (5f26c14 => 153637f)
    Checking out 153637fd67472b39e4eff2e961fd84bfec8a3b65
 
Writing lock file
Generating autoload files
```

You just created your first Magento module using composer!

## Conclusions

I wrote this post/tutorial because although there is a lot of information on how to pull Magento modules into your install using composer, there are not many on how to actually create them. I’m sure this is not the only way of doing so, and would urge anybody with better methods to add to comments below, I really would appreciate any input. For example, the initial step of creating a local file system repository with path mappings, then pulling that into the store, before finally changing to use a github repository does seem a little long winded in my eyes. Indeed, if we knew the structure of our module beforehand we could avoid those initial steps by pushing our module up to github, then pulling straight down into our store, and working on it as normal in the vendor directory. I’d be very interested to hear how others are approaching this and how I can improve my own work flow.




[part-1]: {% post_url 2014-02-15-magento-module-development-using-composer-part-1 %}