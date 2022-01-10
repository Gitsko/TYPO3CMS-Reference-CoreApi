.. include:: /Includes.rst.txt


.. _extension-configuration-files:


========================================================
Configuration Files (ext_tables.php & ext_localconf.php)
========================================================

Files :file:`ext_tables.php` and :file:`ext_localconf.php` are the two
most important files for the execution of extensions
within TYPO3.

These files contain configuration used by the system on almost
every request. Therefore, they should be optimized for speed.

Only add these files if your extension requires any of the configuration
described below.

.. important::

   Since the :file:`ext_tables.php` and :file:`ext_localconf.php` of
   every extension will be concatenated together by TYPO3, you MUST
   follow some rules, such as not use :php:`use` or :php:`declare(strict_types=1)`
   inside these files, see :ref:`rules_ext_tables_localconf_php`.

.. _ext-localconf-php:

ext_localconf.php
=================

*-- optional*

:file:`ext_localconf.php` is always included in global scope of the script,
either frontend or backend.

Should Not Be Used For
----------------------

While you *can* put functions and classes into the script, it is a really bad
practice because such classes and functions would *always* be loaded. It is
better to have them included only as needed.

Registering :ref:`hooks or signals <hooks-concept>`, :ref:`XCLASSes
<xclasses>` or any simple array assignments to
:php:`$GLOBALS['TYPO3_CONF_VARS']` options will not work for the following:

 * class loader
 * package manager
 * cache manager
 * configuration manager
 * log manager (= :ref:`Logging Framework <logging>`)
 * time zone
 * memory limit
 * locales
 * stream wrapper
 * :ref:`error handler <error-handling-extending>`

This would not work because the extension files :file:`ext_localconf.php` are
included (:php:`loadTypo3LoadedExtAndExtLocalconf`) after the creation of the
mentioned objects in the :ref:`Bootstrap <bootstrapping>` class.

In most cases, these assignments should be placed in :file:`typo3conf/AdditionalConfiguration.php`.

Example:

:ref:`Register an exception handler <error-handling-extending>` in :file:`typo3conf/AdditionalConfiguration.php`::

   $GLOBALS['TYPO3_CONF_VARS']['SYS']['debugExceptionHandler'] = \Vendor\Ext\Error\PostExceptionsOnTwitter::class;



Should Be Used For
------------------

These are the typical functions that extension authors should place within :file:`ext_localconf.php`

* Registering :ref:`hooks or signals <hooks-concept>`, :ref:`XCLASSes <xclasses>` or any simple array assignments to :php:`$GLOBALS['TYPO3_CONF_VARS']` options
* Registering additional Request Handlers within the :ref:`Bootstrap <bootstrapping>`
* Adding any PageTSconfig or Default TypoScript via :php:`ExtensionManagementUtility` APIs
* Registering Scheduler Tasks
* Adding reports to the reports module
* Adding slots to signals via Extbase's SignalSlotDispatcher
* Registering Icons to the :ref:`IconRegistry <icon-registration>`
* Registering Services via the :ref:`Service API <services-developer-service-api>`

deprecated

* *Registering Extbase Command Controllers* (Extbase command controllers are deprecated since
  TYPO3 9. Use symfony commands as explained in :ref:`cli-mode`)


.. _ext-tables.php:

ext_tables.php
==============

*-- optional*

:file:`ext_tables.php` is *not* always included in the global scope of the
frontend context.

This file is only included when

* a TYPO3 Backend or CLI request is happening
* or the TYPO3 Frontend is called and a valid Backend User is authenticated

This file usually gets included later within the request and after TCA information is loaded,
and a Backend User is authenticated as well.

.. hint::

   In many cases, the file :file:`ext_tables.php` is no longer needed, since `TCA` definitions
   must be placed in :file:`Configuration/TCA/*.php` files nowadays.


Should Not Be Used For
----------------------

* TCA configurations for new tables. They should go in :file:`Configuration/TCA/tablename.php`
* TCA overrides of existing tables. They should go in :file:`Configuration/TCA/Overrides/tablename.php`
* calling :php:`\TYPO3\CMS\Core\Utility\ExtensionManagementUtility::addToInsertRecords()`
  as this might break the frontend. They should go in :file:`Configuration/TCA/Overrides/tablename.php`
* calling :php:`\TYPO3\CMS\Core\Utility\ExtensionManagementUtility::addStaticFile()`
  as this might break the frontend. They should go in :file:`Configuration/TCA/Overrides/sys_template.php`

For a descriptions of the changes for TCA (compared to older TYPO3 versions), please see
the blogpost `"Cleaning the hood: TCA" by Andreas Fernandez <https://scripting-base.de/blog/cleaning-the-hood-tca.html>`__

More information can be found in the blogpost `"Good practices in extensions
<https://usetypo3.com/good-practices-in-extensions.html>`__ (use TYPO3 blog).

.. hint::

   ext_tables.php is not cached. The files in Configuration/TCA are cached.

Should Be Used For
------------------

These are the typical functions that should be placed inside :file:`ext_tables.php`

