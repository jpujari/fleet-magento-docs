This procedure is a start-to-finish outlining a full migration from an external hosted Magento, to Fleet.


Perform initial Magento config fixup
----

Taken from https://docs.anchor.net.au/system/fleet/Migrating-a-customer

1. [Disable automatic redirects to the Base URL](../configuring-magento-for-fleet/disable-redirects.md)
2. [Push all assets into the database](../configuring-magento-for-fleet/media-in-database.md)
3. Should [fix the cookie domain problem](../configuring-magento-for-fleet/unset-cookie-domain.md) too


Build the new Fleet
----

This is an internal process: [build the fleet](https://docs.anchor.net.au/system/fleet/Building-a-new-Fleet)

Collect the `deploy` user's pubkey (`~deploy/.ssh/id_rsa.pub`) and send it to
the customer, instructing them to use it as a *deploy key* on Github/Bitbucket.
Also give them the name of the fleet that you've created, so they can add the
necessary webhook.

At the same time, ask for the customer's SSH pubkey, so that we can add it to
fleet and allow them to login to the aux box.


Gitify your codebase
----

[Setup your git repo](../getting-started/configuring-revision-control.md) with the necessary branch and hooks.

It goes roughly like:

1. Have a copy of your Magento directory on your workstation (whatever goes in `public_html`, etc)
2. Make a git repo and push it to Github/Bitbucket, here's an example for Github:
    ```
    git init
    git add .
    git commit -m "first commit"
    git remote add origin git@github.com:YOURNAME/propitious-octo-tribble.git
    ```
3. Push to a repo on Github/Bitbucket: `git push -u origin master;  git branch fleet-deploy;  git push origin fleet-deploy`
4. Install the deployment key into your repo, this will allow Fleet to grab a copy of your code when you deploy
5. Add the post-receive hook to call the Fleet's aux box when you commit to the `fleet-deploy` branch

Your git repo is now ready. Send your SSH public key to Anchor so that you can
login to your Fleet via SSH, and also the URL of your Github/Bitbucket repo.


Create the prod environment
----

This is an internal process: [create the *prod* environment](https://docs.anchor.net.au/system/fleet/Create-a-new-environment)

Make sure to add their pubkey, and set the URL of their repo.


Dump the existing DB
----

Make sure all media is synced to the DB, you should've done this earlier. This means that images and other store assets are stored as blobs in there.

Dump the DB:

```text
mysqldump --quick --complete-insert --quote-names magento | gzip > magento-database-dump.sql.gz
```

Put the dump on your workstation or somewhere convenient.


Import the DB
----

These steps are covered here, briefly: [database setup](../configuring-magento-for-fleet/database-setup.md)

1. The very first time, you'll need to login and create the database yourself.
    There is no default setup for this.

        ssh deploy@aux.FLEETNAME.f.nchr.io database connect prod --force

    There will be no feedback as you type each command. Enter the following
    and terminate your input with ^D (Ctrl-D) followed by Enter. The commands
    will then be executed.

    Fill in the database name, username and password that your existing
    Magento installation uses.

        CREATE DATABASE magento_db;
        GRANT ALL PRIVILEGES ON magento_db.* TO 'magento_user' IDENTIFIED BY 'long_and_complex_magento_password' WITH GRANT OPTION;
        FLUSH PRIVILEGES;
        \q

2. Push the database dump into mysql:

        zcat magento-database-dump.sql.gz  |  perl -pe 's/\sDEFINER=`[^`]+`@`[^`]+`/ DEFINER=CURRENT_USER()/'  |  ssh deploy@aux.FLEETNAME.f.nchr.io database connect prod magento_db --force

    Because MySQL is a pain in the arse, we have to mangle the SQL a bit so that triggers can be created successfully.

3. Tell Magento to use a custom admin url, as the Admin node is separate from your frontends.

        echo "UPDATE core_config_data SET value = 'http://admin.prod.FLEETNAME.f.nchr.io/' WHERE path = 'admin/url/custom'" | ssh deploy@aux.FLEETNAME.f.nchr.io database connect prod magento --force
        echo "UPDATE core_config_data SET value = '1' WHERE path = 'admin/url/use_custom'" | ssh deploy@aux.FLEETNAME.f.nchr.io database connect prod magento --force

    Once you've imported your database, loaded and activated an initial release you'll be able to access Magento's admin panel at `https://admin.prod.FLEETNAME.f.nchr.io/admin/`.




Add Redis
----

At some point we need to [setup Redis for sessions and caching](../configuring-magento-for-fleet/configure-cache-and-sessions.md)

Make the edits to your git repo and commit them.


Codebase and config
----

You put your code in a git repo earlier, right? Great! Let's patch the Magento config to work in the Fleet environment.

This assumes you already have Redis configured in Magento.

1. Update `app/etc/local.xml` with the new parameters:  

        --- a/app/etc/local.xml
        +++ b/app/etc/local.xml
        @@ -40,7 +40,7 @@
                     </db>
                     <default_setup>
                         <connection>
        -                    <host><![CDATA[localhost]]></host>
        +                    <host><![CDATA[mysql]]></host>
                             <username><![CDATA[magento]]></username>
                             <password><![CDATA[magento]]></password>
                             <dbname><![CDATA[magento]]></dbname>

2. Commit to the new branch (`fleet-deploy`), push to Bithub, which should
    trigger a release-build on the aux box. This will take about five minutes,
    and you can confirm that the release is being built by running `fleet release list`.


First deploy
----

1. Find the new `RELEASE_ID` in the *name* column:

        $ ssh deploy@aux.migrtest.f.nchr.io release list

        name     status     modified                   message
        -------  ---------  -------------------------  -----------
        9b84481  AVAILABLE  2015-07-30 03:45:32+00:00  First build

2. Load the release into the *prod* environment:

        $ ssh deploy@aux.migrtest.f.nchr.io env load prod 9b84481

3. Wait a little while, when it's finished the release will be loaded:

        $ ssh deploy@aux.migrtest.f.nchr.io env list
        name    status      release    releases  certificate    created                    updated
        ------  --------  ---------  ----------  -------------  -------------------------  -------------------------
        prod    RUNNING                       1  self-signed    2015-07-30 01:00:12+00:00  2015-07-30 01:20:27+00:00

4. Activate the release in the prod environment:

        $ ssh deploy@aux.migrtest.f.nchr.io env activate prod 9b84481

5. Wait a littler while

        $ ssh deploy@aux.migrtest.f.nchr.io env list
        name    status    release      releases  certificate    created                    updated
        ------  --------  ---------  ----------  -------------  -------------------------  -------------------------
        prod    UPDATING  9b84481             1  self-signed    2015-07-30 01:00:12+00:00  2015-07-30 04:53:54+00:00

    When the status changes from UPDATING to RUNNING, your Fleet is now online.


View and testing
----

Hopefully your release was successful. You should now you have a couple of useful DNS names for testing and access:

* [http://admin.prod.migrtest.f.nchr.io/admin/](http://admin.prod.migrtest.f.nchr.io/admin/)
* [http://www.prod.migrtest.f.nchr.io/](http://www.prod.migrtest.f.nchr.io/)

### Troubleshooting

If things aren't working as expected, check the following:

* Getting redirected to the canonical domain: If you originally configured your Magento store to use HTTPS URLs, you will need to access the testing URLs with `https://` instead of http.

* Your images and static assets probably aren't working: you'll need to figure out what's going on here, perhaps your media wasn't synced to the database.

* Can't login to the admin interface: You need to have set a custom hostname for the admin node, but you probably don't need a custom admin *path*. This can cause trouble because your login goes to `/index.php/admin/`, which won't be detected as an admin login attempt, and you'll get kicked to the canonical base URL. So probably disable the use of custom admin path.


NOT YET FINISHED
----

**Need to continue from here to finish the migration and check that the whole process is sane.**


Cloudflare integration
----

Do we do that here?


Enable caching
----

Ensure it's well-behaved too. Is this the right place for it?


SSL certs
----

[Order and install SSL certificates](../how-to/manage-certificates.md), and ensure they're working properly


DNS things
----

Verify that they have a DNS entry that points the root of the domain to `www`


Go-live checklist
-----------------

This list should be a bunch of things that are already working. If you can tick
all the boxes, you're safe to go live on Fleet.

* Your *prod* environment:
    * is running
    * has a loaded release
    * has a suitable certificate associated with it (probably not self-signed)

    It should look something like this:

        deploy@aux-example:~$ fleet env list
        name    status    release      releases  certificate    created                    updated
        ------  --------  ---------  ----------  -------------  -------------------------  -------------------------
        prod    RUNNING   a1b2e3f             2  www_exmpl      2015-06-23 06:16:45+00:00  2015-07-20 05:15:31+00:00


Flip the switch and go live
----

FIXME: how and where is this done?

