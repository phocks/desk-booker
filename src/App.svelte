<!-- App.svelte -->
<script>
  import { onMount, onDestroy } from "svelte";
  import PouchDB from "pouchdb";
  import dayjs from "dayjs";
  import localizedFormat from "dayjs/plugin/localizedFormat";
  dayjs.extend(localizedFormat);

  const db = new PouchDB("storylab");
  const remoteDb = new PouchDB("https://couchdb.phocks.org/storylab");

  let today = dayjs().format("LL");
  let changesListener;
  let remoteSyncListener;
  let nameOfPerson = "";
  let todaysPeople = [];
  let yesterdaysPeople = [];

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
        getToday();
        getYesterday();
      })
      .on("error", (err) => {
        console.error(err);
      });

    getToday();
    getYesterday();

    // Sync up with remote database
    remoteSyncListener = db
      .sync(remoteDb, {
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
    // Cancel listeners on hot reload
    changesListener.cancel();
    remoteSyncListener.cancel();
  });

  function getToday() {
    // See if there's a doc for today, otherwise create it
    db.get(dayjs().format("YYYY-MM-DD"))
      .then(function (doc) {
        // okay, doc contains our document
        todaysPeople = doc.people;
      })
      .catch(function (err) {
        // oh noes! we got an error
        console.log("Doc not found. Let's add it!");
        addToday();
      });
  }

  function getYesterday() {
    // Let's see if yesterday is in there
    const yesterdayDate = dayjs().subtract(1, "day").format("YYYY-MM-DD");

    db.get(yesterdayDate)
      .then(function (doc) {
        yesterdaysPeople = doc.people;
      })
      .catch(function (err) {
        // oh noes! we got an error
        console.log("Yesterday, all my troubles seemed so far away...");
      });
  }

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

  function showDocs() {
    db.allDocs({ include_docs: true, descending: true }, function (err, doc) {
      console.log(doc.rows);
    });
  }

  function handleShowDocsClick() {
    showDocs();
  }

  function togglePerson() {
    if (nameOfPerson === "") {
      alert("Please enter a name...");
      return;
    }

    if (todaysPeople.includes(nameOfPerson)) {
      db.get(dayjs().format("YYYY-MM-DD"))
        .then(function (doc) {
          // update their age
          doc.people = doc.people.filter((person) => person !== nameOfPerson);

          // put them back
          return db.put(doc);
        })
        .then(function () {
          // fetch again
          return db.get(dayjs().format("YYYY-MM-DD"));
        })
        .then(function (doc) {
          console.log("Updated doc", doc);
        });

      return;
    }

    db.get(dayjs().format("YYYY-MM-DD"))
      .then(function (doc) {
        // update their age
        doc.people = [...doc.people, nameOfPerson];

        // put them back
        return db.put(doc);
      })
      .then(function () {
        // fetch again
        return db.get(dayjs().format("YYYY-MM-DD"));
      })
      .then(function (doc) {
        console.log("Updated doc", doc);
      });
  }
</script>

<div class="App">
  <p>Today is {today}</p>
  <p>{yesterdaysPeople.length} in today.</p>
  <ul>
    {#each yesterdaysPeople as peep, i}
      <li>{peep}</li>
    {/each}
  </ul>
  <p>{todaysPeople.length} peeps booked for tomorrow.</p>
  <ul>
    {#each todaysPeople as peep, i}
      <li>{peep}</li>
    {/each}
  </ul>
  <form on:submit|preventDefault={togglePerson}>
    <input bind:value={nameOfPerson} placeholder="Your name" />
    <button type="submit"> Book in (toggle) </button>
  </form>
</div>

<style>
  .App {
    font-size: 22px;
    margin: 15px 20px;
  }

  .App button {
    font-size: 17px;
    background-color: aquamarine;
    border: 2px solid rgb(1, 131, 103);
    border-radius: 3px;
    padding: 5px 10px;
  }

  .App input {
    font-size: 17px;
    padding: 5px 10px;
  }
</style>
