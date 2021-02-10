<!-- App.svelte -->
<script>
  import PouchDB from "pouchdb";
  import dayjs from "dayjs";
  import { onMount } from "svelte";

  const db = new PouchDB("https://couchdb.phocks.org/storylab");

  let people = 0;
  let today = "";

  db.info().then(function (info) {
    console.log(info);
  });

  onMount(async () => {
    // Listen for changes
    db.changes({
      since: "now",
      live: true,
    })
      .on("change", (data) => {
        console.log(data);
      })
      .on("error", (err) => {
        console.error(err);
      });

    db.get(dayjs().format("YYYY-MM-DD"))
      .then(function (doc) {
        // okay, doc contains our document
        console.log(doc._id);
        today = doc._id;
      })
      .catch(function (err) {
        // oh noes! we got an error
        console.log("Doc not found. Let's add it!");
        addToday();
      });
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
        console.log("Probably already have a document for today. Don't worry!");
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

  function handleAddDocClick() {
    addDoc("This is a doc...");
  }

  function handleShowDocsClick() {
    showDocs();
  }

  function addPerson() {
    people++;
  }
</script>

<div class="App">
  <p>Today is: {today}</p>
  <p>There are {people} people booked for tomorrow.</p>
  <button on:click={addPerson}> Add person </button>
  <button on:click={handleAddDocClick}> Add me </button>
  <button on:click={handleShowDocsClick}> Show docs </button>
</div>

<style>
  .App button {
    background-color: aquamarine;
  }
</style>
