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

6.  staleTime:
    <br />
    In React Query terms, `stale` is the opposite of `fresh`. As long as a query is considered fresh, data will only be delivered from the cache. And staleTime is what defines the time (in milliseconds) until a query is considered stale.
    when a query considered `stale` react-query does _nothing_ until one of four condition below is met:

    - The queryKey changes
    - A new observer mounts
    - The window receives a focus event
    - The device goes online

    React Query **will always deliver the data from the cache if it exists, even if that data is no longer fresh**.
    staleTime just tells React Query when to update the cache in the background when a trigger occurs.

7.  gcTime:
    <br />
    Amount of time after a query became `inactive` (_there is no observer for that particular query_) that query become garbage collected.

8.  refetchInterval:
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

9.  Dependent Query:
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

10. Don't destruct the queryClient<br/>
    It's important to note that you can't destructure properties from the QueryClient.

    ```javascript
    const { prefetchQuery } = useQueryClient(); // ❌
    ```

    The reason for this is because the _QueryClient is a class_, and classes can't be destructured in JavaScript without losing the reference to its `this` binding.<br />
    This is not React Query specific, you'll have the same problem when doing something like.

    ```typescript
    const { getTime } = new Date();
    ```

11. Potential Solution to Avoiding Loading State:

    - using queryClient.prefetchQuery()
    - using placeholderData or initialData
    - using both technique

12. initialData vs placeholderData:
    <br />`initialData` is treated as if data is fetched from server and react-query won't make a request for new data until staleTime is passed and a trigger occurred
    <br />`placeholderDate` is treated as placeholder and react-query would instantly request for data regaredless of staleTime and triggers.

13. using queryClient.setQueryData() inside mutation:<br />
    React Query doesn't distinguish where data comes from. Data we write to the cache manually will be treated the same as data put into the cache via any other way – like a refetch or prefetch.
    That means it will also be considered fresh for however long `staleTime` is set to.<br />
    **`setQueryData` as second parameter receives previousData for a particular query key and you can use that plus mutationData that is passed to `onSuccess` to update cache**

14. difference between when `return queryClient.invalidateQueries` and `queryClient.invalidateQueries`:<br/>
    by returning a promise from `onSuccess` (which is what `queryClient.invalidateQueries` returns), React Query can wait for the promise to resolve before it considers the mutation complete – avoiding potential UI flashes where the refetch occurs before the mutation has finished.

15. what happens when `invalidateQueries` in called:<br/>
    when you invalidate a query, it does two things:

    - It refetches all active queries
    - It marks the remaining queries as stale

    you could add a `refetchType` property of `all` to your query options to force all queries, regardless of their status, to refetch immediately.

    ```javascript
    queryClient.invalidateQueries({
      queryKey: ["todos", "list"],
      refetchType: "all",
    });
    ```

16. customizing default:

    - all

    ```javascript
    const queryClient = new QueryClient({
      defaultOptions: {
        queries: {
          staleTime: 10 * 1000,
        },
      },
    });
    ```

    you can set default option for anything except `queryKey`

    - subset

    ```javascript
    queryClient.setQueryDefaults(["todos", "detail"], { staleTime: 10 * 1000 }); // this will fuzzy match
    ```

    - single<br />
      passing option directly to `useQuery/useMutation/etc`

    _the result configuration for each query_

    ```javascript
    const finalOptions = {
      ...queryClientOptions,
      ...setQueryDefaultOptions,
      ...optionsFromUseQuery,
    };
    ```

17. react-query optimizations:

    - Structural Sharing:<br /> you can use the `data object` with React.memo or include it in the dependency array for useEffect or useMemo without worrying about unnecessary effects or calculations.
    - observers: <br /> Observers are the glue between the Query Cache and any React component, and they live outside the React component tree. What this means is that when a queryFn re-runs and the Query cache is updated, at that moment, the Observer can decide whether or not to inform the React component about that change.
    - Tracked Properties:<br />
      When React Query creates the result object returned from useQuery, it does so with `custom getters`.
      <br />Why is this important? Because it allows the Observer to understand and keep track of which fields have been accessed in the render function and in doing so, only re-render the component when those fields actually change.
      <br />For example, if a component doesn't use fetchStatus, it doesn't make sense for that component to re-render just because the fetchStatus changes from idle to fetching and back again. It's Tracked Properties that make this possible and ensure that components are always up-to-date, while keeping their render count to the necessary minimum.
    - select: <br />
      If your queryFn returns extra data that isn't needed in the component, you can use the select option to filter out the data that the component doesn't need – and therefore, subscribe to a subset of the data and only re-render the component when necessary.

18. useDebounced vs abortSignal:<br />
    react-query pass an `signal` parameter to queryFn that can be used to cancel queries for example in search inputs.
    in this case only the last searchTerm would be inside of QueryCache and our server have less load.<br />
    we also can `debounce` a query and only make a request on defined intervals.

19. Error Handling:<br />

    - **YOU SHOULD NOT CATCH THE ERROR INSIDE queryFn YOURSELF WITHOUT THROWING IT OUT AGAIN**

20. Offline Support:<br />
    In the scenario of an offline device, React Query will mark the `fetchStatus` of the query as `paused`, without even attempting to execute the `queryFn`. Then, if and when the device comes back online, React Query will automatically resume the query as normal.

    If we go offline react-query would queue `mutations` and when we came back online do them in the same exact order.
    and if we don't want to get `in between state` when we come back online we can guard the query invalidation.

    ```javascript
    // this tell react query to run invalidateQueries if and only if there is `one` mutation related to ["todos", "list"] is currently running
    // for example if we toggle 10 switches when device was offline, we don't get the in between states with this technique
    if (queryClient.isMutating({ mutationKey: ["todos", "list"] }) === 1) {
      return queryClient.invalidateQueries({ queryKey: ["todos", "list"] });
    }
    ```
