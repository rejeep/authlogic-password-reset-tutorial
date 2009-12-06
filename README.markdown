# Authlogic Password Reset Tutorial

## Introduction

This tutorial will help you set up password reset restfully with Authlogic.
This tutorial is based on
[this blog post](http://www.binarylogic.com/2008/11/16/tutorial-reset-passwords-with-authlogic/)
Hopefully having it as a Git repository, it will always be up to date.

1. A user requests a password reset
2. An email is sent to them with a link to reset their password
3. The user verifies their identity by using the link in the email
4. They are presented with a basic form to change their password
5. After they change their password we log them in

Shoulda
Formtastic
Cucumber
Haml

## Perishable Token

First, you need to add a column in the users table, called
**perishable_token**. A perishable_token is a temporary string used
for identification.

Generate the migration:

    $ script/generate migration add_perishable_token_to_users

And add this contents:
    class AddUsersPasswordResetFields < ActiveRecord::Migration  
      def self.up  
        add_column :users, :perishable_token, :string, :default => "", :null => false
          
        add_index :users, :perishable_token  
      end  
        
      def self.down  
        remove_column :users, :perishable_token  
      end  
    end  
    
Don't forget to migrate the database.
    
    $ rake db:migrate

## Password Resets controller

Next up we add a controller called password resets.

    $ script/generate controller password_resets
    
Add this contents to it:
    
    class PasswordResetsController < ApplicationController

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
          flash[:error] = "No user was found with that email address"
     
          render :action => :new
        end
      end
     
      def edit
      end
     
      def update
        @user.password = params[:user][:password]
        if @user.save
          flash[:success] = "Password successfully updated"
     
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
    
Add a route to that controller:

    map.resources :password_resets, :only => [ :new, :create, :edit, :update ]
    
New view:
    
    %h1 Reset Password
     
    %p Please fill in your email address below.
     
    - form_tag password_resets_path do
      = text_field_tag :email
      = submit_tag "Reset Password"

Edit view:

    %h1 Update your password
     
    - semantic_form_for @user, :url => password_reset_path, :method => :put do |form|
      - form.inputs do
        = form.input :password, :as => :password
      - form.buttons do
        = form.commit_button

You might also want to test the controller. Here's an example with Shoulda.
This test also inclues the case when a user is logged in by using
**should_require_no_user**. If you are not familiar with that, read
[this blog post](http://tuxicity.se/rails/refactor/test/2009/11/24/testing-user-privileges-with-shoulda.html)
If you feel that it is unnessesary, just remove those four lines in the test.

    class PasswordResetsControllerTest < ActionController::TestCase
     
      request_new = lambda do
        get :new
      end
     
      request_create = lambda do
        post :create, :email => Factory(:user).email
      end
     
      request_edit = lambda do
        get :edit, :id => Factory(:user).perishable_token
      end
     
      request_update = lambda do
        @user = Factory(:user)
     
        put :update, :id => @user.perishable_token, :user => { :password => "newpassword" }
      end
     
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
          should_redirect_to("the dashboard") { root_path }
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
          should_redirect_to("the dashboard") { @user }
        end
      end
     
    end
    
As you might noticed, in the password resets controller there was a
method call on the user called
**deliver_password_reset_instructions**.
Lets add that method to the user model:

    class User < ActiveRecord::Base
      def deliver_password_reset_instructions!
        reset_perishable_token!
       
        Notifier.deliver_password_reset_instructions(self)
      end
    end
    
Lets also add a **simple** test for this:

    context "Delivering password instructions" do
      setup do
        @user = Factory(:user)
        @user.deliver_password_reset_instructions!
      end
   
      should "send an email" do
        assert_sent_email
      end
    end    

Now we need to add that mailer method:
    
    class Notifier < ActionMailer::Base
      def password_reset_instructions(user)
        subject       "Password Reset Instructions"
        from          "noreplay@domain.com
        recipients    user.email
        sent_on       Time.now
        body          :edit_password_reset_url => edit_password_reset_url(user.perishable_token)
      end
    end
    
The mailer view:

    %h1= Password Reset Instructions

    %p
      A request to reset your password has been made. If you did not
      make this request, simply ignore this email. If you did make this
      request, follow the link below.

    = link_to "Reset Password!", @edit_password_reset_url

And the mail test:

    class NotifierTest < ActionMailer::TestCase
      context "email password reset instructions" do
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

You might also want to create a Cucumber test for it:

  Scenario: Reset password
    Given I am not logged in
    And there is a user with login "login" and email "user@domain.com" and password "password"
    And I am on the login page
    Then I should see "Forgot Password"
    When I follow "Forgot Password"
    Then I should see "Reset Password"
    And I should see "Please fill in your email address below"
    When I fill in "email" with "user@domain.com"
    And I press "Reset Password"
    Then I should see "Instructions to reset your password have been emailed to you"
    And "user@domain.com" should have an email
    When I open the email
    Then I should see "Password Reset Instructions" in the email body
    When I follow "Reset Password" in the email
    Then I should see "Update your password"
    When I fill in "Password" with "newpassword"
    And I press "Update Password"
    Then I should see "Password successfully updated"
    When I am not logged in
    Then I should be able to log in with login "login" and password "newpassword"


Here are the needed user steps:

    Given /^I am not logged in$/ do
      visit logout_path
    end
   
    Given /^there is a user with login "([^\"]*)" and password "([^\"]*)"$/ do |login, password|
      Factory(:user, :login => login, :password => password)
    end
   
    Given /^there is a user with login "([^\"]*)" and email "([^\"]*)" and password "([^\"]*)"$/ do |login, email, password|
      Factory(:user, :login => login, :email => email, :password => password)
    end
   
    Then /^I should be able to log in with login "([^\"]*)" and password "([^\"]*)"$/ do |login, password|
      UserSession.new(:login => login, :password => password).save.should == true
    end


If you liked this tutorial, I can also recommend this
[Authlogic Activation Tutorial](http://github.com/matthooks/authlogic-activation-tutorial)
