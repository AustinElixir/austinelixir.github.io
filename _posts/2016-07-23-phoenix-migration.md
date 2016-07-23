---
layout: post
title: Migrating a LAMP Stack Application to the Phoenix Framework
---

I have an application with around 100K lines of backend code written in Perl served up in a traditional LAMP stack. I'd like to migrate this application to the Phoenix Framework and I've documented a few ideas that I hope may be helpful to you if you are considering the same option. I'll demonstrate using PHP as it's the more common case.

## Problem Context

Various business constraints prevent me from migrating everything in one monolithic update as would (arguably) be the traditional approach. Hopefully less often as time goes on.

#### Constraints

- Users must be part of the feedback cycle as application logic is migrated into Phoenix because the UI will be affected
- Users must not experience a shocking overnight change to the UI which adds administrative burden i.e. user training
- Sales must not be postponed due to avoidance of technical debt from installing our legacy application
- Each iterative update will remove technical debt by displacing legacy code so we feel relief with each release
- Feature requests will continue to flow and we don't want to accommodate them in two code bases to save development cost

There are also many other reasons to chose an iterative migration approach purely based on technical reasons which closely overlap with the business reasons. I'm sure you can imagine!

I've chosen Phoenix as the new framework for this application and have easily found a good strategy for migrating the application logic steadily over time so all parties are satisfied. Let's take a look!

## Proposed Solution

After much deliberation and testing, I believe the easiest and cleanest approach is to put a reverse proxy up front and segment traffic between the legacy and Phoenix applications. Application logic can be migrated easily and efficiently with each release this way and it keeps your routing very simple.

It would look like this:

<img alt="Nginx reverse proxy for LAMP stack and Phoenix Framework" src="/img/Migrating to Phoenix Framework - Reverse Proxy.png"/>

#### Pros & Cons of the this Approach

- Pros
  - Low overhead and quick to setup
  - Very easy to remove Nginx server after the migration is completed
  - Available routes to each application would be very explicit and prevent confusion with which routes are going to legacy code or to Phoenix
  - Side-by-side performance comparison between frameworks

- Cons
  - Forced to update your URL paths in the UI to point to new routes
  - You may need to enable logging to catch missed UI requests hitting old routes

#### Non Nginx Solution

I also considered creating a reverse proxy in the Phoenix or even the legacy application. A very simple test in Phoenix would be something like this:

{% highlight elixir %}
defmodule VanillaPhx.RProxyController do
  use VanillaPhx.Web, :controller

  def index(conn, _params) do
    response = HTTPotion.get "http://localhost:3000/someroute"
       conn
    |> put_resp_content_type(response.headers.hdrs[:"content-type"])
    |> send_resp(response.status_code, response.body)
  end
end
{% endhighlight %}

If you pull a page from any number of frameworks, such as Ruby on Rails, you'll see common routes like `/assets`, `/css`, or `/js` quite often. So right up front you'll spend time deconflicting routes and strategizing your migration plan. If you install a catch-all route in Phoenix to get responses from the old application, there is a better chance you'll make a mistake because things are not as explicit. Not to mention, all the things a reverse proxy can do that need to be replicated! It's a fair amount of work to be sure.

> If putting in your own reverse proxy is the right thing for your circumstances, go for it!

I'm going to demonstrate what I believe will cover the most use cases with a POC based on a very simple PHP site. I'm not at all a PHP programmer, so I apologize for any idiomatic violatons. It's the same overall approach no matter what you're comming from. I chose to demonstrate coming from no alternative framework at all, that being my personal circumstance with my legacy Perl application.

## Authentication Implications:

This is an important consideration you must give thought to. With a reverse proxy up front, you'll be able to pull cookie data regardless of which application sets it. Just make sure you're not breaking anything :) e.g. In my case, I will be setting session cookies from Phoenix using a token based authentication model. My old code won't understand it so I'll have to upgrade the old code. Just think your situation through and test your ideas early to ensure you've got a solid plan.

## Database Considerations:

Another consideration is the database type and schema. Luckily, <a href="https://hexdocs.pm/ecto/Ecto.Schema.html" target="_blank">Ecto</a> can easily model the existing schema for PostgreSQL, MySQL, MSSQL, and SQLite3. However, if you're existing application doesn't run on anything Ecto supports out of the box, then you'll need to build your own adapter, tackle migrating your database, or use a message bus or similar method to get the information you need. At least in the short term while you migrate your DB later. Too many options to consider in this blog post!

