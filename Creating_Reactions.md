# Creating Reactions for CloudRoutes

## Introduction

With CloudRoutes we have kept the creation of reactions as simple as possible by making the reactions modular. Creating a new reaction doesn't mean that you have to edit the main `web.py` file, but rather requires you to create serveral new files that are dynamically loaded into the application.

When creating a new reaction it is important to define a **short-name** for the reaction. This short-name will be used throughout the various modules you create to uniquely identify the reaction.

**For Example:** We have recently deployed a reaction for Restarting AWS EC2 instances into production, the short-name for this reaction is `aws-ec2restart`. This short-name is unique for this type of reaction and all files created for this reaction reference this shortname.

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

The simpliest example can be seen below.

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

As you can see Jinja2 accepts if statements, in this example if the page is in edit mode `data['edit']` will be `True`. As per the template if `data['edit'` is `True`, the web form field `aws_access_key` will be created and prepopulated with the value of `data['reaction']['data']['aws_access_key']`. If the value of `data['edit']` is `False` the form field will be created and the placeholder value will be displayed.

