# use-phoenix

## Connecting to your Phoenix Socket

Wrap the intended part of your application with a `PhoenixProvider`.

```tsx
// App.tsx
import { PhoenixProvider } from 'use-phoenix';

const Application = () => {
	return <PhoenixProvider>...</PhoenixProvider>;
};
```

Passing a `url` and params to your `PhoenixProvder` will connect to your socket instantly **on mount**:

```tsx
return (
	<PhoenixProvider
		url="ws://localhost:4000/socket"
		options={{
			params: { token: 'xyz' }
		}}
	>
		...
	</PhoenixProvider>
);
```

You could instead use the `usePhoenix` hook to connect lazily using `connect`:

To use this option **do not pass a `url`** into `PhoenixProvider`:

```tsx
// App.tsx
return <PhoenixProvider>...</PhoenixProvider>;
```

Later on when you would like to connect the socket:

```ts
// Component.tsx
import { usePhoenix } from 'use-phoenix';

const Component = () => {
	const { socket, connect } = usePhoenix();

	useEffect(() => {
		connect('ws://localhost:4000/socket', {
			params: { token: 'xyz' }
		});
	}, [connect]);
};
```

`usePhoenix` also provides `isConnected` which becomes `true` when the socket has successfully connected.

## Quick Start - `useEvent` & `useChannel`

You can pass a short circuit expression to delay connection to an event or channel. If for example you are waiting to recieve an id to use from some network request, `useEvent` and `useChannel` will not connect until it is defined. Below is a contrived example:

```jsx
interface PingEvent {
  event: 'ping',
  data: {
    body: string;
  }
}

interface PongEvent {
  event: 'pong';
  data: {
    message: string;
  };
}

interface JoinPayload {
  secret: string;
}

interface PingResponse {
  ok: boolean;
}

  // Channel will not connect until id is defined
  const [channel, { push, data }] = useChannel<JoinPayload>(id && `chat:${id}`);
  //                      ^^^^
  //        data is typed according to `JoinPayload`

  // Events will not be listened to until data.secret is defined
  const { data } = useEvent<PongEvent>(channel, data?.secret && `pong:${data.secret}`);

  const handleClick = () => {
    // ok is typed according to PingResponse
    const { ok } = await push<PingEvent, PingResponse>('ping', { body: 'Hello World' })
  }

  return (
    <div>
      <button onClick={handleClick}>
        ping
      </button>
      <p>{data && data.message}</p>
    </div>
  );
```

# useEvent

`useEvent` is a hook that allows you to succinctly listen in on channel events.

### Example Usage

```ts
...
// Type check your event
type joinEvent = {
	event: 'join',
	data: {
		members: Record<string, User>
	}
}

const [channel, { isSuccess, isError, ...rest }] = useChannel('chat:lobby')

// pass in a channel directly
const { data } = useEvent<JoinEvent>(channel, 'join')

// typed
console.log(data.members)
```

Optionally, if you would rather capture the response in a callback you can (or both):

```ts
const { data } = useEvent<JoinEvent>(channel, 'join', (data) => {
	console.log(response);
});
```

## useChannel

`useChannel` gives you important functions and information about the state of the channel. The following properties are available for `useChannel`

```ts
data: JoinPayload | null; // the join response from the server
status: ChannelStatus;
isSuccess: boolean;
isLoading: boolean;
isError: boolean;
error: any;
push: PushFunction // push a message to the server
leave: () => void // leave the channel
```

### Example Usage

```tsx
const [channel, { push }] = useChannel('room:1');

push('new_msg', { msg: 'Hello World' });
leave();
```

## usePresence

Quickly setup presence tracking by connecting to your presence channel

```tsx
const users = usePresence('room:lobby');
```

the response is transformed to make it easier to use. Typically it is an object of `id: { ..., metas: [{ ... }] }`. In all of my use cases, the `metas` field is always an array of one object, and I found myself having to constantly drill into the first index `metas[0]`. Thus, the hook automatically returns `metas[0]` if the metas field is a single index array.

The response takes the form:

```js
[
	{
		id: number;
		metas: object;
		// any other fields from your server.
	}
]
```

Say you add a `user` field to each presence from your server; you can easily type the field using generics

```tsx
const users = usePresence<{ user: User }>('room:lobby');

// typed User
users[0].user;

// id and metas are automatically typed
users[0].id;
```

Additionally, you can type extra fields into the `metas`:

```tsx
// If you have a custom metas, type it easily
const users = usePresence<void, { lastSeen: string }>('room:lobby');

// typed lastSeen
users[0].metas.lastSeen;
```

## Notes

- `useChannel` does not automatically `leave` the channel when the hook unmounts. That is, the socket will continue to listen in on the channel. It is best to handle leaving the channel explicitly using `leave` if you would like to leave the channel on component unmounts:

  ```ts
  const [channel, { leave }] = useChannel('chat:lobby');

  useEffect(() => {
  	return () => {
  		leave();
  	};
  }, [leave]);
  ```
