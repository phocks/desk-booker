<!-- App.svelte -->
<script>
  import PouchDB from "pouchdb";
  import dayjs from "dayjs";
  import { onMount } from "svelte";

  onMount(async () => {
    db.changes({
      since: "now",
      live: true,
    }).on("change", (data) => {
      console.log(data);
    });
  });

  const db = new PouchDB("https://couchdb.phocks.org/storylab");

  db.info().then(function (info) {
    console.log(info);
  });

  function addTodo(text) {
    var todo = {
      _id: dayjs().format("YYYY-MM-DDTHH:mm:ss:SSSZ"),
      title: text,
      completed: false,
    };

    db.put(todo, function callback(err, result) {
      if (!err) {
        console.log("Successfully posted a todo!");
      }
    });
  }

  function showTodos() {
    db.allDocs({ include_docs: true, descending: true }, function (err, doc) {
      console.log(doc.rows);
    });
  }

  function handleClick() {
    addTodo("Hello world!");
  }

  showTodos();
</script>

<div class="App">
  <button on:click={handleClick}> Click me </button>
</div>

<style>
  .App {
    /* background-color: aquamarine; */
  }
</style>
