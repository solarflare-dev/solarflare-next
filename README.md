# Solarflare

> Interactive Real-Time Grids in React From Your Postgres

The enterprise world is full of data grids, used to present users with millions of rows of tabular data made searchable, sortable and filterable. The source of truth is typically a relational database like Postgres, and the end user experience is something like this:

<img alt="Example data grid" src="./grid.png" width="100%" />

Each column is sortable, filterable and some columns are searchable. Achieving an experience like this in a performant way, on top of a fast-changing dataset is a non-trivial task. You have a couple of options:

1. In your existing application, write raw SQL queries with the ability to handle:

   - Sorting
   - Filtering
   - Pagination
   - Searching

   Additionally, you'll need logic to retrieve and maintain the categorical facets for filtering and searching. You might need to introduce polling or 'push' based logic to keep the data on the user's screen up-to-date.

2. Use a service like Algolia, Elasticsearch or Typesense to index your data and provide a search API. Many of these services get expensive very quickly, charging by the search (i.e. every time a user loads your page or manipulates the table controls) and you still don't get real-time updates. You also have to manage syncing of data into the search service.

3. Solarflare.

## How Solarflare works

Solarflare is a blazing fast, native binary that connects to your Postgres database and automatically ingests live data changes. It provides an intuitive configuration experience, allowing you declaratively define the columns you want to display, the filters you want available and the search fields you want to enable. It then serves this data over a WebSocket connection to an unstyled React component context that allows you to render your own grid. As data changes, tiny delta updates are sent down the wire to your users.

### Advantages

- **Reduce Primary DB Load to Zero:** Solarflare connects to your Postgres database and listens for changes. Your primary database is never queried for the grid data, taking significant load off your database, without you having to worry about the details of replication.
- **Faceted Search:** Solarflare automatically maintains categorical facets, allowing the React component to render rich, relevant filter options that give your users a powerful data exploration experience.
- **Real-Time Updates:** Solarflare sends delta updates to your users in real-time, ensuring that the data on their screen is always up-to-date.
- **Blazing Fast:** Solarflare builds on the latest indexing & search technology written in Rust. Slow page loads based on heavy SQL queries are a thing of the past.
- **Extensible Declartive Configuration:** Solarflare allows you to declaratively define the columns, filters and search fields you want to enable. You can also define custom filters and search fields.
- **In-built Transport Protocol:** Network transport is handled all the way from the Postgres database to the React component, ensuring that you don't have to worry about the details of WebSockets, polling or data formats.

## Getting Started

Let's use Solarflare to take the Humanloop Logs table to the next-level. We're trying to solve these problems:

- High load on primary DB
- Slow page loads
- No real-time updates, except brittle bespoke polling patterns
- Faceted search hard to maintain with custom endpoints like `/datapoints-categorical-column-options`
- Column filtering, searching, sorting also hard to maintain, with custom Python logic required to support each new column

So, let's get started.

1. **Install Solarflare**

   ```bash
   $ brew install solarflare
   ```

1. **Configure Solarflare**

   ```bash
   $ export DATABASE_URL=postgres://user:password@host:port/dbname

   $ solarflare init
   ```

   The terminal-based configuration wizard will guide you through the process of connecting to your Postgres database and configuring the columns, filters and search fields you want to enable, based on introspecting the schema. Ultimately, it produces a `solarflare.toml` file that you can check into your repository. It looks like this:

   ```toml
   [grid]
   name = "logs-table"
   pg_relation = "public.logs"

   pagination = true
   real_time = true

   [[auth]]
   type = "row-level"
   pg_column = "org_id"
   token = "bearer"
   claim = "org_id"

   [[columns]]

   # ID Column. Searchable and sortable.
   name = "id"
   pg_column = "id"
   search = true
   sorting = "alphabetical"

   [[columns]]

   # Version column. Searchable, sortable and filterable.
   name = "version"
   pg_column = "version"
   search = true
   sorting = "alphabetical"
   filter = "categorical"

   [[columns]]

   # Inputs column. Full-text fuzzy search.
   name = "inputs"
   pg_column = "inputs"
   search = true
   sorting = null
   filter = null

   [[columns]]

   # Messages column. Full-text fuzzy search.
   name = "messages"
   pg_column = "messages"
   search = true
   sorting = null
   filter = null

   [[columns]]
   name = "timestamp"
   pg_column = "created_at"
   search = false
   sorting = "date"
   filter = "date_range"

   # etc...
   ```

