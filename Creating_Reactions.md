# Creating Reactions for CloudRoutes

## Introduction

With CloudRoutes we have kept the creation of reactions as simple as possible by making the reactions modular. Creating a new reaction doesn't mean that you have to edit the main `web.py` file, but rather requires you to create several new files that are dynamically loaded into the application.

When creating a new reaction it is important to define a **short-name** for the reaction. This short-name will be used throughout the various modules you create to uniquely identify the reaction.

**For Example:** We have recently deployed a reaction for Restarting AWS EC2 instances into production, the short-name for this reaction is `aws-ec2restart`. This short-name is unique for this type of reaction and all files created for this reaction reference this short-name.

In this guide we will use the `aws-ec2restart` reaction as an example.

## Creating a new reaction

### Step 1: The reaction web form

CloudRoutes is written using the [flask](http://flask.pocoo.org/) framework, a common utility for creating web forms within flask is [wtforms](https://wtforms.readthedocs.org/en/latest/). We utilize wtforms for all web forms within the CloudRoutes GUI, this includes the forms that create reactions.

For this document we will create a new reaction named `some-reaction`; the first step of creating a reaction is to create the web form needed for users. To start the web form we will create a directory in `crweb/reactions` called `some-reaction`. Within that directory we will create a `__init__.py` file that will contain a class that defines the form fields required.

    $ mkdir crweb/reactionforms/some-reaction
    $ vi crweb/reactionforms/some-reaction/__init__.py

Within this file is the actual wtforms code, you can use the [aws-ec2restart](https://github.com/asm-products/cloudroutes-service/blob/master/crweb/reactionforms/aws-ec2restart/__init__.py) reaction as an example.

There are a couple of guidelines when creating a web form for reactions.

* The class must be named ReactForm

When the main web component loads the form it will dynamically load the ReactForm class. This dynamic loading relies on the short-name and is pulled from the URL the user navigates to.

* The class should inherit BaseReactForm

The BaseReactForm contains base fields that every reaction should contain. You should inherit this class to ensure you have at least the base reaction requirements. You can import this class with the following.

    from ..base import BaseReactForm

#### Example reaction form

The simplest example can be seen below.

    from wtforms import Form, TextField
    from wtforms.validators import DataRequired
    from ..base import BaseReactForm
    
    class ReactForm(BaseReactForm):
      ''' Class that creates a Reaction form for the dashboard '''
      example_field = TextField("Example Field", validators=[DataRequired(message="Example Field is a required field")])
      
    if __name__ == '__main__':
      pass

The above will create a form object that has all of the base required fields and a new field named `example_field`. To add more fields simply add them to the class. 

As an example the below class would create a form that had all of the base fields and two additional fields, one for an API key and another for a Resource ID.

    class ReactForm(BaseReactForm):
        ''' Class that creates a Reaction form for the dashboard '''
        api_key = TextField("API Key", validators=[DataRequired(message="API Key is a required field")])
        resource_id = TextField("Resource ID", validators[DataRequired(message="Resource ID is a required field")])


### Step 2: Reaction form HTML

Where Step #1 created the web form object for Flask, Step #2 is about creating the HTML & [Jinja2](http://jinja.pocoo.org/) template files that render the web form. The easiest way to create a new template is to simply copy an existing one; a good example template is the [aws-ec2restart.html](https://github.com/asm-products/cloudroutes-service/blob/master/static/templates/reactions/aws-ec2restart.html) reaction.

The majority of the `aws-ec2restart.html` file is a basic page structure; for the most part the structure of each reaction page does not change from reaction to reaction. The main components that change are the form fields themselves.

#### Example form field

Below is an example of an input field written in HTML and Jinja2.

    <div class="form-group">
      <label for="AWS Access Key" class="col-sm-4 control-label">AWS Access Key</label>
      <div class="col-sm-8">
        <div class="input-group">
          <span class="input-group-btn">
            <button type="button" id="aws-access-key" class="btn btn-default" rel="popover" data-content="This field should contain your AWS Access Key, which can be obtained from the AWS Management Console." title="AWS Access Key"><i class="fa fa-question"></i></button>
          </span>
          {% if data['edit'] %}
            {{ form.aws_access_key(class_="form-control", value=data['reaction']['data']['aws_access_key']) }}
          {% else %}
            {{ form.aws_access_key(class_="form-control", placeholder="AWS Access Key") }}
          {% endif %}
        </div>
      </div>
    </div>

As you can see Jinja2 accepts if statements, in this example if the page is in edit mode `data['edit']` will be `True`. As per the template if `data['edit'` is `True`, the web form field `aws_access_key` will be created and pre-populated with the value of `data['reaction']['data']['aws_access_key']`. If the value of `data['edit']` is `False` the form field will be created and the placeholder value will be displayed.

When the page is rendered the above template code will turn into the below HTML.

    <div class="form-group">
    <label for="AWS Access Key" class="col-sm-4 control-label">AWS Access Key</label>
    <div class="col-sm-8">
    <div class="input-group">
    <span class="input-group-btn">
    <button type="button" id="aws-access-key" class="btn btn-default" rel="popover" data-content="This field should contain your AWS Access Key, which can be obtained from the AWS Management Console." title="AWS Access Key"><i class="fa fa-question"></i></button>
    </span>
    <input class="form-control" id="aws_access_key" name="aws_access_key" placeholder="AWS Access Key" type="text" value="">
    </div>
    </div>
    </div>

##### Classes for form fields

If you look at the Jinja2 code in the template you will notice the `class_="form-control"` was translated to `class="form-control"` in the HTML. The CloudRoutes GUI has several form classes that should be used depending on the form type, below is the list and what they are used for.

* form-control - General purpose bootstrap form field class
* select - Used for `SelectField` form fields and is provided by `bootstrap-multiselect.css`
* multiselect - Used for `MultiSelectField` form fields and is provided by `bootstrap-multiselect.css`

It is important to make sure that you utilize the appropriate class to ensure a consistent visual appearance on the CloudRoutes dashboard.

#### some-reaction.js

When the reaction page is loaded via the main `web.py` a `.js` file of the same name is also loaded. This file is used for the JavaScript required to activate popover help text, however it should also be utilized for any other JavaScript related code that needs to be imported at the footer of the page. Even if popover text or any other JavaScript code is not utilized for this reaction it is required that a `.js` file is present. You can simply create a blank file if necessary.

    $ touch static/templates/reactions/some-reaction.js

#### Processing the form

As a development team our goal is to ensure that everything is modular, when you create a new reaction you do not need to create code to process the web form inputs. This is done automatically via the web application. It is important however to understand how this processing takes place. 

When the web app processes the new reaction the details will be stored into the `reactions` table in RethinkDB, which is a JSON based database. When you edit a reaction the web application will query RethinkDB and store that reactions details into `data['reaction']`; below is an example of the structure of both the database and `data['reaction']`.

    data['reaction'] = {
      "data": {
        "apikey":  "dslfjalskdj32432lajfs233432fcaewrq11c",
        "domain":  "example.com",
        "email": "example@example.com",
        "ip":  "10.0.3.1",
        "name":  "Remove: example.com - 10.0.3.1"
      } ,
      "frequency": 0,
      "id":  "kasdkldj2342-23faew-234fs-a39d519f78",
      "lastrun": 1411916840.440264,
      "name":  "Remove: example.com - 10.0.3.1",
      "rtype":  "cloudflare-ip-remove",
      "trigger": 0,
      "uid":  "kasldflksajl-asfw-1337-1337-asdfa213"
    }

The above example is a reaction entry from a `cloudflare-ip-remove` reaction. When the web application processes the creation form it will place the `name`, `frequency`, and `trigger` fields under the `data['reaction']` dictionary. All other fields are placed into the `data['reaction']['data']` dictionary. This is a similar format to what the actual reaction code will see as well.

### Step 3: Creating the actual reaction code

Steps #1 and #2 were specifically related to creating the web elements of a reaction. Step #3 however is the creation of the reaction module itself. 

#### Creating a new reaction

Reactions are essentially python modules that are called by another process. To create a new reaction you will first need to create a directory using the short-name in the `crbridge/actions` directory, then create a `__init__.py` file that contains the reactions code.

    $ mkdir crbridge/actions/some-reaction
    $ vi crbridge/actions/some-reaction/__init__.py

A good reference file would be the [aws-ec2restart](https://github.com/asm-products/cloudroutes-service/blob/master/crbridge/actions/aws-ec2restart/__init__.py) reaction.

#### How Reactions are called

In each datacenter/monitoring zone there is a process running that is sometimes referred to as **"the sink"**, this process is executing [crbridge/actioner.py](https://github.com/asm-products/cloudroutes-service/blob/master/crbridge/actioner.py). This process will bind a port and listen for [ZeroMQ](http://zeromq.org/) messages, these messages are the results of health checks from both the web application and [cram/worker.py](https://github.com/asm-products/cloudroutes-service/blob/master/cram/worker.py). 

When the `actioner.py` receives a ZeroMQ message it converts the JSON message into a dictionary; the `actioner.py` will then lookup details from Redis and RethinkDB for both the monitor and all associated reactions. For each reaction defined in the monitor it will load the reaction module `actions/<short-name>` and execute either the `failed` or `healthy` methods.

When a monitor, is healthy the method `actioner.py` will call is the reactions `healthy` method. When a monitor is failed the method will be the `failed` method.

The `actioner.py` process calls these methods with 4 objects, `redata`, `jdata`, `rdb_server` and `r_server`. The `redata` object contains a dictionary of the specific reaction pulled from the database/cache. The `jdata` object contains a dictionary of the JSON message received with some additional fields required for executing reactions. The `rdb_server` is an object required for the connection to the RethinkDB instance. The `r_server` is an object required for the connection to the Redis instance.

Below are examples of the `redata` and `jdata` dictionaries.

##### redata

    redata = {
      "data": {
        "apikey":  "dslfjalskdj32432lajfs233432fcaewrq11c",
        "domain":  "example.com",
        "email": "example@example.com",
        "ip":  "10.0.3.1",
        "name":  "Remove: example.com - 10.0.3.1"
      } ,
      "frequency": 0,
      "id":  "kasdkldj2342-23faew-234fs-a39d519f78",
      "lastrun": 1411916840.440264,
      "name":  "Remove: example.com - 10.0.3.1",
      "rtype":  "cloudflare-ip-remove",
      "trigger": 0,
      "uid":  "kasldflksajl-asfw-1337-1337-asdfa213"
    }

As you can see this is exactly what was queried from the database store.

##### jdata

    jdata = {
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
      "check": {
        "status": "healthy",
        "prev_status": "healthy",
        "method": "automatic"
      },
      "cacheonly": False,
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

Depending on the type of monitor the `data` key may contain different keys and values but all other fields from the above example will exist.

#### Processing reactions

Once `actioner.py` invokes the `failed` or `healthy` method from a reaction it is up to the reaction itself to determine what it should do. Should the reaction actually be executed or not depends on the configuration of each action, this decision is solely placed on the reactions code. While this does vary from reaction to reaction there are a couple of set rules for processing reactions.

1) The reaction must honor trigger and frequency settings

In the current release of CloudRoutes we have implemented a feature called `trigger` and `frequency`, these two settings allow users to define the number of `healthy` or `failed` checks that must be returned before reaction execution and the frequency between reaction executions.

The trigger value is stored in `redata['trigger']` and this can be compared with `jdata['failcount']`. The frequency value is the number of seconds between executions, it is stored in `redata['frequency']` and can be compared with `jdata['lastrun']` which is a timestamp of when the reaction was last executed.

Some reactions may require additional processing rules, each reaction is free to define additional fields in the web form that only have the purpose of determining if the reaction should be executed. An example of this is the `jdata['data']['call_on']` field, in the `aws-ec2restart` reaction this field allows users to define if the restart should happen on `healthy` or `failed` monitors.

The below example is an excerpt of the [aws-ec2restart](https://github.com/asm-products/cloudroutes-service/blob/master/crbridge/actions/aws-ec2restart/__init__.py) reaction.

    def failed(redata, jdata, rdb, r_server):
      ''' This method will be called when a monitor has failed '''
      run = True
      ## Check for Trigger
      if redata['trigger'] > jdata['failcount']:
        run = False
    
      ## Check for lastrun
      checktime = time.time() - float(redata['lastrun'])
      if checktime < redata['frequency']:
        run = False
    
      if redata['data']['call_on'] == 'healthy':
        run = False
    
      if run:
        return actionEC2(redata, jdata)
      else:
        return None

The above is a simple example of how to perform the above processing rules.

2) The return code is important

After a reaction has executed, it is important that an appropriate return code is used. The `actioner.py` process is expecting either a `True`, `False` or `None` return; depending on which value is returned will determine the state of the reactions execution. Below is a list of return value and what that value means to `actioner.py`.

* True - Reaction was processed successfully
* False - Reaction attempted to process but failed
* None - Reaction processing was skipped

If we review the sample code above, we can see that the return value is set to `None` if the reactions function `actionEC2` was not executed. This tells the `actioner.py` process that the reaction was not executed.

      if run:
        return actionEC2(redata, jdata)
      else:
        return None

The return value to the `actioner.py` is important as it is used to determine if the `lastrun` value of the reaction should be updated. It is also used for historical tracking of monitors and reactions, and tells users when reactions were executed or not.

## Getting help

CloudRoutes is developed as part of Assembly, if you have any questions developing a new monitor feel free to drop by our [chat](https://assembly.com/chat/cloudroutes) page.
