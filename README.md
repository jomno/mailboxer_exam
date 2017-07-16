# hello gorails

## Gemfile 
```ruby
gem 'devise'
gem 'mailboxer'
```
## devise install bash
```bash
rails g devise:install
rails g devise user
```
## devise setting db/seeds.rb 가상의 유저 생성
```ruby
User.create(email: 'man@gmail.com',password: 'password',password_confirmation: 'password')
User.create(email: 'woman@gmail.com',password: 'password',password_confirmation: 'password')
```
## mailboxer setup bash
```bash
bundle install
rails g mailboxer:install
rake db:migrate
```
## user mailbox connect app/models/user.rb
```ruby
class User < ActiveRecord::Base
  # Include default devise modules. Others available are:
  # :confirmable, :lockable, :timeoutable and :omniauthable
  devise :database_authenticatable, :registerable,
         :recoverable, :rememberable, :trackable, :validatable
  acts_as_messageable
  def name
    "User #{id}"
  end
  
  def mailboxer_email(object)
    nil
  end
end

```
## route setting config/routes.rb
```ruby
devise_for :users
  resources :conversations do
    resources :messages
  end
```
## controller setting /app/controllers/conversations_controller.rb
```ruby
class ConversationsController < ApplicationController
    def index
        @conversations = current_user.mailbox.conversations
    end
    def show
        @conversation = current_user.mailbox.conversations.find(params[:id])
    end
    def new
        @recipients = User.all - [current_user]
    end
    def create
        recipient = User.find(params[:user_id])
        receipt = current_user.send_message(recipient, params[:body], params[:subject])
        redirect_to conversation_path(receipt.conversation)
    end
end
```
## controller setting /app/controllers/messages_controller.rb
```ruby
class MessagesController < ApplicationController
    before_action :set_conversation
    
    def create
        receipt = current_user.reply_to_conversation(@conversation, params[:body])
        redirect_to conversation_path(receipt.conversation)
    end
    
    private
    
    def set_conversation
        @conversation = current_user.mailbox.conversations.find(params[:conversation_id])
    end
end
```
## view setting app/views/conversations/index.html.erb
```html
<h1>All conversations</h1>
<% @conversations.each do |conversation| %>
<div>
    <%= link_to conversation.subject, conversation_path(conversation) %>
</div>
<% end %>

<%= link_to "New Conversation", new_conversation_path %>
```
## view setting app/views/conversations/new.html.erb
```html
<h1>New conversation</h1>

<%= form_tag conversations_path, method: :post do %>
    <div>
        <%= select_tag :user_id, options_from_collection_for_select(@recipients, :id, :email)%>
    </div>
    <div>
        <%= text_field_tag :subject,nil ,placeholder: "Subject"%>
    </div>
    <div>
        <%= text_area_tag  :body,nil,placeholder: "body"%>
    </div>
    <%= submit_tag %>
<% end %>
```
## view setting app/views/conversations/show.html.erb
```html
<h1><%= @conversation.subject %></h1>

<% @conversation.receipts_for(current_user).each do |receipt| %>
<div>
    <div><%= receipt.message.sender.email %> commented: </div>
    <p><%= receipt.message.body %></p>
</div>
<% end %>

<%= form_tag conversation_messages_path(@conversation), method: :post do %>
    <div>
        <%= text_area_tag :body %>
    </div>
    
    <%= submit_tag %>
<%end%>
```