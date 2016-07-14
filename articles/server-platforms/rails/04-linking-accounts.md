---
title: Linking Accounts
description: Link and Unlink accounts from other identity providers using the Rails SDK.
---

<%= include('../../_includes/_package', {
  pkgRepo: 'omniauth-auth0',
  pkgBranch: 'master',
  pkgPath: 'examples/ruby-on-rails-webapp',
  pkgFilePath: null,
  pkgType: 'server'
}) %>

/_ TODO: REMOVE THIS line

**Otherwise, Please follow the steps below to configure your existing Ruby On Rails WebApp to use it with Auth0.**

In this tutorial you'll learn how to use the Auth0 Rails SDK to explicitly link user accounts from different identity providers, allowing a user to authenticate from any of their accounts and still be recognized by your app and associated with the same user profile.

The sample application allows the users to log in with an initial provider, and provides a button that enables them to link another account to the first one using Auth0 Lock. Click [here](/link-accounts) to learn more about account linking.

### 1. Add dependencies

Add the following dependencies to your `Gemfile` and run `bundle install`

${snippet(meta.snippets.dependencies)}

### 2. Handling the second authentication with Lock
In [Step 01] you created a js function to handle the authentication with Lock. You'll use the same pattern for your application to trigger the second authentication.

Add the following method to `/app/assets/javascripts/home.js.rb`:

```js
function linkPasswordAccount(connection){
    var opts = {
			callbackURL: ${ "'<%= Rails.application.secrets.auth0_callback_url %>'" },
      rememberLastLogin: false,
      dict: {
        signin: {
          title: 'Link another account'
        }
      }
    };
    if (connection){
      opts.connections = [connection];
    }
    //open lock in signin mode, with the customized options for linking
    lock.showSignin(opts);
  }
```

### 3. Create the Settings view
This view will contain two forms for linking and unlinking accounts, respectively.
- The first form will show a submit button that, when clicked, will launch Lock allowing the user to login to a second account. Upon successful authentication, the `Auth0Controller` `callback` action will process the account linking using the SDK.
- The second form will have a dropdown listing the user providers, and a submit button. Upon clicking the button, the `SettingsController` `unlink_provider` action will use the SDK to unlink the selected account.

To create the linking form, add the following markup to the settings view:

```ruby
${" <%= form_tag({ action: 'link_provider' },{ class: 'form-horizontal col-xs-10 col-xs-offset-1' }) do %> "}
  <div class="form-group">
    <label class="col-xs-3 control-label">Provider</label>
    <div class="col-xs-9">
      ${ "<%= select_tag(:link_provider, options_for_select(@providers), class: 'form-control') %>" }
    </div>
  </div>
  <div class="form-group">
    <div class="col-xs-offset-3 col-xs-9">
      <!--  TODO: selected from select_tag-->
      <a class="btn btn-success btn-lg" onclick="linkPasswordAccount()">Link Accounts</a>
    </div>
  </div>
${ "<% end%>" }
```

To create the second view, add the following code to the settings view:

```ruby
${ "<%= form_tag({ action: 'unlink_provider', method: 'post' },{ class: 'form-horizontal col-xs-10 col-xs-offset-1' }) do %>" }
      <div class="form-group">
        <label class="col-xs-3 control-label">Provider</label>
        <div class="col-xs-9">
          ${" <%= select_tag(:unlink_provider, options_for_select(@providers), class: 'form-control') %> "}
        </div>
      </div>
      <div class="form-group">
        <div class="col-xs-offset-3 col-xs-9">
          ${ "<%= button_tag(type: 'submit', class: 'btn btn-danger btn-lg') do %>" }
            Unlink Accounts
          ${ "<% end %>" }
        </div>
      </div>
    ${ "<% end %>" }
```

### 4. Create the Settings Controller

Create the `show` action that will continue to retrieve the user information as in the previous steps, but addittionally it will collect the list of the current user's identity providers into the `providers` hash:

```ruby
def show
@user = session[:userinfo]
@providers = user_providers.keys.collect {|x| [x,x]}
# [
#       ['Facebook','facebook'],
#       ['Github','github'],
#       ['Google','google-oauth2'],
#       ['Twitter','twitter']
# ]
end
```

Next, create the `link_provider` and `unlink_provider` actions
- `link_provider` will obtain the second identity from the `linkuserinfo` session hash and call the `link_user_account` method of the ruby SDK. Upon successful linkage, it will redirect the user back to the settings page, flashing a success notice.

```ruby
    def link_provider
    user = session[:userinfo]
    link_user = session[:linkuserinfo]

    v2_creds = { client_id: ENV['AUTH0_CLIENT_ID'],
     token: user[:credentials][:id_token],
     api_version: 2,
     domain: ENV['AUTH0_DOMAIN'] }

    client = Auth0Client.new(v2_creds)

    client.link_user_account(user['uid'], { link_with: link_user[:credentials][:id_token] })

    redirect_to '/settings', notice: "Provider succesfully linked."
  end
```

- `unlink_provider` will obtain the main identity from the `userinfo` session hash and call the `unlink_users_account` method of the ruby SDK, passing the selected provider as a parameter. Then, it will redirect the user back to the settings page, flashing the corresponding notice.

```ruby
def unlink_provider
  user = session[:userinfo]

  v2_creds = { client_id: ENV['AUTH0_CLIENT_ID'],
   token: user[:credentials][:id_token],
   api_version: 2,
   domain: ENV['AUTH0_DOMAIN'] }

  client = Auth0Client.new(v2_creds)
  result = client.unlink_users_account(user['uid'], params['unlink_provider'], user_providers[params['unlink_provider']])
  redirect_to '/settings', notice: "Provider succesfully unlinked."
end
```

Finally, create the private `user_providers` method, which extracts the names of the providers for the current user from the OmniAuth authentication hash.

```ruby
    def user_providers
      #TODO: why is it not working with @user?
      user = session[:userinfo]
      Hash[user['extra']['raw_info']['identities'].collect { |identity| [identity['provider'], identity['user_id']]}]
    end
```
> You can read more about the OmniAuth Auth0 custom strategy and the authentication hash [here](https://github.com/auth0/omniauth-auth0#auth-hash).

### 5.Update the Auth0Controller
You need to update the `callback` action in this controller to handle the second authentication / account linking.

```ruby
class Auth0Controller < ApplicationController
  def callback
    unless session[:userinfo].present?
       session[:userinfo] = request.env['omniauth.auth']
       redirect_to '/dashboard'
       return
    end
    session[:linkuserinfo] = request.env['omniauth.auth']
    redirect_to '/settings/link_provider'
  end
```
