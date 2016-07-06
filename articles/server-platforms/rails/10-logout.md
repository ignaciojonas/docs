---
title: Logout
description: Learn how to log a user out of your rails app using the Authentication API SDK.
---

## Ruby On Rails - Logout
In this step you'll learn how to log a user out of your Rails application using the Auth0 Ruby SDK.

::: panel-info System Requirements
This tutorial and seed project have been tested with the following:
* Ruby 2.3.1
* Rails 5.0.0
:::

<%= include('../../_includes/_package', {
  pkgRepo: 'omniauth-auth0',
  pkgBranch: 'master',
  pkgPath: 'examples/ruby-on-rails-webapp',
  pkgFilePath: null,
  pkgType: 'server'
}) %>

### 1. Logout action

To clear out all the objects stored within the session, call the `reset_session` method within the `logout_controller\logout` method. [Learn more about `reset_session` here](http://api.rubyonrails.org/classes/ActionController/Base.html#M000668).

A typical logout action would look like this:

```ruby
class LogoutController < ApplicationController
  include LogoutHelper
  def logout
    reset_session
    redirect_to logout_url.to_s
  end
end
```

You can take advantage of the SDK for generating the logout URL.

```ruby
module LogoutHelper
  def logout_url
    creds = { client_id: ENV['AUTH0_CLIENT_ID'],
    client_secret: ENV['AUTH0_CLIENT_SECRET'],
    api_version: 1,
    domain: ENV['AUTH0_DOMAIN'] }
    client = Auth0Client.new(creds)
    client.logout_url(root_url)
  end
end
```

**NOTE**: The final destination URL (the `returnTo` value) needs to be in the list of `Allowed Logout URLs`. [Read more about this](/logout#redirecting-users-after-logout).
