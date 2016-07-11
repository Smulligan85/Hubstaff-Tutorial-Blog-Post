HUBSTAFF GEM TUTORIAL

[Add Screenshot of App Homepage]


This tutorial will go over how to incorporate the Hubstaff gem into your Rails application. The Hubstaff gem allows you to easily link a User to their Hubstaff account and retrieve useful information such as custom team reports, project and activity details, screenshots, and much more. The Github repository [link to repository] includes two branches. The master branch is the starter application that this tutorial will walk through and the final-tut branch is the complete application. This tutorial will go over first linking a User to their Hubstaff account and then go over how to retrieve data. We will be retrieving data via two endpoints provided by the Hubstaff API, custom team reports and screenshots. Before we begin you will need to set up a Hubstaff account [link to https://hubstaff.com/]. I also recommend creating some data so that your application will be able to view data, specifically create a organization, project, notes, and a few screenshots. After you have created some data we need to go to the Hubstaff developer page [link to https://developer.hubstaff.com/] to create our application and receive our App Token. Once we have our App Token we’re ready to dive in.

Clone the master branch down and open the application in your editor of choice. First we will open up our Gemfile and add the hubstaff-ruby gem and dotenv gem and run `bundle install`.

```ruby
gem 'hubstaff-ruby', '~> 0.0.1'
gem 'dotenv', '~> 2.1'
```
For this tutorial you may not push anything up publicly, but I since that may change in the future I recommend next opening up our `.gitignore` file and include `.env.local`. We can then create this file in our root directory and place in the following code, switching in your specific App Token:
```ruby
APP_TOKEN=“<App Token Hubstaff Provided>”
```
Now open up our `environment.rb` file and where we will require hubstaff and load our .env file:

```ruby
# Load the Rails application.
require File.expand_path('../application', __FILE__)
require 'hubstaff'
Dotenv.load(".env.local")

# Initialize the Rails application.
Rails.application.initialize!
```
Now that we have our hubstaff gem setup we can begin incorporating it into our User class. Let’s open up our `user.rb` file and include the following lines:

```ruby
class User < ActiveRecord::Base
  include Hubstaff
  serialize :client
  has_secure_password
  validates :email, :presence => true
end
```

Next we will need to create a new migration for the users table to get the `:client` attribute working. In your command line enter:
`rails generate AddClientToUsers client:text`
For the serialize method to work correctly we will need make client a text data type. This will create the following migration table:
```ruby
class AddClientToUsers < ActiveRecord::Migration
  def change
    add_column :users, :client, :text
  end
end
```
Once that file is created we can run rake db:migrate to migrate the database.

Now that our User class has access to a client attribute we can link the logged in user to their Hubstaff account. First let’s reorganize the `application.html.erb` file to include a button that will link to our Hubstaff log in form. Change the nav block to this:
```ruby
  <nav class="navbar navbar-default">
    <div class="container-fluid">
      <div class="navbar-header">
        <% if current_user %>
          <%= link_to "Hubstaff Gem Tutorial", user_path(current_user), class: "navbar-brand" %>
        <% else %>
          <%= link_to "Hubstaff Gem Tutorial", root_path, class: "navbar-brand" %>
        <% end %>
        <ul class="nav navbar-nav">
          <% if current_user %>
            <li><%= link_to "Log Out", logout_path %></li>
          <% else %>
            <li><%= link_to "Sign Up", signup_path %></li>
            <li><%= link_to "Sign In", login_path%></li>
          <% end %>
          <% if current_user && current_user.client.nil? %>
            <button class="btn btn-success navbar-btn"><%= link_to "Connect to Hubstaff", client_path("#{current_user.id}") %></button>
            <% end %>
        </ul>
      </div>
    </div>
  </nav>
```
We will also need to add a two routes one get route to point to the log in form and also a post route to post the log in data to Hubstaff.
```ruby
Rails.application.routes.draw do
  root "welcome#index"

  get 'signup', to: 'users#new', as: 'signup'
  get 'login', to: 'sessions#new', as: 'login'
  get 'logout', to: 'sessions#destroy', as: 'logout'

  get 'users/:id/hubstaff', to: 'users#client', as: 'client'
  post '/auth_client', to: 'users#auth_client'

  resources :users
  resources :sessions
end
```
Next we will build out our log in form. Create a file called `client.html.erb` and add the following form:
```ruby
<%= form_tag(controller: "users", action: "auth_client", method: "post") do %>
  <%= label_tag :email %>
  <%= text_field_tag :hubstaff_email %>
  <%= label_tag :password %>
  <%= password_field_tag :hubstaff_password %>
  <%= submit_tag "Log into Hubstaff" %>
<% end %>
```
Now that we have our routes and form ready, let’s open up our `users_controller.rb` file and add our `client` and `auth_client` action. Add the following code to your UsersController:
```ruby

def client
end

  def auth_client
    user = current_user
    if params[:hubstaff_email].present? && params[:hubstaff_password].present?
      client = Hubstaff::Client.new(params[:hubstaff_email], params[:hubstaff_password])
      if client.nil?
        redirect_to user_path(user), alert: "Unable to Connect to Hubstaff"
      else
        user.client = client
        user.save
        redirect_to user_path(user), notice: "Connected to Hubstaff"
      end
    else
      redirect_to user_path(user), alert: "Unable to Connect to Hubstaff"
    end
  end
```

The `auth_client` method validates that the form params are present and then assigns a new instance of the Hubstaff::Client class to the variable client. If the form was not fully filled in or the log in was unsuccessful the client will return nil and we redirect the user with a flash message. Otherwise the user’s client attribute is assign the the client variable which stores the instance of Hubstaff::Client, saves the user and redirects with a success message. Now the user is fully linked to their Hubstaff account and won’t need to log in again we they use the application!

Now that the user is linked to their Hubstaff account we can use forms to retrieve specific information. As I mentioned in the introduction, this tutorial will go over retrieving custom team report data and screenshots. Lets get started.

First we will add the appropriate post routes to handle our form data. Here is what the final routes file should look like:
```ruby
Rails.application.routes.draw do
  root "welcome#index"

  get 'signup', to: 'users#new', as: 'signup'
  get 'login', to: 'sessions#new', as: 'login'
  get 'logout', to: 'sessions#destroy', as: 'logout'

  get 'users/:id/hubstaff', to: 'users#client', as: 'client'
  post '/auth_client', to: 'users#auth_client'
  post '/custom_report', to: 'users#custom_report'
  post '/screenshots', to: 'users#screenshots'

  resources :users
  resources :sessions
end
```
Next in our `show.html.erb` file we will build our buttons and for the forms we will use Bootstrap modals. Here is what the revised show file will look like:

```ruby
<h1>Welcome <%= @user.email %></h1>

<% if @user.client %>
  <button type="button" class="btn btn-primary btn-lg" data-toggle="modal" data-target="#reportModal">
    Create a custom Hubstaff team report
  </button>

  <!-- Modal -->
  <div class="modal fade" id="reportModal" tabindex="-1" role="dialog" aria-labelledby="myModalLabel">
    <div class="modal-dialog" role="document">
      <div class="modal-content">
        <div class="modal-header">
          <button type="button" class="close" data-dismiss="modal" aria-label="Close"><span aria-hidden="true">&times;</span></button>
          <h4 class="modal-title" id="myModalLabel">Create a Custom Team Report by Date</h4>
        </div>
        <div class="modal-body">
          <%= form_tag(controller: "users", action: "custom_report", method: "post") do %>
            <%= label_tag "Start Date" %>
            <%= text_field_tag :start_date, nil, placeholder: "2016-05-23" %>
            <%= label_tag "End Date" %>
            <%= text_field_tag :end_date, nil, placeholder: "2016-05-23" %><br>
            <%= label_tag "Organizations"%><br>
            <%= text_field_tag :orgs, nil, size: 40, placeholder: "List of organization IDs (comma separated)" %><br>
            <%= label_tag "Projects"%><br>
            <%= text_field_tag :projects, nil, size: 40, placeholder: "List of project IDs (comma separated)" %><br>
            <%= label_tag "Users"%><br>
            <%= text_field_tag :users, nil, size: 40, placeholder: "List of users IDs (comma separated)" %><br>
            <%= label_tag "Show Tasks"%><br>
            <%= select_tag :show_tasks, options_for_select([false, true])%><br>
            <%= label_tag "Show Notes"%><br>
            <%= select_tag :show_notes, options_for_select([false, true])%><br>
            <%= label_tag "Show Activities"%><br>
            <%= select_tag :show_activity, options_for_select([false, true])%><br>
            <%= label_tag "Include Archieved"%><br>
            <%= select_tag :include_archieved, options_for_select([false, true])%><br>
        </div>
        <div class="modal-footer">
            <%= submit_tag "Create Report" %>
          <% end %>
        </div>
      </div>
    </div>
  </div>

  <button type="button" class="btn btn-primary btn-lg" data-toggle="modal" data-target="#screenModal">
    Retrieve Screenshots From Your Hubstaff Account
  </button>

  <!-- Modal -->
  <div class="modal fade" id="screenModal" tabindex="-1" role="dialog" aria-labelledby="myModalLabel">
    <div class="modal-dialog" role="document">
      <div class="modal-content">
        <div class="modal-header">
          <button type="button" class="close" data-dismiss="modal" aria-label="Close"><span aria-hidden="true">&times;</span></button>
          <h4 class="modal-title" id="myModalLabel">Retrieve Screenshots</h4>
        </div>
        <div class="modal-body">
          <%= form_tag(controller: "users", action: "screenshots", method: "post") do %>
            <%= label_tag "Start Time" %>
            <%= text_field_tag :start_time, nil, placeholder: "2016-05-22" %>
            <%= label_tag "Stop Time"%>
            <%= text_field_tag :stop_time, nil, placeholder: "2016-05-24" %><br>
            <%= label_tag "You must choose at least one Organization, Project or User" %><br>
            <%= label_tag "Organizations"%><br>
            <%= text_field_tag :orgs, nil, size: 40, placeholder: "List of organization IDs (comma separated)" %><br>
            <%= label_tag "Projects"%><br>
            <%= text_field_tag :projects, nil, size: 40, placeholder: "List of project IDs (comma separated)" %><br>
            <%= label_tag "Users"%><br>
            <%= text_field_tag :users, nil, size: 40, placeholder: "List of users IDs (comma separated)" %><br>
            <%= label_tag "Offset"%><br>
            <%= text_field_tag :offset, nil, placeholder: "Index of the first record returned (starts at 0)", value: 0 %><br>
        </div>
        <div class="modal-footer">
            <%= submit_tag "Retrieve Screenshots" %>
          <% end %>
        </div>
      </div>
    </div>
  </div>
<% end %>
```

