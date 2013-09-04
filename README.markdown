# Authlogic Password Reset Tutorial

## Introduction

This tutorial will help you set up password reset restfully, with
[Authlogic](http://github.com/binarylogic/authlogic).
This tutorial is based on
[this blog post](http://www.binarylogic.com/2008/11/16/tutorial-reset-passwords-with-authlogic/).
Hopefully, having the tutorial as a Git repository, it will be more up
to date.

To reset a password, the user goes through these steps:

1. The user requests a password reset
2. An email is sent to the user with instructions
3. The user verifies their identity by using the link in the email
4. The user is presented with a simple form for updating the password

This tutorial includes all code, including tests. Below you'll see the
tools that are used. If you're not familiar with any of them, check
them out or replace them with your favorite tools.

* [Shoulda](http://github.com/thoughtbot/shoulda)
* [Cucumber](http://github.com/aslakhellesoy/cucumber)
* [Haml](http://github.com/nex3/haml)
* [Factory Girl](http://github.com/thoughtbot/factory_girl)

_Note that this tutorial assumes you have your user and user session stuff already set up._


## Perishable Token

Perishable token as the name implies is a temporary string. Authlogic
use it for simple identification. Add a column called
**perishable_token** to your table and Authlogic will basically handle
the rest.

Generate the migration:

    $ script/generate migration add_perishable_token_to_users

The migration contents:

    class AddPerishableTokenToUsers < ActiveRecord::Migration
      def self.up
        add_column :users, :perishable_token, :string, :default => "", :null => false
        add_index :users, :perishable_token
      end

      def self.down
        remove_column :users, :perishable_token
      end
    end

Don't forget to migrate the database:

    $ rake db:migrate


## Password Resets controller

Next up we add a controller called _password resets_:

    $ script/generate controller password_resets

With this contents:

    class PasswordResetsController < ApplicationController
      # Method from: http://github.com/binarylogic/authlogic_example/blob/master/app/controllers/application_controller.rb
      before_filter :require_no_user
      before_filter :load_user_using_perishable_token, :only => [ :edit, :update ]
     
      def new
      end
     
      def create
        @user = User.find_by_email(params[:email])
        if @user
          @user.deliver_password_reset_instructions!
          flash[:notice] = "Instructions to reset your password have been emailed to you"
          redirect_to root_path
        else
          flash.now[:error] = "No user was found with email address #{params[:email]}"
          render :action => :new
        end
      end
     
      def edit
      end
     
      def update
        @user.password = params[:password]
        # Only if your are using password confirmation
        # @user.password_confirmation = params[:password]
        
        # Use @user.save_without_session_maintenance instead if you
        # don't want the user to be signed in automatically.
        if @user.save
          flash[:success] = "Your password was successfully updated"
          redirect_to @user
        else
          render :action => :edit
        end
      end
     
     
      private
     
      def load_user_using_perishable_token
        @user = User.find_using_perishable_token(params[:id])
        unless @user
          flash[:error] = "We're sorry, but we could not locate your account"
          redirect_to root_url
        end
      end
    end

Add this route:

    map.resources :password_resets, :only => [ :new, :create, :edit, :update ]

### Views

#### Action: new
##### Erb

    <h1>Reset Password</h1>
     
    <p>Please enter your email address below and then press "Reset Password".</p>
     
    <%= form_tag password_resets_path do %>
      <%= text_field_tag :email %>
      <%= submit_tag "Reset Password" %>
    <% end %>

##### Haml

    %h1 Reset Password
     
    %p Please enter your email address below and then press "Reset Password".
     
    - form_tag password_resets_path do
      = text_field_tag :email
      = submit_tag "Reset Password"

#### Action: edit

##### Erb

    <h1>Update your password</h1>
     
    <p>Please enter the new password below and then press "Update Password".</p>
     
    <%= form_tag password_reset_path, :method => :put do %>
      <%= password_field_tag :password %>
      <%= submit_tag "Update Password" %>
    <% end %>

##### Haml

    %h1 Update your password
     
    %p Please enter the new password below and then press "Update Password".
     
    - form_tag password_reset_path, :method => :put do
      = password_field_tag :password
      = submit_tag "Update Password"


## Functional test

To use this test directly you need the method
**should_require_no_user**. Read the blog post
[Testing User Privileges With Shoulda](http://blog.tuxicity.se/rails/refactoring/testing/2009/11/24/testing-user-privileges-with-shoulda.html)
for more information. If you feel that it is unnessesary, just remove
those four lines from the test.

    class PasswordResetsControllerTest < ActionController::TestCase
      request_new = proc do
        get :new
      end
     
      request_create = proc do
        post :create, :email => Factory(:user).email
      end
     
      request_edit = proc do
        get :edit, :id => Factory(:user).perishable_token
      end
     
      request_update = proc do
        @user = Factory(:user)
        put :update, :id => @user.perishable_token, :user => { :password => "newpassword" }
      end
     
      # Remove these if you don't want to include the Shoulda macros
      should_require_no_user "on GET to :new", &request_new
      should_require_no_user "on POST to :create", &request_create
      should_require_no_user "on GET to :edit", &request_edit
      should_require_no_user "on PUT to :update", &request_update
     
      context "when not logged in" do
        context "on GET to :new" do
          setup &request_new
     
          should_respond_with :success
          should_render_template :new
        end
     
        context "on POST to :create" do
          setup &request_create
     
          should_assign_to :user
          should_respond_with :redirect
          should_redirect_to("the root path") { root_path }
          should "send an email" do
            assert_sent_email
          end
        end
     
        context "on GET to :edit" do
          setup &request_edit
     
          should_assign_to :user
          should_respond_with :success
          should_render_template :edit
        end
     
        context "on PUT to :update" do
          setup &request_update
     
          should_assign_to :user
          should_respond_with :redirect
          should_redirect_to("the users profile") { @user }
        end
      end
    end


## The mail
As you might noticed, in the password resets controller there is a
method call on the user to **deliver_password_reset_instructions!**.

Lets add that method to the user model:

    class User < ActiveRecord::Base
      def deliver_password_reset_instructions!
        reset_perishable_token!
        Notifier.deliver_password_reset_instructions(self)
      end
    end

And test it:

    class UserTest < ActiveSupport::TestCase
      context "A user" do
        setup { @user = Factory(:user) }
     
        context "Delivering password instructions" do
          setup { @user.deliver_password_reset_instructions! }
     
          should_change("perishable token") { @user.perishable_token }
          should "send an email" do
            assert_sent_email
          end
        end
      end
    end

Add the mailer method:

    class Notifier < ActionMailer::Base
      def password_reset_instructions(user)
        subject      "Password Reset Instructions"
        from         "noreplay@domain.com"
        recipients   user.email
        content_type "text/html"
        sent_on      Time.now
        body         :edit_password_reset_url => edit_password_reset_url(user.perishable_token)
      end
    end

### Mailer view

#### Erb

    <h1>Password Reset Instructions</h1>
     
    <p>
      A request to reset your password has been made. If you did not make
      this request, simply ignore this email. If you did make this
      request, please follow the link below.
    </p>
     
    <%= link_to "Reset Password!", @edit_password_reset_url %>

#### Haml

    %h1 Password Reset Instructions
     
    %p
      A request to reset your password has been made. If you did not make
      this request, simply ignore this email. If you did make this
      request, please follow the link below.
     
    = link_to "Reset Password!", @edit_password_reset_url

### Test
Test the mailer:

    class NotifierTest < ActionMailer::TestCase
      context "delivering password reset instructions" do
        setup do
          @user = Factory(:user)
          @user.deliver_password_reset_instructions!
        end
     
        should "send an email" do
          assert_sent_email do |email|
            email.subject =~ /Password Reset Instructions/
            email.body =~ /#{@user.perishable_token}/
            email.body =~ /Password Reset Instructions/
          end
        end
      end
    end


## Cucumber test
And at last, you might also want to create a Cucumber test for
it. Note that you need the
[Email Spec plugin](http://github.com/bmabey/email-spec)
for this feature.

    Feature: Password Reset
      In order to retrieve a lost password
      As a user of this site
      I want to reset it
     
      Scenario: Reset password
        Given I am not logged in
        And a user exists with email: "user@domain.com", password: "password"
        And I am on the login page
        Then I should see "Forgot Password"
        When I follow "Forgot Password"
        Then I should see "Reset Password"
        And I should see "Please enter your email address below"
        When I fill in "email" with "user@domain.com"
        And I press "Reset Password"
        Then I should see "Instructions to reset your password have been emailed to you"
        And "user@domain.com" should have an email
        When I open the email
        Then I should see "Password Reset Instructions" in the email body
        When I follow "Reset Password" in the email
        Then I should see "Update your password"
        When I fill in "password" with "newpassword"
        And I press "Update Password"
        Then I should see "Your password was successfully updated"
        When I am not logged in
        Then I should be able to log in with email "user@domain.com" and password "newpassword"
     
      Scenario: Reset password no account
        Given I am not logged in
        And I am on the login page
        Then I should see "Forgot Password"
        When I follow "Forgot Password"
        Then I should see "Reset Password"
        And I should see "Please enter your email address below"
        When I fill in "email" with "user@domain.com"
        And I press "Reset Password"
        Then I should see "No user was found with email address user@domain.com"  

### User steps

    Given /^I am not logged in$/ do
      visit logout_path
    end
     
    Then /^I should be able to log in with email "([^\"]*)" and password "([^\"]*)"$/ do |email, password|
      UserSession.new(:email => email, :password => password).save.should == true
    end
     
    # If you use Pickle (http://github.com/ianwhite/pickle) this step is already defined
    Given /^a user exists with email: "([^\"]*)", password: "([^\"]*)"$/ do |email, password|
      Factory(:user, :email => email, :password => password)
    end


## Sum up

Feel free to contact me if there's anything in the tutorial that is
incorrect or if you have any good improvement suggestions.

If you liked this tutorial, I can also recommend the
[Authlogic Activation Tutorial](http://github.com/matthooks/authlogic-activation-tutorial).