In my humble opinion, migrating the application logic is enough work. You don't want mess with your database anymore than you absolutely have to. I think you can do it if you must, but it's helpful to limit the moving pieces to make this transition as stable as you can. In my case, I'll be migrating from MySQL to PostgreSQL after Phoenix is fully implemented.

## Analysis

As you can see, I've laid out some fundamental considerations for routing, authentication, and databases. Your intimate knowledge of your application is ultimately what matters. I hope some of the points I've made assist you in selecting a path forward.

## Proof-of-Concept

The goal here is to get the fundamentals down and understand the overall strategy for migration. You should be able to take this example and tweak it to test out ideas that are specific to your needs. We're only covering routing and migrating application code from legacy into Phoenix and connecting to the same PostgreSQL server.

We'll walk through the a simple example of using Nginx as a reverse proxy. Each step will be provided and you'll be able to implement it as a POC using the provided git repository.

### Code Repository

For simplicity, I've setup a single repository which contains a Vagrantfile that will provide:

- Phoenix, Elixir, and Erlang
- Nginx
- Apache+PHP
- Postgresql

There are two folders:

- **conf**: _Server Configs_
- **phx_reference**: _Modules and Configs for Reference_
- **vanilla**: _PHP Site_

Clone the repo into your local folder:

`git clone https://github.com/AustinErlang/PhoenixAppMigration`

Install <a href="https://www.vagrantup.com" target="_blank">Vagrant</a> If you don't already have it.

`cd PhoenixAppMigration && vagrant up`

Also keep an open session into your virtual host.

`vagrant ssh`

### What is done for you

**a. Postgresql Setup**

In the `conf` directory, there is a `posts.sql` file which is automatically imported when you build your vagrant box.

The schema for the `posts` table is created with this SQL:

{% highlight sql %}
CREATE TABLE posts (
    id integer NOT NULL,
    title character varying(255),
    body text,
    created_at timestamp without time zone,
    updated_at timestamp without time zone
);
{% endhighlight %}

The postgresql server should be running and the user/pass is `postgres` / `postgres`

**b. Apache and PHP Setup**

In the `conf` directory, you'll find an apache config file for a single site. During the vagrant build it was copied to `/etc/apache2/sites-enabled/vanilla_site.conf`. The home directory for the PHP site is `/vagrant/vanilla/`. You can restart apache if you make any changes with `sudo service apache2 restart`.

In the `vanilla` folder, you'll find two simple PHP scripts called `index.php` and `updates.php`. The idea is just to provide <a href="https://en.wikipedia.org/wiki/Create,_read,_update_and_delete" target="_blank">CRUD</a> operations on a table called `posts` without any fancy framework as many sites still work this way. The intent is to migrate some of these operations into Phoenix so you can see the basic idea.

You should be able to test the legacy app here: <a href="http://localhost:8090/" target="_blank">http://localhost:8090/</a>. Create some test posts and ensure the <a href="https://en.wikipedia.org/wiki/Create,_read,_update_and_delete" target="_blank">CRUD</a> operations are all working.

### What you must do

**a. Nginx Proxy Setup**

Here's a diagram showing what we're going to setup with Nginx as our reverse proxy for Apache and Phoenix.

<img alt="Nginx reverse proxy for LAMP stack and Phoenix Framework" src="/img/Migrating to Phoenix Framework - Reverse Proxy.png"/>

> The main thing to understand is that we're going to be rewriting the URL to strip off `^/newsite/` from the path. This allows Phoenix to have routes as if it were at the top level so when you remove this proxy server later it's a breeze.

> Phoenix prepends the URL with `/newsite/` for the client so they can get back to the Phoenix server via the reverse proxy.

If you've verified Apache is serving on <a href="http://localhost:8090/" target="_blank">8090</a>, setup Nginx to proxy port 8080 requests to this server.

{% highlight nginx %}
server {
	listen 8080 default_server;

	# Upgrade connection for the live reload websocket
	location /newsite/phoenix/live_reload/socket/websocket {
		rewrite ^/newsite/(.*)$ /$1 break;
		proxy_pass http://localhost:4000;
		proxy_http_version 1.1;
		proxy_set_header Upgrade $http_upgrade;
		proxy_set_header Connection "upgrade";
	}

	# Proxy to Phoenix Framework
	location /newsite {
		rewrite ^/newsite/(.*)$ /$1 break;
		proxy_pass http://localhost:4000;
	}

	# Proxy to Apache site
	location / {
		proxy_pass http://localhost:8090;
	}
}
{% endhighlight %}

