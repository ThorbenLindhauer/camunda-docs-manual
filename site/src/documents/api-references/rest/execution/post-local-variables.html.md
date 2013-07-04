Update/Delete Local Execution Variables
=======================================

Updates or deletes the variables in the context of an execution. The updates do not propagate upwards in the execution hierarchy.
Updates precede deletes. So if a variable is updated AND deleted, the deletion overrides the update.


Method
------

POST `/execution/{id}/localVariables`


Parameters
----------

#### Path Parameters

<table class="table table-striped">
  <tr>
    <th>Name</th>
    <th>Description</th>
  </tr>
  <tr>
    <td>id</td>
    <td>The id of the execution to set variables for.</td>
  </tr>
</table>
  

#### Request Body

<p>
  A json object with the following properties:
</p>
<table class="table table-striped">
  <tr>
    <th>Name</th>
    <th>Description</th>
  </tr>
  <tr>
    <td>modifications</td>
    <td>A json object containing variable key-value pairs. Each key is a variable name and each value a json variable value object.
    A variable value object has has the property `value`, which is the value to update.</td>
  </tr>
  <tr>
    <td>deletions</td>
    <td>An array of `String` keys of variables to be deleted.</td>
  </tr>
</table>


Result
------

This method returns no content.


Response codes
--------------  

<table class="table table-striped">
  <tr>
    <th>Code</th>
    <th>Media type</th>
    <th>Description</th>
  </tr>
  <tr>
    <td>204</td>
    <td>application/json</td>
    <td>Request successful.</td>
  </tr>
  <tr>
    <td>500</td>
    <td>application/json</td>
    <td>Update or delete could not be executed, for example because the execution does not exist.</td>
  </tr>
</table>

Example
-------

#### Request

POST `/execution/anExecutionId/localVariables`

Request body:

    {"modifications": 
        {"aVariable": {"value": "aValue"},
        "anotherVariable": {"value": 42}},
    "deletions": [
        "aThirdVariable", "FourthVariable"
    ]}

#### Response

Status 204. No content.