* Registering of Backend modules or Backend module functions
* Adding Context-Sensitive-Help docs via ExtensionManagementUtility API
* Adding TCA descriptions (via :php:`ExtensionManagementUtility::addLLrefForTCAdescr()`)
* Adding table options via :php:`ExtensionManagementUtility::allowTableOnStandardPages`
* Assignments to the global configuration arrays :php:`$TBE_STYLES` and :php:`$PAGES_TYPES`
* Adding new fields to User Settings ("Setup" Extension)


.. _ext-tables-localconf-best-practices:
.. _rules_ext_tables_localconf_php:

Rules and best practices
========================

The following apply for both :php:`ext_tables.php` and :php:`ext_localconf.php`.

.. important::

   Since the :file:`ext_tables.php` and :file:`ext_localconf.php` of
   every extension will be concatenated together by TYPO3, you MUST
   follow some rules, such as not use :php:`use` or :php:`declare(strict_types=1)`
   inside these files. More information below:

As a rule of thumb: Your :file:`ext_tables.php` and :file:`ext_localconf.php` files must be designed in a way
that they can safely be read and subsequently imploded into one single
file with all configuration of other extensions!

These files contain configuration used by the system on almost
every request. Therefore, they should be optimized for speed.

-  You **MUST NOT** use a :php:`return` statement in the files global scope -
   that would make the cached script concept break.

-  You **MUST NOT** rely on the PHP constant :php:`__FILE__` for detection of
   include path of the script - the configuration might be executed from
   a cached file with a different location and therefore such information should be derived from
   e.g. :php:`\TYPO3\CMS\Core\Utility\GeneralUtility::getFileAbsFileName()` or
   :php:`\TYPO3\CMS\Core\Utility\ExtensionManagementUtility::extPath()`.


-  You **MUST NOT** use :php:`use` inside :file:`ext_localconf.php` or :file:`ext_tables.php` since this can lead to conflicts with other :php:`use` in files of other extensions.

.. code-block:: diff

   // do NOT use use:
   -use TYPO3\CMS\Core\Resource\Security\FileMetadataPermissionsAspect;
   -
   -$GLOBALS['TYPO3_CONF_VARS']['SC_OPTIONS']['t3lib/class.t3lib_tcemain.php']['processDatamapClass'][] = FileMetadataPermissionsAspect::class;
    // Use the full class name instead:
   +$GLOBALS['TYPO3_CONF_VARS']['SC_OPTIONS']['t3lib/class.t3lib_tcemain.php']['processDatamapClass'][] = \TYPO3\CMS\Core\Resource\Security\FileMetadataPermissionsAspect::class;

-  You **MUST NOT** use :php:`declare(strict_types=1)` and similar directives which must be placed
   at the very top of files: once all files of all extensions are merged, this condition is not
   fulfilled anymore leading to errors. So these must **never** be used here.

.. code-block:: diff

   // do NOT use declare strict and other directives which MUST be placed at the top of the file
   -declare(strict_types=1)


-  You **MUST NOT** check for values of :php:`TYPO3_MODE` or :php:`TYPO3_REQUESTTYPE`
   constants (e.g. :php:`if (TYPO3_MODE === 'BE')`) within these files as it limits the functionality
   to cache the whole systems' configuration. Any extension author should remove the checks if not
   explicitly necessary, and re-evaluate if these context-depending checks could go inside
   the hooks / caller function directly., e.g. do not do:

.. code-block:: diff

   // do NOT do this:
   -if (TYPO3_MODE === 'BE')

-  You **SHOULD** check for the existence of the constants :php:`defined('TYPO3_MODE') or die();`
   at the top of :file:`ext_tables.php` and :file:`ext_localconf.php` files to make sure the file is
   executed only indirectly within TYPO3 context. This is a security measure since this code in global
   scope should not be executed through the web server directly as entry point.

.. code-block:: php

   <?php
   // put this at top of every ext_tables.php and ext_localconf.php
   defined('TYPO3_MODE') or die();

-  You **SHOULD** use the extension name (e.g. "tt_address") instead of :php:`$_EXTKEY`
   within the two configuration files as this variable will be removed in the future. This also applies
   to :php:`$_EXTCONF`.

-  However, due to limitations to TER, the :php:`$_EXTKEY` option **MUST** be kept within an extension's
   :ref:`ext_emconf.php <extension-declaration>`.

-  You **SHOULD** use a directly called closure function to encapsulate all
   locally defined variables and thus keep them out of the surrounding scope. This
   avoids unexpected side-effects with files of other extensions.

The following example contains the complete code::

    <?php
    defined('TYPO3_MODE') or die();

    (function () {
        // Add your code here
    })();


Additionally, it is possible to extend TYPO3 in a lot of different ways (adding TCA, Backend Routes,
Symfony Console Commands etc) which do not need to touch these files.

Additional tips:

-  :php:`TYPO3\CMS\Core\Package\PackageManager::getActivePackages()` contains information about
   whether the module is loaded as *local* or *system* type in the `packagePath` key,
   including the proper paths you might use, absolute and relative.
