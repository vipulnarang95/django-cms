:sequential_nav: both

.. _apphooks_introduction:

========
Apphooks
========

Right now, our Django Polls application is statically hooked into the project's ``urls.py``. This
is all right, but we can do more, by attaching applications to django CMS pages.


Create an apphook
=================

We do this with an **apphook**, created using a :class:`CMSApp <cms.app_base.CMSApp>` sub-class,
which tells the CMS how to include that application.


Create the apphook class
------------------------

Apphooks live in a file called ``cms_apps.py``, so create one in your Polls/CMS Integration
application. This is a very basic example of an apphook for a Django-CMS application:

.. code-block:: python
    :caption:polls/cms_apps.py

    from cms.app_base import CMSApp
    from cms.apphook_pool import apphook_pool


    @apphook_pool.register  # register the application
    class PollsApphook(CMSApp):
        name = "Polls Application"
        app_name = "polls"

        def get_urls(self, page=None, language=None, **kwargs):
            return ["polls.urls"]

Alternatively, you can also specify the URL patterns directly, for instance:

.. code-block:: python

    from django.urls import path
    from polls import views

    ...
    class PollsApphook(CMSApp):
        ...

        def get_urls(self, page=None, language=None, **kwargs):
            return [
                path('<int:pk>/results/', views.ResultsView.as_view(), name='results'),
                path('<int:pk>/vote/', views.vote, name='vote'),
                path('<int:pk>/', views.DetailView.as_view(), name='detail'),
                path('', views.IndexView.as_view(), name='index'),
            ]

In this ``PollsApphook`` class, we have done several key things:

* ``name`` is a human-readable name, and will be displayed to the user in the *Advanced settings*
  of the CMS pages attaching to this apphook.
* ``app_name`` this optional attribute gives the system a unique way to refer to the apphook. It is
  used the create a reverse mapping for the URL's namespace.
* ``get_urls()`` method is what actually hooks the application in, returning a list of URL
  configurations that will be made active wherever the apphook is used - in this case, it will
  either use the ``urls.py`` from ``polls``, or declare its own list of URL patterns.


Remove the old ``polls`` entry from the project's ``urls.py``
-------------------------------------------------------------

You must now remove the entry for the Polls application::

    path('polls/', include('polls.urls', namespace='polls'))

from your project's ``urls.py``.

Not only is it not required there, because we reach the polls via the apphook
instead, but if you leave it there, it will conflict with the apphook's URL handling. You'll
receive a warning in the logs::

    URL namespace 'polls' isn't unique. You may not be able to reverse all URLs in this namespace.


Restart the runserver
=====================

**Restart the runserver**. This is necessary because we have created a new file containing Python
code that won't be loaded until the server restarts. You have to restart the server each time you
want to apply a modification made to this file or any views attached to thereof. Restarting the
server after a change can be prevented, if the :ref:`ApphookReloadMiddleware` has been added to the
``MIDDLEWARE`` in your ``settings.py``.



.. _apply_apphook:

Apply the apphook to a page
===========================

Now we need to create a new page, and attach the Polls application to it through this apphook.

Create and save a new page. In its *Advanced settings* (from the toolbar, select
*Page > Advanced settings...*) choose "Polls Application" from the *Application* pop-up menu, and
save once more.

.. image:: /introduction/images/select-application.png
   :alt: select the 'Polls' application
   :width: 400
   :align: center

Your apphook won't work until the page has been published, therefore publish it.

Refresh the page, and you'll find that the Polls application is now available
directly from the new django-CMS page.

..  important::

    Don't add child pages to a page with an apphook.

    The apphook "swallows" all URLs below that of the page, handing them over to the attached
    application. If you have any child pages of the apphooked page, django CMS will not be
    able to serve them reliably.

This apphook's primary view is handled by the class `polls.views.IndexView`. This view then
is in full control for rendering that page, just like a normal Django view would do.


Using placeholders in apphooked views
-------------------------------------

It also is possible to use CMS placeholders in apphooked views, combining the functionality of
third party applications with all the features Django-CMS plugins can offer. To fully utilize
these possibilities, one must change the template used by the view, and add one or more
placeholders to it. For example:

.. code-block:: django

    {% load cms_tags %}

    <html>
        <head>
        ...
        </head>
        <body>
            <!-- some HTML required by the apphook view -->
            {% placeholder "Main Content" %}
            <!-- more HTML required by the apphook view -->
        </body>
    </html>

This will then allow the CMS user to edit the placeholder's content while using a Django
view responsible to fill the application context.


.. _multi_apphook:

Attaching an apphooked application multiple times
=================================================

Sometimes one might want to use an apphooked application on more than one CMS page. Then however,
it is desirable being able to configure that apphook, so that different instances can offer
distinguished services.

Namespacing your apphooks makes it possible to manage additional database-stored apphook
configuration, on an instance-by-instance basis. In order to do so, we have to declare a
Django model:

