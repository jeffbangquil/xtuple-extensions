## xTuple Extension Tutorial
### Part II: Extending a business object

Having completed **Part I** of our tutorial, we can now manage `IceCreamFlavors` per the customer's requirements. Now we need to be able to extend the `Contact` business object to add `IceCreamFlavor` as a field.

### Tables

In a perfect world, we would just go into the `cntct` table and add a column. This is not an option. We're writing a humble extension here! We have no authority to make changes to core tables in the `public` schema.

We'll take the next-easiest approach. Let's create a new table that will function as a link table between `contact` and `icflav`, and then extend the `Contact` ORM. The good news is that when we complete this plumbing in the database, the `Contact` business object will appear in the application as if this field were in it from the beginning. Open a new file `database/source/cntcticflav.sql`:

```javascript
select xt.create_table('cntcticflav');

select xt.add_column('cntcticflav','cntcticflav_id', 'serial', 'primary key');
select xt.add_column('cntcticflav','cntcticflav_cntct_id', 'integer', 'references cntct (cntct_id)');
select xt.add_column('cntcticflav','cntcticflav_icflav_id', 'integer', 'references xt.icflav (icflav_id)');

comment on table xt.cntcticflav is 'Joins Contact with Ice cream flavor';
```

Don't forget to add this new file to the `manifest.js` file, underneath the definition for `icflav.sql`.

### ORMs

We need to extend the pre-existing `Contact` ORM to have it include `IceCreamFlavor` as a new field. By convention, ORM definitions which extend existing ORMs should go in the `database/orm/ext` directory. Make that directory and add into it a file named `contact.json`:

```javascript

  {
    "context": "icecream",
    "nameSpace": "XM",
    "type": "Contact",
    "table": "xt.cntcticflav",
    "isExtension": true,
    "isChild": false,
    "comment": "Extended by Icecream",
    "relations": [
      {
        "column": "cntcticflav_cntct_id",
        "inverse": "id"
      }
    ],
    "properties": [
     {
        "name": "favoriteFlavor",
        "toOne": {
         "type": "IceCreamFlavor",
         "column": "cntcticflav_icflav_id"
        }
     }
    ],
    "isSystem": true
  },
  {
    "context": "icecream",
    "nameSpace": "XM",
    "type": "ContactListItem",
    "table": "xt.cntcticflav",
    "isExtension": true,
    "isChild": false,
    "comment": "Extended by Icecream",
    "relations": [
      {
        "column": "cntcticflav_cntct_id",
        "inverse": "id"
      }
    ],
    "properties": [
     {
        "name": "favoriteFlavor",
        "toOne": {
         "type": "IceCreamFlavor",
         "column": "cntcticflav_icflav_id"
        }
     }
    ],
    "isSystem": true
  }
]
```

What we're doing here is pointing back to the original `Contact` ORM and telling it that there's another bit of data for it, in the `xt.cntcticflav` table. Note that we're adding the field both to the editable object and the list item object. That's because one drives the workspace and the other drives the list, and we're going to want both to have access to the new `favoriteFlavor` field.

### Models

We don't need to add anything to the model layer. The new field to `Contact` will be pulled up reflectively from the ORM. The link table itself has no ORM and therefore needs no model.

### The Cache

We are going to use a `XV.Picker` in the `Contact` workspace, which will rely on the Ice Cream Flavor collection to be cached in the browser. Let's set that up now, in the file `client/models/startup.js`

```javascript
XT.cacheCollection("XM.iceCreamFlavors", "XM.IceCreamFlavorCollection");
```

That was easy (don't forget reference this in the `package.js` file, underneath `ice_cream_flavor.js`!). **Verify** that this worked by refreshing the browser, opening up the Javascript console, and entering the line `XM.iceCreamFlavors`. The console should display the collection with all the flavors you added in **Part I**. 

### Widgets

Next is to create a widget for the selection of `IceCreamFlavors`. In this case, we choose a `XV.Picker` over an `XV.RelationalWidget` because there will be a limited number of options and we will not need full-fledged search capabilites.

Create a new directory in `client/widgets`. The directory will need to be referenced by the root `package.js`, and it should have its own `package.js`, pointing to a new file `picker.js`, with the following contents:

```javascript
enyo.kind({
  name: "XV.IceCreamFlavorPicker",
  kind: "XV.PickerWidget",
  collection: "XM.iceCreamFlavors"
});
```

Note that we set the collection of the picker to be the cache that we've just set up. This kind is a good illustration of the power of the way that we use Object-Oriented behavior on the client-side. All of the functionality this picker will need is shared among all pickers. The code lives in `XV.PickerWidget`. All we have to do to make a picker widget backed by this particular collection is point that general code at our new cache, and all the details will take care of themselves.

### Extending views

The last step is to add the new `IceCreamFlavorPicker` to the `Contact` workspace, by adding it into the component array. Of course, we're not allowed to change the core source of `XV.ContactWorkspace`. We have to inject it in from the extension. Luckily, our core workspaces give you an easy way to do this. We can add the following code to `client/views/workspace.js`.

```javascript
var extensions = [
  {kind: "onyx.GroupboxHeader", container: "mainGroup", content: "_iceCreamFlavor".loc()},
  {kind: "XV.IceCreamFlavorPicker", container: "mainGroup", attr: "favoriteFlavor" }
];

XV.appendExtension("XV.ContactWorkspace", extensions);
```

**Verify** this step by setting some flavors in with the contacts and making seeing that they stick if you back out and go back into them.

We can use the same trick to add this picker to the advanced search options for contact, by adding the following code into a new file `client/widgets/parameter.js` file.

```javascript
extensions = [
  {kind: "onyx.GroupboxHeader", content: "_iceCreamFlavor".loc()},
  {name: "iceCreamFlavor", label: "_favoriteFlavor".loc(),
    attr: "favoriteFlavor", defaultKind: "XV.IceCreamFlavorPicker"}
];

XV.appendExtension("XV.ContactListParameters", extensions);
```

Add it to the `package.js` file underneath `picker.js`. **Verify** this step by pressing the magnifying glass icon when you're in the Contact list view, and filtering based on ice cream flavor. Only those contacts that you've set up to have that flavor should get fetched.

Congratulations! You've added the new business object to `Contact`. If you're still hungry to learn more about the capabilities of the xTuple stack, read on to [Part III](TUTORIAL3.md) to see what sorts of bells and whistles we can add to what we've built.
