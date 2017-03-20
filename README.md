loopback-ds-tree-mixin
=============
This module is designed for the [Strongloop Loopback](https://github.com/strongloop/loopback) framework.
It provides a mixin that makes it easy to transform a model into a tree
using the Array of Ancestors pattern, more info [here](https://docs.mongodb.org/manual/tutorial/model-tree-structures-with-ancestors-array)

### Some notes
* This module has only been tested to work with MongoDB. Feel free to test it
on other Databases and PR if something needs to be added.
* At the moment only some basic features are covered, like get the tree and create nodes.
More features will be added down the road

# Installation
```bash
npm install loopback-ds-tree-mixin
```

## server.js
In your `server/server.js` file add the following line before the
`boot(app, __dirname);` line.

```javascript
...
var app = module.exports = loopback();
...
// Add Readonly Mixin to loopback
require('loopback-ds-tree-mixin')(app);
...
```

CONFIG
=============

To use with your Models add the `mixins` attribute to the definition object of your model config.

```json
{
    "name": "Categories",
    "properties": {
        "category": "String",
        "description": "String"

    },
    "mixins": {
        "Tree": {}
    }
}
```

USAGE
=============
The module returns a promise for each call.

Simple example, get the entire tree

```javascript
var Category = app.models.Category;
Category.asTree({})
  .then(function (results) {
  console.log(results);
})
  .catch(function (err) {
    console.log(err);
  });
```

Get all children of a given parent. We can pass a query in the form of an object
`{id : 1}` or `{category:'books'}` or any valid loopback query. The
`where` property is added to your object. You can also pass a string like
`1` which has to be the parent ID or you can pass an object containing the
property id (like the entire parent model)

```javascript
var Category = app.models.Category;
Category.asTree({category : 'Books'})
  .then(function (results) {
  console.log(results);
});
```

```javascript
var Category = app.models.Category;
Category.asTree('5616204f1c6a443c256c7a2f')
  .then(function (results) {
  console.log(results);
});
```

```javascript
var Category = app.models.Category;
Category.findOne({where : {category : 'Books'}})
.then(function(Parent) {
//note, that we will use the Parent.id in our query to locate children
  Category.asTree(Parent)
    .then(function (results) {
    console.log(results);
  });
});
```

## Some cool options
There are times where you want the children attached to the parent.
In this case we can use the `withParent` option

```javascript
var Category = app.models.Category;
Category.asTree('5616204f1c6a443c256c7a2f',{withParent:true})
  .then(function (results) {
  //the results will contain the parent
  /*{
    "category" : "Parent category",
    "children" : [....]
  }*/
  console.log(results);
});
```

In some (not so rare) cases you may need to perform complex operations
on the tree, or you may need a quick reference to all nodes. A quick
solution is to have the entire tree in a flat form where each element
is connected by reference (because javascript) to the appropriate node.
To achieve this we use the `returnEverything` option. We will get back
an object containing both the tree and a flat array

```javascript
var Category = app.models.Category;
Category.asTree('5616204f1c6a443c256c7a2f',{returnEverything:true})
  .then(function (results) {
  //the results will contain the parent
  /*
  returns {tree: treeStructuredArray, flat: aFlatArrayOfTheTree}
  */
  console.log(results);
});
```

### API
The mixin adds a `/asTree` end point to your model. It takes 2 parameters,
the parent query (`{"category":"Books"}`) and the options object.
Only the parent query is required

# Adding nodes
To add a node you need to provide the parent and the node.

```javascript
//Example providing a query object as parent
var Category = app.models.Category;
  Category.addNode({permalink: 'a-category'},
    {
      category : 'A sub category',
      permalink : 'a-sub-category',
      active : true
    })
    .then(function (newItem) {
      console.log(newItem);
    })
    .catch(function (err) {
      console.log('Err', err);
    });
```

Again, you can provide an object, a string or a loopback model as a
parent, just like the `asTree()` method.

```javascript
//Example providing a string ID as parent
var Category = app.models.Category;
  Category.addNode('5616204f1c6a443c256c7a2f',
    {
      category : 'A sub category',
      permalink : 'a-sub-category',
      active : true
    })
    .then(function (newItem) {
      console.log(newItem);
    })
    .catch(function (err) {
      console.log('Err', err);
    });
```

```javascript
//Example providing a loopback model as parent
var Category = app.models.Category;
Category.findOne({where : {category : 'Books'}})
.then(function(Parent) {
  Category.addNode(Parent,
    {
      category : 'A sub category',
      permalink : 'a-sub-category',
      active : true
    })
    .then(function (newItem) {
      console.log(newItem);
    })
});
```

### API
The mixin adds a `/addNode` end point to your model. It takes 2 parameters,
the parent query (`{"category":"Books"}`) and the new node object.
Both parameters are required

# Deleting nodes
You can safely delete a node via the `deleteNode` method. By safely
we mean that you can be sure that the children will be handled accordingly.
If a node has X amount of children, and you just delete it via the
loopback API, then all the children still point to that parent. By using
the `deleteNode` method, you ensure that these children will become
orphaned so that you can take care of them later on as you please.
You can also delete the parent along with the children in one call
by adding the `withChildren` option.
Just as above, you can locate the node you want to delete either
as an ID string, a loopback query or a loopback model.

```javascript
//Default operation. all children will become orphaned
//set withChildren to true to delete the children along with the parent
var Category = app.models.Category;
Category.deleteNode({category : 'Books'},{withChildren:false})
.then(function(result) {
    //True if all went well
})
.catch(function(err){

});
```

### API
The mixin adds a `/deleteNode` end point to your model. It takes 2 parameters,
the node query (`{"category":"Books"}`) and the options object.
Only the first parameter is required

# Moving nodes
Moving a node consists of the following actions :
* Find the node
* Find the parent
* Make the node a child of the parent
* Update all (if any) of the nodes children to the new path

```javascript
var Category = app.models.Category;
//move category sci-fi under the category books
Category.moveNode({permalink: 'sci-fi'},{permalink: 'Books'})
.then(function(result) {
    //get the sci-fi category back with the path updated
})
.catch(function(err){

});
```

### API
The mixin adds a `/moveNode` end point to your model. It takes 2 parameters,
the node query (`{"category":"Books"}`) and the parent query object (same as node).
Both parameters are required

# Saving a json tree
In some cases you might need to update the entire tree based on a json
representation. Maybe your front-end sends the json of the tree after
the user has finished re-ordering it. Assuming that the tree is formed
more or less as it is in your model, and that the children nodes live
in a `children` array, you can pass this json and the mixin will update
your DB accordingly.

### Notes
* ONLY parent - ancestors are updated. Updating other properties is risky
when considering that every model is different.
* On a front-end we usually show just a part of the tree, root nodes are
 hidden from the user in most cases. If that is true, then you need to
 pass the option prependRoot to the method (as usually, can be a query, stringID or model).
 If you don't do this, then the tree will break
 * This is feature is not 100% solid. It worked on many test cases but
 as most front-end plugins transform the JSON they produce, there might
 be several cases where this fails.

```javascript
var Category = app.models.Category;
//req.body.tree is the JSON representation
//req.body.prependRoot is the root i want to append
Category.saveJsonTree(req.body.tree,{prependRoot : req.body.prependRoot || false})
      .then(function (result) {
        res.send(result);
      })
      .catch(function (err) {

      });
```

### API
The mixin adds a `/saveJsonTree` end point to your model. It takes 2 parameters,
the JSON tree (valid Array tree) and the options object.
Only the first parameter is required.

# Events
The mixin emits events on the loopback Event bus for certain operations.
- `lbTree.add.success` when a node was added. Returns the new node
* `lbTree.move.before` Just before start the moving operation. Return the previous parent, the new parent and the node.
* `lbTree.move.after` After the moving operation. Return the previous parent, the new parent and the node.
* `lbTree.move.childrenFound` During the move when we have found all the children. Return the children.
* `lbTree.move.parent` During the move we have the parent. Returns the parent.
* `lbTree.move.newPath` The new path after the move ended. Returns the node.
* `lbTree.delete` Node deleted. Returns {success : true/false}
* `lbTree.saveJsTree` When a JSTree operations is done. Returns the tree

```javascript
var Category = app.models.Category;

Category.on('lbTree.move.childrenFound',function(children){
    console.log("childrenFound",children);
});
Category.on('lbTree.move.parent',function(children){
    console.log("parent",children);
});
```