.. code-block:: python
    :caption:polls/models.py

    class PollConfig(models.Model):
        cmsapp = None

        namespace = models.CharField(
            "Instance namespace",
            default=None,
            max_length=50,
            unique=True,
        )

        # application specific fields used to configure our apphook

        def __str__(self):
            return self.namespace

The attribute ``cmsapp`` shall point on an apphook class, for instance ``PollsApphook``. If it is
set to ``None``, then Django-CMS will take care of doing so.

The field ``namespace`` is mandatory and shall be decalred to be unique. It is used as identifier,
whenever the ower of a CMS page attaches it to a given apphook. Next, we must tell our apphook to
use that configuration model.

To capture the configuration that different instances of an apphook can take, a Django model needs
to be created – each apphook instance will be an instance of that model, and administered through
the Django admin in the usual way:

.. code-block:: python
    :caption:polls/admin.py

    from django.contrib import admin
    from polls.model import PollConfig

    @admin.register(PollConfig)
    class PollConfigAdmin(admin.ModelAdmin):
        ...


Adopt the apphook to be configurable
------------------------------------

.. code-block:: python

    ...
    from polls import models

    ...
    class PollsApphook(CMSApp):
        name = "Polls Application"
        app_name = "polls"
        app_config = models.PollConfig

        def get_urls():
            ...

If you want to attach an application multiple times to different pages, then the class defining the apphook *must*
have an ``app_name`` attribute. It does three key things:

* It provides the *fallback namespace* for views and templates that reverse URLs.
* It exposes the *Application instance name* field in the page admin when applying an apphook.
* It sets the *default apphook instance name* (which you'll see in the *Application instance name* field).

We'll explain these with an example. Let's suppose that your application's views or templates use
``reverse('polls:index')`` or ``{% url 'polls:index' %}``.

In this case the namespace of any apphooks must match ``polls``. If they don't, pages using them will
throw up a ``NoReverseMatch`` error.

You can set the namespace for the instance of the apphook in the *Application instance name* field.
However, you'll need to set that to something *different* if an instance with that value already
exists. In this case, as long as ``app_name = "polls"`` it doesn't matter; even if the system
doesn't find a match with the name of the instance it will fall back to the one hard-wired into the
class.

In other words, setting ``app_name`` correctly guarantees that URL-reversing will work, because it
sets the fallback namespace appropriately.


Set a namespace at instance-level
---------------------------------

This apphook uses a configuration model by pointing attribute ``app_config`` to model
``PollConfig``. This alters the form in the *Advanced Settings* of the CMS page's admin by adding
another select field below the *Application* field named *Application configurations*. This field
then allows the user to pick one of the prepared poll configurations. If selected, it will override
the ``app_name`` *if* a match is found.

This arrangement allows you to use multiple application instances and namespaces if that
flexibility is required, while guaranteeing a simple way to make it work when it's not.

Django's :ref:`django:topics-http-reversing-url-namespaces` documentation provides more information
on how this works, but the simplified version is:

1. First, it'll try to find a match for the *Application instance name*.
2. If it fails, it will try to find a match for the ``app_name``.


Accessing the configuration instance from the apphook
-----------------------------------------------------

In ``get_urls()`` of our apphook class, we route onto different Django views. These views then are
responsible in setting up a context and rendering to HTML. These views shall behave differently,
depending on where they have been inserted into the CMS page tree. We therefore have to fetch the
proper configuration object, depending on the current CMS page. This can be done in every method
of the view class, where the ``request``-object is available:

.. code-block:: python
    :caption:polls/views.py

    from django.urls import Resolver404, resolve
    from django.utils.translation import get_language_from_request, override
    from django.views.generic import ListView
    from cms.apphook_pool import apphook_pool

    def get_app_instance(request):
        """
        Returns a tuple containing the current namespace and the AppHookConfig instance
        :param request: request object
        :return: namespace, config
        """
        app, config, namespace = None, None, ''
        if getattr(request, 'current_page', None) and request.current_page.application_urls:
            app = apphook_pool.get_apphook(request.current_page.application_urls)
            if app and app.app_config:
                try:
                    with override(get_language_from_request(request, check_path=True)):
                        namespace = resolve(request.path_info).namespace
                        config = app.get_config(namespace)
                except Resolver404:
                    pass
        return namespace, config

    class IndexView(ListView):
        ...
        def get_context_data(self, **kwargs):
            context = super().get_context_data(**kwargs):
            namespace, config = get_app_instance(self.request)
            # adopt the context using the config instance:
            qs = self.get_queryset()
            context['entries'] = qs.filter(...)  # filter by some config criteria
            return context

Here, our list view class ``IndexView`` uses a ``PollConfig`` instance to filter the given queryset
by criteria defined in that model. This can be anything and is left out to the implementors of that
class.