You can simply move the provided config if you like:

`sudo cp /vagrant/conf/nginx_proxy /etc/nginx/sites-enabled/ && sudo service nginx restart`

Now test the functionality of the php application using the proxied port <a href="http://localhost:8080/" target="_blank">http://localhost:8080/</a>.

If that's working, you're ready to start working in Phoenix!

**b. Phoenix.New**

Now we're going to create a phoenix project called `vanilla_phx` and test the server. The vagrant box should have everything you need to execute these commands and start serving a default phoenix page quickly.

{% highlight bash %}
cd /vagrant
mix phoenix.new vanilla_phx
cd vanilla_phx
sed -i 's/vanilla_phx_dev/vanilla/' /vagrant/vanilla_phx/config/dev.exs
mix phoenix.server
{% endhighlight %}

> Notice I have shimmed a sed command in here to set the database name to "vanilla". This matches the database name used by the legacy php code.

After these commands you should see a default page at <a href="http://localhost:4000/" target="_blank">http://localhost:4000/</a>.

Make sure you can also access this default page via <a href="http://localhost:8080/newsite/" target="_blank">http://localhost:8080/newsite/</a>.

> The page may not show up correctly as we need to adjust a few things. As long as the request makes it through move to the next step.

**c. Creating a Model**
  
With a working phoenix server, we can now create a model for the existing table in our database.

Create a file called `/vagrant/vanilla_phx/web/models/posts.ex` with this code:

{% highlight elixir %}
defmodule VanillaPhx.Posts do
  use VanillaPhx.Web, :model

  schema "posts" do
    field :title, :string
    field :body, :string
    field :created_at, Ecto.DateTime
    field :updated_at, Ecto.DateTime
  end

  @doc """
  Builds a changeset based on the `struct` and `params`.
  """
  def changeset(struct, params \\ %{}) do
    struct
    |> cast(params, [:title, :body])
    |> validate_required([:title, :body])
  end
end
{% endhighlight %}

If your Phoenix server is running kill it by holding `Ctrl` and hitting `c` twice.

Ensure your code is ok by compiling it first with `mix compile`. You shouldn't see any errors as all this was tested.

Now start your Phoenix server with: `iex -S mix phoenix.server`

In the iex shell, you can test your model. You should have already created at least one post using the PHP legacy application. Take one of the ID numbers for a post you created and try to query the repo with it.

{% highlight shell %}
iex(1)> alias VanillaPhx.Repo
iex(2)> alias VanillaPhx.Posts
iex(3)> Repo.get(Posts, 4)
[debug] QUERY OK db=1.7ms
SELECT p0."id", p0."title", p0."body", p0."created_at", p0."updated_at" FROM "posts" AS p0 WHERE (p0."id" = $1) [4]
%VanillaPhx.Posts{__meta__: #Ecto.Schema.Metadata<:loaded, "posts">,
 body: "hoasuthoasu osuh osenh aonsuh snuaoe", created_at: nil, id: 4,
 title: "Test3", updated_at: nil}
iex(10)> 
{% endhighlight %}

As you can see, the model returns a struct with the data from the posts table.

Now you've verified your model works and you can query your table. Let's use this model to replace the update operation from the PHP application.

