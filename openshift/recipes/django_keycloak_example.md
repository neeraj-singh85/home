# Integrating Keycloak and Python Django on OpenShift
In this recipe we will set up Keycloak authentication for a Python Django project and run it in MiniShift. If using minishift I suggest increasing memory by first doing: `minishift config set memory 4096`. The recipe sets up a build of vanillla Keycloak but it should be possible to use Redhat SSO if you already have that set up. 

## Starting Keycloak service on MiniShift
We create a build from an inline dockerfile:
```
$ oc new-build jboss/keycloak-postgres:latest --name=keycloak --dockerfile='FROM jboss/keycloak-postgres:latest
 
USER root

#Give correct permissions when used in an OpenShift environment.
RUN chown -R jboss:0 $JBOSS_HOME/standalone && \
    chmod -R g+rw $JBOSS_HOME/standalone

USER jboss'
```
This might take a little while to build. In the meantime, we will also be needing a Postgresql database:
```
$ oc new-app -e POSTGRESQL_ADMIN_PASSWORD=foo \
             -e POSTGRESQL_USER=keycloak \
             -e POSTGRESQL_PASSWORD=keycloak \
             -e POSTGRESQL_DATABASE=keycloak \
             centos/postgresql-95-centos7 --name postgres-95
```
(Notice that secrets should probably be used for passwords in production)

Then (once the Keycloak build is completed, to see progress check **Builds** -> **Builds** in the left menu) we deploy the Keycloak build, from UI: **Add to Project**, **Deploy Image** and find it in Image Stream tag.

(For me it was under: `myproject / keycloak : latest`)

We need to set some **Environment Variables** for `keycloak`:
```
POSTGRES_PORT_5432_TCP_ADDR=postgres-95.myproject.svc
POSTGRES_PASSWORD=keycloak
KEYCLOAK_USER=test
KEYCLOAK_PASSWORD=test
```
(Notice that secrets should probably be used for passwords in production)

Now you can click **Create**.

### Create route for Keycloak
Under **Applications** in the left menu select **Routes** and then **Create Route** in the upper right corner. 

Name your route, _e.g._ keycloakroute, make sure that the service is set to your keycloak service and click **Create**

## Getting a sample Django project and modify it to use Keycloak for authentication
OpenShift comes with a sample Django template and [repository](https://github.com/openshift/django-ex). 
We are going to need to edit the Django project so start by cloning it into your own Github account and 
check it out to your machine. I have done so and my [repository](https://github.com/jonalv/django-ex)
contains a version of _django-ex_ with all the changes described in this text so go ahead and use it if you don't feel like programming Python right now :)

### Modifications needed in our Django app
We will be using the [Django OIDC libraries](https://github.com/jhuapl-boss/boss-oidc) (see also this [excellent blog post](http://blog.jonharrington.org/static/integrate-django-with-keycloak/) if you want some more information)
We need to add the following lines `requirements.txt`:
```
git+https://github.com/jhuapl-boss/django-oidc.git
git+https://github.com/jhuapl-boss/drf-oidc-auth.git
git+https://github.com/jhuapl-boss/boss-oidc.git
```
Then in `settings.py` we need to add the applications:
```
    'bossoidc',
    'djangooidc',
```
specify our authentication backends:
```
AUTHENTICATION_BACKENDS = (
    'django.contrib.auth.backends.ModelBackend',
    'bossoidc.backend.OpenIdConnectBackend',
)
```
and finally add some configuration code:
```
auth_uri = "http://keycloakroute-myproject.192.168.64.2.nip.io/auth/realms/sample"
client_id = "webapp"
public_uri = "http://django-psql-persistent-myproject.192.168.64.2.nip.io/"

from bossoidc.settings import *
configure_oidc(auth_uri, client_id, public_uri)
```

I had some trouble with Django 1.8 so I went ahead and changed version in requirements.txt:

```
django>=1.11
```
Finally, let's make som modifications so that we have a page that requires the user to be authenticated as well as the one which does not. Make the `url.py` look like:

```
from django.conf.urls import include, url
from django.contrib import admin

from welcome.views import index, health, secure

urlpatterns = [
    # Examples:
    # url(r'^$', 'project.views.home', name='home'),
    # url(r'^blog/', include('blog.urls')),

    url(r'^$', index),
    url(r'^secure$', secure),
    url(r'^health$', health),
    url(r'^admin/', include(admin.site.urls)),
    url(r'openid/', include('djangooidc.urls')),
]
```
and add the following view to `welcome/views.py`:
```
from django.contrib.auth.decorators import login_required

[...]

@login_required
def secure(request):
    return HttpResponse('This page is only visible for the logged in user. <a href="/openid/logout">Logout</a>')
```
### Configuring Keycloak 
Log in to the **Administration console** of Keycloak with the admin username and password you gave when configuring Keycloak (test, test). We need to create a **Realm** in Keycloak, we will call it **sample**. Then we need to create a user, in the left menu choose **Users** and then in the upper right corner **Add user**. Give the new user a username and a password (in the **Credentials** tab).

We also need to create a Client, which we will give the **Client id**: webapp, and then give it the **Valid Redirect URI**: `http://django-psql-persistent-myproject.192.168.64.2.nip.io/*`

### Deploying our modified Django app
We will use the OpenShift Django + PostgreSQL template and point it to our modified Django source code.
Click **Add to Project**, **Browse Catalog**, **Python**, and then Select **Django + PostgreSQL(Persistent)**

Now point it to your **Git repository URL** (I will use https://github.com/jonalv/django-ex.git)

And click **Create**

This might take a little while but once it is all up we just need to make migrations and migrate the database for our Django app and then we are done. In OpenShift click **Applications**, **Pods** and locate the running `django-psql-persistent-X-XXXX` pod. Click it and then click on the **Terminal** tab and run:
```
$ python manage.py makemigrations bossoidc
$ python manage.py migrate bossoidc
```

you should be able to visit the Django web page, and if you try, _i.e._ http://django-psql-persistent-myproject.192.168.64.2.nip.io/secure you should first have to sign in to Keycloak before you are allowed to continue.

