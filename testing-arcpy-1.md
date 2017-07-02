# Testing with ArcPy

A common task that can be automated via [arcpy](http://pro.arcgis.com/en/pro-app/arcpy/get-started/what-is-arcpy-.htm) 
is transforming geodatabase data and schemas.  We may want to use [FieldMappings](http://pro.arcgis.com/en/pro-app/arcpy/classes/fieldmappings.htm), add or remove fields, and calculate new values derived from existing fields; all of these operations can be test-driven.

## Test Fixture

To start with, we'll create a file geocodatabase (FGDB); this database will be a [fixture](https://en.wikipedia.org/wiki/Test_fixture#Software), used to hold known data that will be used by our tests.  Since a FGDB is just a directory, we can check it into our project, along with our tests.

When creating a test fixture, we want to use the minimum set of data needed to prove out our logic; in practice, this usually means tables with one or two rows--we'll be running the tests many, many times, so we want them to be as fast as possible!

Note:  ArcMap may lock the database if we have it open; because of this, it's a good idea to add `.lock` files in the test data directory to your `.gitignore` file.

Once we have our fixture created, our project structure should look something like this:

```bash
├── my_project
│   ├── __init__.py
│   └── my_module.py
├── setup.py
└── tests
    ├── __init__.py
    ├── fixtures
    │   └── test_data.gdb
    │       ├── <a whole bunch of FGDB files>
    └── test_my_module.py
```

## Setting up and tearing down

We want each test to run from a fresh, clean state, and to be independent of all the other tests.  We can accomplish this by copying our fixture data into memory, and cleaning up in the `tearDown` method of the python [unittest](https://docs.python.org/2/library/unittest.html) module; this method will automatically be run after each test.

The [Copy](http://pro.arcgis.com/en/pro-app/tool-reference/data-management/copy.htm) tool [doesn't support](https://pro.arcgis.com/en/pro-app/tool-reference/tool-errors-and-warnings/001001-010000/tool-errors-and-warnings-00251-00275-000260.htm) `in_memory` workspaces, so we can use a create and append pattern, with a template:

```python
import os
import unittest
import arcpy


class MyModuleTest(unittest.TestCase):

    TEST_GDB = 'in_memory'

    def setup_data(self, table_name='MyTable'):
        input_file = os.path.abspath('tests/fixtures/test_data.gdb/{}'.format(table_name))
        output_file = '{}/{}'.format(self.TEST_GDB, table_name)
        arcpy.CreateFeatureclass_management(
            out_path=self.TEST_GDB,
            out_name=table_name,
            template=input_file
        )
        arcpy.Append_management(
            inputs=input_file,
            target=output_file,
            schema_type='NO_TEST'
        )
        return output_file
```

To clean up, we call the [Delete](http://pro.arcgis.com/en/pro-app/tool-reference/data-management/delete.htm) tool in our `tearDown` method:

```python
    def tearDown(self):
        arcpy.Delete_management(self.TEST_GDB)
```

Now we're ready to write our first test!

## Adding a Field

Say we want our business code to add a `SUM` field to the table, and calculate some value.  We'll write our test first:

```python
def test adds_sum_field(self):
    feature_class = self.setup_data('SumData')
    field_name = 'SUM'
    add_sum_field(feature_class)
    field_names = [field.name for field in arcpy.ListFields(feature_class)]
    self.assertTrue(field_name in field_names)
```

and watch it fail:

```shell
$ pytest
============================= test session starts =============================
platform win32 -- Python 2.7.12, pytest-3.1.2, py-1.4.34, pluggy-0.4.0
rootdir: C:\develop\testing-arcpy, inifile:
collected 1 items

tests\test_my_module.py F

================================== FAILURES ===================================
______________________ MyModuleTest.test_adds_sum_field _______________________

self = <tests.test_my_module.MyModuleTest testMethod=test_adds_sum_field>

    def test_adds_sum_field(self):
        feature_class = self.setup_data('SumData')
        field_name = 'SUM'
        field_names = [field.name for field in arcpy.ListFields(feature_class)]
>       self.assertTrue(field_name in field_names)
E       AssertionError: False is not true

tests\test_my_module.py:32: AssertionError
========================== 1 failed in 11.63 seconds ==========================
```

Next we add our implementation code to the business module:

```python
import arcpy


def add_sum_field(feature_class):
    arcpy.AddField_management(
        in_table=feature_class,
        field_name='SUM',
        field_type='DOUBLE'
    )
```

And rerun our tests:

```shell
$ pytest
============================= test session starts =============================
platform win32 -- Python 2.7.12, pytest-3.1.2, py-1.4.34, pluggy-0.4.0
rootdir: C:\develop\testing-arcpy, inifile:
collected 1 items

tests\test_my_module.py .

========================== 1 passed in 11.61 seconds ==========================
```

## Calculating a Field Value

Now we want to add some logic to calculate our `SUM` value; again, we'll write the test first:

```python
    def test_calculates_sum(self):
        feature_class = self.setup_data('SumData')
        add_sum_field(feature_class)
        with arcpy.da.SearchCursor(feature_class, ['SUM']) as cursor:
            for row in cursor:
                self.assertEqual(row[0], 2)
```

and watch it fail:

```shell
$ pytest
============================= test session starts =============================
platform win32 -- Python 2.7.12, pytest-3.1.2, py-1.4.34, pluggy-0.4.0
rootdir: C:\develop\testing-arcpy, inifile:
collected 2 items

tests\test_my_module.py .F

================================== FAILURES ===================================
______________________ MyModuleTest.test_calculates_sum _______________________

self = <tests.test_my_module.MyModuleTest testMethod=test_calculates_sum>

    def test_calculates_sum(self):
        feature_class = self.setup_data('SumData')
        add_sum_field(feature_class)
        with arcpy.da.SearchCursor(feature_class, ['SUM']) as cursor:
            for row in cursor:
>               self.assertEqual(row[0], 2)
E               AssertionError: None != 2

tests\test_my_module.py:42: AssertionError
===================== 1 failed, 1 passed in 12.76 seconds =====================
```

then implement our business logic:

```python
import arcpy


def add_sum_field(feature_class):
    arcpy.AddField_management(
        in_table=feature_class,
        field_name='SUM',
        field_type='DOUBLE'
    )

    fields = ['FIRST_VALUE', 'SECOND_VALUE', 'SUM'] 
    with arcpy.da.UpdateCursor(feature_class, fields) as cursor:
        for row in cursor:
            row[2] = row[0] + row[1]
            cursor.updateRow(row)
```

And check that our tests pass:

```shell
$ pytest
============================= test session starts =============================
platform win32 -- Python 2.7.12, pytest-3.1.2, py-1.4.34, pluggy-0.4.0
rootdir: C:\develop\testing-arcpy, inifile:
collected 2 items

tests\test_my_module.py ..

========================== 2 passed in 13.22 seconds ==========================
```