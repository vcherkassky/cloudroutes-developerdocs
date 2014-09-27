# Creating Reactions for CloudRoutes

## Introduction

With CloudRoutes we have kept the creation of reactions as simple as possible by making the reactions modular. Creating a new reaction doesn't mean that you have to edit the main `web.py` file, but rather requires you to create serveral new files that are dynamically loaded into the application.

When creating a new reaction it is important to define a **short-name** for the reaction. This short-name will be used throughout the various modules you create to uniquely identify the reaction.

**For Example:** We have recently deployed a reaction for Restarting AWS EC2 instances into production, the short-name for this reaction is `aws-ec2restart`. This short-name is unique for this type of reaction and all files created for this reaction reference this shortname.

In this guide we will use the `aws-ec2restart` reaction as an example.

## Creating a new reaction

### Step 1: The reaction creation web form - wtforms

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

The above will create a form object that has all of the based required forms and a new field named `example_field`.
