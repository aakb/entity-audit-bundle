# EntityAudit Extension for Doctrine2

[![Build Status](https://travis-ci.org/simplethings/EntityAudit.svg?branch=master)](https://travis-ci.org/simplethings/EntityAudit)

This extension for Doctrine 2 is inspired by [Hibernate Envers](http://www.jboss.org/envers) and
allows full versioning of entities and their associations.

## How does it work?

There are a bunch of different approaches to auditing or versioning of database tables. This extension
creates a mirroring table for each audited entitys table that is suffixed with "_audit". Besides all the columns
of the audited entity there are two additional fields:

* rev - Contains the global revision number generated from a "revisions" table.
* revtype - Contains one of 'INS', 'UPD' or 'DEL' as an information to which type of database operation caused this revision log entry.

The global revision table contains an id, timestamp, username and change comment field.

With this approach it is possible to version an application with its changes to associations at the particular
points in time.

This extension hooks into the SchemaTool generation process so that it will automatically
create the necessary DDL statements for your audited entities.

## Installation (In Symfony2 Application)

### Installing the bundle

Simply run assuming you have installed composer.phar or composer binary:

``` bash
$ php composer.phar require simplethings/entity-audit-bundle
```

### Enable the bundle

Finally, enable the bundle in the kernel:

``` php
// app/AppKernel.php

public function registerBundles()
{
    $bundles = array(
        //...
        new SimpleThings\EntityAudit\SimpleThingsEntityAuditBundle(),
        //...
    );
    return $bundles;
}
```

### Configuration

Load extension "simple_things_entity_audit" and specify the audited entities (yes, that ugly for now!)

##### app/config/config.yml
```yml
simple_things_entity_audit:
    audited_entities:
        - MyBundle\Entity\MyEntity
        - MyBundle\Entity\MyEntity2
```
If you need to exclude some entity properties from triggering a revision use:

##### app/config/config.yml
```yml
simple_things_entity_audit:
    global_ignore_columns:
        - created_at
        - updated_at
```

### Creating new tables

Call the command below to see the new tables in the update schema queue.

```bash
./app/console doctrine:schema:update --dump-sql 
```

**Notice**: EntityAudit currently **only** works with a DBAL Connection and EntityManager named **"default"**.


## Installation (Standalone)

For standalone usage you have to pass the entity class names to be audited to the MetadataFactory
instance and configure the two event listeners.

```php
use Doctrine\ORM\EntityManager;
use Doctrine\Common\EventManager;
use SimpleThings\EntityAudit\AuditConfiguration;
use SimpleThings\EntityAudit\AuditManager;

$auditconfig = new AuditConfiguration();
$auditconfig->setAuditedEntityClasses(array(
    'SimpleThings\EntityAudit\Tests\ArticleAudit',
    'SimpleThings\EntityAudit\Tests\UserAudit'
));

$auditconfig->setGlobalIgnoreColumns(array(
    'created_at',
    'updated_at'
));

$evm = new EventManager();
$auditManager = new AuditManager($auditconfig);
$auditManager->registerEvents($evm);

$config = new \Doctrine\ORM\Configuration();
// $config ...
$conn = array();
$em = EntityManager::create($conn, $config, $evm);
```

## Usage

Querying the auditing information is done using a `SimpleThings\EntityAudit\AuditReader` instance.

In Symfony2 the AuditReader is registered as the service "simplethings_entityaudit.reader":

```php
class DefaultController extends Controller
{
    public function indexAction()
    {
        $auditReader = $this->container->get('simplethings_entityaudit.reader');
    }
}
```

In a standalone application you can create the audit reader from the audit manager:

```php
$auditReader = $auditManager->createAuditReader($entityManager);
```

### Find entity state at a particular revision

This command also returns the state of the entity at the given revision, even if the last change
to that entity was made in a revision before the given one:

```php
$articleAudit = $auditReader->find(
    'SimpleThings\EntityAudit\Tests\ArticleAudit',
    $id = 1,
    $rev = 10
);
```

Instances created through `AuditReader#find()` are *NOT* injected into the EntityManagers UnitOfWork,
they need to be merged into the EntityManager if it should be reattached to the persistence context
in that old version.

### Find Revision History of an audited entity

```php
$revisions = $auditReader->findRevisions(
    'SimpleThings\EntityAudit\Tests\ArticleAudit',
    $id = 1
);
```

A revision has the following API:

```php
class Revision
{
    public function getRev();
    public function getTimestamp();
    public function getUsername();
}
```

### Find Changed Entities at a specific revision

```php
$changedEntities = $auditReader->findEntitiesChangedAtRevision(10);
```

A changed entity has the API:

```php
class ChangedEntity
{
    public function getClassName();
    public function getId();
    public function getRevisionType();
    public function getEntity();
}
```

### Find Current Revision of an audited Entity

```php
$revision = $auditReader->getCurrentRevision(
    'SimpleThings\EntityAudit\Tests\ArticleAudit',
    $id = 3
);
```

## Setting the Current Username

Each revision automatically saves the username that changes it. For this to work you have to set the username.
In the Symfony2 web context the username is automatically set to the one in the current security token.

In a standalone app or Symfony command you have to set the username to a specific value using the `AuditConfiguration`:

```php
// Symfony2 context
$container->get('simplethings_entityaudit.config')->setCurrentUsername('beberlei');

// Standalone app
$auditConfig = new \SimpleThings\EntityAudit\AuditConfiguration();
$auditConfig->setCurrentUsername('beberlei');
```

## Viewing auditing

A default Symfony2 controller is provided that gives basic viewing capabilities of audited data.

To use the controller, import the routing **(don't forget to secure the prefix you set so that
only appropriate users can get access)**

##### app/config/routing.yml
```yml
simple_things_entity_audit:
    resource: "@SimpleThingsEntityAuditBundle/Resources/config/routing.yml"
    prefix: /audit
```

This provides you with a few different routes:

 * ```simple_things_entity_audit_home``` - Displays a paginated list of revisions, their timestamps and the user who performed the revision
 * ```simple_things_entity_audit_viewrevision``` - Displays the classes that were modified in a specific revision
 * ```simple_things_entity_audit_viewentity``` - Displays the revisions where the specified entity was modified
 * ```simple_things_entity_audit_viewentity_detail``` - Displays the data for the specified entity at the specified revision
 * ```simple_things_entity_audit_compare``` - Allows you to compare the changes of an entity between 2 revisions

## TODOS

* Currently only works with auto-increment databases
* Proper metadata mapping is necessary, allow to disable versioning for fields and associations.
* It does NOT work with Joined-Table-Inheritance (Single Table Inheritance should work, but not tested)
* Many-To-Many associations are NOT versioned