1. **Configure Postgres instance to enable logical replication**

   ```sql
   ALTER SYSTEM SET wal_level = logical;
   ```

   then restart your Postgres instance.

1. **Start Solarflare**

   ```bash
   $ solarflare start --admin-port 8080
   ```

   Solarflare now connects to your database and begins indexing the `logs` table. Before this process is complete, the search API is already working, but the results are not yet complete. You can load the page http://localhost:8080 to see the status of the indexing process, as well as the health and performance statistics of the Solarflare service. You can also explore the served tables and columns in an admin view.

   A good way to seamlessly transition from your old implementation to the new one is to ping `/status` to know when the index is ready.

   ```curl
   GET http://localhost:8080/status

   {
     "status": "indexing",
     "tables": {
        "logs": {
           "status": "indexing",
           "progress": 0.42,
           "processed_records": 238746,
           "remaining_records": 568443
        }
     }
   }
   ```

1. **Generate frontend types**

   For frontend development, Solarflare provides a fully-typed experience based on the configuration in your solarflare.toml and the database schema.

   ```bash
   $ solarflare codegen --outdir ~/humanloop/frontend/src/types
   ```

   This places a `solarflare.d.ts` file in your frontend types directory, which we'll import in the next step.

1. **Integrate Solarflare into the frontend application**

   Solarflare is primarily a React hook. We understand that you want full control over the look and feel of your application and want to build fine-grained interactions at every level. So we provide a hook and a collection of totally unstyled 'headless' components to get the data, maintain a stateful relationship with the Solarflare service and organize the data into a component tree. Your components only need to think about rendering and styling.

   Next, install the Solarflare provider in your `layout.tsx`, or wherever you wrap your application in providers:

   ```jsx
   import { SolarflareProvider, createSolarflare } from "solarflare";
   import { type DB } from "./types/solarflare";

   export const { useTable } = createSolarflare<DB>();
   //             ^^^^^^^^ Fully-typed hook.

   const App = () => {
     return (
       <SolarflareProvider url="http://localhost:8080">
         <App />
       </SolarflareProvider>
     );
   };
   ```

   Then, in your grid component, `useTable` gives you everything you need to render beautiful, interactive grids.

   ```jsx
   import { useTable } from "./layout";

   const LogsTable = () => {
     const {
       data,
       loading,
       facets,
       filters,
       setFilters,
       search,
       setSearch,
       sort,
       setSort,
       pagination,
     } = useTable("logs-table");

     return (
       <MyGrid
         data={data}
         loading={loading}
         facets={facets}
         filters={filters}
         setFilters={setFilters}
         search={search}
         setSearch={setSearch}
         sort={sort}
         setSort={setSort}
         pagination={pagination}
       />
     );
   };
   ```

   What's happening here?

   The Solarflare client is performing data fetching and maintaining filter and sort state internally. You simply render out the components for showing the available filter & facet options. The `facets` object contains the categorical facets for each filterable column, and the `data` object contains the paginated data to render in your grid.

   Callbacks like `setSearch` are fully-typed, meaning you can only make queries to search columns that are configured on the Solarflare service to be searchable.

   When you call `setSearch`, the Solarflare client requests the new data from the server and the `data` object is updated with the new data, and your component re-renders.

   Some query optimizations can be done client-side, like debouncing requests and caching results. In some cases, it is possible to know that a search will return a subset of already cached results, and the Solarflare client can return results immediately. Or we can render results from the cache while waiting for the server to respond with a (possibly) more up to date result set. Buttery smooth grids across millions of rows, with minimal work.

   You bring Postgres, we give you an interactive data grid.

### Real-time

You can pass the `subscribe` prop to the `useTable` hook to get real-time updates.

```jsx
const {
  data,
  loading,
  facets,
  filters,
  setFilters,
  search,
  setSearch,
  sort,
  setSort,
  pagination,
} = useTable("logs-table", { subscribe: true });
```

In this mode, the current sort, filter and search state is maintained on a stateful connection at the Solarflare service. Each time new rows are ingested from Postgres, the Solarflare service determines which currently connected-clients need that row and sends a light delta update to the client. You don't have to do anything except pass `subscribe: true` to the hook.
