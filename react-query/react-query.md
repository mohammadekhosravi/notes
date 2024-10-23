### React Query Notes

1. Don't need to catch error inside queryFn.

```javascript
// you need to do this if you are fetching inside useEffect for example
const fetchRepos = async () => {
  try {
    const response = await fetch("https://api.github.com/orgs/TanStack/repos");

    if (response.ok) {
      const data = await response.json();
      return data;
    } else {
      throw new Error(`Request failed with status: ${response.status}`);
    }
  } catch (error) {
    // handle network errors
  }
};

// but in react-query you do this
function useRepos() {
  return useQuery({
    queryKey: ["repos"],
    queryFn: async () => {
      const response = await fetch(
        "https://api.github.com/orgs/TanStack/repos",
      );

      if (!response.ok) {
        throw new Error(`Request failed with status: ${response.status}`);
      }

      return response.json();
    },
  });
}
```

There are a couple things to notice here.

First, we were able to get rid of our try/catch code. In order to tell React Query that an error occurred, and therefore, to set the status of the query to error, all you have to do is throw an error in your queryFn.

Second, we were able to return response.json() directly. As you know, your query function should return a promise that eventually resolves with the data you want to cache. That's exactly what we're doing here, since response.json() returns a promise that resolves with the parsed JSON data.

2. Purpose of `isLoading`:

   query `status` have 3 state:

- pending: there is no data in cache
- error: there is an error
- success: data is in cache

  when we use `enabled` inside `useQuery` by default `status === 'pending'` because there is no data in cache but we are not fetching data actively
  so there is another property `fetchStatus` that tell us if we are fetching data.
  so if `status === 'pending' && fetchingStatus === 'fetching'` we show `<Loading />`. but there is a shorthand.
  `isLoading = status === 'pending' && fetchingStatus === 'fetching'`

3. `isLoading === false` doesn't mean we have data

```javascript
const data = undefined; // There's no data in the cache
const status = "pending"; // There's no data in the cache
const fetchStatus = "idle"; // The queryFn isn't currently being executed
const isLoading = status === "pending" && fetchStatus === "fetching"; // false

if (isLoading) {
  // there is no data in the cache
  // and the queryFn is currently being executed
  return <div>...</div>;
}

if (status === "error") {
  // there was an error fetching the data to put in the cache
  return <div>There was an error fetching the issues</div>;
}

return (
  <p>
    <ul>
      {/* data could be undefined we need to explicitly check for status === 'success' */}
      {data.items.map((issue) => (
        <li key={issue.id}>{issue.title}</li>
      ))}
    </ul>
  </p>
);
```

4. alternative to `enabled`

instead of using `isLoading` and `status === 'success'` we can use a simple react pattern,
assume condition is `enabled: searchTerm !== ''` instead we can only render the component when there is a `searchTerm`

```javascript
const Component = () => {
  return searchTerm === "" ? (
    <div>Plase Enter a Search Term</div>
  ) : (
    <ShowResultWithReactQury />
  );
};
```

5. queryKeys

queryKeys don't need to be only 'string', because despite of useEffect dependencies that have problem with object and arrays
react-query hash queryKeys so you can use object and arrays without any problem.

```javascript
useQuery({
  queryKey: ["book", { sort }],
});
```

6. staleTime:
   <br />
   In React Query terms, `stale` is the opposite of `fresh`. As long as a query is considered fresh, data will only be delivered from the cache. And staleTime is what defines the time (in milliseconds) until a query is considered stale.
   when a query considered `stale` react-query does _nothing_ until one of four condition below is met:

   - The queryKey changes
   - A new observer mounts
   - The window receives a focus event
   - The device goes online

   React Query **will always deliver the data from the cache if it exists, even if that data is no longer fresh**.
   staleTime just tells React Query when to update the cache in the background when a trigger occurs.

7. gcTime:
   <br />
   Amount of time after a query became `inactive` (_there is no observer for that particular query_) that query become garbage collected.

8. refetchInterval:
   used for polling data, `refetchInterval` is smart and if we trigger refetch with other triggers it's counter would reset.

   - you can pass a function to `refetchInterval` for long running task that can finish and if return `false` it would not refetch anymore

   ```typescript
   refetchInterval: (query) => {
     if (query.state.data?.finished) {
       return false;
     }

     return 3000; // 3 seconds
   };
   ```

9. Dependent Query:
   when we need to fetch two or more seperate endpoint for getting necessary data we can use this technique:

   ```typescript
   async function fetchMovie(title) {
     const response = await fetch(
       `https://ui.dev/api/courses/react-query/movies/${title}`,
     );

     if (!response.ok) {
       throw new Error("fetch failed");
     }

     return response.json();
   }

   async function fetchDirector(id) {
     const response = await fetch(
       `https://ui.dev/api/courses/react-query/director/${id}`,
     );

     if (!response.ok) {
       throw new Error("fetch failed");
     }

     return response.json();
   }

   async function getMovieWithDirectorDetails(title) {
     const movie = await fetchMovie(title);
     const director = await fetchDirector(movie.director);

     return { movie, director };
   }
   ```

   but this way we tie this two fetch together(error, pending and ...)<br />
   _another way_: use the same technique we use for fetching on demand: `enabled`
   ![Dependent Query](https://github.com/mohammadekhosravi/notes/blob/master/react-query/dependent.png)
