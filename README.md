# django-generate-series

Use Postgres' generate_series to create sequences with Django's ORM

## Goals

When using Postgres, the set-returning functions allow us to easily create sequences of numbers, dates, datetimes, etc. Unfortunately, this functionality is not currently available within the Django ORM.

This project makes it possible to create such sequences, which are returned as a Django QuerySet. For instance, assuming you have an Order model, you can create a set of sequential dates and then annotate each with the number of orders placed on that date. This will ensure you have no date gaps in the resulting QuerySet. To get the same effect without this package, additional post-processing of the QuerySet with Python would be required.

_New in version 1.0.0_: you can now generate cartesian products with a QuerySet or an iterable. For instance, you can create a sequence of dates that are repeated for each item in a QuerySet or iterable.

- **sequence**: Formally, "a list of objects (or events) which have been ordered in a sequential fashion; such that each member either comes before, or after, every other member."

  In django-generate-series, we can generate sequences of integers, decimals, dates, datetimes, as well as the equivalent ranges of each of these types.

- **term**: The *n*th item in the sequence, where '*n*th' can be found using the id of the model instance.

  This is the name of the field in the model which contains the term value.

## API

The package includes a `generate_series` function from which you can create your own series-generating QuerySets. The field type passed into the function as `output_field` determines the resulting type of series that can be created.

### `generate_series` keyword arguments

- **_start_** - The value at which the sequence should begin (required)
- **_stop_** - The value at which the sequence should end. For range types, this is the lower value of the final term (required)
- **_step_** - How many values to step from one term to the next. For range types, this is the step from the lower value of one term to the next. (required for non-integer types)
- **_span_** - When generating a sequence of ranges (except for date and datetime ranges), this specifies the difference between the lower value of each term and its upper value. Typically, this is the same as the `step` value. (optional, defaults to 1 if neeeded in the query)
- **_output_field_** - A django model field class, one of BigIntegerField, IntegerField, DecimalField, DateField, DateTimeField, BigIntegerRangeField, IntegerRangeField, DecimalRangeField, DateRangeField, or DateTimeRangeField. If not provided, the field will be determined from the type of the `start` input. (optional)
- **_include_id_** - If set to True, an auto-incrementing `id` field will be added to the QuerySet.
- **_max_digits_** - For decimal types, specifies the maximum digits.
- **_decimal_places_** - For decimal types, specifies the number of decimal places.
- **_default_bounds_** - In Django 4.1+, allows specifying bounds for list and tuple inputs. See [Django docs](https://docs.djangoproject.com/en/dev/releases/4.1/#django-contrib-postgres)
- **_queryset_** - If provided, each `pk` in the QuerySet will be combined with the generated series as the cartesian product. This is useful for creating a series that is repeated for each `pk` in the QuerySet. (optional, only one of `queryset` or `iterable` can be provided)
- **_iterable_** - If provided, the iterable will be combined with the generated series as the cartesian product. This is useful for creating a series that is repeated for each item in the iterable. (optional, only one of `queryset` or `iterable` can be provided)

### `generate_series` return value

The function returns a QuerySet of the specified type. The QuerySet will have a `term` field that contains the sequence values. If `include_id` is set to True, the QuerySet will also have an `id` field that contains the sequence values. Finally, if `queryset` or `iterable` is provided, the QuerySet will have a `value` field that contains the primary key values from the provided QuerySet or the items in the provided iterable.

## Basic Examples

### Create a bunch of sequential integers

```python
from django.db import models
from django_generate_series.models import generate_series

integer_sequence_queryset = generate_series(
    start=0, stop=1000, output_field=models.IntegerField,
)

for item in integer_sequence_queryset:
    print(item.term)
```

Result:

    term
    ----
    0
    1
    2
    3
    4
    5
    6
    7
    8
    9
    10
    ...
    1000

#### `generate_series` returns a QuerySet!

Keep in mind that `generate_series` returns a QuerySet, so you can chain the result with other QuerySet methods, use it in subqueries, filter it, etc. For instance, continuing from the previous example:

```python
for item in integer_sequence_queryset.filter(term__lt=5):
    print(item.term)
```

Result:

    term
    ----
    0
    1
    2
    3
    4

### Create a sequence of dates from now until a year from now

```python
from django.db import models
from django.utils import timezone
from django_generate_series.models import generate_series

now = timezone.now().date()
later = (now + timezone.timedelta(days=365))

date_sequence_queryset = generate_series(
    start=now, stop=later, step="1 days", output_field=models.DateField,
)

for item in date_sequence_queryset:
    print(item.term)
```

Result:

    term
    ----
    2022-04-27
    2022-04-28
    2022-04-29
    2022-04-30
    2022-05-01
    2022-05-02
    2022-05-03
    ...
    2023-04-27

_Note: See [the usage examples in the docs](https://django-generate-series.readthedocs.io/en/latest/usage_examples.html) and the example project in the tests directory for further examples of usage._

## Usage with partial

If you often need sequences of a given field type or with certain args, you can use [partial](https://docs.python.org/3/library/functools.html#functools.partial).

Example with default `include_id` and `output_field` values:

```python
from functools import partial

int_and_id_series = partial(generate_series, include_id=True, output_field=BigIntegerField)

qs = int_and_id_series(1, 100)
```
