Badges
---------------------------
This is an LTI-enabled service that allows you to award badges
(Mozilla Open Badges, specifically) to students in a course
based on their accomplishments in the course. Currently this
will will only work with Canvas. You can see (and use!) Canvabadges
at https://canvabadges.herokuapp.com.

Canvabadges now supports multiple badges per course, and has better
support for launching from multiple courses in the same session.

The following setup of Canvabadges is more specific to the Hotchalk Ember environment.

## Setup

Canvabadges is a Rack compliant Sinatra app. Deployment is done using Phusion Passenger and Apache.  We need a SSL certificate so we don't get security errors in canvas when the app is embedded. The setup follows these assumptions:

 - The canvabadges root is located in **/srv/app/canvabadges**
 - Data directory for the app's database is **/srv/data**
 - Name of the badges instance is **badges-sbx.hotchalkember.com**

Sample Virtual host Apache 2.2 configuration:

```apacheconf
<VirtualHost *:80>
  ServerName badges-sbx.hotchalkember.com
  RewriteEngine on
  RewriteCond %{HTTPS} off
  RewriteRule ^/(.*) https://%{HTTP_HOST}/$1 [NC,R,L]
</VirtualHost>

<VirtualHost *:443>
  ServerName badges-sbx.hotchalkember.com
  SSLEngine ON
  SSLCertificateFile /etc/apache2/ssl/badges.crt
  SSLCertificateKeyFile /etc/apache2/ssl/badges.key

  DocumentRoot /srv/app/canvabadges/public
  PassengerRoot /var/lib/gems/2.1.0/gems/passenger-5.0.20
  PassengerMaxPoolSize  1
  SetEnv SESSION_KEY <RANDOM_STRING>
  SetEnv DATABASE_URL sqlite3:////srv/data/production.sqlite3
 <Directory /srv/app/canvabadges/public>
    Allow from all
    Options -MultiViews
  </Directory>

  LogLevel warn
  ErrorLog /var/log/apache2/canvabadges-error.log
  CustomLog /var/log/apache2/canvabadges-access.log combined
</VirtualHost>

```
  
  Now in the canvabadges root folder:

```bash
# install gems...
sudo bundle install

# Set the data model to support multiple Canvas instances (such as QA, Prod, Sandbox
irb
require './canvabadges.rb'
#  create a record matching our own canvabadges instance
#  twitter_login set to false. LTI credentials created by hand later on
#  (twitter_login lets anyone generate an LTI key and secret with a twitter login)
d = Domain.create(:host => "badges-sbx.hotchalkember.com", :name => "Hotchalk Ember Canvabadges")
o = Organization.create(:host => "badges-sbx.hotchalkember.com", :settings => {
  'name' => "Hotchalk Ember Canvabadges", 
  'description' => "Badging app",
  'twitter_login' => false,
  'url' => 'https://badges-sbx.hotchalkember.com',
  'image' => '<optional>',
  'email' => '<optional>',
  'oss_oauth' => true
})

#Sample external config specific for sandbox
ExternalConfig.create(:config_type => 'canvas_oss_oauth', :value => "<canvas developer key id>", :shared_secret => "<canvas developer secret>", :domain => "sandbox.hotchalkember.com", :app_name => "Hotchalk Ember Sandbox", :organization_id => o.id)
#Notice the reference to the main badges organization "o.id" in the tenant external configuration.
#Additional external configs for QA or Prod are created in the same way.
exit

# Create LTI configuration. Key and secrets to use in Ember
irb
require './canvabadges.rb'
#  create a new LTI configuration
conf = ExternalConfig.generate("My Magic LTI Config")
#  print out the results
puts "key:    #{conf.value}"
puts "secret: #{conf.shared_secret}"
exit
```
Canvas developer keys are created by logging in as a site admin and going to 
`https://<canvas instance>/developer_keys` and creating one. You can use the
image at `https://badges-sbx.hotchalkember.com/logo.png` as the developer key image. For
the redirect URI enter `https://badges-sbx.hotchalkember.com/oauth_success`. 
```

## TODO

- Per-badge option to auto-publish
- Shared pool of badges for entire organization
- API for querying for things like leaderboards
- Way for admins to manually modify specific badge issuances

[![Build Status](https://travis-ci.org/whitmer/canvabadges.png)](https://travis-ci.org/whitmer/canvabadges)
