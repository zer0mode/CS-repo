## Collec-science multi-instance config [CS-MI]

###### Proc steps

\_\_\_\_\_[Note 0](#note-0)  
\_\_\_**[Create multi-instance tree structure](#create-multi-instance-tree-structure)**  
\_\_\_\_\_[CS-MI tree example](#cs-mi-tree-example)  
\_\_\_\_[Extract keys and values](#extract-keys-and-values)  
\_\_\_\_\_[ini file structure](#ini-file-structure)  
\_\_\_**[Enable multi-instance mode](#enable-multi-instance-mode)**  
\_\_\_**[Configure databases and credentials](#configure-databases-and-credentials)**  
\_\_\_\_[Start from scratch](#start-from-scratch)  
\_\_\_\_[Duplicate existing database](#duplicate-existing-database)  
\_\_\_**[Configure multi-instance.conf](#configure-multi-instanceconf)**  
\_\_\_[Additional instances](#additional-instances)

---

##### Note 0

A working collec-science should be running on the system prior to creating a collec instance. For further information see the collec science [install guide][] and the [installation procedure][]. The mechanism of multi-instance functionality is presented in the chapter _2.2.5 Configurer le dossier d'installation_.

[install guide]: https://github.com/Irstea/collec/blob/master/database/documentation/collec_installation_configuration.pdf
[installation procedure]: https://github.com/Irstea/collec/blob/master/install/deploy_new_instance.sh

### Create multi-instance tree structure

##### CS-MI tree example

	/
	|-- var
	|   |-- www
	|   |   |-- collec-science
	+   |   |   |-- collec ( -> /path/to/collec-lastver)
	    +   |   |   |-- param
		+   |   |   |--param.inc.php
		    |   |   |
		    |   +   |
		    |       +
		    |
		    |-- bin* ( -> collec)
		    |
		    |-- first-instance
		    |   |-- bin* ( -> ../bin)
		    |   \-- param.ini
		    |
		    |-- second-instance
		    |   |-- bin* ( -> ../bin)
		    |   \-- param.ini
		    |
		    +

>	<sup>-> _\<symbolic link\>_</sup>  
>	<sup>+ _additional content</sup>_

Create the structure

`cd /path/to/collec-science`  
`sudo ln -s /path/to/collec-lastver collec`  _<sup>**`collec-lastver`** can reside in **`collec-science`** folder<sup>_  
`sudo ln -s collec bin`  
`sudo mkdir first-instance`  _<sup>this is the instance folder (see the [tree example](#cs-mi-tree-example))<sup>_  
`sudo ln -s ../bin first-instance/bin` 

#### Extract keys and values

##### ini file structure
Comments should be prefixed with semicolon character  
<sup>info @ [php.net](http://php.net/manual/en/function.parse-ini-file.php#refsect1-function.parse-ini-file-changelog)</sup>

Here below there are 4 variables. The value of #var is 222. Strings containing non-alphanumeric characters should be enclosed with double-quotes _`""`_.
``` ini
;this is a comment
var = 1
#var = 222 ;https://en.wikipedia.org/wiki/INI_file#Comments_2  
string = this works
string_special = "that's special ;"
```

A typical minimal CS-MI param.ini file should contain

``` ini
; First instance param
; Database management
BDD_login = username
BDD_passwd = password
BDD_dsn = "pgsql:host=localhost;dbname=dbname"

; Rights management, logins and log records
GACL_dblogin = username
GACL_dbpasswd = password
GACL_dsn = "pgsql:host=localhost;dbname=dbname"
```
_You can create the **`first-instance/param.ini`** file using the above example and skip to the [next chapter](#enable-multi-instance-mode). However, verify that keys correspond with those in the **`param.inc.php.dist`** of the deployed application._

---
Use the default **`param.inc.php.dist`** or already configured **`param.inc.php`** file for the new instance. The _Minimal configuration **(A)**_ option below will create the **`param.ini`** file. Choose the _optional configuration **(B)**_ for more complex instance configuration.

* _Minimal configuration requirements **(A)**_  
`sudo sed -e 's/^\$//' -e 's/;$//' -ne '/titre =\|login =\|passwd =\|dsn =/p' -e '1i ; First instance config' collec/param/param.inc.php.dist > first-instance/param.ini`

  - `'s/^\$//'`					_create keys - remove leading `$` characters_
  - `'s/;$//'`					_remove ending semicolons `;`_
  - `-n '/titre =\|login =\|passwd =\|dsn =/p'`	_keep the keys `titre`, `login`, `passwd`, `dsn`_
  - `'1i ; First instance config'`		_add header_

> * Including all variables in scope _( optional and instance specific configuration ) **(B)**_  
> `sudo sed -e 's/^[^$].*$/;&/' -e 's/\$//g' -e 's/;$//' -e 's/^\(paramI\|SMARTY\)/;&/' collec/param/param.inc.php.dist > second-instance/param.ini`  
>   - `'s/^[^$].\+$/;&/'` _or_ `'s/^[^$].*$/;&/'`	_add ini comments_
>   - `'s/\$//g'`					_remove _'$'_ character from variable names_  
>   - `'s/;$//'`						_remove semicolons at the end of lines_  
>   - `'s/^\(paramI\|SMARTY\)/;&/'`			_comment param.ini and smarty variables_  
>
> * Removing php comments - for complete clean-up run  
>   `sudo sed -e 's/^[\*/<>? ][\*/<>? ]*/;/' -e 's/\$//g' -e 's/;$//' -e 's/^\(paramI\|SMARTY\)/;&/' collec/param/param.inc.php.dist > second-instance/param.ini`  
>   - `-r 's/^[\*/><? ]+/;/'`	_extended regular expressions version_ _or shorter_ `'s/^[\*/><? ]\+/;/'`  

### Enable multi-instance mode

1. Change the login credentials in the new **`first-instance/param.ini`**.

2. To enable the multi-instance _'mode'_ uncomment the lines [144][] and [145][] in **`/var/www/collec-science/collec/param/param.inc.php`**.

``` php
//$chemin = substr($_SERVER["DOCUMENT_ROOT"],0, strpos($_SERVER["DOCUMENT_ROOT"],"/bin"));
//$paramIniFile = "$chemin/param.ini";
```
[144]: https://github.com/Irstea/collec/blob/8ecbf85f555f13fbcc61d8bd07d0d4b3c1692a23/param/param.inc.php.dist#L144
[145]: https://github.com/Irstea/collec/blob/8ecbf85f555f13fbcc61d8bd07d0d4b3c1692a23/param/param.inc.php.dist#L145

### Configure databases and credentials

Modify the connection details in the created **`param.ini`** file to match the configuration in **`pg_hba.conf`** and enable the DB access. You might need to change the access rights.

`sudo chgrp www-data first-instance/param.ini`  
`sudo chmod 750 first-instance/param.ini`

#### Start from scratch
To start a fresh instance create a new database using the collec-science [init script]<sup id="n1">[(1)](#f1)</sup>. Don't forget to chose a database name suiting the new instance and to configure its access in the postgresql pg_hba.conf

<sup id="f1">(1) https://github.com/Irstea/collec/blob/03dc3942bb46ddf5c37f114210dbffc641ff22e1/install/deploy_new_instance.sh#L40</sup>

[init script]: https://github.com/Irstea/collec/blob/hotfix-2.0.2/install/init_by_psql.sql

#### Duplicate existing database

If the database contains data which can be used in the new instance :

`sudo -u postgres pg_dump myhotdb > path/to/hotDBexport.sql` _export the existing DB_  
`sudo -u postgres psql -c "CREATE DATABASE newinstancedb OWNER collec;"` _create a new empty DB_
> <sup>make sure the _OWNER_ has been previously created</sup>

`sudo -u postgres psql newinstancedb < path/to/hotDBimport.sql` _and reimport the exported DB_

> Consider reading the chapter about upgrading the database - _2.6.4 Mise à jour de la structure de la base de données_ if you're planning to set-up multiple instances based on an updated collec-science version or if the imported database version does not match with the one currently installed on the system.

### Configure multi-instance.conf

Create a new **`first-instance.conf`** in */etc/apache2/sites-available/*

`sudo cp -a /etc/apache2/sites-available/collec-science.conf /etc/apache2/sites-available/first-instance.conf`

Replace current domain name with the _first-instance_ domain.
> \# you must change lines 9, 10, 12 and 15, 16 (replace collec.mysociety.com by your fqdn)  
> <sup>https://github.com/Irstea/collec/blob/8ecbf85f555f13fbcc61d8bd07d0d4b3c1692a23/install/apache2/collec-science.conf#L1</sup>

`<Document root>` and `<Directory>` have to be redefined as well. Modify paths on lines [37][], [39][] and [45][] to  
_`/var/www/collec-science/first-instance/bin`_.

[37]: https://github.com/Irstea/collec/blob/8ecbf85f555f13fbcc61d8bd07d0d4b3c1692a23/install/apache2/collec-science.conf#L37
[39]: https://github.com/Irstea/collec/blob/8ecbf85f555f13fbcc61d8bd07d0d4b3c1692a23/install/apache2/collec-science.conf#L39
[45]: https://github.com/Irstea/collec/blob/8ecbf85f555f13fbcc61d8bd07d0d4b3c1692a23/install/apache2/collec-science.conf#L45

Enable the new site and reload apache2

`sudo a2ensite first-instance`  
`sudo service apache2 reload`

---

The new instance should be up. _~~Check if the site title corresponds with the `APPLI_titre` set in the instance's param.ini.~~_ _APPLI\_titre_ is obsolette and has been managed as _**`APPLI_title`**_ via database since version 2.1.

### Additional instances

Refer to the [Proc steps](#proc-steps) to properly configure all the instance elements.

- Create a new database
- Duplicate the instance folder and edit the **`param.ini`** according to the connection details
- Recreate the _`bin`_ link if necessary
- Duplicate the **`first-instance.conf`** file and change the domain name details

---

The additional instance should be also up.
