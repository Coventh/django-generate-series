# django-generate-series

Use Postgres' generate_series to create sequences with Django's ORM

https://django-generate-series.readthedocs.io/

## Goals

When using Postgres, the set-returning functions allow us to easily create sequences of numbers, dates, datetimes, etc. Unfortunately, this functionality is not currently available within the Django ORM.

This project makes it possible to create such sequences, which can then be used with Django QuerySets. For instance, assuming you have an Order model, you can create a set of sequential dates and then annotate each with the number of orders placed on that date. This will ensure you have no date gaps in the resulting QuerySet. To get the same effect without this package, additional post-processing of the QuerySet with Python would be required.

## API

The package includes a `generate_series` function from which you can create your own series-generating QuerySets. The field type passed into the function as `output_field` determines the resulting type of series that can be created.

```python
# Create a bunch of sequential integers
integer_sequence_queryset = generate_series(
    0, 1000, output_field=models.IntegerField,
)

for item in integer_sequence_queryset:
    print(item.term)
```

Result:

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

```python
# Create a sequence of dates from now until a year from now
now = timezone.now().date()
later = (now + timezone.timedelta(days=365))

date_sequence_queryset = generate_series(
    now, later, "1 days", output_field=models.DateField,
)

for item in date_sequence_queryset:
    print(item.term)
```

Result:

    2022-04-27
    2022-04-28
    2022-04-29
    2022-04-30
    2022-05-01
    2022-05-02
    2022-05-03
    ...
    2023-04-27

*Note: See the docs and the example project in the tests directory for further examples of usage.*

## Terminology

Although this packages is named django-generate-series based on Postgres' [`generate_series` set-returning function](https://www.postgresql.org/docs/current/functions-srf.html), mathematically we are creating a [sequence](https://en.wikipedia.org/wiki/Sequence) rather than a [series](https://en.wikipedia.org/wiki/Series_(mathematics)).

- **sequence**: Formally, "a list of objects (or events) which have been ordered in a sequential fashion; such that each member either comes before, or after, every other member."

    In django-generate-series, we can generate sequences of integers, decimals, dates, datetimes, as well as the equivalent ranges of each of these types.

- **term**: The *n*th item in the sequence, where '*n*th' can be found using the id of the model instance.

    This is the name of the field in the model which contains the term value.
