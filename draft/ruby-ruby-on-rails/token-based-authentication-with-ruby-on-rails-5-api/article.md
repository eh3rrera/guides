With API-only applications gaining popularity and  Rails 5 being just around the corner, the most ubiquitous methods of authentication are now becoming token-based. In this tutorial, a short overview of token-based authentication will be given and implemented into a Rails 5 API-only application.


## What is token-based authentication?
 
 Token-based authentication (also known as [JSON Web Token authentication](https://jwt.io/)) is a new way of handling authentication of users in applications. It is an alternative to session-based authentication. The most notable difference between the two is that session-based authentication is persisted server-side i.e a record is created for each logged-in user. Token-based authentication is stateless - it does not store anything on the server, but creates a unique encoded token that is checked every time a request is made.  

### Benefits of token-based authentication
There are several benefits to using such approach. The main ones are:
Cross-domain / CORS: cookies + CORS don't play well across different domains. A token-based approach allows you to make AJAX calls to any server, on any domain because you use an HTTP header to transmit the user information.

- JWT are stateless there is no need to keep a session store, the token is a self-contanined entity that conveys all the user information. 


- Decoupling: you are not tied to a particular authentication scheme. The token might be generated anywhere, hence the API can be called from anywhere with a single way of authenticating all the calls.

- Mobile ready: Cookies are a problem when it comes to storing user information on native mobile applications. Adopting a token-based approach simplifies the process significantly.

- CSRF: Because the application does not rely on cookies for authentication, it is invulnerable cross site request attacks.

- Performance: In terms of server-side load, a network roundtrip (e.g. finding a session on database) is likely to take more time than calculating an HMACSHA256 code to validate a token and parsing its contents.

### How does it work?
The way token-based authentication works is simple: The user enters his/her credentials and sends a request to the server. If the credentials are correct, the server creates a unique HMACSHA256 encoded token also known as JSON web token (JWT). The client stores the JWT and makes all subsequent requests to the server with the token attached. The server authenticates the user by comparing the JWT sent with the request to the one it has stored in the database. Here is a simple diagram of the process:

![token-based auth](http://i.imgur.com/xkvip2y.jpg)


## Setting up a token-based authentication with Rails 5


Enought theory, it's time for practice. The first step is to create a new Rails 5 API-only application:

```bash
rails _5.0.0.beta3_ new api-app --api
```


Using `--api-only` when create an  API-only application. [API-only](http://edgeguides.rubyonrails.org/api_app.html) applications are a new feature that is included in Rails 5. They are trimmed down versions of standard Rails applications that have features like `.erb` views, helpers and assets removed because of redundancy. They come with special middlewares such as `ActionController::API` , request throttling, easily configurable CORS and other custom-waived features for building APIs.

To make set up token authentication for the newly created application, several steps have to be taken:

* A model that can be accessed
* A way of encoding and decoding JWT tokens
* Methods for checking if the user is authenticated.
* Controllers for creating and logging in users.
* Routes for creating users and logging them in and out.

### Creating the user model
 First, the user model must be created:
```bash
 rails g model User name email password_digest
```
 Run the migrations:
```bash
 rails db:migrate
```
 By running these methods, we create a user model in with name, e-mail and password fields and have its schema migrated in the database.
 
The method `has_secure_password`  has to be added to the model to make sure the password is properly encrypted into the database:
 
 ```ruby
#app/user.rb
 
class User < ApplicationRecord
  has_secure_password
end

```
### Encoding and decoding JWT tokens

Once the user model is done, the implementation of the JWT token generation can start. First, the `jwt` gem will make encoding and decoding of HMACSHA256 tokens available in the Rails application. Let's install it:

 
 ```ruby
  gem 'jwt'

```
And install it:

```bash
bundle install

```
Once the  gem is installed, it can be accessed through the `JWT` global variable. Here are the two methods that have to be created:
 ```ruby
 
 # lib/json_web_token.rb

class JsonWebToken
  class << self
    def encode(payload, exp = 24.hours.from_now)
      payload[:exp] = exp.to_i
      JWT.encode(payload, Rails.application.secrets.secret_key_base)
    end

    def decode(token)
      body = JWT.decode(token, Rails.application.secrets.secret_key_base)[0]
      HashWithIndifferentAccess.new body
    rescue
      nil
    end
  end
end
```
The first method,`encode`, includes three parameters - takes the user id, the expiration time (1 day) and the unique secret key of your rails application in order to create a key. The second method,`decode` takes the token and uses the application's secret key to decode it.