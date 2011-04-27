An Hour with Django and Python
==============================

Brian Gershon

April 27, 2011

---

Who uses Python?
----------------
Python has a broad development community

* Web developers
* System administrators
* Mad scientists (NOAA, NASA, Google)

See also [http://wiki.python.org/moin/OrganizationsUsingPython](http://wiki.python.org/moin/OrganizationsUsingPython)

---

Why Python?
-----------
* Clean and natural syntax
* Cross platform - offers options for devs and deployment
* All the bells and whistles: OO, Exceptions, Generators, Decorators, ...
* Mature libraries
* Enjoyable to write (and read)

---

What does Python look like?
---------------------------
	!python
	import random
	
	def getSeattleWeather():
		""" What is the current weather like in Seattle? """
		weather = 'sunny'
		if not niceWeather():
			common_weather = ['cloudy', 'rainy', 'pissy', 'blistery']
			weather = random.choice(common_weather)
		return weather

* modules
* whitespace / blocks are indented (4 spaces)
* triple-quoted strings

---

Another snippet of Python
-------------------------
List comprehensions provide a more concise way to create lists in situations where map() and filter() and/or nested loops would currently be used.

	!python
	# return a list of event names if the event happens today
	eventsToday = [event.name for event in events if event.today]
	
instead of

	!python
	events_today = []
	for event in events:
		if event.today:
			events_today.append(event)

---

Who uses Django?
----------------
* Disqus.com website commenting platform [Scaling talk at DjangoCon2010](http://python.mirocommunity.org/video/1886/djangocon-2010-scaling-the-wor)
* PBS [article](http://www.chicagopython.com/news/articles/2010/nov/19/pbsorg-goes-django)
* [National Geographic](http://www.nationalgeographic.com)
* Discover Channel
* MSNBC Everyblock
* NW Avalanche Center [http://nwac.us](http://nwac.us). Air Charity Network. (sites I worked on)
* GeoDjango-based sites (complete open source GIS platform)

---

Why Django?
-----------
* A nice MVC framework in Python
* Open Source community
* Culture of test driven development
* Extensibility: [http://djangopackages.com](http://djangopackages.com)
* Well documented / Get up to speed quickly
* Django grew from "building intensive web applications on journalism deadlines"
* Note: MVC with slightly different names: Models -> Views (controllers) -> Templates

---

Django features we'll cover
----------------------------
* Object Relational Manager (ORM): Models, QuerySets, Managers
* Django Admin: "A dynamic admin interface"
* Templating
* Routing
* Getting Up and Running
* Testing

---

Django: What is the ORM?
------------------------
* A way to map objects to a relational database.
* A Python database-access API.

---

Django ORM: Models
------------------
	!python
	import datetime
	from django.db import models

	class Passenger(models.Model):
	    """ Holds information specific to patients. """
	    first_name = models.CharField("First name", max_length=50) # required
	    flight = models.ForeignKey(Flight)

	    date_of_birth = models.DateField(
	        "Date of Birth", 
	        help_text="Please enter dates in the following format: mm/dd/yyyy")

	    @property
	    def full_name(self):
	        return "%s %s" % (self.first_name, self.last_name)

	    @property
	    def age(self):
	        delta = datetime.datetime.now().date() - self.date_of_birth
	        return delta.days/365

	    def __unicode__(self):
	        return u'%s' % self.full_name

---

What about floating helper functions?
-------------------------------------
* Helpers can take on a life of their own
* Suggest moving helper functionality into Models (as properties) and Managers (as common querysets). Need to weigh what code to put in model and which into views (controllers). *Discussing Managers shortly.*
* Consider a new app

---

Django Admin
------------
"A dynamic admin interface: it's not just scaffolding -- it's the whole house"

Demo

... also walk through a typical Django application and its files.

---

Django ORM: Querying
--------------------

$ ./manage.py shell
	
	!python
	>>> from demo.flight_system.models import Flight, Passenger
	>>> Passenger.objects.all()
	>>> Passenger.objects.filter(first_name__startswith='J')
	>>> p = Passenger(first_name="Joe", last_name="Bar", flight=Flight(id=1))
	>>> p.save()
	
Can chain queryset methods, query runs when you first use the result set.

	!python
	Passenger.objects.filter(first_name__startswith='J')\
		.exclude(date_of_birth__gt=datetime.date(2005, 1, 3))\
		.order_by('-date_of_birth', 'last_name')

---

Django ORM: Queryset Managers
-----------------------------
The default object manager is a model's 'objects' attribute e.g. Passenger.objects.all()

Define your own Manager to house common "helpers" that involve table queries. Also mix and match with built-in queryset methods (.filter, .exclude, .order_by, etc.)

	!python
	class FlightLegPilotObjectManager(models.Manager):
	    def allAwaitingPFRs(self):
	        """ Returns all FlightLegPilot records of type 'pilot' that have status of FLIGHTLEG_STATUS_PFR_NEEDED_VALUE.
	        """
	        return self.get_query_set()\
	            .filter(status=getStateForWorkflow("FlightLegPilot", FLIGHTLEG_PILOT_STATUS_APPROVED_VALUE))\
	            .filter(flightleg__status=getStateForWorkflow("FlightLeg", FLIGHTLEG_STATUS_PFR_NEEDED_VALUE))\
	            .filter(pilot_type='pilot')
	
	class FlightLegPilot(models.Model):
	    pilot = models.ForeignKey(Pilot)
	    flightleg = models.ForeignKey(FlightLeg)
	    status = models.ForeignKey(State)
	    pilot_type = models.CharField(
	        "Is this pilot a pilot or co-pilot for this flight leg?",
	        max_length=10, choices = PILOT_TYPE_CHOICES)
	    objects = FlightLegPilotObjectManager()

---

Django Routing
--------------
	!python
	from django.conf.urls.defaults import patterns, url
	from app_trips.views.flight_details import view_flight_details
	from app_trips.views.trip_list import view_trip_list, \
		view_trip_list_selected_trip

	urlpatterns = patterns('',
	# view flight
		url(r'^(?P<flight_id>\d+)/view/$', view_flight_details,
	    	{},
			name='flight_details'),

	# open trips queue
	    url(r'^view/trips/$', view_trip_list,
	        {},
	        name='trip_list'),
        
	    url(r'^view/trips/(?P<trip_id>\d+)/$', view_trip_list_selected_trip,
	        {},
	        name='trip_list_selected_trip'),
	)

---

Django Views (controllers)
--------------------------

	!python
	from django.contrib.auth.decorators import login_required
	from django.shortcuts import get_object_or_404, render_to_response
	from django.template import RequestContext
	from settings import GOOGLE_MAPS_API_KEY
	from app_trips.models import FlightLegPilot, FlightLeg

	@login_required
	def view_flight_details(request, flight_id):
	    """ Details for a flight. Includes manifest (passenger, companions, etc.)"""
	    flight = get_object_or_404(FlightLeg, id=flight_id)

	    # Get flight manifest
	    manifest = flight.trip.trip_manifest
	    is_canceled = flight.status

	    return render_to_response('flight_details.html',
	            {
	                'flight' : flight,
	                'manifest' : manifest,
	                'is_canceled' : is_canceled,
	                'GOOGLE_MAPS_API_KEY': GOOGLE_MAPS_API_KEY,
	            }, 
	            context_instance=RequestContext(request))

---

Django Templates
----------------
Template inheritance, tags (`{% url %}`), filters (`|date`), variables (`{{}}`)

	!html
	{% extends "base.html" %}
	{% block content %}
	<tr>
    	<th>Date </th>
    	<td>{{ flight.departure_date|date:"D, d M Y" }}</td>
  	</tr>
  	<tr>
    	<th>Departure</th>
    	<td>
			{{ flight.departure_airport }}
			<a href="{{ flight.departure_airport.airnav_url}}" target="_blank">
				<img src="/media/images/icon_airnav.png" title="Airnav Information" />
			</a>
    	</td>
  	</tr>
    {% if user_profile.is_coordinator or user_profile.is_administrator %}
        <a href="{% url flightsystem_edit_patient manifest.passenger.id %}">
            {{ manifest.passenger.name }}
        </a>
    {% endif %}
	{% endblock %}

---

Getting Up and Running
----------------------
* Dependencies: Python, Django, a relational database (SQLite for dev, MySQL, PostgreSQL, MS SQL)
* Django Installation Docs [http://docs.djangoproject.com/en/dev/topics/install](http://docs.djangoproject.com/en/dev/topics/install)
* Recommend virtualenv for creating a nice isolated Python environment [http://www.virtualenv.org/en/latest/](http://www.virtualenv.org/en/latest/) and [Tools of the Modern Python Hacker: ](http://www.clemesha.org/blog/modern-python-hacker-tools-virtualenv-fabric-pip)
* Check out the Django Tutorial: [http://docs.djangoproject.com/en/1.3/](http://docs.djangoproject.com/en/1.3/)

---

Test Driven Development
-----------------------
* Test Driven Development good for design, maintenance, and refactoring
* Refactoring an important part of keeping code lean and maintainable
* Having an automated test suite makes this possible/feasible (e.g. Flights->Trips, FlightLeg->Flights)
* Walk through some tests.
* Fixtures vs data helpers. Testing views.

---

Additional Resources
--------------------
* Seattle Python Interest Group (SeaPIG)
* DjangoSeattle.org (local group I co-founded)
* PyCon 2012 in March in Santa Clara, CA. http://us.pycon.org/2012/
* DjangoCon 2011 in Sep in Portland OR. Last year's slides: http://djangocon.us/wiki/slides/
* Google App Engine
* "Dive Into Python" [Online book](http://diveintopython.org/)

----

Other Python web-frameworks
---------------------------
* Pyramid (was Pylons) http://docs.pylonsproject.org/
* Google App Engine http://code.google.com/appengine/
* http://web2py.com/
* http://pyroutes.com/
* http://www.tornadoweb.org/
* http://toastdriven.com/
* http://bottlepy.org/docs/dev/index.html
* https://github.com/breily/juno
* http://webpy.org/
* http://flask.pocoo.org/
* https://github.com/agiliq/so-starving

---

Closing
-------
* Other features we didn't cover: Signals, Forms, custom Tags, Generic Views, Caching, Authentication, Flatpages, Context Processors, etc.  <strong>See "Other batteries included" in the Django documentation.</strong>
* Q&A
* Presentation available at [https://github.com/briangershon/djangointro](https://github.com/briangershon/djangointro)
* Presentation written in Markdown and compiled into a slideshow by [Landslide](http://pypi.python.org/pypi/landslide)
* An Hour with Django and Python by Brian Gershon is licensed under a [Creative Commons Attribution-ShareAlike 3.0 Unported License](http://creativecommons.org/licenses/by-sa/3.0/).