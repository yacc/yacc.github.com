---
layout: post
title: API design
tagline: Elements of good api design
description: Elements of good api design
category: coding
tags: [design patterns,api,authentication, rails, ruby]
---
{% include JB/setup %}

<h2>What are some elements of good api design ?</h2>
<p>
This post presents a collection of principles I followed when developing the <a href="https://sensr.net">Sensr.net API</a>, you can learn more about the sensr.net APIs by reading our <a href="http://yacc.github.com/sensrapi-tutorials/">tutorial.</a>. This is the first of several posts that will cover some of the aspects presented here.
</p>
<p>
I'm going to go over some of the basics of API design and illustrate some of the points with example implementation in Ruby on Rails.
If you're on a Rack stack, note that a nice alternative to implementing an API from the ground up is to go with <a href="https://github.com/intridea/grape">Grape</a>, an opinionated micro-framework for creating REST-like APIs in Ruby.
<ol>
	<li>Maintainability: api must be easy to test and the code easy to understand.</li>
	<li>Version: support multiple version</li>
	<li>Authentication: restrict access to private resources.</li>
	<li>Extendability: must be easy to add new API calls as new resources become available.</li>
	<li>Predictability: avoid surprising developers even if it means not following best pratices.</li>
	<li>Documentation: up-to-date and extensive documentation.</li>
	<li>Throtling</li>
	<li>Generating SDKs</li>
</ol>	
</p>

<h2>Maintainability and code structure</h2>
How do you organize your code so the API is easy to maintain ? I needed a little more control than what <a href="https://github.com/intridea/grape">Grape</a> was offering me, so intead I went for a Presenter pattern. The <a href="http://www.amazon.com/gp/product/0321127420">presenter pattern</a>
 is a way to compose a single unified object that can be utilized to properly output data for a specific purpose.
This implementaiton yields flexibility, customization, and testability instead of ActiveRecord's as_json rigidity. We basicaly setup presenters for our models and utilize them in our controllers to provide the desired structure of our JSON responses.  

<h3>Code structure</h3>
Our code is now organized this way, for all version of the api, every model has a corresponding presenter and controller.
This leads to a code that is nicely organized. Here, I'm adding a namespace to distinguish between internal API '/i/v3/' calls and user level ones '/u/v3/'

{% highlight ruby%}
app/models/user.rb
          | camera.rb
app/controllers/u/v3/base_controller.rb
                    |users_controller.rb
                    |cameras_controller.rb
               /i/v3/base_controller.rb
                    |users_controller.rb
app/presenter/u/v3/base_presenter.rb
                    |users_presenter.rb                     
                    |cameras_presenter.rb                     
{% endhighlight %}

<h3>Base Controllers</h3>
Let's take a look at our controllers first. All controllers inherit from a base controller. 
The base controller has two purposes, encapsulate behavior common to all controllers and -very important- expose a description of the resources available, we will use this later for documentation. 

{% highlight ruby%}
class U::V3::BaseController < ApplicationController  
  respond_to :json
  attr_reader :current_user, :current_tenant
  
  # this is used for API discovery and documentation
  def resources
    render :json => U::V3::ResourcesPresenter.new.description
  end
  def users
    render :text => U::V3::UserPresenter.new.description
  end
  def cameras
    render :text => U::V3::CameraPresenter.new.description
  end
end
{% endhighlight %}

<h3>Controllers</h3>

Now a resource controller is fairly straighforward. It has a before_filter to validate the token and inherits from the base controller. You'll notice that the code is well scoped and versionned. The resource is either looked up by query paramaters or in the case of the user as the owner of the API session token.
Once the resource is loaded, we just call instantiate the resource presenter and call 'render :json' on it. 

{% highlight ruby%}
class U::V3::UsersController < U::V3::BaseController

  before_filter :set_resource_owner_by_oauth_token,  :except => :register
  oauth_required :scope => "user", :except => :register
  
  def me
    logger.info "[API Client: #{oauth.client}] with scope #{oauth.scope} and tenant  [#{oauth.identity}]"
    render :json => U::V3::UserPresenter.new(current_user), :status => :ok
  end
end
{% endhighlight%}

<h3>Base api presenter</h3>
{% highlight ruby%}
class ApiPresenter
  def initialize(resource=nil,options={})
    @options = options
    if Rails.env.test? || Rails.env.build? || Rails.env.development? 
      @basepath = 'https://sensrapi.dev'
    elsif Rails.env.staging?
      @basepath = 'https://api.stagingserver.com'
    else
      @basepath = 'https://api.sensr.net'          
    end    
  end
end
{% endhighlight%}

<h3>User presenter</h3>
{% highlight ruby%}
class U::V3::UserPresenter < ApiPresenter
  attr_reader :user

  def initialize(user=nil,options={})
    super
    @user = user
  end

  def description
    File.read("#{Rails.root}/app/assets/resources/u_v3_users.json").gsub("https://api.sensr.net", @basepath)
  end
  
  def as_json(options={})
    @options.merge!(options)
    data = {
      :user => {
        :id => @user.id,
        :email => @user.email,
        :name  => @user.name,
      },
      :urls => {
        :cancel => "#{@basepath}/u/v3/users/#{@user.id}/cancel",
        :update => "#{@basepath}/u/v3/users/#{@user.id}/update",
        :my_cameras => "#{@basepath}/u/v3/cameras/owned",
        :my_clips => "#{@basepath}/u/v3/clips/owned"      }
    }
    if @options[:include] == :cameras
      data[:user][:cameras] = @user.cameras.collect { |camera| U::V3::CameraPresenter.new(camera, {:include => :ftp_user}) }
    end
    data
  end
     
end
{% endhighlight%}

<h2>Versionning</h2>
<p>
Some of the principles I followed when developing our API
<ol>
	<li>Always include a version when releasing an API.</li>
	<li>Developers must specify a version when calling the API</li>
	<li>Specify the version with a 'v' prefix (ex: /u/v3/cameras.json)</li>
</ol>
</p>
<p>
As a developer, I like seeing the version of the api on the URL rather than in the HTTP header.
With the code structure and the namespacing introduced above in the presenters and the controller, we can now release new versions of our API along side the old ones. We can apply our namescoping to our models, controllers, presenters, and specs.
</p>

The other aspecty will be presented in follow-up posts.