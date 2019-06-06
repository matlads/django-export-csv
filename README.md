# django-export-csv
[![Build Status](https://travis-ci.org/oddcc/django-export-csv.svg?branch=master)](https://travis-ci.org/oddcc/django-export-csv)
[![Coverage Status](https://coveralls.io/repos/github/oddcc/django-export-csv/badge.svg?branch=develop)](https://coveralls.io/github/oddcc/django-export-csv?branch=develop)

[Chinese](https://github.com/oddcc/django-export-csv/blob/master/README_CN.md)
## Introduction
a CSV exporter for Django
this tool create a shortcut to render a queryset to a CSV steaming HTTP response. 

## install
Run:
```
pip install django-export-csv
```
Support Python 2.7 and 3.5, Django >= 1.8.

## usage
let your Class-based views which inherit `ListView` or `MultipleObjectMixin` also inherit `QueryCsvMixin`, then you can use `render_csv_response` to turn a queryset into a response with a CSV attachment. `render_csv_response` takes a `QuerySet` or a `ValuesQuerySet` instance:

### CBV
```python
from django_export_csv import QueryCsvMixin
from django.views.generic.list import ListView

from .models import Student


class StudentListView(QueryCsvMixin, ListView):
    queryset = Student.objects.all()

    def get(self, *args, **kwargs):
        return self.render_csv_response(queryset)
```

### FBV
```python
from django_export_csv import render_csv_response


def student_list_view(request):
    if request.method == 'GET':
        queryset = Student.objects.all()
        return render_csv_response(queryset)
```

## custom CSV
once you inherit `QueryCsvMixin`, then you can use following arguments to custom CSV export:

- Use `fields` - (default: `[]`) to set fields you want to export, or use `exclude` - (default: `[]`) exclude which you don't want to. If use neither, it will export all non relational fields by default.
- If you want to export relation fields, just implement your `serializer`, it should take model object as argument and return something you want.
    1. Use `extra_field` - (default: `[]`) to set relation and custom field names.
    2. Use `field_serializer_map` - (default: `{}`) to map field names to your serializers, key is field name, value is corresponding serializer.
    3. Note that fields in `extra_field` must be a corresponding serializer in `field_serializer_map` to work.
- If you want to change the way information export (like `True`/`False` ==> Yes/No), just implement `serializer` and add it to `field_serializer_map`.
- Other arguments:
    1. `filename` - (default: `None`), this argument should be a `str`; if not given(means use default), the CSV filename will be generated by model name.
    2. `add_datestamp` - (default: `False`), this argument should be boolean, if it is `True`, filename will be add a datestamp.
    3. `use_verbose_names` - (default: `True`), this argument should be boolean, if it is `True`, CSV header will use model field's verbose_name.
    4. `field_order` - (default: `[]`), this should be a list to determine field order, any fields not specified will follow those in the list.
    5. `field_header_map` - (default: `{}`), A dictionary mapping model field's name to CSV column header name. Has a higher priority than the `use_verbose_names`.

e.g:

```python
# data_init.py
import datetime
from .models import Student, College


def create_student_and_get_queryset():
    college1, _ = College.objects.get_or_create(name="College 1st")
    college2, _ = College.objects.get_or_create(name="College 2nd")

    Student.objects.get_or_create(
        name='Jim', age=18, is_graduated=False, birthday=datetime.date(1998,6,6), college=college1
    )
    Student.objects.get_or_create(
        name='Bing', age=22, is_graduated=True, birthday=datetime.date(1994, 2, 6), college=college1
    )
    Student.objects.get_or_create(
        name='Monica', age=25, is_graduated=True, birthday=datetime.date(1991, 2, 6), college=college2
    )

    return Student.objects.all()
```

```python
# views.py
from django_export_csv import QueryCsvMixin
from django_export_csv import render_csv_response
from django.views.generic.list import ListView

from .models import Student
from .data_init import create_student_and_get_queryset


def boolean_serializer(value):
    if value == True:
        return 'Y'
    else:
        return 'N'
        
        
def college_serializer(obj):
    return obj.college.name


# CBV
class StudentListView(QueryCsvMixin, ListView):
    filename = 'export_student_list'
    add_datestamp = True
    use_verbose_names = True
    exclude = ['id']
    field_order = ['name', 'is_graduated']
    field_header_map = {'is_graduated': 'Graduated'}
    field_serializer_map = {'is_graduated': boolean_serializer, 'college': college_serializer}
    extra_field = ['college']
    queryset = create_student_and_get_queryset()

    def get(self, *args, **kwargs):
        return self.render_csv_response(self.get_queryset())
        

# FBV
def student_list_view(request):
    filename = 'export_student_list'
    add_datestamp = True
    use_verbose_names = True
    exclude = ['id']
    field_order = ['name', 'is_graduated']
    field_header_map = {'is_graduated': 'Graduated'}
    field_serializer_map = {'is_graduated': boolean_serializer, 'college': college_serializer}
    extra_field = ['college']

    if request.method == 'GET':
        queryset = create_student_and_get_queryset()
        return render_csv_response(
            queryset, filename=filename, add_datestamp=add_datestamp, use_verbose_names=use_verbose_names,
            exclude_field=exclude, field_order=field_order, field_header_map=field_header_map,
            field_serializer_map=field_serializer_map, extra_field=extra_field
        )
```