**d. Setting Phoenix Path to /newsite/**

- `vanilla_phx/config/config.exs`

**Change**

{% highlight elixir %}
url: [host: "localhost"],
{% endhighlight %}

**To**

{% highlight elixir %}
url: [host: "localhost", path: "/newsite"],
{% endhighlight %}

- `vanilla_phx/web/static/css/phoenix.css`

**Change**

{% highlight css %}
background-image: url("/images/phoenix.png");
{% endhighlight %}

**To**

{% highlight css %}
background-image: url("/newsite/images/phoenix.png");
{% endhighlight %}

> **Note:** After you edit css or js files you may need to remove the cached files in: `priv/static/`

**e. Updating a Post**

Next we need to add the necessary code to process the update request from the web form coming from the PHP application.

Please create these files and/or directories yourself as they are described.

I'll use relative paths assuming you're in the `/vagrant/vanilla_phx/web/` directory.

- `router.ex`

**Change**

{% highlight elixir %}
...
  scope "/", VanillaPhx do
    pipe_through :browser # Use the default browser stack

    get "/", PageController, :index
  end
...
{% endhighlight %}

**To**

{% highlight elixir %}
...
  scope "/", VanillaPhx do
    pipe_through :browser # Use the default browser stack

    get "/", PageController, :index
    resources "/posts", PostController, only: [:index, :edit, :update]
  end
...
{% endhighlight %}

The addition of the `resources` route will provide three additional paths which are compliant with REST path standards.

{% highlight bash %}
$ mix phoenix.routes
page_path  GET    /                VanillaPhx.PageController :index
post_path  GET    /posts           VanillaPhx.PostController :index
post_path  GET    /posts/:id/edit  VanillaPhx.PostController :edit
post_path  PATCH  /posts/:id       VanillaPhx.PostController :update
           PUT    /posts/:id       VanillaPhx.PostController :update
{% endhighlight %}

> Note the presence of `post_path` and the atoms `:edit`, `:update`, and `:index` to the right. This will be key for URL generation in the templates.

- `controllers/post_controller.ex`

{% highlight elixir %}
defmodule VanillaPhx.PostController do
  use VanillaPhx.Web, :controller
  alias VanillaPhx.Repo
  alias VanillaPhx.Posts

  def index(conn, _params) do
    render conn, "index.html"
  end

  def edit(conn, %{"id" => id}) do
    post = Repo.get(Posts, id)
    changeset = Posts.changeset(post)
    render conn, "edit.html", post: post, changeset: changeset
  end

  def update(conn, %{"id" => id, "posts" => %{"title" => title, "body" => body}}) do
  	post = Repo.get(Posts, id)
  	changeset = Posts.changeset(post, %{title: title, body: body})
  	Repo.update(changeset)
  	conn
  	|> put_layout(false)
  	|> render("index.html")
  end
end

{% endhighlight %}

> This simple code will take the URL parameters, create a changeset from the posts model, and then update the table with the new title and body.

- `views/post_view.ex`

{% highlight elixir %}
defmodule VanillaPhx.PostView do
  use VanillaPhx.Web, :view
end
{% endhighlight %}

> This is the default view code which will import a render/3 function from Phoenix.view.

- `templates/post/index.html.eex`

{% highlight shell %}
Post Updated! <a href='/index.php'>Return</a>
{% endhighlight %}

- `templates/post/edit.html.eex`

{% highlight shell %}
<h1>Edit Post</h1>

<%= form_for @changeset, post_path(@conn, :update, @post.id), fn f -> %>
  <div class="form-group">
    <%= text_input f, :title, placeholder: "Title", class: "form-control" %>
  </div>
  <div class="form-group">
    <%= textarea f, :body, placeholder: "Body", class: "form-control" %>
  </div>
  <%= submit "Save Post", class: "btn btn-primary" %>
<% end %>
{% endhighlight %}

> Notice `post_path(@conn, :update, @post.id)` and how that correlates with the routes we showed earlier.

You're welcome to test this now with a manually crafted url such as:

`http://localhost:8080/newsite/posts/{ID NUMBER HERE}/edit`

> If your changes don't automatically rebuild, it's probably due to a known issue with Virtualbox and sendfile. Just kill your server and restart it with `iex -S mix phoenix.server`.

**f. Update Legacy Application UI**

The final step in our simple POC is to update index.php to point to `/newsite`.

- `/vagrant/vanilla/index.php`

**Change**

{% highlight php %}
echo "<a href='/updates.php?action=edit&id=". $row['id'] ."'>" . $row['title'] . "</a><br/>";
{% endhighlight %}

**To**

{% highlight php %}
echo "<a href='/newsite/posts/". $row['id'] ."/edit'>" . $row['title'] . "</a><br/>";
{% endhighlight %}

Test it: <a href="http://localhost:8080/index.php" target="_blank">http://localhost:8080/index.php</a>

Now if you click any post and submit the form to update it, Phoenix will process the request and make the change! Now you have seen a full example of how this migration process could work. Cheers!

### Conclusion

Hopefully now you understand how you might migrate your application to use the Phoenix Framework. Phoenix makes this very easy by allowing a central place to add a prefix to all URLs generated by the application code. Static paths in Javascript or CSS can easily be modified in batch after the migration is over with search and replace, sed, or anything of the like.

I hope this has been helpful, please join our <a href="https://groups.google.com/forum/#!forum/austinerlang" target="_blank">Google Group</a> to provide us valued feedback or just have a general discussion on the topic.