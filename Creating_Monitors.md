# Creating Monitors for CloudRoutes

## Introduction

With CloudRoutes we have kept the creation of monitors as simple as possible by making the monitors modular. Creating a new monitor doesn't require you to edit the main `web.py` file, but rather requires you to create several new files that are dynamically loaded into the application.

When creating a new monitor it is important to define a **short-name** for the monitor. This short-name will be used throughout the various modules you create to uniquely identify the monitor.

**For example:** An HTTP GET based monitor that searches for a specific keyword, has a short-name of `http-keyword`. This short-name is unqiue for this type of monitor and all files created for this monitor reference this short-name.

In this guide we will be using the `http-keyword` monitor as reference, this monitor is a Non-API based monitor and is a good example of how simple a monitor is to create.

## Creating a new monitor

### Step 1: The monitor web form

CloudRoutes is written using the [flask](http://flask.pocoo.org/) framework, a common utility for creating web forms within flask is [wtforms](https://wtforms.readthedocs.org/en/latest/). We utilize wtforms for all web forms within the CloudRoutes GUI, this includes the forms that create monitors.

For this document we will create a new monitor named `some-monitor`; the first step of creating a monitor is to create the web form needed for users. To start the web form we will create a directory in `crweb/monitorforms/` called `some-monitor`. Within that directory we will create a `__init__.py` file that will contain a class that defines the form fields required 

    $ mkdir crweb/monitorforms/some-monitor
    $ vi crweb/monitorforms/some-monitor/__init__.py

Within this file is the actual wtforms code, you can use the [http-keyword](https://github.com/asm-products/cloudroutes-service/blob/master/crweb/monitorforms/http-keyword/__init__.py) monitor as an example.

There are a couple of guidelines when creating a web form for monitors.

* The class must be named CheckForm

When the main web component loads the form it will dynamically load the CheckForm class. This dynamic loading relies on the short-name and is pulled from the URL the user navigates to.

* The class should inherit either DatacenterCheckForm or BaseCheckForm

When creating a web form the CheckForm should inherit either the BaseCheckForm or the DatacenterCheckForm classes. You can import these classes with the following

**BaseCheckForm**

    from ..base import BaseCheckForm

**DatacenterCheckForm**

    from ..datacenter import DatacenterCheckForm

The BaseCheckForm class contains the `name` form field and is the lowest level of wtform classes that should be used with monitor forms.

The DatacenterCheckForm class imports the TimerCheckForm which also imports the BaseCheckForm. If you import the DatacenterCheckForm class you will inherit the `name`, `timer`, and `datacenter` fields.

Most monitors should inherit the DatacenterCheckForm, only monitors that do not run in an interval such as API monitors should inherit the BaseCheckForm directly.

#### Example monitor form

The below example will show an example monitor form class that has two fields, one for an IP and one for a Port.

    from wtforms import Form, TextField
    from wtforms.validators import DataRequired, IPAddress, NumberRange
    from ..datacenter import DatacenterCheckForm
    
    class CheckForm(DatacenterCheckForm):
      ''' Class that creates a Check form for the dashboard '''
      ip = TextField("IP", validators=[IPAddress(message='Does not match IP address format')])
      port = TextField("Port", validators=[NumberRange(message='Port must be a number between 1 and 65535')])
    
    if __name__ == '__main__':
      pass

If we wanted to add a second field that contains a `hostname` we could add it by just adding another `TextField` object to the class.

    class CheckForm(DatacenterCheckForm):
      ''' Class that creates a Check form for the dashboard '''
      ip = TextField("IP", validators=[IPAddress(message='Does not match IP address format')])
      port = TextField("Port", validators=[NumberRange(message='Port must be a number between 1 and 65535')])
      hostname = TextField("Hostname", validators=[DataRequired(message='Hostname is a required field')])

### Step 2: The monitor form HTML

Where Step #1 defines what fields should be present in the form Step #2 actually renders the web form page. The easiest way to create a new web form page is to simply copy an existing template and modify the input fields; a good reference would be the [http-keyword](https://github.com/asm-products/cloudroutes-static/blob/master/templates/monitors/http-keyword.html) template.

#### Example form field

The below is an example form input field written in HTML and [Jinja2](http://jinja.pocoo.org/); Jinja2 is the templating language used in Flask.

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

As you can see Jinja2 offers the ability to use if statements within the template. In the above example if we are in edit mode `data['edit']` will be `True` and the form will be pre-filled with the value of `data['monitor']['data']['host']`. If the value of `data['edit']` is `False` the form field will be created and the placeholder value will be displayed.

The below HTML is an example of the above templates rendering when `data['edit']` is `False`.

    <div class="form-group">
    <label for="Host" class="col-sm-4 control-label">Domain to request</label>
    <div class="col-sm-8">
    <input class="form-control" id="host" name="host" placeholder="example.com" type="text" value="">
    </div>
    </div>

#### some-monitor.js

When the monitor page is loaded via the main `web.py` a `.js` file of the same name is also loaded. This file is used for the JavaScript required to activate popover help text, however it should also be utilized for any other JavaScript related code that needs to be imported at the footer of the monitor page.

Even if popover text or any other JavaScript code is not utilized for this monitor it is required that a `.js` file is present. You can create a blank file if necessary.

    $ touch static/templates/monitors/some-monitor.js

#### Processing the form

As a development team our goal is to ensure that everything is modular, when you create a new monitor you do not need to create code to process the web form inputs. This is done automatically via the web application. It is important however to understand how this processing takes place.

When the web app processes the new monitor the details will be stored into the monitors table in RethinkDB, which is a JSON based database. When you edit a monitor the web application will query RethinkDB and store that monitors details into `data['monitor']`; below is an example of the structure of both the database and `data['monitor']`.

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

In simpler terms, the data key can change between monitor types however the other fields in `data['monitor']` are meta fields that exist for every monitor.


### Step 3: Creating the actual monitor code

Step #1 and #2 were all about creating the web elements of a monitor. Step #3 is actually creating the monitor module itself. There are two main types of monitors in CloudRoutes, API based monitors and Non-API based monitors.

#### API Based Monitors

An example of an API based monitor would be the [datadog-webhook](https://github.com/asm-products/cloudroutes-service/blob/master/crweb/monitorapis/datadog-webhook/__init__.py) monitor. The end point for API based monitors is `/api/<short-name>/<monitor id>`. The short-name in our example would be `some-monitor` and the ID would be the `id` key for the monitor in the database. When this end point is called the web application will try to load a python module `monitorapis/<short-name>`. If the module does not exist there is an error, if the module does exist then the web application will call the `webCheck` method from that module.

##### Example API Monitor

The following is an example of a simple API based monitor that always marks the monitor failed when called.

    def webCheck(request, monitor, cid, atype, rdb):
      ''' Process the webbased api call '''
      replydata = { 
        'headers': { 'Content-type' : 'application/json' }
        }
      jdata = request.json
       
      ## Delete the Monitor
      monitor.get(cid, rdb)
      if jdata['check_key'] == monitor.url and atype == monitor.ctype:
        monitor.healthcheck = "failed"
        result = monitor.webCheck(rdb)
      replydata['data'] = "{'success':'True'}"
      return replydata

When the `webCheck` method is called it will be given 5 arguments; `request`, `monitor`, `cid`, `atype` and `rdb`. The `request` argument is the full `request` object from Flask, this contains all POST data and Headers of the API request. The `monitor` argument is an object for the `Monitor` class, in the example above we use the `get`, `healthcheck` and `webCheck` methods from this class. The `cid` is the monitor ID value passed from the URL, this is not a validated ID and should be treated the same as any user input. The `atype` argument is the type of API being requested, this is essentially the `ctype` key in the monitors meta data. The `rdb` object is a connection object to the RethinkDB database store.

In the above example if the POST data contains a JSON string that has a key `check_key` and that key is the same as the `monitor.url` objects value, and the `atype` value is the same as the `monitor.cytype` objects value. The `monitor.healthcheck` object will be set to `failed` and the `monitor.webCheck` method will be called. This method will send a health check message to the backend [crbridge](https://github.com/asm-products/cloudroutes-service/tree/master/crbridge) process. This process will process the failed monitor and perform necessary reactions.

To get started with a new API based monitor you will first need to create a new directory with the short-name under the `crweb/monitorapis` directory and then create an `__init__.py` file that contains the API processing code.

    $ mkdir crweb/monitorapis/some-monitor
    $ vi crweb/monitorapis/some-monitor/__init__.py

#### Non-API Based Monitors

Non-API based monitors are monitors that are run via [cram](https://github.com/asm-products/cloudroutes-service/tree/master/cram), these monitors are executed from CloudRoutes. You can think of these as external monitors. At the moment of this writing most of these have to do with checking a server/application externally. Using the [http-keyword](https://github.com/asm-products/cloudroutes-service/tree/master/cram/checks/http-keyword) monitor as an example is the best place to start. All monitors that run through `cram` are python modules placed into the `checks/` directory. 

Much like the wtforms module in Step #1 to create a new monitor simply create a directory with the short-name and a `__init__.py` file.

    $ mkdir cram/checks/some-monitor
    $ vi cram/checks/some-monitor/__init__.py

The only requirement for this monitor is to have a single method called `check`, this method will be given `data` which is a dictionary that contains the monitors information from the database as well as some extra information from the `cram/control.py` process.

##### Example of `data`

The below is an example of what the `data` dictionary contains.

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

The key item of this monitor is the `data['data']` dictionary, the `data['data']` dictionary holds all of the form values from Step #1 and #2. For most monitors the details are in this dictionary.

##### What to do after the health check is performed

The actual code to perform the health check really depends on the health check itself, but once you determine if the check was "healthy" the check function should return `True`. If the monitor is determined "failed" the return value should be `False`.

###### Example Health Check: http-get-statuscode

The following monitor code is from the `http-get-statuscode` monitor and can be used as a guide on how to write a `cram` based monitor.

    def check(data):
      """ Perform a http get request and validate the return code """
      headers = { 'host' : data['data']['host'] }
      timeout = 3.00
      url = data['data']['url']
      try:
        result = requests.get(url, timeout=timeout, headers=headers, verify=False)
      except:
        return False
      rcode = str(result.status_code)
      if rcode in data['data']['codes']:
        return True
      else:
        return False

###### Example Health Check: always-true

The following monitor code will always return `True` which means the monitor itself will always be `healthy`.

    def check(data):
      ''' Always return healthy '''
      return True


## Getting help

At this point you should have at least a mostly working monitor, if not and you need some help you can reachout to our [Assembly](https://assembly.com/chat/cloudroutes) page.
