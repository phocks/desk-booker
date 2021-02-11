<!-- App.svelte -->
<script>
  import { onMount, onDestroy } from "svelte";
  import PouchDB from "pouchdb";
  import dayjs from "dayjs";
  import localizedFormat from "dayjs/plugin/localizedFormat";
  dayjs.extend(localizedFormat);

  const db = new PouchDB("storylab");
  const remoteDb = new PouchDB("https://couchdb.phocks.org/storylab");

  let numberOfPeople = 0;
  let today = dayjs().format("LL");
  let changesListener;

  // Do once when component mounted
  onMount(async () => {
    // Set up out changes listener
    changesListener = db
      .changes({
        since: "now",
        live: true,
        include_docs: true,
      })
      .on("change", (data) => {
        console.log("Changes", data);
        numberOfPeople = data.doc.people.length;
      })
      .on("error", (err) => {
        console.error(err);
      });

    // See if there's a doc for today, otherwise create it
    db.get(dayjs().format("YYYY-MM-DD"))
      .then(function (doc) {
        // okay, doc contains our document
        numberOfPeople = doc.people.length;
      })
      .catch(function (err) {
        // oh noes! we got an error
        console.log("Doc not found. Let's add it!");
        addToday();
      });

    // Sync up with remote database
    db.sync(remoteDb, {
      live: true,
      retry: true,
    })
      .on("change", function (change) {
        // yo, something changed!
      })
      .on("paused", function (info) {
        // replication was paused, usually because of a lost connection
      })
      .on("active", function (info) {
        // replication was resumed
      })
      .on("error", function (err) {
        // totally unhandled error (shouldn't happen)
      });
  });

  // Do this on unMount
  onDestroy(async () => {
    // Cancel the listener on hot reload
    changesListener.cancel();
  });

  function addToday() {
    // A new day a new doc
    const today = {
      _id: dayjs().format("YYYY-MM-DD"),
      people: [],
    };

    db.put(today)
      .then((result) => {
        console.log(result);
      })
      .catch((err) => {
        console.error(err);
      });
  }

  function addDoc(text) {
    const doc = {
      _id: dayjs().format("YYYY-MM-DDTHH:mm:ss:SSSZ"),
      title: text,
      completed: false,
    };

    db.put(doc, function callback(err, result) {
      if (!err) {
        console.log("Successfully posted a doc!");
      }
    });
  }

  function showDocs() {
    db.allDocs({ include_docs: true, descending: true }, function (err, doc) {
      console.log(doc.rows);
    });
  }

  function handleShowDocsClick() {
    showDocs();
  }

  function addPerson() {
    numberOfPeople++;

    db.get(dayjs().format("YYYY-MM-DD"))
      .then(function (doc) {
        // update their age
        doc.people = [...doc.people, "person"];

        // put them back
        return db.put(doc);
      })
      .then(function () {
        // fetch again
        return db.get(dayjs().format("YYYY-MM-DD"));
      })
      .then(function (doc) {
        console.log(doc);
      });
  }
</script>

<div class="App">
  <p>Today is: {today}</p>
  <p>There are {numberOfPeople} people booked for tomorrow.</p>
  <button on:click={addPerson}> Add person </button>
  <button on:click={handleShowDocsClick}> Show docs </button>
</div>

<style>
  .App button {
    background-color: aquamarine;
  }
</style>
