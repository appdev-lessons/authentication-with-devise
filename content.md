# Authentication with Devise: Full Reference

The [Devise gem](https://github.com/plataformatec/devise) is probably the most popular authentication library in the Rails ecosystem.

## Add sign-in/sign-out

 - Add `gem "devise"` to your Gemfile and `bundle`
 - Run `rails g devise:install`

Devise will give you some setup instructions. We don't need to worry about most of them, but we do need to set a root URL. Usually, you will point the root URL to the index action of some important resource in your application: In `config/routes.rb`:

```ruby
root "photos#index"
```

This is just a shortcut for setting a root URL the old way,

```ruby
get "/", :controller => "photos", :action => "index"
```

Next, we need to secure one of our models with Devise.

### Generate a new model with Devise

Use the following command to generate a User model with Devise built-in. Replace the column names with ones that are relevant to your application. 

```
rails g devise user username:string avatar_url:string
```

Examine the generated migration file and make any desired changes (i.e., default values, uniqueness constraints, indexes)

`rake db:migrate` and restart your server.

## Benefits

The big benefits that we now get for free from Devise are:

 - RCAVs that handle sign up, sign in, and sign out -- all done, for free!
    - Visit `/rails/info` and search for "user" to see the routes that were automatically written. It's up to use to link to these routes in our UI wherever we think it appropriate; e.g. in the navbar.
    - The most commonly used routes are:
        - Sign up: `new_user_registration_path`
        - Sign in: `new_user_session_path`
        - Sign out: `destroy_user_session_path`
        - Edit profile: `edit_user_registration_path`
 - **The `current_user` helper method, available within all views and controllers, that will retrieve the row from the Users table for whoever is currently signed in.**
 - The `before_action :authenticate_user!` filter that we can use in our controllers to ensure someone is signed in before accessing any actions within that controller.

 If you want to skip this action on any of your controller actions that are inheriting from `ApplicationController`, you can add this to that specific controller and action (the `:only =>` specifies the action that will be skipped):

```ruby
class UsersController < ApplicationController
  skip_before_action(:authenticate_user!, { :only => [:index] })
  # ...
```

There are lots more (password reset emails, etc), but these are the first ones that we care about.

## Customizing Devise Views

Devise does an incredible amount of work for us out of the box, but at some point, we will want to customize it; at the very least, we will want to make the sign-up and sign-in forms look nicer.

Assuming we are using `before_action :authenticate_user!` in our `ApplicationController` to force visitors to sign up or sign in before doing anything else, then the sign in page is our landing page, after all (try going to Twitter, Facebook, etc when signed out -- the landing page is really a sign-in page with some extra info thrown in). So it would be nice to be able to make it pretty.

More importantly, we will likely want to allow users to provide more information when they sign up or edit their profiles than simply their email addresses and passwords.

### Step One: Get Our Hands On The View Templates

The first problem is that we don't even have any code to edit in order to customize the view templates for sign-in, sign-up, edit profile, etc. That's because, by default, these templates are wrapped up inside the Devise gem.

It's really easy to have Devise generate copies of these templates that we can edit, though. And then our edited versions will take precedence. Simply run

```
rails g devise:views
```

There will now be a folder in your `app/views` folder called `devise`, with a whole bunch of stuff in it. What we are most interested in are the contents of the `registrations` and `sessions` subfolders:

- `registrations/new.html.erb` is the sign up form, 
- `registrations/edit.html.erb` is the edit profile form, and 
- `sessions/new.html.erb` is the sign in form.

### Step Two: Modify The Markup

Devise is using some of the helper methods we've just learned about to draw the `<form>` and `<input>` elements:

```erb
<!-- app/views/devise/registrations/new.html.erb -->

<h2>Sign up</h2>

<%= simple_form_for(resource, as: resource_name, url: registration_path(resource_name)) do |f| %>
  <%= f.error_notification %>

  <div class="form-inputs">
    <%= f.input :email,
                required: true,
                autofocus: true,
                input_html: { autocomplete: "email" }%>
```

#### Add HTML Around The Helpers

You can write whatever HTML you want *around* these helpers; for example, Devise has already wrapped the label/input pairs inside a `<div>`:

```erb{4,7}
<!-- app/views/devise/registrations/new.html.erb -->

<!-- ... -->
  <div class="form-inputs">
    <%= f.input :email,
    <!-- ... -->
  </div>
```

Peruse the [Bootstrap conventions for form markup](https://getbootstrap.com/docs/5.3/forms/overview/) to get some inspiration for modifying these classes.

#### Add CSS Classes To The Helpers

You can also add CSS classes to the input tags, like so:

```erb{6}
  <div class="form-inputs">
    <%= f.input :email,
                required: true,
                autofocus: true,
                input_html: { autocomplete: "email" },
                class: "my-custom-class" %>
```

#### Add Additional Input Helpers
    
There are various types of `____field` helpers; the most common are

 - `f.text_field`
 - `f.email_field`
 - `f.password_field`
 - `f.number_field`
 - `f.check_box`
 - `f.collection_select` -- this one is special. It is similar to the `select_tag` and `options_from_collection_for_select` helpers we discussed in class, but rolled into one. The syntax looks like this (assuming you have a Company table and you want the user to belong to one company):

        <%= f.collection_select :company_id, Company.all, :id, :name %>

You can add as many of these as you need for the additional columns you've included in your user model, e.g. on the sign-up form:

```erb{9-11}
<!-- app/views/devise/registrations/new.html.erb -->

<h2>Sign up</h2>
<!-- ... -->
    <%= f.input :email,
                required: true,
                autofocus: true,
                input_html: { autocomplete: "email" }%>
    <%= f.input :avatar_url,
            required: true,
            autofocus: true %>
    <!-- ... -->
```

and on the edit profile form:

```erb{9-11}
<!-- app/views/devise/registrations/edit.html.erb -->

<h2>Edit <%= resource_name.to_s.humanize %></h2>
<!-- ... -->
    <%= f.input :password,
                hint: "leave it blank if you don't want to change it",
                required: false,
                input_html: { autocomplete: "new-password" } %>
    <%= f.input :avatar_url,
            required: true,
            autofocus: true %>
    <!-- ... -->
```

#### Example Bootstrapped Devise Forms

[Here are some examples of the Devise forms customized with Bootstrap classes.](https://github.com/firstdraft/bootstrapped_devise_forms) You can copy-paste them into your `app/views/devise/` folder to use as a starting point, if you wish.

### Step Three: Allow Additional Parameters Through Security

The last step we need to take is to whitelist these additional attributes as things that we will allow users to modify about themselves. To do this, you need to go to your `ApplicationController` and add the following code:

```ruby{3-9}
class ApplicationController < ActionController::Base
  before_action :authenticate_user!
  before_action :configure_permitted_parameters, if: :devise_controller?

  def configure_permitted_parameters
    devise_parameter_sanitizer.permit(:sign_up, :keys => [:username, :avatar_url])

    devise_parameter_sanitizer.permit(:account_update, :keys => [:avatar_url])
  end
end
```

You need to add the name of each column you want to be able to modify to the respective array of symbols. In the example, I have whitelisted both `:username` and `:avatar_url` to be modified upon sign-up, but only `:avatar_url` can be modified upon account update. You have to decide what makes sense in your app and modify the two forms accordingly:

- `app/views/devise/registrations/new.html.erb`
- `app/views/devise/registrations/edit.html.erb`

Then add to the `configure_permitted_parameters` method in `ApplicationController`.

---
