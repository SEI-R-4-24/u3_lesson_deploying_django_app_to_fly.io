# Deploying a Django app to [fly.io](https://fly.io)
![](https://encrypted-tbn0.gstatic.com/images?q=tbn:ANd9GcTorjndFm_1AcoKCTBGzcJdHSPqWfmkvY5gTi98TV3b4L3mNMYzmbPc54sq-6k4ZzFbFQ&usqp=CAU)
## Overview

This post has two main sections:

1. <b>Setting up</b>: steps on how to make your local Django app ready for going to production on any hosting platform.
2. <b>Deploying to [fly.io](https://fly.io)</b>: deploying your production-ready Django app to [fly.io](https://fly.io).

## Setting Up

You will need to create a virtual environment. Since you have not had a dedicated environment for this app, you will also need to install all the packages you used in creating your app into this environment. 

To create a dedicated Python development environment run the following command.

```shell
# Linux/Unix/macOS
python3 -m venv .venv
source .venv/bin/activate

#once activated you should see this prompt:
(.venv) $

# Windows
$ python -m venv .venv
$ .venv\Scripts\activate
#once activated you should see this prompt:
(.venv) $
```

Once this environment is activated you will need to install all packages you used to create your app (best practice is to always create a new active environment for each new project at the beginning of your project):

You will need to install the following packages for your Django app:

```shell
pip install django
pip install boto3 # only if you are using S3 buckets for uploads
```

The great thing is that Fly.io provides a single-node/high availability PostgreSQL cluster out of the box for us to use. It's also easy to set it up when configuring your deployment. We'll go over how in the next steps.

You can also use [neon.tech](http://neon.tech) to host your PostgreSQL database for free. 

The config updates required to change our database are explained in the next steps.

## Environment Variables

First of all, we want to store the configuration separate from our code and load them at runtime. This allows us to keep one settings.py file and still have multiple environments (i.e. local/staging/production).

One popular options is the usage of the environs package. We'll use django-environ to configure our Django application.

Make sure your Python virtual environment is activated and let's install the django-environ:

```shell
python -m pip install django-environ==0.9.0
```

In our settings.py file, we can define the casting and default values for specific variables, for example, setting DEBUG to False by default.

```python
# settings.py
from pathlib import Path
import environ  # <-- Updated!

env = environ.Env(  # <-- Updated!
    # set casting, default value
    DEBUG=(bool, False),
)
```

django-environ can take environment variables from the .env file. Go ahead and create this file in your root directory and also don't forget to add it to your .gitignore in case you are using Git for version control (you should!). We don't want those variables to be public, this will be used for the local environment and be set separately in our production environment.

Make sure you are taking the environment variables from your .env file:

```python
# settings.py
# Build paths inside the project like this: BASE_DIR / 'subdir'.
BASE_DIR = Path(__file__).resolve().parent.parent

# Take environment variables from .env file
environ.Env.read_env()  # <-- Updated!
```
We can now set the specific environment variables in the .env file:

```.env
# .env
SECRET_KEY=justhaveyourcatdanceonyourkeyboard
DEBUG=True
```
Check that there are no quotations around strings neither spaces around the =.

Coming back to our settings.py, we can then read SECRET_KEY and DEBUG from our environment variables:

```python
# settings.py
# SECURITY WARNING: keep the secret key used in production secret!
SECRET_KEY = env('SECRET_KEY')  # <-- Updated!

# SECURITY WARNING: don't run with debug turned on in production!
DEBUG = env('DEBUG')  # <-- Updated!
```

The last environment variable to be set is the DATABASE_URL. env.db() will read from the DATABASE_URL variable.

```python
# settings.py
DATABASES = {
    # read os.environ['DATABASE_URL']
    'default': env.db()  # <-- Updated!
}
```

Here we can define our local database, adding it to the .env file:

```.env
# .env
SECRET_KEY=justhaveyourcatdanceonyourkeyboard
DEBUG=True
DATABASE_URL=postgres://postgres:postgres@localhost:5432/catcollector  # <-- Updated!
```

## Psycopg

To interact with our Postgres database, we'll use the most popular PostgreSQL database adapter for Python, the psycopg package. With your virtual environment activated, go ahead and installed it:

```shell
python -m pip install psycopg2-binary
```

⚠️ For production, [it's advised to use the source distribution](https://psycopg.org/docs/install.html?highlight=binary#psycopg-vs-psycopg-binary) instead of the binary package (psycopg2-binary). ```python -m pip install psycopg2==2.9.5``` however fly.o causes problems for some users and runs fine with the binary package.

## Gunicorn

Gunicorn (Green Unicorn) is a Python WSGI HTTP Server recommended for Unix and one of the easiest to start with. It can be installed using pip:

```shell
python -m pip install gunicorn==20.1.0
```

## Static Files

Handling static files in production is a bit more complex than in development. One of the easiest and most popular ways to serve our static files in production is using the WhiteNoise package, which serves them directly from our WSGI Server (Gunicorn). Install it with:


```shell
python -m pip install whitenoise==6.3.0
```

A few changes to our settings.py are necessary.

* Add the WhiteNoise to the MIDDLEWARE list right after the SecurityMiddleware.
* Set the STATIC_ROOT to the directory where the collectstatic management command will collect the static files for deployment.
* (Optional) Set STATICFILES_STORAGE to CompressedManifestStaticFilesStorage to have compression and caching support.

```python
# settings.py
MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'whitenoise.middleware.WhiteNoiseMiddleware',  # <-- Updated!
    ...
]

...
STATIC_ROOT = BASE_DIR / 'staticfiles'  # <-- Updated!

STATICFILES_STORAGE = 'whitenoise.storage.CompressedManifestStaticFilesStorage'  # <-- Updated!
```

It's recommended to use WhiteNoise also in development to keep consistent behavior between development and production environments. The easiest way to do that is to add whitenoise.runserver_nostatic to our INSTALLED_APPS right before the built-in staticfiles app:

```python
# settings.py
INSTALLED_APPS = [
    ...
    'whitenoise.runserver_nostatic',  # <-- Updated!
    'django.contrib.staticfiles',
    ...
]
```

## ALLOWED_HOSTS and CSRF_TRUSTED_ORIGINS

As a security measure, we should set in ALLOWED_HOSTS, a list of host/domain names that our Django website can serve. For development we might include localhost and 127.0.0.1 and for our production we can start with .fly.dev (or the provider's subdomain you chose) and update for the dedicated URL once your app is deployed to the hosting platform.

CSRF_TRUSTED_ORIGINS should also be defined with a list of origins to perform unsafe requests (e.g. POST). We can set the subdomain https://*.fly.dev (or the provider's subdomain you chose) until our deployment is done and we have the proper domain for our website.

```python
# settings.py
ALLOWED_HOSTS = ['localhost', '127.0.0.1', '.fly.dev']  # <-- Updated!

CSRF_TRUSTED_ORIGINS = ['https://*.fly.dev']  # <-- Updated!
```
## Installed packages

Make sure you have all necessary installed packages are tracked and listed in your requirements.txt by running:

```shell
pip freeze > requirements.txt
```

This command generates the requirements.txt file if it doesn't exist. (It is the Django equivalent to the package.json in NodeJS) Yours will vary depending on the installed packages.

```python
# requirements.txt
asgiref==3.7.2
boto3==1.26.158
botocore==1.29.158
Django==4.2.2
django-environ==0.9.0
gunicorn==20.1.0
jmespath==1.0.1
psycopg2-binary==2.9.6
python-dateutil==2.8.2
s3transfer==0.6.1
six==1.16.0
sqlparse==0.4.4
typing_extensions==4.6.3
urllib3==1.26.16
whitenoise==6.3.0
```

With our Django application prepped and ready for production hosting, we'll take the next step and deploy our app to Fly.io!

# Deploying to Fly.io 🚀

[flyctl](https://fly.io/docs/hands-on/install-flyctl/) is the command-line utility provided by Fly.io.

If not installed yet, follow these [instructions](https://fly.io/docs/hands-on/install-flyctl/), [sign up](https://fly.io/docs/hands-on/sign-up/) and [log in](https://fly.io/docs/hands-on/sign-in/) to Fly.io.

## Launching our App

Fly.io allows us to deploy our Django app as long as it's packaged in a Docker image. However, we don't need to define our <span style="color:lightblue">Dockerfile</span> manually. Fly.io detects our Django app and automatically generates all the necessary files for our deployment. Those are:

* <span style="color:lightblue">Dockerfile</span> contains commands to build our image.
* <span style="color:lightblue">.dockerignore</span> list of files or directories Docker will ignore during the build process.
* <span style="color:lightblue">fly.toml</span> configuration for deployment on Fly.io.
All of those files are templates for a simple Django apps and can be modified according to your needs.

Before deploying our app, first we need to configure and launch our app to Fly.io by using the flyctl command fly launch. During the process, we will:

* <b>Choose an app name:</b> this will be your dedicated fly.dev subdomain.
* <b>Select the organization:</b> you can create a new organization or deploy to your personal account (connect to your Fly account, visible only to you).
* <b>Choose the region for deployment:</b> Fly.io initially suggests the closest to you, you can choose another region if you prefer.
* <b>Set up a Postgres database cluster:</b> flyctl offers a single node "Development" config that is designed so we can turn it into a high-availability cluster by adding a second instance in the same region. [Fly Postgres is a regular app you deploy on Fly.io, not a managed database](https://fly.io/docs/postgres/getting-started/what-you-should-know/). 
    * Select Y for the first question: Would you like to set up a Postgresql database now? Yes 
    *  Select Y for the question: Scale single node pg to zero after one hour? Yes
    * Select N for the question: Would you like to set up an Upstash Redis database now? No
* If you use Neon.tech to host your database you can select N to all the database question, [you will have to add your connection variables by updating your secrets. ](https://fly.io/docs/reference/secrets/#setting-secrets)

This is what it looks like when we run fly launch:

```shell
fly launch

Creating app in ../flyio/icollectcats
Scanning source code
Detected a Django app
? Choose an app name (leave blank to generate one): icollectcats
? Select Organization: Jan Horak (personal)
? Choose a region for deployment:Ashburn, Virginia (US) (iad)
App will use 'iad' region as primary
Created app icollectcats in organization personal
Admin URL: https://fly.io/apps/icollectcats
Hostname: icollectcats.fly.dev
Set secrets on icollectcats: SECRET_KEY  <-- # SECRET_KEY is set here!
? Would you like to set up a Postgresql database now? Yes
? Select configuration: Development - Single node, 1x shared CPU, 256MB RAM, 1GB disk
? Scale single node pg to zero after one hour? Yes
Creating postgres cluster in organization personal
Creating app...
Setting secrets on app icollectcats-db...
Provisioning 1 of 1 machines with image flyio/postgres:14.6
Waiting for machine to start...
Machine 32874445c04218 is created
==> Monitoring health checks
  Waiting for 32874445c04218 to become healthy (started, 3/3)

Postgres cluster icollectcats-db created
  Username:    postgres
  Password:    <your-internal-postgres-password>
  Hostname:    icollectcats-db.internal
  Proxy port:  5432
  Postgres port:  5433
  Connection string: postgres://postgres:<your-internal-postgres-password>@icollectcats-db.internal:5432

Save your credentials in a secure place -- you will not be able to see them again!

Connect to postgres
Any app within the jannyb organization can connect to this Postgres using the above connection string

Now that you have set up Postgres, here is what you need to understand: https://fly.io/docs/postgres/getting-started/what-you-should-know/
Checking for existing attachments
Registering attachment
Creating database
Creating user

Postgres cluster icollectcats-db is now attached to icollectcats
The following secret was added to icollectcats:  <-- # DATABASE_URL is set here!
  DATABASE_URL=postgres://icollectcats:<your-postgres-password>@top2.nearest.of.icollectcats-db.internal:5432/icollectcats?sslmode=disable
Postgres cluster icollectcats-db is now attached to icollectcats
? Would you like to set up an Upstash Redis database now? No
Creating database migrations
Wrote config file fly.toml

Your Django app is ready to deploy!

For detailed documentation, see https://fly.dev/docs/django/
```

Scroll up and copy and save the database information to add to your local .env if you want to connect to the same database on your local machine as you continue to work on your app.

During the process, the SECRET_KEY and DATABASE_URL will be automatically set to be used on your production deployment. Those are the only ones we need at the moment but if you have any other secrets, [check here how to set them](https://fly.io/docs/reference/secrets/#setting-secrets). You can also list all your application secret names:

```shell
fly secrets list

NAME            DIGEST                  CREATED AT
DATABASE_URL    cc999c17fa021988        2023-02-07T19:48:55Z
SECRET_KEY      e0a6dbbd078004f7        2023-02-07T19:47:33Z
```

fly launch sets up a running app, creating the necessary files: Dockerfile, .dockerignore and fly.toml.

Don't forget to replace demo.wsgi in your Dockerfile with your Django project's name:

```python
# Dockefile
...
# replace demo.wsgi with <project_name>.wsgi
CMD ["gunicorn", "--bind", ":8000", "--workers", "2", "website.wsgi"]  # <-- Updated!
```

For security reasons, we'll add .env to our .dockerignore file - so Docker doesn't include our secrets during the build process.

```.env
# .dockerignore
fly.toml
.git/
*.sqlite3
.env  # <-- Updated!
```
This means the environment variables (i.e. SECRET_KEY) stored in the .env file won't be available at build time, neither the secrets automatically set by Fly.io on your application during the fly launch process.

Set a default SECRET_KEY using the get_random_secret_key function provided by Django that will be used at build time. At runtime, we'll use the SECRET_KEY set by Fly.io, i.e. the default value only applies to the build process.

```python
# settings.py
from django.core.management.utils import get_random_secret_key
...
# SECURITY WARNING: keep the secret key used in production secret!
SECRET_KEY = env.str('SECRET_KEY', default=get_random_secret_key())  # <-- Updated!
```

This keeps our environment variables safe and makes sure they will be set for the different environments.

Now that we have set our app name, we can update our settings.py with the dedicated subdomain we chose (or that was generated for us):

```python
# settings.py
ALLOWED_HOSTS = ['localhost', '127.0.0.1', 'icollectcats.fly.dev']  # <-- Updated!

CSRF_TRUSTED_ORIGINS = ['https://icollectcats.fly.dev']  # <-- Add this line!
```
Finally, in the [[statics]] section on fly.toml file, we define the guest_path, which is the path inside our container where the files will be served directly to the users, bypassing our web server. In the settings.py we defined the STATIC_ROOT:

```python
# settings.py
STATIC_ROOT = BASE_DIR / 'staticfiles'  # <-- Updated!
```
The Dockerfile generated by fly launch defines our WORKDIR as /code. That's where our static files will be collected: /code/staticfiles. Let's go ahead and update our fly.toml to serve those files directly:

```shell
# fly.toml
[[statics]]
  guest_path = "/code/staticfiles"  # <-- Updated!
  url_prefix = "/static"
```


# Deploying our App

Great! All ready and it's finally time to deploy our app:

```shell
fly deploy
...
 1 desired, 1 placed, 1 healthy, 0 unhealthy [health checks: 1 total, 1 passing]
--> v0 deployed successfully
```

Our app is now up and running! ⚙️ Try:

```shell
fly open
```

YAY! 🎉 We just deployed our Django app to production! How great is that?



# Credits:
* [This blog post](https://fly.io/django-beats/deploying-django-to-production/)
* https://fly.io/docs/django/getting-started/
* https://fly.io/docs/reference/secrets/#setting-secrets 