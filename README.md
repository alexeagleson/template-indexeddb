# Running the Example

Clone the repository and serve the `index.html` file.

See [here](#browsing-your-database) for instructions on how to view the sample data in the browser.

Continue below for the complete text of the blog tutorial, or follow along [ADD BLOG LINK]().

# Full Tutorial

# If you are interested in a video version of this tutorial, check out the link below. You can follow along with the code in this blog.  _(The video is entirely optional, every step and instruction is covered in the blog post.)_

![Video goes here]()

1. [What is IndexedDB?](#what-is-indexeddb)
1. [Important Terms](#important-terms)
1. [How to use IndexedDB](#how-to-use-indexeddb)
1. [Browsing your Database](#browsing-your-database)
1. [Limitations](#limitations)
1. [Further Learning](#further-learning)
1. [Wrapping Up](#wrapping-up)

## What is IndexedDB?

IndexedDB is an in-browser database that you can use to store large quantities of data to support your web page or web app. The information is stored using a simple key-value pair similar to the way you may already be familiar with using Javascript objects.

If you are just looking for the simplest possible way to store some data on the users's side that will survive through browser refreshes and closes, then you may be better off starting with the simple [local storage](https://developer.mozilla.org/en-US/docs/Web/API/Window/localStorage) API.  Local storage supports up to 5MB of data with simple key-value pairs like Javascript objects.

However if you are interested in learning about a much more robust and feature-rich client side storage method that supports many of the same features as a real full fledged database system -- then IndexedDB may be the right choice for you.  

IndexedDB is supported in most modern browsers and allows you to  store up to 50% of a user's free hard drive space (before the browser will begin dropping data).  To better understand the restrictions for storage and space with IndexedDB MDN has a [great resource](https://developer.mozilla.org/en-US/docs/Web/API/IndexedDB_API/Browser_storage_limits_and_eviction_criteria).


## Important Terms

### Database

A database is an _organized_ collection of data.  Whereas something comparable like the hard drive on your computer might be optimized to store large amounts of mostly unorganized data that is searched occasionally, a database instead assumes the data will be searched for frequently and is optimized to make sure these searches are as fast as possible.

### Cursor

A _cursor_ represents your current position when viewing the data in your database.  Cursors in IndexedDB can be used on entire object stores or even indexes that have been limited to a certain type of documents.  They offer the ability to iterate from one document to the next within the database rather than having to query all the data and store it in memory on the client application (in this case, our Javascript program).

### Transaction

A _transaction_ in database context is an operation or multiple operations that must all run successfully, otherwise none of them will be run at all.  

To understand why transactions are necessary, the most common example is transferring money between accounts in a bank database.  A transfer operation includes both `remove money` from one account and `add money` to another.  If the `add money` operation fails for any reason, you also need the `remove money` operation to fail as well, otherwise you would end up with a pretty nasty scenario where the money is simply "gone".

### Schema

The _schema_ of your database refers to the shape of your data.  For example we will be using a database that keeps track of _cars_ in our example.  There are endless different pieces of information you could imagine that relate to cars: colour, make, model, condition, trim, VIN, year, etc.

Our schema defines which of these properties are tracked and stored in our database.  So in our example we are only using _colour_ and _make_.  We also have an _id_ value that serves as a way to identify objects in our database.  

### Index

An _index_ in database terminology is just like an index you would use at the end of a book.  It is a basically a map of one set of values to another set.  

At the end of a book an index is a map of words to page numbers.  They allow you as the reader to have the ability to quickly find concepts you are looking for without having to look through the book page by page.

The concept is exactly the same for computers.  When looking at a huge database, without any index, your search will start at the very beginning and look at absolutely everything until it finds what it's looking for.  Adding an _index_ will create a structure in memory that makes those lookups faster and easier.  An index takes up space in memory, so they are often considered to be a tradeoff of space vs. speed.  In most cases, that tradeoff is well worth it.

The most common use of an index in a database is on the _primary key_ which is something unique (like an ID number) about the item stored in your database.  For cars it might be the VIN, for books the ISBN, etc, etc.  

## How to use IndexedDB

```js
// 1
const indexedDB =
  window.indexedDB ||
  window.mozIndexedDB ||
  window.webkitIndexedDB ||
  window.msIndexedDB ||
  window.shimIndexedDB;

if (!indexedDB) {
  console.log("IndexedDB could not be found in this browser.");
}

// 2
const request = indexedDB.open("CarsDatabase", 1);
```
1. IndexedDB can potentially be referred to by different names depending on which browser you are using.  Fortunately they all have the same API, so this will simply hop through a list of all possibilities until it finds one that exists and save it in your indexedDB variable.  If one is not found, a message will be sent to the log and the rest of the code will fail.  

2. Makes an "open" request to the database _version 1_. The first parameter is the name you want to call your DB and the second parameter is the version. If you decide to update the structure later you can increment this number to ensure all users are using the latest version of the schema.

Next we need to listen for different possible _events_ that can occur when an open request is made. The possible events are `success`, `error` and `upgradeneeded`. Let's begin by handling the error case:

### Error Event

```js
request.onerror = function (event) {
  console.error("An error occurred with IndexedDB");
  console.error(event);
};
```

The most likely error you may encounter is if you are running your browser in **private** or **incognito** modes, IndexedDB may not be supported. Be sure to disable those modes if you are using IndexedDB.

### Upgradeneeded Event

This event fires when either the database version number is incrementing, or a new database is being created. 

Whenever this occurs you need to define the shape of the database. So we will do that here:

```js
request.onupgradeneeded = function () {
  //1
  const db = request.result;
  
  //2
  const store = db.createObjectStore("cars", { keyPath: "id" });
  
  //3
  store.createIndex("cars_colour", ["colour"], { unique: false });

  // 4
  store.createIndex("colour_and_make", ["colour", "make"], {
    unique: false,
  }); 
};
```

We'll break it down line by line to understand each piece:

1. The result of the request is the database object itself.  We are inside the `onupgradeneeded` event so we can assume the database exists, otherwise the `onerror` function would have triggered.

2. IndexedDB works with the concept of _object stores_.  These are essentially names of collections of data.  You can have as many of these as you like in a single database.  Think of them like _tables_ or _collections_ if you have used other databases with those terms.  `keyPath` is the name of the field on the object that IndexedDB will use to identify it.  Typically this is a unique number.  

    You can also add the `autoincrement: true` to have it set to a unique id manually that you don't need to set yourself.  The first item you insert would have an `id` of 0, then second item and `id` of 1, and so on.  

    We are going to use cars an an example, so I have named my object store `cars`.

3. Adding [indexes](#index) allows us to search inside of our object store by specific terms aside from just the value defined as the `keyPath`.  This index will allow us to search for car objects by their `colour` property (pardon the Canadian spelling).

4. Similar you can create what are called _compound indexes_ which are indexes that can lookup with a combination of more than one term.  In this case it will allow us to find cars providing both the make and colour.

Now that we have established out [schema](#schema) we are ready to add data and query to find it.  This can be done once the database is open, which will be confirmed when the `success` event triggers.

```js
request.onsuccess = function () {
  console.log("Database opened successfully");

  const db = request.result;

  // 1
  const transaction = db.transaction("cars", "readwrite");
  
  //2
  const store = transaction.objectStore("cars");
  const colourIndex = store.index("cars_colour");
  const makeModelIndex = store.index("colour_and_make");

  //3
  store.put({ id: 1, colour: "Red", make: "Toyota" });
  store.put({ id: 2, colour: "Red", make: "Kia" });
  store.put({ id: 3, colour: "Blue", make: "Honda" });
  store.put({ id: 4, colour: "Silver", make: "Subaru" });

  //4
  const idQuery = store.get(4);
  const colourQuery = colourIndex.getAll(["Red"]);
  const colourMakeQuery = makeModelIndex.get(["Blue", "Honda"]);

  // 5
  idQuery.onsuccess = function () {
    console.log('idQuery', idQuery.result);
  };
  colourQuery.onsuccess = function () {
    console.log('colourQuery', colourQuery.result);
  };
  colourMakeQuery.onsuccess = function () {
    console.log('colourMakeQuery', colourMakeQuery.result);
  };

  // 6
  transaction.oncomplete = function () {
    db.close();
  };
};
```

1. In order to perform any operation on our DB we must create a [transaction](#transaction).  A transaction can be a single operation or multiple operations that must all succeed, otherwise none of them will.  Further down we will add four "cars" to our database one by one, but if any of those inserts failed for any reason then all four of them would fail because they happen on this single transaction we have created.

2. Here we need to ge ta reference to our object store that holds the cars.  We also get a reference to our indexes.  These are simply just getting references to the values that we created on the database in the previous section.

3. The `put` method on an object store is how we add data to our database.  Based on the schema we created we will add a bunch of objects (cars).  The ID I have given them is simply a unique number, you can also use the _autoincrement_ value described previously when creating the object store to avoid having to set this value manually.

4. These are our queries.  You can always query an item directly with the value of your `keyPath` as we have here on the first line.  On our second line we use the `getAll` method which will return an array with every result it finds.  We are searching against our `cars_colour` index for "Red".  We should expect to find two results.  The final line searches for one result against our compound index for any vehicle with a colour of "Blue" and a make of "Honda".

5. These are `success` event handlers, they will fire when the query finishes and run whatever code is inside of them.  They will not fire until the `result` value is populated on the query so it is safe to check it, as we do in these functions by logging it to the console.

6. Lastly, since this is our only operation we will close our connection to the database when the transaction finishes.  You don't need to manually fire the transaction with IndexedDB it will simply run on its own.

If you take each of the above code (every sample block in the examples into a `.js` file and run it in the browser (with private/incognito modes off) your results will look like.  Take note of each of the logged values matching what we queried for.

![IndexedDB Example](https://res.cloudinary.com/dqse2txyi/image/upload/v1637686766/blogs/indexeddb/indexeddb-example_lsbrvi.png)


### Removing Data

Data in IndexedDB can be deleted with an API similar to how it is queried.  The simplest method is to delete an entry directly by its known key:

```js
const deleteCar = store.delete(1);

deleteCar.onsuccess = function () {
  console.log("Red Toyota has been removed");
};
```

If you don't know the key and want to remove based on the value of one of your indexes, you can do that too:

```js
const getRedCarKey = colourIndex.getKey(["Red"]);

getRedCarKey.onsuccess = function () {
  const deleteCar = store.delete(redCarKey.result);

  deleteCar.onsuccess = function () {
    console.log("Red car has been removed");
  };
};
```

For updating data instead of removing it entirely, you can simply re-insert the same data as long as it has the same key, with the new values you wish to be reflected.  


## Browsing your Database

Browsers make it trivially simple to view the contents of your store.  First open up the developer console with `F12`.  

On Chrome you will find it under the `Application` -> `Storage` -> `IndexedDB`.

![IndexedDB Chrome](https://res.cloudinary.com/dqse2txyi/image/upload/v1637686833/blogs/indexeddb/indexeddb-chrome_lyisrf.png)

On Firefox it's under `Storage` -> `Indexed DB`.

![IndexedDB Firefox](https://res.cloudinary.com/dqse2txyi/image/upload/v1637686846/blogs/indexeddb/indexeddb-firefox_ahbaei.png)

## Limitations

There are a couple limitations to be aware of when using IndexedDB.

The first is relevant to any client-side storage solution you might use, in that you should not ever rely on it existing for your application to function.  Remember that the user can clear their private data and storage at any time.  Any data you save should always be _supplementary_ to your application and easy to replace if removed.

The second is related to performance.  IndexedDB is known to be quite fast on inserting reasonably large quantities of data on a single transaction, but can slow down significantly when these inserts/updates are made across multiple transactions.

The solution is to simply be aware of this limitation and ensure you are developing your application to batch data modifications into as few transactions as possible.  If that is not possible, take the time to research and considered if IndexedDB is the right tool for your project.  There are [alternatives](https://rxdb.info/) out there.  

## Further Learning

There is even more to IndexedDB than is covered in this beginner's tutorial.  For example if you intend to store large amounts of data, potentially more than some users would be able to store in memory off a single query you will be interested in the concept of [cursors](https://javascript.info/indexeddb#cursors).

Both javascript.info and MDN cover IndexedDB extremely in-depth, check them out if you want to go deeper with INdexedDB:

- [IndexedDB on javascript.info](https://javascript.info/indexeddb)
- [IndexedDB on MDN](https://developer.mozilla.org/en-US/docs/Web/API/IndexedDB_API/Using_IndexedDB)

## Wrapping Up

You should now have a good understanding about how to create your own React component library.  Doing so can not only teach you a lot about how the Javascript package management ecosystem works, but it can be a great way to make code that you use across multiple projects easily available with a simple command.

Please check some of my other learning tutorials.  Feel free to leave a comment or question and share with others if you find any of them helpful:

- [How to Create and Publish a React Component Library ](https://dev.to/alexeagleson/how-to-create-and-publish-a-react-component-library-2oe)

- [Running a Local Web Server](https://dev.to/alexeagleson/understanding-the-modern-web-stack-running-a-local-web-server-4d8g)

- [ESLint](https://dev.to/alexeagleson/understanding-the-modern-web-stack-linters-eslint-59pm)

- [Prettier](https://dev.to/alexeagleson/understanding-the-modern-web-stack-prettier-214j)

- [Babel](https://dev.to/alexeagleson/building-a-modern-web-stack-babel-3hfp)

- [React & JSX](https://dev.to/alexeagleson/understanding-the-modern-web-stack-react-with-and-without-jsx-31c7)

- [Webpack: The Basics](https://dev.to/alexeagleson/understanding-the-modern-web-stack-webpack-part-1-2mn1)

- [Webpack: Loaders, Optimizations & Bundle Analysis](https://dev.to/alexeagleson/understanding-the-modern-web-stack-webpack-part-2-49bj)

- [Webpack: DevServer, React & Typescript](https://dev.to/alexeagleson/understanding-the-modern-web-stack-webpack-devserver-react-typescript-4b9b)

---

For more tutorials like this, follow me <a href="https://twitter.com/eagleson_alex?ref_src=twsrc%5Etfw" class="twitter-follow-button" data-show-count="false">@eagleson_alex</a><script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script> on Twitter
