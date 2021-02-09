<!-- App.svelte -->
<script>
  import PouchDB from "pouchdb";
  import dayjs from "dayjs";
  import { onMount } from "svelte";

  onMount(async () => {
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
  });

  const db = new PouchDB("https://couchdb.phocks.org/storylab");

  db.info().then(function (info) {
    console.log(info);
  });

  function addDoc(text) {
    var todo = {
      _id: dayjs().format("YYYY-MM-DDTHH:mm:ss:SSSZ"),
      title: text,
      completed: false,
    };

    db.put(todo, function callback(err, result) {
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
</script>

<div class="App">
  <button on:click={handleAddDocClick}> Add me </button>
  <button on:click={handleShowDocsClick}> Show docs </button>
</div>

<style>
  .App {
    /* background-color: aquamarine; */
  }
</style>
