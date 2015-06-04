---
title:  Decoupling your Magento module code from the core
date:   2015-06-04
excerpt: How to remove hard coded dependencies from Magento core in your own modules, and why it's a good thing.
---

I saw [James Cowie][James Cowie] using this pattern on the [second magento fireside talk about testing][]. Its pretty cool so I have broken it down here for both my own reference, and for anybody else that might be interested.

## What's the problem?

Typically we might write our models like so

```php
<?php

class Foo_Bar_Model_Baz extends Mage_Catalog_Model_Abstract
{
    
    public function getDatabaseValues()
    {
        return $this->_databaseAdapter->getDatabaseValues();
    }
    
}
```

Take a look at the following code:

```php
<?php

class Foo_Bar_Model_Baz extends Mage_Catalog_Model_Abstract
{
    
    private $_databaseAdapter;
    
    public function __construct($databaseAdapter = null)
    {
        $this->_databaseAdapter = is_null($databaseAdapter)
            ? new Foo_Bar_Model_Baz_Adapter_Database()
            : $databaseAdapter;
    }
    
    public function getDatabaseValues()
    {
        $values = $this->_databaseAdapter->getDatabaseValues();
        
        
        
        return $this->_databaseAdapter->getDatabaseValues();
    }
    
}
```

```php
<?php

class Foo_Bar_Model_Baz_Adapter_Database
{
    
    public function getDatabaseValues()
    {
        return Mage::getModel('core/config')->getDistroServerVars();
    }
    
}
```

[James Cowie]: http://twitter.com/jcowie
[second magento fireside talk about testing]: https://www.youtube.com/watch?v=fee1CGNPWIg