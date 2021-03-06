odoo-saas-tools
=================
Odoo (OpenERP) addons to provide SaaS

Structure of SaaS system:
=========================

* SaaS Portal - main database for control servers and clients, manage client templates and plans.
* SaaS Servers - technical databases to control client databases. SaaS server create, edit, delete databases. Each SaaS Server can be installed on a separate machine (e.g. VPS)
* SaaS Clients - client database to be used by customers. Each SaaS Client is attached to a SaaS Server.

Features
========

* create SaaS Client databases:
  * manually from SaaS Portal backend
  * by client after choosing subdomain (similar to https://www.odoo.com/page/start ) - module saas_portal_start
  * by client after choosing database template (e.g. template for POS, template for ECommerce etc) with auto-generated subdomain (e.g. demo-12345.odoo.com) - module saas_server_templates
  * by client after singing up - module saas_portal_signup
  * by client via Trial button (saas_portal_sale_online)
* prepare templates for new SaaS Client database. You are able to connect to template database, install modules you need, edit configuration, edit access rights for customer etc. Such template database will be exactly what a customer will see after database creating.
* connect to existed SaaS Client database as administrator
* control SaaS Client database from SaaS Portal backend:
  * install, update, delete addons
  * configure parameters (e.g. Max Allowed Users)
  * grant or remove access rights for database users
  * block database (users will be logged out immediately and will not able to log in back until unblocking)
  * rename database (e.g. to change domain name)
  * delete database
* collect information from client databases (count of users, disk space usage, etc.)
* sale subscription (saas_portal_sale, saas_portal_sale_online)
* notify customers about subscription expiration
* control system via external tool (see section API Integration below)
* users can see all SaaS status (Max users, Expiration date etc) at Settings->Dashboard

Usage
=====

1. Configure Odoo installation
   * set dbfilter in config file or run odoo with --db-filter parameter, e.g. ^%h$
     * be sure, that you don't use db_name in config file and don't run odoo with -d (--database) parameter
     * check [odoo documentation](https://www.odoo.com/documentation/8.0/reference/cmdline.html) for understanding db filter and database parameters.

   * execute commands below to allow create databases with dots in name:

   > cd path/to/odoo

   > sed -i 's/matches="[^"]*"//g' addons/web/static/src/xml/base.xml

   * If you run odoo locally, add domains you are going to use to /etc/hosts. E.g.

   > 127.0.0.1	odoo.local # portal

   > 127.0.0.1	s1.odoo.local # server

   > 127.0.0.1	t1.odoo.local # template

   > 127.0.0.1	t2.odoo.local # template

   > 127.0.0.1	client-x.odoo.local

   > 127.0.0.1	client-y.odoo.local

   > 127.0.0.1	client-z.odoo.local

   * Redirect requests to domains above to localhost:8069 (e.g. via nginx)
     * be sure, that odoo gets host header, e.g. for nginx:

     > proxy_set_header Host $host;

     > proxy_set_header X-Real-IP       $remote_addr;

     > proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

     > proxy_set_header X-Forwarded-Proto $scheme;

   * install dependencies:
     * oauthlib:
       * pip install oauthlib
     * requests
       * pip install requests --upgrade

2. Create two databases (via /web/database/manager):

   * Main Database, e.g. **odoo.local**:
     * install saas_portal and saas_portal_* (optional) modules
   * Server Database, e.g. **s1.odoo.local**
     * install saas_server

3. Configure **Server Database**
   * Activate the developer mode via "(?)/About" menu at the top right-hand corner
   * Open Settings/Users/OAuth Providers - SaaS
     * click [Edit]
     * update domain name at "Authentication URL" and "Validation URL", change http to https if needed. E.g.
       * http://**odoo.local**/oauth2/auth
       * http://**odoo.local**/oauth2/tokeninfo
	 * Edit Body according to your brand
	 * click [Save]
     * remember value of Client ID field. It's a database.uuid of SaaS Server.
   

4. Configure **Main Database**:
   * open Settings/SaaS Portal Settings
     * set *Base SaaS domain*, e.g. **odoo.local**
     * click Apply (do it even if you didn't make changes)

5. Register Server Database in Main Database
   * open SaaS/SaaS/Servers
     * click [Create]
     * set Database Name, e.g. **s1.odoo.local**
     * fix autogenerated Database UUID to actual one (see previous section)
     * click [Save]

6. Create Plan
   * open Saas/SaaS/Plans
     * click [Create]
     * set Plan's name, e.g. "POS + ECommerce"
     * set SaaS Server
     * set Template DB: type name, e.g. **t1.odoo.local**, and click *Create "__t1.odoo.local__"*
     * click [Save]
     * click [Create Template DB].
     * wait couple minutes while Database is being created.
	 * click [Sync server]

7. Prepare Template Database for Plan
   * click [Log in to template DB] at Plan's form
     * be sure that you allow pop-ups in your browser
   * install modules that will be used for Plan, e.g. *point_of_sale*, *website_sale*
   * make any other changes in database if needed. E.g. configure
     chart of accounts.
   * open Settings/Users/Users - onwer_template. Configure Access Rights for Owner.
	 
8. Try to create database from template
   * open SaaS/Saas/Plans - select plan
     * click [Create Client]
     * set DB Name, e.g. client-x.odoo.local
     * click [Create]
     * wait couple minutes while Database is being created.
	 * click [Sync server]
   * open SaaS/SaaS/Client - choose client
     * click [Configure]
	   * open Parameters tab
	   * add parameter "Max Users", set Value 2
	   * click [Execute]
	   * click [Close]
	 * click [Log in]
	   * you see created database
	   * try to create new user to test "Max Users" setting

9. Get more
   * check description of other saas_* modules to get more features

Client database customization
==============================

The key question of client database customization is how to restrict
administration rights. For example, client users should not be able to
uninstall **saas_client** module, edit ir.confir_parameter records
etc. The simplest solution is don't give to customer access to
administration. To make more flexible configuration, you could install
following modules on SaaS Client databases:

* [ir_rule_protected](https://github.com/yelizariev/addons-yelizariev/tree/9.0/ir_rule_protected) - makes impossible for non-superuser admin edit\delete protected ir.rule
* [access_restricted](https://github.com/yelizariev/addons-yelizariev/tree/9.0/access_restricted) - makes impossible for administrator set (and see) more access rights (groups) than he already have. (follow the link for more description)
* [hidden_admin](https://github.com/yelizariev/addons-yelizariev/tree/9.0/hidden_admin) - makes admin (user and partner) invisible
* [access_apps](https://github.com/yelizariev/addons-yelizariev/tree/9.0/access_apps) - allows to have administrators which don't have access to Apps
* [access_settings_menu](https://github.com/yelizariev/addons-yelizariev/tree/9.0/access_settings_menu) - allows to show settings menu for non-admin

API integration
===============

To control SaaS via external tool [built-in XML-RPC](https://www.odoo.com/documentation/8.0/api_integration.html) can be used.

Example in python language:

    # Import libs
    import json
    import xmlrpclib
    import requests

    # Define credentials
    main_url = 'http://odoo.local'
    main_db = 'odoo.local'
    admin_username = 'admin'
    admin_password = 'admin'

    # Authenticate
    common = xmlrpclib.ServerProxy('{}/xmlrpc/2/common'.format(main_url))
    admin_uid = common.authenticate(main_db, admin_username, admin_password, {})
    models = xmlrpclib.ServerProxy('{}/xmlrpc/2/object'.format(main_url))

    # Signup a user
    # (would raise error, if user already exists)
    client_username = 'client-email@example.com'
    client_name = 'Client Name'
    client_password = 'Client Password'
    models.execute_kw(main_db, admin_uid, admin_password, 'res.users', 'signup', [{
        'login': client_username,
        'name': client_name,
        'password': client_password,
    }])

    # Authenticate the user at Main Database
    client_uid = common.authenticate(main_db, client_username, client_password, {})

    # Get user session at Main Database if needed
    params = {'db': main_db, 'login': client_username, 'password': client_password}
    data = json.dumps({'jsonrpc': '2.0', 'method': 'call', 'params': params})
    r = requests.post('%s/web/session/authenticate' % main_url,
                      data=data,
                      headers={'Content-Type':'application/json'})
    if not r.json()['result']['uid']:
        raise Exception('Authenticaion failed')
    client_session_id = r.json()['result']['session_id']


    # Create new Client database
    plan_id = 1  # specify plan you need
    client_db = 'client.odoo.local'

    # you can keep client_db empty to generate it automatically
    # from "DB Names" parameter in Plan's form

    client_db = False
    res = models.execute_kw(main_db, admin_uid, admin_password,
                            'saas_portal.plan', 'create_new_database',
                            [plan_id], {'dbname': client_db, 'user_id':client_uid})

    res['url']  # contains url for new database with client's access token.
    saas_portal_client_id = res['id']

    # Configure system
    data = {
        # configure addons
        'update_addons': [],
        'install_addons': ['sale', 'point_of_sale', 'stock', 'access_settings_menu', access_apps'],
        'uninstall_addons': [],
        # grant access to owner
        'access_owner_add': ['base.group_sale_manager', 'stock.group_stock_manager', 'access_settings_menu.group_show_settings_menu'],
        # restrict access for all users
        'access_remove': ['access_apps.group_show_modules_menu'],
        'params': [
             {'key': 'saas_client.max_users', 'value': 10, 'hidden': True}
        ],
    }
    res = models.execute_kw(main_db, admin_uid, admin_password,
                            'saas.config', 'do_upgrade_database',
                            [data, saas_portal_client_id])

Notes abouts API integration

* Be sure, that Portal module is installed at Main Database
* Be sure, that "Allow external users to sign up" option from "Settings/General Settings" is enabled (this option is only available in Debug mode)
* To find new signuped user open "Settings/Users" at Main Database and delete filter "Regular users only"
* don't use trailing slash at main_url
* Access token is expired in one hour
* In case of log out, client has to click "Log in via SaaS Portal". Client will be navigated to Portal database and can use client_username and client_password. After that the client will be returned back to his database. Important thing here, is that the client is not able to use client_password at login page of his database.
