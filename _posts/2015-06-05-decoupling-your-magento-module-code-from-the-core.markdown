---
title:  Decoupling your Magento module code from the core
date:   2015-06-05
excerpt: How to remove hard coded dependencies from Magento core in your own modules, and why it's a good thing.
---

I saw [James Cowie][James Cowie] using this pattern on the [second magento fireside talk about testing][]. Its pretty cool so I have broken it down here for both my own reference, and for anybody else that might be interested.

## What's the problem?

Typically a model may look like so

```php
<?php

class Foo_Bar_Model_Baz
{
    
    public function sillyExample($to_merge = array())
    {
        $values = Mage::getModel('core/config')->getDistroServerVars();
        
        if(stripos($values['base_url'], 'localhost')){
            $values = array_merge($values, $to_merge);
        }
        
        return $values;
    }
    
} ?>
```

Our ```sillyExample``` method is used to get some server vars, and merging in our own vars if the base url of our site contains the word localhost. A very silly and simple example, but it shows the issue perfectly.

You might think this isnt too bad; I though the same until recently. The real problem starts when we want to test this model (which we should). Following unit testing best practices, we shouldnt be running any code outside of our class. Remember, unit testing means we are testing in isolation, not calling other parts of our system (thats what integration & acceptance tests are for). We need to get rid of that hard coded ```core/config``` model so that we can mock any calls to it.

## What will a mock do for us?

A mock allows us to simulate calls to other objects. It also allows us to fake the return values of those calls. This way we can control the flow of the system and test various cases. Think about it, if we call our ```core/config``` model, its always going to do the same thing. So if I happened to have my local version running on ```http://testing.dev``` then I will never reach inside my if check.

Once we have our ```core/config``` object mocked, we can force its method ```getDistroServerVars``` to return whatever we want. We will run tests to make sure we go inside the if statement (by making sure the base_url key contains localhost), as well as tests that remain outside the statement, therefore covering all paths through our code.

## Removing the hard coded model with an injected dependency

Lets get to work and remove that call to the ```core/config``` model.

```php
<?php

class Foo_Bar_Model_Baz
{
    
    private $_configAdapter;
    
    public function __construct($configAdapter = null)
    {
        $this->_configAdapter = is_null($configAdapter)
            ? new Foo_Bar_Model_Baz_Adapter_Config()
            : $configAdapter;
    }
    
    public function sillyExample($to_merge = array())
    {
        $values = $this->_configAdapter->getDistroServerVars();
        
        if(stripos($values['base_url'], 'localhost')){
            $values = array_merge($values, $to_merge);
        }
        
        return $values;
    }
    
}?>
```

```php
<?php

class Foo_Bar_Model_Baz_Adapter_Config
{
    
    public function getDistroServerVars()
    {
        return Mage::getModel('core/config')->getDistroServerVars();
    }
    
}?>
```

Take a look at our ```Foo_Bar_Model_Baz``` class now. Upon instantiation we can now pass in a config adapter. If we dont pass one, we automatically use ```Foo_Bar_Model_Baz_Adapter_Config``` which is our implemetation using the ```core/config``` model. However, it also means we can create a mock of the adapter and pass that into our constructor instead.

## Testing Foo\_Bar\_Model\_Baz

Here is what a typical test might look like. I have added some docs which may help understand the process if testing is new to you.

```php
<?php

class Foo_Bar_Model_Baz_Test extends PHPUnit_Framework_TestCase
{
    /**
     * Use the test annotation so we dont have to prefix all of
     * our test functions with the word test. It just makes
     * them more readable.
     * 
     * @test
     */
    public function it_merges_an_array_when_the_base_url_contains_localhost()
    {
        
        /**
         * ARRANGE
         * 
         * Create a mock of our adapter. We also force
         * getDistroServerVars to return a specific value.
         */
        $configMock = $this->getMockBuilder('Foo_Bar_Model_Baz_Adapter_Config')
                          ->getMock();
        
        $configMock->method('getDistroServerVars')
                  ->willReturn(array(
                       'base_url' => 'http://madeup-localhost.dev'
                  ));
        
        /**
         * Create our class under test and pass in our mock object.
         */
        $baz_model = new Foo_Bar_Model_Baz($configMock);
        
        /**
         * Here we define a simple array that we expect to get
         * in return from the call to our sillyExample method
         */
        $expected = array(
            'base_url' => 'http://madeup-localhost.dev',
            'key' => 'value'
        );
        
        /**
         * ACT
         * 
         * Perform the actual call to our method and store the
         * result.
         */
        $actual = $baz_model->sillyExample(array('key' => 'value'));
        
        /**
         * ASSERT
         * 
         * Check that the value we expected and the value returned
         * from our sillyExample method match.
         */
        $this->assertSame($expected, $actual);
    }
}?>
```

We would also probably perform tests where localhost does not appear in the url, and where no values are passed in to be merged. This would make sure we cover our code and any posible edge cases. Its a matter of testing what will help you sleep at night really. If there's potential for it to break, then it should probably be tested.

## Summary

So there we have it. Move those core calls into separate classes, then inject them into your domain to keep things super clean. It'll make things a lot easier to test!

[James Cowie]: http://twitter.com/jcowie
[second magento fireside talk about testing]: https://www.youtube.com/watch?v=fee1CGNPWIg