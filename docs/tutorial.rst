Tutorial
========

.. note::

    The purpose of this section is not to replace the excellent `Django REST Framework documentation <https://www.django-rest-framework.org>`_ nor the `Datatables manual <https://datatables.net/manual/>`_, it is just to give you hints and gotchas for using your datatables compatible API.


Backend code
------------

So we have the following backend code, nothing very complicated if you are familiar with Django and Django REST Framework:

albums/models.py:

.. code:: python

    from django.db import models
    
    
    class Genre(models.Model):
        name = models.CharField('Name', max_length=80)
    
        class Meta:
            ordering = ['name']
    
        def __str__(self):
            return self.name
    
    
    class Artist(models.Model):
        name = models.CharField('Name', max_length=80)
    
        class Meta:
            ordering = ['name']
    
        def __str__(self):
            return self.name
    
    
    class Album(models.Model):
        name = models.CharField('Name', max_length=80)
        rank = models.PositiveIntegerField('Rank')
        year = models.PositiveIntegerField('Year')
        artist = models.ForeignKey(
            Artist,
            models.CASCADE,
            verbose_name='Artist',
            related_name='albums'
        )
        genres = models.ManyToManyField(
            Genre,
            verbose_name='Genres',
            related_name='albums'
        )
    
        class Meta:
            ordering = ['name']
    
        def __str__(self):
            return self.name

albums/serializers.py:

.. code:: python

    from rest_framework import serializers
    from .models import Album
    
    
    class AlbumSerializer(serializers.ModelSerializer):
        artist_name = serializers.ReadOnlyField(source='artist.name')
        genres = serializers.SerializerMethodField()
    
        def get_genres(self, album):
            return ', '.join([str(genre) for genre in album.genres.all()])
    
        class Meta:
            model = Album
            fields = (
                'rank', 'name', 'year', 'artist_name', 'genres',
            )

albums/views.py:

.. code:: python

    from django.shortcuts import render
    from rest_framework import viewsets
    from .models import Album
    from .serializers import AlbumSerializer
    
    
    def index(request):
        return render(request, 'albums/albums.html')
    
    
    class AlbumViewSet(viewsets.ModelViewSet):
        queryset = Album.objects.all().order_by('rank')
        serializer_class = AlbumSerializer

urls.py:

.. code:: python
    
    from django.conf.urls import url, include
    from rest_framework import routers
    from albums import views
    
    
    router = routers.DefaultRouter()
    router.register(r'albums', views.AlbumViewSet)
    
    
    urlpatterns = [
        url('^api/', include(router.urls)),
        url('', views.index, name='albums')
    ]

A minimal datatable
-------------------

In this example, we will build a simple table that will list music albums, we will display 3 columns, the album rank, name and release year.
For the sake of simplicity we will also use HTML5 data attributes (which are supported by Datatables).

.. code:: html

    <!doctype html>
    <html lang="en">
    <head>
      <meta charset="utf-8">
      <title>Rolling Stone Top 500 albums of all time</title>
      <meta name="description" content="Rolling Stone magazine's 2012 list of 500 greatest albums of all time with genres.">
      <link rel="stylesheet" href="//cdnjs.cloudflare.com/ajax/libs/twitter-bootstrap/4.0.0/css/bootstrap.css">
      <link rel="stylesheet" href="//cdn.datatables.net/1.10.16/css/dataTables.bootstrap4.min.css">
    </head>

    <body>
      <div class="container">
        <div class="row">
          <div class="col-sm-12">
            <table id="albums" class="table table-striped table-bordered" style="width:100%" data-server-side="true" data-ajax="/api/albums/?format=datatables">
              <thead>
                <tr>
                  <th data-data="rank">Rank</th>
                  <th data-data="name">Album name</th>
                  <th data-data="year">Year</th>
                </tr>
              </thead>
            </table>
          </div>
        </div>
      </div>
      <script src="//code.jquery.com/jquery-1.12.4.js"></script>
      <script src="//cdn.datatables.net/1.10.16/js/jquery.dataTables.min.js"></script>
      <script src="//cdn.datatables.net/1.10.16/js/dataTables.bootstrap4.min.js"></script>
      <script>
          $(document).ready(function() {
              $('#albums').DataTable();
          });
      </script>
    </body>
    </html>

And that's it ! At this point, you should have a fully functional Datatable with search, ordering and pagination !

What we just did:

- included all the necessary CSS and JS files
- set the table ``data-server-side`` attribute to ``true``, to tell Datatables to use the server-side processing mode
- set the table ``data-ajax`` to our API URL with ``?format=datatables`` as query parameter
- set a ``data-data`` attribute for the two columns to tell Datatables what properties must be extracted from the response
- and finally initialized the Datatable via a javascript one-liner.


