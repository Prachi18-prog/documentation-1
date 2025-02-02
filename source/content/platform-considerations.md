---
title: Platform Considerations
description: A list of the Pantheon platform considerations for your Drupal or WordPress sites.
tags: [infrastructure]
categories: []
---
This page is used to keep track of common platform considerations, mostly derived from Pantheon's distributed nature. Check back often, as we are keeping it up to date as we make improvements to address these limitations.

## Browser Support for Pantheon's Dashboard
In order to focus internal development and engineering work, the Pantheon Dashboard supports the following browsers:

+------------------------+--------+---------+-------+------+---------------------+
|                        | Chrome | Firefox | Opera | Edge | Safari              |
+========================+========+=========+=======+======+=====================+
| **Versions Supported** | Evergreen Browsers - Last 4     | Current + Last Year |
+------------------------+---------------------------------+---------------------+

## Batch Uploads

The [max execution time](/timeouts/#user-configurable-timeouts) for PHP scripts on the platform is 120 seconds. Batch uploads, like one might see importing products with [WooCommerce](https://wordpress.org/plugins/woocommerce/) can encounter this limit.

In this or similar instances, consider performing larger operations locally, then importing the code, files, and database back up to the platform.

## Batched Data Export to File

In Test and Live environments on plans with multiple application containers, it is difficult to batch export data to a file. Many contrib plugins and modules are not designed to support multiple application containers. It might be possible to get data export working, but that may require additional effort and custom code.

Often, modules and plugins do this type of batch export by continuously appending data to the same file in each request in the batch process. With multiple application containers, the result is that several containers will attempt to add data to the same file at once, while simultaneously syncing their own version of the updated file to other appservers and receiving updates from other application containers. The exported data will likely be incomplete.

A non-batched export of a dataset small enough to complete within the set timeout for web requests will likely work.

### Potential Workarounds

1. Have each request in the data export write to its own `tmp` file, then concatenate these at the end. This solution requires that the [Persistent Temporary Path Workaround](/tmp/#persistent-temporary-path-workaround) is in place.

2. Do small batches and add enough time between each request in the batch process to allow the updated file to be synced between all application containers.

### Alternative Approaches

Running the export from the command line using tools like [Terminus](/terminus/), [Drush](/drush/), [WP-CLI](/wp-cli/) and cron will produce a better result. Larger data sets can be exported, as command line processes have longer timeouts than HTTP requests. For more details, see [Timeouts on Pantheon](/timeouts/). The export won't need to be batched and can therefore run to completion on a single application container.

Most often, the best solution is to implement data exports as a web service, incrementally exchanging the data with the target system.

## CORS

For sites that need to provide services with Cross-Origin Resource Sharing (CORS), adding the proper header should enable the resource. See  [https://enable-cors.org/server_php.html](https://enable-cors.org/server_php.html)

Sites that consume services using CORS, such as Amazon S3 CORS, do work on Pantheon.

For WordPress users, you can use the [WP-CORS plugin](https://wordpress.org/plugins/wp-cors/), or add the following to the active theme's `function.php`:

```php
add_filter('allowed_http_origins', 'pantheon_allowed_origins');

function pantheon_allowed_origins($urls) {
    $urls[] = 'https://www.example.com';
    return $urls;
}
```

In the example above, `$urls[]` is defined as a URL for which cross-domain requests are allowed. Note that the protocol (`http` or `https`) and any subdomains (like `www`) are relevant. Here's an example of a larger array allowing requests from multiple URLS:

```php
$urls[] = array( 'https://www.example.com', 'http://www.example.com', 'https://example.com', 'http://example.com' ) ;
```

## CSS Preprocessors
Pantheon does not currently support LESS or Sass/Compass CSS preprocessor languages. LESS and Sass will need to be pre-compiled to make traditional CSS stylesheets before being pushed to the platform.

## Database Stored Procedures

MySQL stored procedures are not supported. Due to the nature of the platform, there is no guarantee that they will  persist following a database migration. You can avoid the use of stored procedures by using parameterized queries or [object-relational mapping](https://en.wikipedia.org/wiki/Object-relational_mapping).

## Email and Deliverability

Because of the cloud-based nature of Pantheon's infrastructure, we cannot ensure high-deliverability email originating from your Application Containers, as they have no fixed location. While all sites have access to a local Postfix service for testing and development, we recommend using an external SMTP gateway (SendGrid, for example) in production to ensure that your email is delivered.

See [the email documentation](/email) for more details and suggestions.

## Emoji Support

Emoji support is available out of the box on WordPress and Drupal 8. Drupal 7 sites can enable Emoji support by following this procedure:

For new _or_ existing Drupal 7 installs, add the following lines to `sites/default/settings.php`:

```php
$databases['default']['default']['charset'] = 'utf8mb4';
$databases['default']['default']['collation'] = 'utf8mb4_general_ci';
```

Existing sites that already have an active database must install the [UTF8MB4 Convert](https://www.drupal.org/project/utf8mb4_convert) Drush command and convert the database. Note that this is not a Drupal module that can be enabled, it's a Drush command that should be placed within `/sites/all/drush`. Once you've installed the command in `/sites/all/drush`, you must clear Drush cache for the new command to run. Clear Drush cache using [Terminus](/terminus/):

```bash
terminus drush <site>.<env> -- cc drush
```

Start by making a [backup](/backups/) of the site database, then place the site in maintenance mode and run the following:

```bash
terminus drush <site>.<env> -- utf8mb4-convert-databases
```

This will convert the database tables in the existing installation to the proper encoding to support emoji characters. After making the conversion, test it out by placing an emoji in the site text.

## General PHP Framework Support
Pantheon does not currently support any PHP frameworks outside of Drupal and WordPress. The platform is only optimized for Drupal or WordPress and no others. Although PHP will run, we can not assist you in getting the non-approved frameworks running in any way.

## Highly Populated Directories
If you have individual directories with tens of thousands of files (e.g. an image repository) it may be necessary to refactor this file structure to see good performance on Pantheon. The danger zone begins at around 50,000 files in a single directory, and performance drops off suddenly at over 100,000 files.

Drupal itself is capable of managing uploaded content into different directories based on the date or user, which is preferable to dumping all uploads into a single place. Refactoring an existing large-scale site with this issue is usually simply a matter of re-arranging the files and then updating the files table in Drupal.

Consider the [File (field) Paths](https://www.drupal.org/project/filefield_paths) module to help resolve issues with highly populated directories.

## .htaccess
Pantheon sites use NGINX to concurrently serve requests. The NGINX web server ignores distributed configuration files such as `.htaccess` for reduced resource consumption and increased efficiency. This configuration is standard across all Pantheon sites, and modifications to the `nginx.conf` file are not supported.

For details, see [Configure Redirects](/redirects/#php-vs-htaccess).

### Drupal False Positive
Drupal 7 and 8 checks for arbitratry code execution prevention by looking for a specific string in the `.htaccess` file. Since Pantheon uses NGINX as described above, this message can be safely ignored. For more details, see [this Drupal.org issue](https://www.drupal.org/project/drupal/issues/2150399) on `SA-CORE-2013-003`.

## Image Optimization Tools
Image optimization libraries such as advpng, OptiPNG, PNGCRUSH, jpegtran, jfifremove, advdef, pngout, jpegoptim have to be installed on the server. At this time, they are not supported. For more information see [Modules and Plugins with Known Issues.](/modules-plugins-known-issues#imageapi-optimize)

## Inactive Site Freezing
Sandbox sites that are over four months old that have not had code commits or other Git activity for three months are "frozen". All requests to a frozen site will return a `530 Site Frozen` error code, and the site's Dashboard will be unavailable.

You can easily reactivate a site by visiting your Pantheon User Dashboard, select the frozen site there, then click **Unfreeze site**. Within a few minutes, the site will be ready for development again. If you experience any issues, a backup of the site is available and can be restored via the Site Dashboard. Please note that only files that have been committed will be available after unfreezing.

## IP-Address Based Security Schemes
IP-based security is not recommended on Pantheon - or any cloud platform because the actual IP address where code executes from can change as application containers are migrated throughout the infrastructure.

For more information, see [Dynamic Outgoing IP Addresses](/outgoing-ips).

## Large Code Repository
When a code repo is larger than 2GB, it increases the possibility of Git errors when committing code on Pantheon. We suggest keeping multimedia assets out of the repo by moving them to a media file storage service such as [Amazon S3](https://aws.amazon.com/s3/), and using version control to track URLs.

## Large Files
Due to the configuration of the [Pantheon Filesystem](/files/), Pantheon's file serving infrastructure is not optimized to store and deliver very large files. Files over 100MB cannot be uploaded through WordPress or Drupal, and must be added by [SFTP or rsync](/rsync-and-sftp/). Files over 256MB will fail no matter how they are uploaded. Transfers with files over 50MB will experience noticeable degradation in performance.

| File Size       | Platform Compatibility          | Notes                               |
|:--------------- | ------------------------------- |------------------------------------ |
| ≤ 100MB         | <span  style="color:green">✔</span>  | Can be uploaded via any means       |
|   100MB - 256MB | <span  style="color:orange">✔</span> | Must be uploaded over SFTP or rsync |
| > 256MB         | <span  style="color:red">❌</span>   | Must be hosted via 3rd party CDN    |

If you are distributing large binaries or hosting big media files, we recommend using a CDN like Amazon S3 as a cost-effective file serving solution that allows uploads directly to S3 from your site without using Pantheon as an intermediary.

 - Drupal sites can use a module such as [S3 File System](https://www.drupal.org/project/s3fs).
 - WordPress sites can use plugins such as [S3 Uploads](https://github.com/humanmade/S3-Uploads) or [WP Offload Media](https://deliciousbrains.com/wp-offload-media/).

Be aware, even when using an external CDN to host files, you cannot upload files over 100MB through the CMS. Upload these files directly to the CDN (here's Amazon's documentation for [uploading to an S3 bucket](https://docs.aws.amazon.com/AmazonS3/latest/user-guide/upload-objects.html)).

See our documentation for [Drupal](/drupal-s3) and [WordPress](/wordpress-s3/) for more information about integrating S3 with your Pantheon site.

### Upload Speed
Uploading large files over a slow local internet connection can cause the process to hit our [Connection Timeout](/timeouts/#timeouts-that-are-not-configurable) of 59 seconds. For example, a 10MB file uploaded on a 2Mbps connection may take too long and fail. You can use an upload time calculator like [this one](https://downloadtimecalculator.com/Upload-Time-Calculator.html) to help determine if your local internet connection is impeding file uploads to Pantheon.

## Large (>100GB) File Backups
Large backups take longer, use more resources, and have a higher likelihood of failing.  Additionally, a 100GB compressed tarball is in itself not particularly convenient for anyone.  For this reason, scheduled backups do not backup files for sites with footprints over 200GB (although code and database are backed-up as normal).  Despite the lack of backups, file content is highly durable and stored on multiple servers.

## Maintenance Mode
Pantheon may send a [generic Maintenance Mode message](/errors-and-server-responses#pantheon-503-target-in-maintenance) during platform problems; this message cannot be customized.

Built-in Maintenance Mode for both Drupal and WordPress sites can be customized; clear caches when toggling.

## Modules and Plugins with Known Issues
See [Modules and Plugins with Known Issues](/modules-plugins-known-issues) for a list of Drupal modules and WordPress plugins that are not supported and/or require workarounds.

## Multisite

Pantheon supports designated use cases for [WordPress Site Networks](/guides/multisite) created by WordPress' Multisite feature.

We do not support [Drupal Multisite](https://www.drupal.org/docs/7/multisite-drupal/multi-site-sharing-the-same-code-base). See blog posts: [Why Drupal Multisite is not Enterprise Grade](https://pantheon.io/blog/why-drupal-multisite-not-enterprise-grade) and [Much Ado About Drupal Multisite](https://pantheon.io/blog/much-ado-about-drupal-multisite).

## nginx.conf

Pantheon does not currently support modifying the nginx.conf per site, as we run a highly tuned universal configuration file.  All of the containers run a standard profile, and we have opted to keep this configuration to keep the nginx.conf lean.

## Node.js
Node.js is not available in the platform. If running node.js services is a hard requirement for your Drupal or WordPress application, the node.js service must to be hosted on a different remote server outside of Pantheon.

## One Application per Site

Each site supports a single Drupal or WordPress application. Placing a WordPress application to behave as the blog for a Drupal site, for example, is unsupported.

### Domain Masking or URL Forwarding
Domain masking allows you to serve two entirely different and separate sites over a single common domain. For example, using one system as a front end for marketing efforts and another for blog content:

* Main Site: `https://www.example-site.com/`
* Blog: `https://www.example-site.com/blog/`

Domain masking is available to contract customers as an add-on. If you require domain masking, ask your Customer Success Manager (CSM) or [contact us](https://pantheon.io/contact-us). Customers may also set up domain masking using a third-party CDN service, but please note that third-party services are outside [Pantheon's scope of support](/support/#scope-of-support).

### Additional Databases
While you are able to import an additional database to an environment, only the Pantheon database will be preserved when the application container is updated. This means you can use an additional database for running migration scripts, but should not rely on it nor write any new data to it.

## Oracle Database Drivers
Pantheon does not currently support directly connecting to Oracle databases. Customers have successfully used the [Pantheon Secure Integration](https://pantheon.io/features/secure-integration) to connect to an external API on top of their Oracle databases.

## PHP Configuration
`php.ini` cannot be customized or overridden on the Platform. See [Securely Working with phpinfo](/phpinfo/) for more information on PHP configuration.

## PHP/Java Bridge
Pantheon does not currently support the [PHP/Java Bridge](http://php-java-bridge.sourceforge.net/pjb/).

## PHP Short Tags
PHP short tags (`<? ... ?>`) are not supported on Pantheon. The [PHP Manual](https://secure.php.net/manual/en/language.basic-syntax.phpmode.php) recommends not utilizing short tags because they are not supported on every server. Additionally, using short tags can interfere with embedding PHP in XML. Avoiding their use leads to more portable, re-distributable code.

## Rename/Move Files or Directories
### Files
Like file directories, files on Pantheon cannot be renamed or moved. Our SFTP mode doesn’t support the `mv` command, which is what most apps use when renaming or moving files. The workaround is to delete the old file and upload the new file.

### Directories
File directories on Pantheon's file serving infrastructure cannot be moved or renamed. The workaround is to create a new directory, move all the files from inside the old directory into the new one, and delete the old directory.

## Server Side Includes (SSI)
Pantheon does not and will not support Server Side Includes. We recommend converting those to use PHP includes.

## Streaming Media
Because Pantheon does not provide [transcoding](https://en.wikipedia.org/wiki/Transcoding#Re-encoding.2Frecoding), bandwidth-adaptive media delivery, or support for large files (see below), [streaming media](https://en.wikipedia.org/wiki/Streaming_media) is not possible directly from the platform.

However, you can run a great streaming media website. To do so, Pantheon recommends that you find a service to handle the transcoding and streaming, whether that's [YouTube](https://www.youtube.com/), [Brightcove](https://www.brightcove.com/), [Vimeo](https://vimeo.com/), [Soundcloud](https://soundcloud.com/), or another provider. These services provide all the necessary components for great streaming media.

It is also possible to deliver smaller media files from Pantheon using [progressive download](https://en.wikipedia.org/wiki/Progressive_download), but the media will not automatically adapt to the bandwidth and capabilities of the device browsing the site, nor does Pantheon support "seeking" to arbitrary playback positions based on time. The actual media formats (encodings, containers, file name extensions) are unrestricted.

## Terminus Support
[Terminus](/terminus), our command-line tool for power users, is designed for 'nix-type operating systems like MacOS and Linux. While some people have installed Terminus on Windows using the [Git BASH on Git for Windows](https://git-for-windows.github.io/) or [Windows Subsystem for Linux](https://docs.microsoft.com/en-us/windows/wsl/install-win10) shells, this is not officially supported.

If you're a Windows user, consider using a virtualization tool like [VirtualBox](https://www.virtualbox.org/) to run a virtualized 'nix-type environment for tools like Terminus.

## Write Access on Environments
For Dev environments in SFTP mode, the entire codebase is writable. However the platform is designed to keep only the codebase under version control.  This means that the only writable paths are `sites/default/files` for Drupal sites and `wp-content/uploads` for WordPress sites.

Any modules for Drupal or plugins for WordPress that need to write to the codebase (and assume write access) need a symlink added so that they will instead write to the file system. For more information, read [Using Extensions That Assume Write Access](/assuming-write-access/).

## XML-RPC

The [XML-RPC PHP extension](https://www.php.net/manual/en/intro.xmlrpc.php) is, as of this doc's last update, listed as experimental, and not included on the Platform. Consider the [XML-RPC for PHP](http://gggeek.github.io/phpxmlrpc/) library as an alternative.
