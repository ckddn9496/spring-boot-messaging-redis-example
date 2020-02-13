You will use the `Redis template` to `send messages`
, and you will register the `Receiver` with `the message listener container` so that it will `receive messages`.
The `connection factory` drives both `the template and the message listener container`
, `letting them connect to the Redis server`.


`RedisTemplate` - `send message`
`Receiver`, `MessageListenerContainer` - `receive message`
`ConnectionFactory` - `connect to the redis server`