.. hint::

    Perhaps you noticed that we didn't use all fields from our serializer in the above example, that's not a problem, django-rest-framework-datatables will automatically filter the fields that are not necessary when processing the request from Datatables.

.. important::

    To sum up, **the most important things** to remember here are:
    
    - don't forget to add ``?format=datatables`` to your API URL
    - you must add a **data-data attribute** or specify the column data property via JS for each columns, the name must **match one of the fields of your DRF serializers**.


A more complex and detailed example
-----------------------------------

In this example we want to display more informations about the album:

- the album artist name (``Album.artist`` is a foreignkey to ``Artist`` model)
- the genres (``Album.genres`` is a many to many relation with ``Genre`` model)

The HTML/JS code will look like this:

.. code:: html

    <!doctype html>
    <html lang="en">
    <head>
      <meta charset="utf-8">
      <title>Rolling Stone Top 500 albums of all time</title>
      <meta name="description" content="Rolling Stone magazine's 2012 list of 500 greatest albums of all time with genres.">
      <link rel="stylesheet" href="//cdnjs.cloudflare.com/ajax/libs/twitter-bootstrap/4.0.0/css/bootstrap.css">
      <link rel="stylesheet" href="//cdn.datatables.net/1.10.16/css/dataTables.bootstrap4.min.css">
    </head>

    <body>
      <div class="container">
        <div class="row">
          <div class="col-sm-12">
            <table id="albums" class="table table-striped table-bordered" style="width:100%" data-server-side="true" data-ajax="/api/albums/?format=datatables">
              <thead>
                <tr>
                  <th data-data="rank">Rank</th>
                  <th data-data="artist_name" data-name="artist__name">Artist</th>
                  <th data-data="name">Album name</th>
                  <th data-data="year">Year</th>
                  <th data-data="genres" data-name="genres__name">Year</th>
                </tr>
              </thead>
            </table>
          </div>
        </div>
      </div>
      <script src="//code.jquery.com/jquery-1.12.4.js"></script>
      <script src="//cdn.datatables.net/1.10.16/js/jquery.dataTables.min.js"></script>
      <script src="//cdn.datatables.net/1.10.16/js/dataTables.bootstrap4.min.js"></script>
      <script>
          $(document).ready(function() {
              $('#albums').DataTable();
          });
      </script>
    </body>
    </html>

Notice that artist and genres columns have an extra data attribute: ``data-name``, this attribute is necessary to tell to the django-rest-framework-datatables builtin filter backend what field part to use to filter and reorder the queryset. The builtin filter will add ``__icontains`` to the string to perform the filtering/ordering.

We could also have written that in a more conventional form (without data attributes), for example:

.. code:: html

    <!doctype html>
    <html lang="en">
    <head>
      <meta charset="utf-8">
      <title>Rolling Stone Top 500 albums of all time</title>
      <meta name="description" content="Rolling Stone magazine's 2012 list of 500 greatest albums of all time with genres.">
      <link rel="stylesheet" href="//cdnjs.cloudflare.com/ajax/libs/twitter-bootstrap/4.0.0/css/bootstrap.css">
      <link rel="stylesheet" href="//cdn.datatables.net/1.10.16/css/dataTables.bootstrap4.min.css">
    </head>

    <body>
      <div class="container">
        <div class="row">
          <div class="col-sm-12">
            <table id="albums" class="table table-striped table-bordered" style="width:100%">
              <thead>
                <tr>
                  <th>Rank</th>
                  <th>Artist</th>
                  <th>Album name</th>
                  <th>Year</th>
                  <th>Year</th>
                </tr>
              </thead>
            </table>
          </div>
        </div>
      </div>
      <script src="//code.jquery.com/jquery-1.12.4.js"></script>
      <script src="//cdn.datatables.net/1.10.16/js/jquery.dataTables.min.js"></script>
      <script src="//cdn.datatables.net/1.10.16/js/dataTables.bootstrap4.min.js"></script>
      <script>
          $(document).ready(function() {
              $('#albums').DataTable({
                  'serverSide': true,
                  'ajax': '/api/albums/?format=datatables',
                  'columns': [
                      {'data': 'rank'},
                      {'data': 'artist_name', 'name': 'artist__name'},
                      {'data': 'name'},
                      {'data': 'year'},
                      {'data': 'genres', 'name': 'genres__name'},
                  ]
          
              });
          });
      </script>
    </body>
    </html>

.. hint::

    Here our "fields parts" are ``artist__name`` and ``genres__name``, but we could also use the dot syntax ``artist.name`` and ``genres.name``, both are recognised !

.. hint::

    If you need more complex filtering and ordering, you can always implement your own filter backend by inheriting from ``rest_framework_datatables.DatatablesFilterBackend``.

.. important::

    To sum up, for **foreign keys and relations** you need to specify a **name for the column** otherwise filtering and ordering will not work.


You can see this code live by running the :doc:`example app <example-app>`.
