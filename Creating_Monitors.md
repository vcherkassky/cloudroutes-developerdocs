# Creating Monitors for CloudRoutes

## Introduction

With CloudRoutes we have kept the creation of monitors as simple as possible by making the monitors modular. Creating a new monitor doesn't require you to edit the main `web.py` file, but rather requires you to create several new files that are dynamically loaded into the application.

Because monitors are based on a modular systems it is important to define a shortname for the monitor that will be used to identify the module. 

**For example:** An HTTP GET based monitor that searches for a specific keyword, has a shortname of `http-keyword`. This shortname is used in filenames and as the `ctype` value stored in the database. `ctype` is an abbreviation for **Check Type** so you can think of this shortname as a unique classifier for the monitor.

In this document we will be using the `http-keyword` monitor as reference, this monitor is a Non-API based monitor and is a good example of how simple a monitor is to create.

## Creating a new monitor

### Step 1: The monitor creation form - wtforms

CloudRoutes is written using the [flask](http://flask.pocoo.org/) framework, a common utility for creating web forms within flask is [wtforms](https://wtforms.readthedocs.org/en/latest/). We utilize wtforms for all web forms within the CloudRoutes gui, this includes the forms that create monitors.

For this document we will create a new monitor named `some-monitor`; the first step of creating a monitor is to create the web form needed for users. To start the web form we will create a directory in `cloudroutes-crweb/monitorforms/` called `some-monitor`. Within that directory we will create a `__init__.py` file that will contain a class that defines the form fields required 

    $ mkdir cloudroutes-crweb/monitorforms/some-monitor
    $ vi cloudroutes-crweb/monitorforms/some-monitor/__init__.py

Within this file is the actual wtforms code, you can use the [http-keyword](https://github.com/asm-products/cloudroutes-crweb/blob/master/monitorforms/http-keyword/__init__.py) monitor as an example.

There are a couple of guidelines when creating a web form for monitors.

* The class must be named CheckForm

When the main web component loads the form it will dynamically load the CheckForm class. This dynamic loading relies on the shortname and is pulled from the URL the user navigates to.

* The class should inherit either DatacenterCheckForm or BaseCheckForm

When creating a web form the CheckForm should inherit either the BaseCheckForm or the DatacenterCheckForm classes. You can import these classes with the following

**BaseCheckForm**
    from ..base import BaseCheckForm

**DatacenterCheckForm**
    from ..datacenter import DatacenterCheckForm

The BaseCheckForm class contains the `name` form field and is the lowest level of wtform classes that should be used with monitor forms.

The DatacenterCheckForm class imports the TimerCheckForm which also imports the BaseCheckForm. If you import the DatacenterCheckForm class you will inherit the `name`, `timer`, and `datacenter` fields.

Most monitors should inherit the DatacenterCheckForm, only monitors that do not run in an interval such as API monitors should inherit the BaseCheckForm directly.

### Step 2: The monitor creation form - html

Where Step #1 defines what fields should be present in the form Step #2 actually creates the web form page. The easiest way to create a new web form page is to simply copy an existing page and modify the input fields; a good reference would be the [http-keyword](https://github.com/asm-products/cloudroutes-static/blob/master/templates/monitors/http-keyword.html) html page.

#### Form inputs

The below is an example form input written in HTML and [Jinja2](http://jinja.pocoo.org/).

    <div class="form-group">
    <label for="Host" class="col-sm-4 control-label">Domain to request</label>
      <div class="col-sm-8">
      {% if data['edit'] %}
      {{ form.host(class_="form-control", value=data['monitor']['data']['host']) }}
      {% else %}
      {{ form.host(class_="form-control", placeholder="example.com") }}
      {% endif %}
      </div>
    </div> 

As you can see Jinja2 offers the ability to use if statements within the template. In the above example if we are in edit mode `data['edit']` will be `True` and the form will be prefilled with the value of `data['monitor']['data']['host']`. When generating the edit page in flask, the main web application will lookup the existing monitors information and store it within the dictionary `data['monitor']`.

The below is an example of what data is in an `http-keyword` monitor.

    data['monitor'] = {
      "ctype":  "http-keyword" ,
      "data": {
        "datacenter": [
          "dc1queue" ,
          "dc2queue"
        ] ,
        "host":  "example.com" ,
        "keyword":  "Test" ,
        "name":  "Status Check" ,
        "present":  "True" ,
        "reactions": [
          "c1c0240e-1333-1333-1333-122131112" ,
          "c107250a-1333-1333-13313-abcdefghi73"
        ] ,
        "regex":  "True" ,
        "timer":  "5mincheck" ,
        "url": "http://127.0.0.1/blah.html",
      } ,
      "failcount": 1583 ,
      "id":  "adfasdlkjfsdkljasdf98f0-1a2dbdea2675" ,
      "name":  "Example HTTP" ,
      "status":  "healthy" ,
      "uid":  "sdfsaasdfjlaksdfaskj369-15888dd98382" ,
      "url":  "asdfweqrue0rj2302309rur20cdsa09dafw09iacs09caswekflkwjqfklwejfjf.qwerzPHUz7heZ6VxA"
    }

When a monitor form is submitted the web application will process the form and define the `ctype`, `name`, `failcount`, `status`, `uid`, and `url` keys. The application will then take all of the forms inputs and put them into a dictionary under the `data` key. This system gives us the ability to create new monitors without having to redefine or customize anything outside of the web form itself.

In simplier terms, the data key can change between monitor types however the other fields in `data['monitor']` are meta fields that exist for every monitor.

#### some-monitor.js

When the monitor html page is loaded a .js file of the same name is also loaded. This is meant to be used for popover help text however it can pretty much be used for anything javascript related. Even if you don't plan on utilizing java script for the monitor creation the .js file is required. You can simply create a blank one if necessary.

### Step 3: Creating the actual monitor

Step #1 and #2 were all about creating the web elements of a monitor. Step #3 is actually creating the monitor module itself. There are two main types of monitors in CloudRoutes, API based monitors and Non-API based monitors.

#### API Based Monitors

An example of an API based monitor would be the [datadog-webhook](https://github.com/asm-products/cloudroutes-crweb/blob/master/monitorapis/datadog-webhook/__init__.py) monitor. The end point for API based monitors is `/api/<shortname>/<monitor id>`. The shortname in our example would be `some-monitor` and the ID would be the `id` key for the monitor in the database. When this end point is called the web application will try to load a python module `monitorapis/<shortname>`. If the module does not exist there is an error, if the module does exist then the web application will call the `webCheck` method from that module.

The arguments passed to webCheck will be the full request, monitor object, monitor id, shortname of the monitor type, and an object for the database store.

#### Non-API Based Monitors

Non-API based monitors are monitors that are run via [cram](https://github.com/asm-products/cloudroutes-cram), these monitors are executed from CloudRoutes. You can think of these as external monitors. At the moment of this writing most of these have to do with checking a server/application externally. Using the [http-keyword](https://github.com/asm-products/cloudroutes-cram/tree/master/checks/http-keyword) monitor as an example is the best place to start. All monitors that run through cram are python modules placed into the `checks/` directory. 

Much like the wtforms module in Step #1 to create a new monitor simply create a directory with the shortname and a `__init__.py` file.

    $ mkdir checks/some-monitor
    $ vi checks/some-monitor/__init__.py

The only requirement for this monitor is to have a single method called `check`, this method will be given data which is a dictionary that contains the monitors key information.

##### data

The below is an example of what the data dictionary contains.

    data = {
      "status": "failed",
      "uid": "1232131231231231231-111-15888dd98382", 
      "zone": "Digital Ocean - sfo1", 
      "cid": "232132312312312313123-aea-qer2-vs4e3", 
      "url": "Twerewu230432423owrjewoj3fw3r-.2342432fserw323eaew1234567890204zT6el98CmmI2X30SwCo", 
      "ctype": "http-keyword", 
      "failcount": "412", 
      "time_tracking": {
        "control": 1411488928.422103, 
        "ez_key": "key@example.com", 
        "env": "Prod"
      }, 
      "data": {
        "regex": "True", 
        "datacenter": [
          "dc2queue",
          "dc1queue"
        ], 
        "name": "Some Monitor",
        "keyword": "Test",
        "reactions": [
          "1232432jsad-aefawewr2-adsfa-q23261c5",
          "asfkldjsafj0eq2.-23rq23=afsedfadc359"
        ], 
        "url": "http://example.com/hello.txt",
        "timer": "5mincheck", 
        "host": "example.com",
        "present": "True"
      }, 
      "name": "Some Monitor"
    }

The key item of this monitor is the `data` key, the `data` key holds all of the form values from Step #1 and #2. For most monitors the details are in the data key.

##### What to do after the health check is performed

The actual code to perform the health check really depends on the health check itself, but once you determine if the check was "healthy" you can return with True. If the monitor is determined "failed" the return value should be False.

A simple bare bones health check would look similar to the following code.

    def check(data):
      ## Perform some health check
      if results:
        return True ## Healthy
      else:
        return False ## Failed

At this point you should have a basic working Monitor.

## Getting help

CloudRoutes is developed as part of Assembly, if you have any questions developing a new monitor feel free to drop by our [chat](https://assembly.com/chat/cloudroutes) page.
