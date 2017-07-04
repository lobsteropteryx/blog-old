# Testing with ArcPy - Isolation and Mocking

[Test fixtures]() can help us run our each of our tests against a clean set of known data, but arcpy can still throw a few curve balls at us--there are global singletons we need to be aware of (i.e., `arcpy.env`), and many GP tools can have side effects (such as changing the current working directory) or limitations of their own (i.e., the [Project](http://pro.arcgis.com/en/pro-app/tool-reference/data-management/project.htm) and [Copy](http://pro.arcgis.com/en/pro-app/tool-reference/data-management/copy.htm) tools don't support in-memory workspaces).  In addition, just loading arcpy has a significant performance cost, and we want our tests to be as fast as possible.

We also want to be sure that we're testing **our** business logic and not arcpy's; ideally our tests will be making assertions about the state of the system after some public method has been called, and not testing the implementation details--we care about *what* the state of the system looks like, not *how* it got there.

To avoid some of the difficulties around the arcpy module, and keep focused on our own, custom, business logic, we can use a couple of techniques:  **Isolating** arcpy within our project, and **Mocking** it when we need to test some adjacent logic.

## Isolating arcpy calls

In a project of even moderate size and complexity, it's almost always worth wrapping arcpy calls, keeping them isolated to their own module(s), rather than sprinkling them throughout the codebase.  In practice, this often means that we want only a single `import arcpy` statement in our project, with the majority of our business logic in other modules.  By doing so, we can protect ourselves against future changes and decrease our iteration time substantially.

### Protection against changes 

We don't have control over the arcpy module, and there's no guarantee that the function signatures or pubic interfaces won't change the next time we upgrade ArcMap; we might be asked to port our application to use ESRI's new [Python API](https://developers.arcgis.com/python/), or even [GDAL](http://www.gdal.org/)!  By keeping our arcmap calls isolated, we minimize the number of places we need to make code changes in the future if our underlying GIS system changes.

### Iteration time

There's no getting around it:  Testing with arcpy is SLOW.  As the codebase grows and the number of tests increase, it can make sense to run a subset of the tests while you're iterating, and the full suite less frequently, before checking in.  By keeping arcpy calls isolated, we can get orders of magnitude better performance out of our tests when working in a part of the codebase that doesn't need arcpy.

By way of example, consider a small module that updates field values in a geodatabase; we'll call it `updater.py`:

```python
import arcpy

def update_field(feature_class):
    fields = ['MY_STRING_FIELD'] 
    with arcpy.da.UpdateCursor(feature_class, fields) as cursor:
        for row in cursor:
            row[0] = init_cap(row[0])
            cursor.updateRow(row)
            
def init_cap(field_value):
    return " ".join(word.capitalize() for word in field_value.split())
```

Our tests for the `init_cap` method might look like:

```python
from unittest import TestCase
from updater import init_cap

class TestInitCap(TestCase):
    def test_returns_empty_string_if_field_is_empty(self):
        expected = ''
        actual = init_cap('')
        self.assertEqual(actual, expected)
        
    def test_capitalizes_first_letter_of_single_word(self):
        expected = 'Foo'
        actual = init_cap('foo')
        self.assertEqual(actual, expected)
        
    def test_capitalizes_first_letter_of_multiple_words(self):
        expected = 'Foo Bar'
        actual = init_cap('foo bar')
        self.assertEqual(actual, expected)        

    def test_does_not_capitalize_words_that_start_with_numbers(self):
        expected = '3g'
        actual = init_cap('3g')
        self.assertEqual(actual, expected)                
```

Even without using a single arcpy call, the runtime of our tests is terrible:

```bash
$ pytest tests/test_updater.py
============================= test session starts =============================
platform win32 -- Python 2.7.12, pytest-3.1.2, py-1.4.34, pluggy-0.4.0
rootdir: C:\develop\testing-arcpy, inifile:
collected 4 items

tests\test_updater.py ....

========================== 4 passed in 14.35 seconds ==========================
```

By splitting the `init_cap` method out into its own module, we can avoid importing arcpy, and get a much better runtime:

```bash
$ pytest tests/test_updater.py
============================= test session starts =============================
platform win32 -- Python 2.7.12, pytest-3.1.2, py-1.4.34, pluggy-0.4.0
rootdir: C:\develop\testing-arcpy, inifile:
collected 4 items

tests\test_updater.py ....

========================== 4 passed in 0.03 seconds ===========================
```

That's almost 500 times faster!

## Mocking arcpy calls

Sometimes we can't help but have arcpy objects mixed in with our business logic; we may be working closely with domain objects in the GIS, (i.e. [MapDocument](http://desktop.arcgis.com/en/arcmap/latest/analyze/arcpy-mapping/mapdocument-class.htm) or [Layer](http://desktop.arcgis.com/en/arcmap/latest/analyze/arcpy-mapping/layer-class.htm) objects), and our business logic might be interacting with them directly.  In these cases, we can [mock](https://en.wikipedia.org/wiki/Mock_object) the arcpy calls and objects, to "get at" our actual business logic and test its functionality.

### Listing unique workspaces within a map document
Suppose we want to get a list of the unique workspaces used by a particular map document; we might be setting up a new server instance, and we need to register the data sources that our map services will consume.  There could be SDE connections, shapefiles, and cloud-based services all within a single map, and each data source could be used by multiple layers.

We can get a list of `Layer` objects from the `MapDocument.ListLayers` method, and we'll have to ask each layer for its [workspacePath]() property to build our list.  Once we have the list of *all* workspaces for all the layers, it's easy to narrow that down to the unique ones.  Our test might look like this:

```python

from unittest import TestCase
from mock import MagicMock, patch
from arcpy_helper.ArcpyHelper import list_workspaces_for_mxd

@patch('my_module.arcpy_helper.arcpy')
class TestListDataSources(TestCase):

    @staticmethod
    def create_mock_layer(workspace):
        layer = MagicMock()
        layer.supports = MagicMock(return_value=True)
        layer.workspacePath = workspace
        return layer

    def test_lists_unique_data_sources(self, mock_arcpy):
        data_source1 = 'layer1'
        data_source2 = 'layer2'
        layer1 = self.create_mock_layer(data_source1)
        layer2 = self.create_mock_layer(data_source2)
        mock_arcpy.mapping.ListLayers = MagicMock(return_value=[layer1, layer1, layer2])
        expected = sorted([data_source2, data_source1])
        actual = sorted(list_workspaces_for_mxd({}))
        self.assertEqual(expected, actual)
```

