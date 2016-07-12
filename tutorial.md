![Tutorial Homepage](/images/homepage.png)

This tutorial will go over how to incorporate the Hubstaff gem into your Rails application. The Hubstaff gem allows you to easily link a User to their Hubstaff account and retrieve useful information such as custom team reports, project and activity details, screenshots, and much more. The [Github repository](https://github.com/Smulligan85/Hubstaff-Gem-Tutorial) for this tutorial includes two branches. The master branch is the starter application that this tutorial will walkthrough and the final-tut branch is the complete application. The starter applicaiton is a basic Rails app that includes a User model and some basic authentication. This tutorial will go over first linking a User to their Hubstaff account and then go over how to retrieve data. We will be retrieving data via two endpoints provided by the Hubstaff API, custom team reports and screenshots. Before we begin you will need to set up a [Hubstaff account](https://hubstaff.com/). I also recommend creating some data so that your application will be able to view data, specifically create a organization, project, notes, and a few screenshots. After you have created some data we need to go to the [Hubstaff developer page](https://developer.hubstaff.com/) to create our application and receive our App Token. Click My Apps and create a new application. Once we have our App Token we’re ready to dive in.

Clone the master branch down and open the application in your editor of choice. First we will open up our Gemfile and add the hubstaff-ruby gem and dotenv gem and run `bundle install`.

```ruby
gem 'hubstaff-ruby', '~> 0.0.1'
gem 'dotenv', '~> 2.1'
```
For this tutorial you may not push anything up publicly, but since that may change in the future I recommend next opening up our `.gitignore` file and include `.env.local`. We can then create this file in our root directory and place in the following code, swapping in your specific App Token:
```ruby
APP_TOKEN=“<App Token Hubstaff Provided>”
```
Now open up our `environment.rb` file where we will require hubstaff and load our .env.local file:
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
Next we will need to create a new migration for the users table to get the `:client` attribute working. In your terminal enter:
`rails generate migration AddClientToUsers client:text`
For the serialize method to work correctly we will need to make client a text data type. This will create the following migration table:
```ruby
class AddClientToUsers < ActiveRecord::Migration
  def change
    add_column :users, :client, :text
  end
end
```
Once we have our migration setup we can run `rake db:migrate` to migrate the database.

Now that our User class has access to a client attribute we can link the logged in user to their Hubstaff account. First let’s reorganize the `application.html.erb` file to include a button that will link to our Hubstaff log in form. Change the nav block to the following:
```erb
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
We will also need to add two routes; a get route to point to the log in form and a post route to post the log in data to Hubstaff.
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
```erb
<%= form_tag(controller: "users", action: "auth_client", method: "post") do %>
  <%= label_tag :email %>
  <%= text_field_tag :hubstaff_email %>
  <%= label_tag :password %>
  <%= password_field_tag :hubstaff_password %>
  <%= submit_tag "Log into Hubstaff" %>
<% end %>
```
Now that we have our routes and form ready, let’s open up our `users_controller.rb` file and add our `#client` and `#auth_client` action. Add the following code to your UsersController:
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
The `#auth_client` method validates that the form params are present and then assigns a new instance of the Hubstaff::Client class to the client variable. If the form was not fully filled in or the log in was unsuccessful the client will return nil and we redirect the user with a flash message. Otherwise the user’s client attribute is assigned to the client variable which stores the instance of Hubstaff::Client, saves the user and redirects with a success message. Now the user is fully linked to their Hubstaff account and won’t need to log in again when they use the application!

Now that the user is linked to their Hubstaff account we can use forms to retrieve specific information. As I mentioned in the introduction, this tutorial will go over retrieving custom team report data and screenshots. Check out the [documentation](https://github.com/hookengine/hubstaff-ruby) if you would like to learn about additional methods provided by the Hubstaff gem. Lets get started.

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
Next in our `show.html.erb` file we will build our buttons and forms using Bootstrap modals. Here is what the revised show file will look like:
```erb
<h1>Welcome <%= @user.email %></h1>

<% if @user.client %>
  <button type="button" class="btn btn-primary btn-lg" data-toggle="modal" data-target="#reportModal">
    Create a custom Hubstaff team report
  </button>

  <!-- Custom Report Modal -->
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

  <!-- Screenshot Modal -->
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
Finally we have to write the methods to retrieve the custom report data and screenshots. Let’s begin with the `#get_custom_report` method.
```ruby
  def get_custom_report
    user = current_user
    options = {}
    options[:orgs] = params[:orgs] unless params[:orgs] == ""
    options[:projects] = params[:projects] unless params[:projects] == ""
    options[:users] = params[:users] unless params[:users] = ""
    options[:show_tasks] = params[:show_tasks]
    options[:show_notes] = params[:show_notes]
    options[:show_activity] = params[:show_activity]
    options[:include_archieved] = params[:include_archieved]
    @report = user.client.custom_date_team(params[:start_date], params[:end_date], options)
  end
```
Before calling the Hubstaff gem method `#custom_data_team` we need to build our options hash with the params provided by the form. We also need to add some validations so that we don’t pass an empty string as a value. It’s important to use the proper key names for using all the Hubstaff methods that take a options hash correctly, please reference the [documentation](https://github.com/hookengine/hubstaff-ruby) to confirm the proper key names needed for each method. 

With the options hash built, we will simply pass it as the third argument and assign the returned JSON to @report.  As a side note, all Hubstaff methods will return JSON data. Now it’s only a matter of extracting the JSON to display nicely on our view page. Create a file called `custom_report.html.erb` and place the following code in it:
```erb
<% @report["organizations"].each do |org| %>
  <h1>Organization Name: <%= org["name"] %></h1>
  <% org["dates"].each do |date| %>
    <h2>Date: <%= date["date"] %></h2>
    <% date["users"].each do |user| %>
      <h3>User: <%= user["name"] %></h3>
        <% user["projects"].each do |project| %>
          <h1>Project: <%= project["name"] %></h1>
        <% end %>
    <% end %>
  <% end %>
<% end %>
```
Awesome now we can retrieve a custom team report from our Hubstaff account!
![Custom Report](/images/custom_report.png)
[Insert image of custom report]

Next let's create a method to retrieve screenshots. Back in our `users_controller.rb` file add the following method:
```ruby
def get_screenshots
  user = current_user
  options = {}
  options[:orgs] = params[:orgs] unless params[:orgs] == ""
  options[:projects] = params[:projects] unless params[:projects] == ""
  options[:users] = params[:users] unless params[:users] == ""
  options[:offset] = params[:offset]
  @screenshots = user.client.screenshots(params[:start_time], params[:stop_time], options)
end
```
This method is very similar to our `#get_custom_report` method, where we build a options hash based on the form params and pass that hash as the third argument to our `#screenshots` method. Now all that is left to do is display each screenshot. As I mentioned before all methods provided by the Hubstaff gem return JSON. Specifically, the `#screenshots` JSON output contains a url key with a value equal to the screenshot url. Let's extract the url address and display the each image on our view page.

Create a file called `screenshots.html.erb` and add the following code:
```ruby
<% @screenshots["screenshots"].each do |screen| %>
  <%= image_tag "#{screen['url']}"%><br><br><br>
<% end %>
```
Now when you submit the form you should be redirected to a page displaying all the screenshots that fit within the given form parameters.

The Hubstaff gem allows a Rails application to easily integrate and retrieve useful information from Hubstaff. In addition to the two methods this tutorial covered, the Hubstaff gem contains many more methods for retrieving a wide variety of data. I hope you found this tutorial helpful.

