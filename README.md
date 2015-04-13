# jquery_alchemy

`jquery_alchemy` is a python module which (automatically) creates client-side validation rules for the [http://jqueryvalidation.org/](jQuery Validation Plugin) from [http://www.sqlalchemy.org/](SQLAlchemy) columns, based on their data types.

It is designed to work with web frameworks which use templates, such as [http://www.pylonsproject.org/](Pyramid) in conjunction with orm form-generation modules, such as [https://wtforms-alchemy.readthedocs.org/en/latest/](wtforms_alchemy), [https://colanderalchemy.readthedocs.org/en/latest/](deform (colander_alchemy)) or [http://docs.formalchemy.org/](formalchemy) (though it can work equally well without these).

This is very much a work-in-progress - it currently maps from many of the 'simple' data types, and provides validation for required (i.e. non-empty) fields and field length.  However there are many more mappings which could be done between sqlalchemy column types and jquery validation types... patches welcome!

## Use

The module accepts an sqlalchemy Base class as the argument, then parses this class looking at columns for explicit rule configuration (in the `info` dictionary `jquery_validate` key) or generating implicit rules based on the column types.

Explicit configuration will prevent implicit configuration being generated for that column - so you need to specify all configuration options explicitly.

The returned dictionary should be converted to a json hash (using `json` or equivalrnt module) to use in the javascript in your template. (Note that you may need to prevent escaping or filtering in your template).

## Simple Example

The following example assumes use with `Pyramid`, `WTForms` and `Mako` templating, but it should be easy to understand the use of `jquery_alchemy` with other frameworks from this.

Note the explicit configuration specified for the `uid` field - the others will be automatically generated.

```python
# models.py

# Import the appropriate method from the module
from jquery_alchemy import jquery_alchemy

from sqlalchemy import (
    Base,
    Column,
    Integer,
    Unicode,
    )

from sqlachemy_utils import EmailType
from wtforms import SubmitField
from wtforms_alchemy import ModelForm

class User(Base):
    __tablename__ = 'users'
    id = Column(Integer, autoincrement=True, primary_key=True)
    firstname = Column(Unicode(50),
                       nullable=False,)
    lastname = Column(Unicode(50),
                      nullable=False,)
    email = Column(EmailType(255),
                   unique=True,
                   nullable=False,)
    uid = Column(Integer(8),
                 info={'jquery_validate':{'required': True, #  explicit config
                                          'minlength': 8,
                                          'number': True}},)

class UserForm(ModelForm):
    class Meta:
        model = User

    submit = SubmitField('Submit')

# Populate the 'rules' dictionary
jquery_rules = jquery_alchemy(User)

```

```python
# views.py

import json

# Import the rules dictionary from models
from .models import (
    User,
    jquery_rules
    )

@view_config(route_name='myform', renderer='templates/myform.mako')
def myform(request):
    user = User()
    form = UserForm(request.POST)
    if request.method == 'POST' and form.validate():
        form.populate_obj(user)
        DBSession.add(user)
        return HTTPFound(location=request.route_url('home'))
    # Pass a json dump of the rules dictionary to the template
    return {'form':form, 'rules_json': json.dumps(jquery_rules)}

```

```mako
## myform.mako

<script type="text/javascript">
  $().ready(function() {
    ## Use the (un-escaped) json rules dictionary
    $("#myform").validate(${rules_json|n});
  });
</script>

<form id="myform" action="${request.route_url('myform')}" method="post">
% for field in form:
  ${field.label}: ${field}
  <br/>
</form>
% endfor
```

This will generate the following rules dictionary:

```html
<!-- generated rules -->
<script type="text/javascript">
  $().ready(function() {
    $("#myform").validate({
                            "rules": {
                                "lastname": {
                                    "required": true,
                                    "maxlength": 50
                                },
                                "id": {
                                    "integer": true,
                                    "required": true
                                },
                                "firstname": {
                                    "required": true,
                                    "maxlength": 50
                                },
                                "email": {
                                    "required": true,
                                    "email": true,
                                    "maxlength": 255
                                },
                                "uid": {
                                    "required": true,
                                    "minlength": 8,
                                    "number": true
                                }
                            }
                        });
  });
</script>
```